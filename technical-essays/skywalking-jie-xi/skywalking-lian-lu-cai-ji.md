---
description: Skywalking trace collect
---

# skywalking 链路采集

&#x20;链路采集的复杂性在于数据结构的复杂。我们将从Trace的数据结构开始讲起，最后再以一个实际采集的例子结束。

## Trace数据结构

### TraceSegment

* `TraceSegment`用于记录所在线程( Thread )的链路，一个`TraceSegment`由多个Span组成。
* 一次分布式链路追踪，可以包含多个`TraceSegment` ，因为存在跨进程的操作( RPC 、MQ 等等)，或者垮线程的操作( 并发执行、异步回调等等 )。

`TraceSegment`数据结构及主要方法：

```
/**
 * {@link TraceSegment} is a segment or fragment of the distributed trace. See https://github.com/opentracing/specification/blob/master/specification.md#the-opentracing-data-model
 * A {@link TraceSegment} means the segment, which exists in current {@link Thread}. And the distributed trace is formed
 * by multi {@link TraceSegment}s, because the distributed trace crosses multi-processes, multi-threads. <p>
 */
public class TraceSegment {
    /**
     * The id of this trace segment. Every segment has its unique-global-id.
     */
    //表示此segment的id
    private String traceSegmentId;

    /**
     * The refs of parent trace segments, except the primary one. For most RPC call, {@link #ref} contains only one
     * element, but if this segment is a start span of batch process, the segment faces multi parents, at this moment,
     * we only cache the first parent segment reference.
     * <p>
     * This field will not be serialized. Keeping this field is only for quick accessing.
     */
    //指向parent的指针
    private TraceSegmentRef ref;

    /**
     * The spans belong to this trace segment. They all have finished. All active spans are hold and controlled by
     * "skywalking-api" module.
     */
    //这次trace包含的span
    private List<AbstractTracingSpan> spans;

    /**
     * The <code>relatedGlobalTraceId</code> represent the related trace. Most time it related only one
     * element, because only one parent {@link TraceSegment} exists, but, in batch scenario, the num becomes greater
     * than 1, also meaning multi-parents {@link TraceSegment}. But we only related the first parent TraceSegment.
     */
    //此次追踪的全局id
    private DistributedTraceId relatedGlobalTraceId;

    //是否忽略
    private boolean ignore = false;

    //span是否超过上线
    private boolean isSizeLimited = false;

    private final long createTime;

    /**
     * Establish the link between this segment and its parents.
     *
     * @param refSegment {@link TraceSegmentRef}
     */
    public void ref(TraceSegmentRef refSegment) {
        if (null == ref) {
            this.ref = refSegment;
        }
    }

    /**
     * Establish the line between this segment and the relative global trace id.
     */
    //将全局追踪id与此segment关联
    public void relatedGlobalTrace(DistributedTraceId distributedTraceId) {
        if (relatedGlobalTraceId instanceof NewDistributedTraceId) {
            this.relatedGlobalTraceId = distributedTraceId;
        }
    }

    /**
     * After {@link AbstractSpan} is finished, as be controller by "skywalking-api" module, notify the {@link
     * TraceSegment} to archive it.
     */
    //归档，对将完成的span添加到list中
    public void archive(AbstractTracingSpan finishedSpan) {
        spans.add(finishedSpan);
    }

    /**
     * Finish this {@link TraceSegment}. <p> return this, for chaining
     */
    public TraceSegment finish(boolean isSizeLimited) {
        this.isSizeLimited = isSizeLimited;
        return this;
    }
  
    /**
     * This is a high CPU cost method, only called when sending to collector or test cases.
     *
     * @return the segment as GRPC service parameter
     */
     //转成proto的格式
    public SegmentObject transform() {
        SegmentObject.Builder traceSegmentBuilder = SegmentObject.newBuilder();
        traceSegmentBuilder.setTraceId(getRelatedGlobalTrace().getId());
        /*
         * Trace Segment
         */
        traceSegmentBuilder.setTraceSegmentId(this.traceSegmentId);
        // Don't serialize TraceSegmentReference

        // SpanObject
        for (AbstractTracingSpan span : this.spans) {
            traceSegmentBuilder.addSpans(span.transform());
        }
        traceSegmentBuilder.setService(Config.Agent.SERVICE_NAME);
        traceSegmentBuilder.setServiceInstance(Config.Agent.INSTANCE_NAME);
        traceSegmentBuilder.setIsSizeLimited(this.isSizeLimited);

        return traceSegmentBuilder.build();
    }
}
```

