---
description: Skywalking metrics collection
---

# skywalking 指标采集

## metrics数据结构

Counter、Gauge、Histogram是skywalking支持的metrics累行，均继承自`BaseMeter`，`BaseMeter`有个`MeterId`字段，还封装了`getName`、`getTag`、`transform`(转成grpc格式)等方法。

```
public abstract class BaseMeter {
    protected final MeterId meterId;
}
```

MeterId类如下，包括name, type(三种指标类型的枚举), tags, labels字段。

```
public class MeterId {

    private final String name;
    private final MeterType type;
    private final List<MeterTag> tags;

    // Labels are used to report meter to the backend.
    private List<Label> labels;
    }
```

### Counter是只增不减

```
public class Counter extends BaseMeter {

    //线程安全的double增加器
    protected final DoubleAdder count;
    protected final CounterMode mode;
    private final AtomicReference<Double> previous = new AtomicReference();

    public Counter(MeterId meterId, CounterMode mode) {
        super(meterId);
        this.count = new DoubleAdder();
        this.mode = mode;
    }

    public void increment(double count) {
        this.count.add(count);
    }

    public double get() {
        return count.doubleValue();
    }

    @Override
    public MeterData.Builder transform() {
        // using rate mode or increase
        final double currentValue = get();
        double count;
        //根据不同的mode计算不同的结果
        //INCREMENT模式直接返回当前值
        //RATE模式返回当前-上次报告的值
        if (Objects.equals(mode, CounterMode.RATE)) {
            final Double previousValue = previous.getAndSet(currentValue);

            // calculate the add count
            //特殊处理提一次汇报
            if (previousValue == null) {
                count = currentValue;
            } else {
                count = currentValue - previousValue;
            }
        } else {
            count = currentValue;
        }

        //组装并发返回构造器
        final MeterData.Builder builder = MeterData.newBuilder();
        builder.setSingleValue(MeterSingleValue.newBuilder()
            .setName(getName())
            .addAllLabels(transformTags())
            .setValue(count).build());

        return builder;
    }
}
```

### Gauge可以表示数值的浮动

```
public class Gauge extends BaseMeter {
    private static final ILog LOGGER = LogManager.getLogger(Gauge.class);
    protected Supplier<Double> getter;

    public Gauge(MeterId meterId, Supplier<Double> getter) {
        super(meterId);
        this.getter = getter;
    }

    /**
     * Get value
     */
    public double get() {
        final Double data = getter.get();
        return data == null ? 0 : data;
    }

    @Override
    public MeterData.Builder transform() {
        double count;
        try {
            count = get();
        } catch (Exception e) {
            LOGGER.warn(e, "Cannot get the count in meter:{}", meterId.getName());
            return null;
        }

        final MeterData.Builder builder = MeterData.newBuilder();
        builder.setSingleValue(MeterSingleValue.newBuilder()
            .setName(getName())
            .addAllLabels(transformTags())
            .setValue(count).build());

        return builder;
    }
    }
```

### Histogram直方图

用桶来统计每个value有多少个计数

```
public class Histogram extends BaseMeter {
    //value从下到大排列的桶
    protected final Bucket[] buckets;

  
    public Histogram(MeterId meterId, List<Double> steps) {
        super(meterId);
        this.buckets = initBuckets(steps);
    }

    /**
     * Add value into the histogram, automatic analyze what bucket count need to be increment [step1, step2)
     */
    //对某个值的计数+1
    public void addValue(double value) {
        Bucket bucket = findBucket(value);
        if (bucket == null) {
            return;
        }

        bucket.increment(1L);
    }

    
    //二分法查找桶的位置
    //初始化的时候保障value序列是从小到大的
    private Bucket findBucket(double value) {
        int low = 0;
        int high = buckets.length - 1;

        while (low <= high) {
            int mid = (low + high) / 2;
            if (buckets[mid].bucket < value)
                low = mid + 1;
            else if (buckets[mid].bucket > value)
                high = mid - 1;
            else
                return buckets[mid];
        }

        // because using min value as bucket, need using previous bucket
        low -= 1;

        return low < buckets.length && low >= 0 ? buckets[low] : null;
    }

    private Bucket[] initBuckets(List<Double> steps) {
        return steps.stream().map(Bucket::new).toArray(Bucket[]::new);
    }

    @Override
    public MeterData.Builder transform() {
        final MeterData.Builder builder = MeterData.newBuilder();

        // get all values
        List<MeterBucketValue> values = Arrays.stream(buckets)
                                              .map(Histogram.Bucket::transform).collect(Collectors.toList());

        return builder.setHistogram(MeterHistogram.newBuilder()
                                                  .setName(getName())
                                                  .addAllLabels(transformTags())
                                                  .addAllValues(values)
                                                  .build());
    }

    /**
     * Histogram bucket
     */
    protected static class Bucket {
        protected double bucket;
        protected AtomicLong count = new AtomicLong();

        public Bucket(double bucket) {
            this.bucket = bucket;
        }

        public void increment(long count) {
            this.count.addAndGet(count);
        }

        public MeterBucketValue transform() {
            return MeterBucketValue.newBuilder()
                                   .setBucket(bucket)
                                   .setCount(count.get())
                                   .build();
        }

        @Override
        public boolean equals(Object o) {
            if (this == o)
                return true;
            if (o == null || getClass() != o.getClass())
                return false;
            Bucket bucket1 = (Bucket) o;
            return bucket == bucket1.bucket;
        }

        @Override
        public int hashCode() {
            return Objects.hash(bucket);
        }
    }
}
```