### TraceSegmentRef

`TraceSegmentRef`是用来让`TraceSegment`指向`父TraceSegment`，一般情况下一个`TraceSegment`只有一个`父TraceSegment`(比如RPC调用)，但也会存在多个父`TraceSegment`的情况，比如批处理任务-MQ的批量消费。

```
public class TraceSegmentRef {
    //引用类型 跨线程、跨进程
    private SegmentRefType type;
    //全局traceid
    private String traceId;
    //parent的segmentid
    private String traceSegmentId;
    private int spanId;
    //parent的服务名
    private String parentService;
    //parent的具体实例
    private String parentServiceInstance;
    //进入parentService 的那个请求
    private String parentEndpoint; 
    private String addressUsedAtClient;
 }
```

### Span

Span的分支比较多，最主要的实现是`EntrySpan`、`ExitSpan`、`LocalSpan`。\


![Span](<../../.gitbook/assets/image (9).png>)

#### AbstractSpan&#x20;

最基础的Span结构

```
public abstract class AbstractTracingSpan implements AbstractSpan {
    /**
     * Span id starts from 0.
     */
    protected int spanId;
    /**
     * Parent span id starts from 0. -1 means no parent span.
     */
    protected int parentSpanId;
    //记录Tags集合
    protected List<TagValuePair> tags;
    protected String operationName;
    protected SpanLayer layer;
    /**
     * The span has been tagged in async mode, required async stop to finish.
     */
    protected volatile boolean isInAsyncMode = false;
    /**
     * The flag represents whether the span has been async stopped
     */
    private volatile boolean isAsyncStopped = false;

    /**
     * The context to which the span belongs
     */
    protected final TracingContext owner;

    /**
     * The start time of this Span.
     */
    protected long startTime;
    /**
     * The end time of this Span.
     */
    protected long endTime;
    /**
     * Error has occurred in the scope of span.
     */
    protected boolean errorOccurred = false;

    protected int componentId = 0;

    /**
     * Log is a concept from OpenTracing spec. https://github.com/opentracing/specification/blob/master/specification.md#log-structured-data
     */
    protected List<LogDataEntity> logs;
    }
```

#### StackBasedTracingSpan

栈基础span，用栈结构来存储span，多了栈深度这个字段，其子类`EntrySpan`和`ExitSpan`可以多次`start`和`finish`。

```
public abstract class StackBasedTracingSpan extends AbstractTracingSpan {
    protected int stackDepth;
    protected String peer;
    @Override
    //当stackDepth==0时，才真正结束
    public boolean finish(TraceSegment owner) {
        if (--stackDepth == 0) {
            return super.finish(owner);
        } else {
            return false;
        }
    }
}
```

#### EntrySpan

`EntrySpan`是重复利用的，且只会记录最靠近服务侧的信息。

例如链路:Tomcat->SpringMVC->Controller ，Tomcat和SpringMVC都会创建`EntrySpan` , 但只会记录SpringMVC的信息，startTime除外。记录最靠近服务的组件信息，会包含更多的服务信息。

```
public class EntrySpan extends StackBasedTracingSpan {

    private int currentMaxDepth;

    public EntrySpan(int spanId, int parentSpanId, String operationName, TracingContext owner) {
        super(spanId, parentSpanId, operationName, owner);
        this.currentMaxDepth = 0;
    }

    /**
     * Set the {@link #startTime}, when the first start, which means the first service provided.
     */
    @Override
    public EntrySpan start() {
        //只有当第一次进入才会startTime
        if ((currentMaxDepth = ++stackDepth) == 1) {
            super.start();
        }
        //每次都清空记录的layer、tags、logs
        clearWhenRestart();
        return this;
    }

    @Override
    //当前栈深度是最大栈深度时候才去tag
    public EntrySpan tag(String key, String value) {
        if (stackDepth == currentMaxDepth || isInAsyncMode) {
            super.tag(key, value);
        }
        return this;
    }
}}
```

#### ExitSpan

`ExitSpan`是Trace链路的退出点，用来表示结束此Trace。

`ExitSpan`只能记录最开始的ExitSpan信息，因为他最靠近我们的业务服务。例如链路：MyServcie->Dubbo->HTTP client，ExitSpan只会记录Dubbo相关的信息。

```
 // ExitSpan是更靠进消费端的信息 EntrySpan是更靠近服务端的信息
public class ExitSpan extends StackBasedTracingSpan implements ExitTypeSpan {

    public ExitSpan(int spanId, int parentSpanId, String operationName, String peer, TracingContext owner) {
        super(spanId, parentSpanId, operationName, peer, owner);
    }

    public ExitSpan(int spanId, int parentSpanId, String operationName, TracingContext owner) {
        super(spanId, parentSpanId, operationName, owner);
    }

    /**
     * Set the {@link #startTime}, when the first start, which means the first service provided.
     */
    @Override
    public ExitSpan start() {
        //记录刚开始的时间
        if (++stackDepth == 1) {
            super.start();
        }
        return this;
    }

    @Override
    //只会记录第一个ExitSpan的信息
    public ExitSpan tag(String key, String value) {
        if (stackDepth == 1 || isInAsyncMode) {
            super.tag(key, value);
        }
        return this;
    }
```

所以，EntrySpan和ExitSpan记录的都是更靠近我们自身业务的一些链路信息，这个可以保证再有多个EntrySpan和ExitSpan组件是，能了解到与我们业务相关的更多细节。

#### LocalSpan

表示本地方法调用，不能多次start和finish。

#### NoopSpan

当前Span没有任何操作，其方法都是空实现，是为了统一处理Span。配置 `IgnoredTracerContext` 一起使用，在 `IgnoredTracerContext` 声明单例 ，以减少不收集 Span 时的对象创建，达到减少内存使用和 GC 时间。

## Context

TracingContext是是链路追踪的上下文。其继承结构如下：

![TracingContext impl](<../../.gitbook/assets/image (2).png>)

### AbstractTracerContext

是顶层抽象逻辑，封装了对跨进程和跨线程的方法。

```
/**
 * The <code>AbstractTracerContext</code> represents the tracer context manager.
 */
public interface AbstractTracerContext {
    /**
     * Prepare for the cross-process propagation. How to initialize the carrier, depends on the implementation.
     *
     * @param carrier to carry the context for crossing process.
     */
    //跨进程 将context注入到ContextCarrier
    void inject(ContextCarrier carrier);

    /**
     * Build the reference between this segment and a cross-process segment. How to build, depends on the
     * implementation.
     *
     * @param carrier carried the context from a cross-process segment.
     */
    //跨进程 将context提取到ContextCarrier
    void extract(ContextCarrier carrier);

    /**
     * Capture a snapshot for cross-thread propagation. It's a similar concept with ActiveSpan.Continuation in
     * OpenTracing-java How to build, depends on the implementation.
     *
     * @return the {@link ContextSnapshot} , which includes the reference context.
     */
    //跨线程 生成快照
    ContextSnapshot capture();

    /**
     * Build the reference between this segment and a cross-thread segment. How to build, depends on the
     * implementation.
     *
     * @param snapshot from {@link #capture()} in the parent thread.
     */
    //跨线程 提取快照信息
    void continued(ContextSnapshot snapshot);

    /**
     * Get the global trace id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    String getReadablePrimaryTraceId();

    /**
     * Get the current segment id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    String getSegmentId();

    /**
     * Get the active span id, if needEnhance. How to build, depends on the implementation.
     *
     * @return the string represents the id.
     */
    int getSpanId();

    /**
     * Create an entry span
     *
     * @param operationName most likely a service name
     * @return the span represents an entry point of this segment.
     */
    AbstractSpan createEntrySpan(String operationName);

    /**
     * Create a local span
     *
     * @param operationName most likely a local method signature, or business name.
     * @return the span represents a local logic block.
     */
    AbstractSpan createLocalSpan(String operationName);

    /**
     * Create an exit span
     *
     * @param operationName most likely a service name of remote
     * @param remotePeer    the network id(ip:port, hostname:port or ip1:port1,ip2,port, etc.). Remote peer could be set
     *                      later, but must be before injecting.
     * @return the span represent an exit point of this segment.
     */
    AbstractSpan createExitSpan(String operationName, String remotePeer);

    /**
     * @return the active span of current tracing context(stack)
     */
    AbstractSpan activeSpan();

    /**
     * Finish the given span, and the given span should be the active span of current tracing context(stack)
     *
     * @param span to finish
     * @return true when context should be clear.
     */
    boolean stopSpan(AbstractSpan span);

    /**
     * Notify this context, current span is going to be finished async in another thread.
     *
     * @return The current context
     */
    AbstractTracerContext awaitFinishAsync();

    /**
     * The given span could be stopped officially.
     *
     * @param span to be stopped.
     */
    void asyncStop(AsyncSpan span);

    /**
     * Get current correlation context
     */
    CorrelationContext getCorrelationContext();
}
```