## Metrics的采集

Metrics的收集有`MeterService`负责，调用`register`方法对meter进行注册，将其添加到`ConcurrentHashMap<MeterId, BaseMeter> meterMap` 里：

```
    //添加meter
    public <T extends BaseMeter> T register(T meter) {
        if (meter == null) {
            return null;
        }
        if (meterMap.size() >= Config.Meter.MAX_METER_SIZE) {
            LOGGER.warn(
                "Already out of the meter system max size, will not report. meter name:{}", meter.getName());
            return meter;
        }

        final BaseMeter data = meterMap.putIfAbsent(meter.getId(), meter);
        return data == null ? meter : (T) data;
    }
```

`MeterService`启动的时候，会启动一个定时任务，每隔`REPORT_INTERVAL`间隔一下run方法：

```
    @Override
    public void boot() {
        if (Config.Meter.ACTIVE) {
            //每隔REPORT_INTERVAL间隔一下run方法
            reportMeterFuture = Executors.newSingleThreadScheduledExecutor(
                new DefaultNamedThreadFactory("MeterReportService")
            ).scheduleWithFixedDelay(new RunnableWithExceptionProtection(
                this,
                t -> LOGGER.error("Report meters failure.", t)
            ), 0, Config.Meter.REPORT_INTERVAL, TimeUnit.SECONDS);
        }
    }
```

`run`方法就是将`meterMap`传给`MeterSender`：

```
    public void run() {
        if (meterMap.isEmpty()) {
            return;
        }
        //将meterMap传给MeterSender
        sender.send(meterMap, this);
    }
```

那么是怎么调用`register`方法，将metrics添加进来的呢？ 其实是skywalking拦截了指标类的构造方法，对其构造方法进行增强。

```
    @Override
    public void onConstruct(EnhancedInstance objInst, Object[] allArguments) {
        final MeterId meterId = (MeterId) allArguments[0];
        final Counter.Mode mode = (Counter.Mode) allArguments[1];

        final org.apache.skywalking.apm.agent.core.meter.Counter counter =
            new org.apache.skywalking.apm.agent.core.meter.Counter(MeterIdConverter.convert(meterId),
                mode == Counter.Mode.RATE ? CounterMode.RATE : CounterMode.INCREMENT);

        if (METER_SERVICE == null) {
            METER_SERVICE = ServiceManager.INSTANCE.findService(MeterService.class);
        }
        //register当前指标并设置到DynamicField上
        objInst.setSkyWalkingDynamicField(METER_SERVICE.register(counter));
    }
```

## Metrics的发送

`MeterSender`被调用`send`之后，简单的转换数据结构，并通过grpc发送。

```
    public void send(Map<MeterId, BaseMeter> meterMap, MeterService meterService) {
        //判断grpc的channel是不是连接
        if (status == GRPCChannelStatus.CONNECTED) {
            StreamObserver<MeterData> reportStreamObserver = null;
            final GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);
            try {
                reportStreamObserver = meterReportServiceStub.withDeadlineAfter(
                    GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS
                ).collect(new StreamObserver<Commands>() {
                    @Override
                    public void onNext(Commands commands) {
                    }

                    @Override
                    public void onError(Throwable throwable) {
                        status.finished();
                        if (LOGGER.isErrorEnable()) {
                            LOGGER.error(throwable, "Send meters to collector fail with a grpc internal exception.");
                        }
                        ServiceManager.INSTANCE.findService(GRPCChannelManager.class).reportError(throwable);
                    }

                    @Override
                    public void onCompleted() {
                        status.finished();
                    }
                });

                final StreamObserver<MeterData> reporter = reportStreamObserver;
                //转换数据结构并发送
                transform(meterMap, meterData -> reporter.onNext(meterData));
            } catch (Throwable e) {
                if (!(e instanceof StatusRuntimeException)) {
                    LOGGER.error(e, "Report meters to backend fail.");
                    return;
                }
                final StatusRuntimeException statusRuntimeException = (StatusRuntimeException) e;
                if (statusRuntimeException.getStatus().getCode() == Status.Code.UNIMPLEMENTED) {
                    LOGGER.warn("Backend doesn't support meter, it will be disabled");

                    meterService.shutdown();
                }
            } finally {
                if (reportStreamObserver != null) {
                    reportStreamObserver.onCompleted();
                }
                status.wait4Finish();
            }
        }
    }

    protected void transform(final Map<MeterId, BaseMeter> meterMap,
                             final Consumer<MeterData> consumer) {
        // build and report meters
        boolean hasSendMachineInfo = false;
        for (BaseMeter meter : meterMap.values()) {
            final MeterData.Builder dataBuilder = meter.transform();
            if (dataBuilder == null) {
                continue;
            }

            // only send the service base info at the first data
            if (!hasSendMachineInfo) {
                dataBuilder.setService(Config.Agent.SERVICE_NAME);
                dataBuilder.setServiceInstance(Config.Agent.INSTANCE_NAME);
                dataBuilder.setTimestamp(System.currentTimeMillis());
                hasSendMachineInfo = true;
            }
            //消费meter 即reportStreamObserver.onNext(data)
            consumer.accept(dataBuilder.build());
        }
    }
```

除了默认的`MeterSender`之外，还有`KafkaMeterSender`的覆盖实现，可以将`metrics`数据发送到消息队列。

## Metrics接收

我们在看一下aop的代码，看下metrics是如何接收的，进入org.apache.skywalking.oap.server.receiver.meter 包：

![](<../../.gitbook/assets/image (6).png>)

TODO：