### TracingContext

#### inject&#x20;

注入context到`ContextCarrier`

```
   @Override
    public void inject(ContextCarrier carrier) {
        this.inject(this.activeSpan(), carrier);
    }

    /**
     * Inject the context into the given carrier and given span, only when the active span is an exit one. This method
     * wouldn't be opened in {@link ContextManager} like {@link #inject(ContextCarrier)}, it is only supported to be
     * called inside the {@link ExitTypeSpan#inject(ContextCarrier)}
     *
     * @param carrier  to carry the context for crossing process.
     * @param exitSpan to represent the scope of current injection.
     * @throws IllegalStateException if (1) the span isn't an exit one. (2) doesn't include peer.
     */
    public void inject(AbstractSpan exitSpan, ContextCarrier carrier) {
        if (!exitSpan.isExit()) {
            throw new IllegalStateException("Inject can be done only in Exit Span");
        }

        ExitTypeSpan spanWithPeer = (ExitTypeSpan) exitSpan;
        String peer = spanWithPeer.getPeer();
        if (StringUtil.isEmpty(peer)) {
            throw new IllegalStateException("Exit span doesn't include meaningful peer information.");
        }

        carrier.setTraceId(getReadablePrimaryTraceId());
        carrier.setTraceSegmentId(this.segment.getTraceSegmentId());
        carrier.setSpanId(exitSpan.getSpanId());
        carrier.setParentService(Config.Agent.SERVICE_NAME);
        carrier.setParentServiceInstance(Config.Agent.INSTANCE_NAME);
        carrier.setParentEndpoint(first().getOperationName());
        carrier.setAddressUsedAtClient(peer);

        this.correlationContext.inject(carrier);
        this.extensionContext.inject(carrier);
    }
```

#### extract

从`ContextCarrier`提取context

```
   public void extract(ContextCarrier carrier) {
        TraceSegmentRef ref = new TraceSegmentRef(carrier);
        this.segment.ref(ref);
        this.segment.relatedGlobalTrace(new PropagatedTraceId(carrier.getTraceId()));
        AbstractSpan span = this.activeSpan();
        if (span instanceof EntrySpan) {
            span.ref(ref);
        }

        carrier.extractExtensionTo(this);
        carrier.extractCorrelationTo(this);
    }
```

#### capture

获取快照

```
  public ContextSnapshot capture() {
        ContextSnapshot snapshot = new ContextSnapshot(
                segment.getTraceSegmentId(),
                activeSpan().getSpanId(),
                getPrimaryTraceId(),
                first().getOperationName(),
                this.correlationContext,
                this.extensionContext
        );

        return snapshot;
    }
```

#### continued

从快照获取信息

```
   @Override
    public void continued(ContextSnapshot snapshot) {
        if (snapshot.isValid()) {
            TraceSegmentRef segmentRef = new TraceSegmentRef(snapshot);
            this.segment.ref(segmentRef);
            this.activeSpan().ref(segmentRef);
            this.segment.relatedGlobalTrace(snapshot.getTraceId());
            this.correlationContext.continued(snapshot);
            this.extensionContext.continued(snapshot);
            this.extensionContext.handle(this.activeSpan());
        }
    }
```

#### span相关的操作

包括`createEntrySpan`、`createExitSpan`、`createLocalSpan`、`activeSpan`、`stopSpan`。

```
  /**
     * Create an entry span
     *
     * @param operationName most likely a service name
     * @return span instance. Ref to {@link EntrySpan}
     */
    @Override
    public AbstractSpan createEntrySpan(final String operationName) {
        //是否需要限制
        if (isLimitMechanismWorking()) {
            NoopSpan span = new NoopSpan();
            return push(span);
        }
        AbstractSpan entrySpan;
        TracingContext owner = this;
        //从栈里取
        final AbstractSpan parentSpan = peek();
        final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
        //能取到并且是EntrySpan
        if (parentSpan != null && parentSpan.isEntry()) {
            /*
             * Only add the profiling recheck on creating entry span,
             * as the operation name could be overrided.
             */
            profilingRecheck(parentSpan, operationName);
            parentSpan.setOperationName(operationName);
            entrySpan = parentSpan;
            //复用此EntrySpan重新开始start
            return entrySpan.start();
        } else {
            //没有就新建一个EntrySpan添加到栈里
            entrySpan = new EntrySpan(
                    spanIdGenerator++, parentSpanId,
                    operationName, owner
            );
            entrySpan.start();
            return push(entrySpan);
        }
    }

    /**
     * Create a local span
     *
     * @param operationName most likely a local method signature, or business name.
     * @return the span represents a local logic block. Ref to {@link LocalSpan}
     */
    @Override
    public AbstractSpan createLocalSpan(final String operationName) {
        if (isLimitMechanismWorking()) {
            NoopSpan span = new NoopSpan();
            return push(span);
        }
        //获取父span的id
        AbstractSpan parentSpan = peek();
        final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
        //新建一个localSpan添加到栈里
        AbstractTracingSpan span = new LocalSpan(spanIdGenerator++, parentSpanId, operationName, this);
        span.start();
        return push(span);
    }

    /**
     * Create an exit span
     *
     * @param operationName most likely a service name of remote
     * @param remotePeer    the network id(ip:port, hostname:port or ip1:port1,ip2,port, etc.). Remote peer could be set
     *                      later, but must be before injecting.
     * @return the span represent an exit point of this segment.
     * @see ExitSpan
     */
    @Override
    public AbstractSpan createExitSpan(final String operationName, final String remotePeer) {
        if (isLimitMechanismWorking()) {
            NoopExitSpan span = new NoopExitSpan(remotePeer);
            return push(span);
        }

        AbstractSpan exitSpan;
        AbstractSpan parentSpan = peek();
        TracingContext owner = this;
        //栈顶是ExitSpan
        if (parentSpan != null && parentSpan.isExit()) {
            exitSpan = parentSpan;
        } else {
            //否则就新建一个ExitSpan添加到栈里
            final int parentSpanId = parentSpan == null ? -1 : parentSpan.getSpanId();
            exitSpan = new ExitSpan(spanIdGenerator++, parentSpanId, operationName, remotePeer, owner);
            push(exitSpan);
        }
        exitSpan.start();
        return exitSpan;
    }
       /**
     * @return the active span of current context, the top element of {@link #activeSpanStack}
     */
    @Override
    //就是栈顶的span
    public AbstractSpan activeSpan() {
        AbstractSpan span = peek();
        if (span == null) {
            throw new IllegalStateException("No active span.");
        }
        return span;
    }

    /**
     * Stop the given span, if and only if this one is the top element of {@link #activeSpanStack}. Because the tracing
     * core must make sure the span must match in a stack module, like any program did.
     *
     * @param span to finish
     */
    @Override
    public boolean stopSpan(AbstractSpan span) {
        AbstractSpan lastSpan = peek();
        //必须要是栈顶的span
        if (lastSpan == span) {
            //弹出栈并且调用finish方法，会将此span归档到segment
            if (lastSpan instanceof AbstractTracingSpan) {
                AbstractTracingSpan toFinishSpan = (AbstractTracingSpan) lastSpan;
                if (toFinishSpan.finish(segment)) {
                    pop();
                }
            } else {
                pop();
            }
        } else {
            throw new IllegalStateException("Stopping the unexpected span = " + span);
        }

        finish();

        return activeSpanStack.isEmpty();
    }
```

#### finish

结束此context

```
   /**
     * Finish this context, and notify all {@link TracingContextListener}s, managed by {@link
     * TracingContext.ListenerManager} and {@link TracingContext.TracingThreadListenerManager}
     */
    //结束此context
    private void finish() {
        if (isRunningInAsyncMode) {
            asyncFinishLock.lock();
        }
        try {
            boolean isFinishedInMainThread = activeSpanStack.isEmpty() && running;
            if (isFinishedInMainThread) {
                /*
                 * Notify after tracing finished in the main thread.
                 */
                TracingThreadListenerManager.notifyFinish(this);
            }

            if (isFinishedInMainThread && (!isRunningInAsyncMode || asyncSpanCounter == 0)) {
                //结束此segment,调用listener通知结束，这里有TraceSegmentServiceClient,负责segment的发送
                TraceSegment finishedSegment = segment.finish(isLimitMechanismWorking());
                TracingContext.ListenerManager.notifyFinish(finishedSegment);
                running = false;
            }
        } finally {
            if (isRunningInAsyncMode) {
                asyncFinishLock.unlock();
            }
        }
    }
```

### ContextManager

顾名思义，是对context的管理。将context封装成ThreadLocal类型，让context只对此线程可见。将TraceContext的方法再进行封装，即每次都要从ThreadLocal里get TraceContext。

```
public class ContextManager implements BootService {
    private static final String EMPTY_TRACE_CONTEXT_ID = "N/A";
    private static final ILog LOGGER = LogManager.getLogger(ContextManager.class);
    private static ThreadLocal<AbstractTracerContext> CONTEXT = new ThreadLocal<AbstractTracerContext>();
    private static ThreadLocal<RuntimeContext> RUNTIME_CONTEXT = new ThreadLocal<RuntimeContext>();
    //创建context的服务
    private static ContextManagerExtendService EXTEND_SERVICE;
}
```

`ContextManagerExtendService`负责Context的创建，可以使`TraceContext`和`IgnoredTracerContext`，`IgnoredTracerContext`表示和此上下文关联的链路都不采集，其内部只有一个NoopSpan的单例。方法如下：

```
   public AbstractTracerContext createTraceContext(String operationName, boolean forceSampling) {
        AbstractTracerContext context;
        /*
         * Don't trace anything if the backend is not available.
         */
        //配置不采集
        if (!Config.Agent.KEEP_TRACING && GRPCChannelStatus.DISCONNECT.equals(status)) {
            return new IgnoredTracerContext();
        }

        int suffixIdx = operationName.lastIndexOf(".");
        //包含不采集后缀
        if (suffixIdx > -1 && Arrays.stream(ignoreSuffixArray)
                                    .anyMatch(a -> a.equals(operationName.substring(suffixIdx)))) {
            context = new IgnoredTracerContext();
        } else {
            //调用samplingService判断是否需要采集
            SamplingService samplingService = ServiceManager.INSTANCE.findService(SamplingService.class);
            if (forceSampling || samplingService.trySampling(operationName)) {
                context = new TracingContext(operationName, spanLimitWatcher);
            } else {
                context = new IgnoredTracerContext();
            }
        }

        return context;
    }
```

### ContextCarrier

用来存储`TracingContext`的快照，用作跨线程传输`Context`。

## 链路采集实例

### Tomcat

拦截`org.apache.catalina.core.StandardHostValve`的`invoke`方法，拦截器是`org.apache.skywalking.apm.plugin.tomcat78x.TomcatInvokeInterceptor` 。`before`方法开始EntrySpan，`after`方法结束Span

```
    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Request request = (Request) allArguments[0];
        ContextCarrier contextCarrier = new ContextCarrier();

        //从http头中获取链路信息
        CarrierItem next = contextCarrier.items();
        while (next.hasNext()) {
            next = next.next();
            next.setHeadValue(request.getHeader(next.getHeadKey()));
        }
        String operationName =  String.join(":", request.getMethod(), request.getRequestURI());
        //根据contextCarrier生成EntrySpan
        AbstractSpan span = ContextManager.createEntrySpan(operationName, contextCarrier);
        //添加tags信息
        Tags.URL.set(span, request.getRequestURL().toString());
        Tags.HTTP.METHOD.set(span, request.getMethod());
        //设置组件名
        span.setComponent(ComponentsDefine.TOMCAT);
        //设置layer
        SpanLayer.asHttp(span);

        if (TomcatPluginConfig.Plugin.Tomcat.COLLECT_HTTP_PARAMS) {
            collectHttpParam(request, span);
        }
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        Request request = (Request) allArguments[0];
        HttpServletResponse response = (HttpServletResponse) allArguments[1];

        AbstractSpan span = ContextManager.activeSpan();
        if (IS_SERVLET_GET_STATUS_METHOD_EXIST && response.getStatus() >= 400) {
            span.errorOccurred();
            Tags.HTTP_RESPONSE_STATUS_CODE.set(span, response.getStatus());
        }
        // Active HTTP parameter collection automatically in the profiling context.
        if (!TomcatPluginConfig.Plugin.Tomcat.COLLECT_HTTP_PARAMS && span.isProfiling()) {
            collectHttpParam(request, span);
        }
        ContextManager.getRuntimeContext().remove(Constants.FORWARD_REQUEST_FLAG);
        //停止此span
        ContextManager.stopSpan();
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
                                      Class<?>[] argumentsTypes, Throwable t) {
        AbstractSpan span = ContextManager.activeSpan();
        span.log(t);
    }
```

### Dubbo

拦截`org.apache.dubbo.monitor.support.MonitorFilter`类的`invoke`方法。其拦截器是`org.apache.skywalking.apm.plugin.asf.dubbo.DubboInterceptor`

Dubbo要区分服务提供者和服务调用者，提供者要创建EntrySpan，调用者要创建ExitSpan。

```
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        Invoker invoker = (Invoker) allArguments[0];
        Invocation invocation = (Invocation) allArguments[1];
        RpcContext rpcContext = RpcContext.getContext();
        boolean isConsumer = rpcContext.isConsumerSide();
        URL requestURL = invoker.getUrl();

        AbstractSpan span;

        final String host = requestURL.getHost();
        final int port = requestURL.getPort();

        boolean needCollectArguments;
        int argumentsLengthThreshold;
        //消费端，创建ExitSpan
        if (isConsumer) {
            final ContextCarrier contextCarrier = new ContextCarrier();
            span = ContextManager.createExitSpan(generateOperationName(requestURL, invocation), contextCarrier, host + ":" + port);
            //invocation.getAttachments().put("contextData", contextDataStr);
            //@see https://github.com/alibaba/dubbo/blob/dubbo-2.5.3/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/RpcInvocation.java#L154-L161
            //将carrier的items添加到dubbo的rpcContext,链路信息随着dubbo请求一起发送
            CarrierItem next = contextCarrier.items();
            while (next.hasNext()) {
                next = next.next();
                rpcContext.setAttachment(next.getHeadKey(), next.getHeadValue());
                if (invocation.getAttachments().containsKey(next.getHeadKey())) {
                    invocation.getAttachments().remove(next.getHeadKey());
                }
            }
            needCollectArguments = DubboPluginConfig.Plugin.Dubbo.COLLECT_CONSUMER_ARGUMENTS;
            argumentsLengthThreshold = DubboPluginConfig.Plugin.Dubbo.CONSUMER_ARGUMENTS_LENGTH_THRESHOLD;
       //提供端，创建EntrySpan
        } else {
            ContextCarrier contextCarrier = new ContextCarrier();
            CarrierItem next = contextCarrier.items();
            while (next.hasNext()) {
                next = next.next();
                next.setHeadValue(rpcContext.getAttachment(next.getHeadKey()));
            }

            span = ContextManager.createEntrySpan(generateOperationName(requestURL, invocation), contextCarrier);
            span.setPeer(rpcContext.getRemoteAddressString());
            needCollectArguments = DubboPluginConfig.Plugin.Dubbo.COLLECT_PROVIDER_ARGUMENTS;
            argumentsLengthThreshold = DubboPluginConfig.Plugin.Dubbo.PROVIDER_ARGUMENTS_LENGTH_THRESHOLD;
        }

        Tags.URL.set(span, generateRequestURL(requestURL, invocation));
        collectArguments(needCollectArguments, argumentsLengthThreshold, span, invocation);
        span.setComponent(ComponentsDefine.DUBBO);
        SpanLayer.asRPCFramework(span);
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        Result result = (Result) ret;
        try {
            if (result != null && result.getException() != null) {
                dealException(result.getException());
            }
        } catch (RpcException e) {
            dealException(e);
        }
       //方法结束，停止span
        ContextManager.stopSpan();
        return ret;
    }
```

### tomcat->servlet->dubbo consumer  --远程--- > dubbo provider

设想这样一个系统，我们用tomcat部署了一个web应用，其中用到了Dubbo的Rpc。那我们的链路是这样的:

1. 进入tomcat，创建一个EntrySpan，他会从Http Header里提取链路信息，这里为空。
2. 进入我们自己的servlet，调用dubbo
3. Dubbo消费端发起rpc调用，创建ExitSpan，将链路信息写入dubbo 的rpcContext
4. Dubbo服务端收到请求后，从rpcContext读取链路信息，创建新的链路上下文。
5. Dubbo服务端方法返回，结束链路，调用stopSpan，因为栈空了，则context.finish,链路信息发送到aop。
6. Dubbo服务端收到请求，方法返回，结束链路，调用stopSpan。
7. Tomcat方法返回，结束链路，调用stopSpan，因为栈空了，则context.finish,链路信息发送到aop。
8. aop收到完整的链路信息，组装数据。

## Trace数据接收

TODO:
