---
description: Skywalking logs collect
---

# skywalking 日志采集

首先要理解这里的log不是skywalking的log，而是你部署项目所产生的log，

skywalking自己实现了日志系统，不用log4j这些原因可能是，应用里可能用log4j，这样拦截的时候可能会出现日志混乱。

## Logs采集

skywalking针对log4j不同版本、logback都自定义了`CustomAppender`，即自定义日志导出，但`CustomAppender`的`append`方法都是空实现，其实是对这些组件又进行了增强。

![](<../../.gitbook/assets/image (4).png>)

看一下针对log4j v1的拦截器，拦截了`GRPCLogClientAppender`的`append`方法，其拦截器是：

`org.apache.skywalking.apm.toolkit.activation.log.log4j.v1.x.log.GRPCLogAppenderInterceptor`

看一下`beforeMethod`方法：

```
  public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        if (Objects.isNull(client)) {
            //找到LogReportServiceClient,即log的发送器
            client = ServiceManager.INSTANCE.findService(LogReportServiceClient.class);
            if (Objects.isNull(client)) {
                return;
            }
        }
        //获取到log4j的LoggingEvent
        LoggingEvent event = (LoggingEvent) allArguments[0];
        if (Objects.nonNull(event)) {
            //转成skywaking定义logData，并交给找到LogReportServiceClient
            client.produce(transform((AppenderSkeleton) objInst, event));
        }
    }
```

tranform方法

```
    private LogData transform(final AppenderSkeleton appender, LoggingEvent event) {
        //一些列的结构转化
        LogData.Builder builder = LogData.newBuilder()
                .setTimestamp(event.getTimeStamp())
                .setService(Config.Agent.SERVICE_NAME)
                .setServiceInstance(Config.Agent.INSTANCE_NAME)
                .setTraceContext(TraceContext.newBuilder()
                        .setTraceId(ContextManager.getGlobalTraceId())
                        .setSpanId(ContextManager.getSpanId())
                        .setTraceSegmentId(ContextManager.getSegmentId())
                        .build())
                .setTags(LogTags.newBuilder()
                        .addData(KeyStringValuePair.newBuilder()
                                .setKey("level").setValue(event.getLevel().toString()).build())
                        .addData(KeyStringValuePair.newBuilder()
                                .setKey("logger").setValue(event.getLoggerName()).build())
                        .addData(KeyStringValuePair.newBuilder()
                                .setKey("thread").setValue(event.getThreadName()).build())
                        .build())
                .setBody(LogDataBody.newBuilder().setType(LogDataBody.ContentCase.TEXT.name())
                                    .setText(TextLog.newBuilder().setText(transformLogText(appender, event)).build()).build());
        //添加trace相关的信息，这里是让log与trace关联
        return -1 == ContextManager.getSpanId() ? builder.build()
                : builder.setTraceContext(TraceContext.newBuilder()
                        .setTraceId(ContextManager.getGlobalTraceId())
                        .setSpanId(ContextManager.getSpanId())
                        .setTraceSegmentId(ContextManager.getSegmentId())
                        .build()).build();
    }
```

其他日志组件的逻辑也是与此类似。

## Logs发送

`LogReportServiceClient`负责log的接收和发送。

`LogReportServiceClient`是`BootService`，必然有`boot`方法，看下其`boot`方法的实现，其初始化了`DataCarrier`并将自己作为`consumer`传入(`LogReportServiceClient`实现了`IConsumer`接口)

```
@Override
public void boot() throws Throwable {
    //初试话DataCarrier
    carrier = new DataCarrier<>("gRPC-log", "gRPC-log",
                                Config.Buffer.CHANNEL_SIZE,
                                Config.Buffer.BUFFER_SIZE,
                                BufferStrategy.IF_POSSIBLE
    );
    //添加consumer逻辑
    carrier.consume(this, 1);
}
```

`produce`负责接收数据：

```
    public void produce(LogData logData) {
        //将logData交给carrier
        if (Objects.nonNull(logData) && !carrier.produce(logData)) {
            if (LOGGER.isDebugEnable()) {
                LOGGER.debug("One log has been abandoned, cause by buffer is full.");
            }
        }
    }
```

`consume`方法负责发送数据:

```
    public void consume(final List<LogData> dataList) {
        if (CollectionUtil.isEmpty(dataList)) {
            return;
        }

        if (GRPCChannelStatus.CONNECTED.equals(status)) {
            GRPCStreamServiceStatus status = new GRPCStreamServiceStatus(false);

            StreamObserver<LogData> logDataStreamObserver = logReportServiceStub
                .withDeadlineAfter(Collector.GRPC_UPSTREAM_TIMEOUT, TimeUnit.SECONDS)
                .collect(
                new StreamObserver<Commands>() {
                    @Override
                    public void onNext(final Commands commands) {
                     //空实现
                    }

                    @Override
                    public void onError(final Throwable throwable) {
                        status.finished();
                        LOGGER.error(throwable, "Try to send {} log data to collector, with unexpected exception.",
                                     dataList.size()
                        );
                        ServiceManager.INSTANCE
                            .findService(GRPCChannelManager.class)
                            .reportError(throwable);
                    }

                    @Override
                    public void onCompleted() {
                        status.finished();
                    }
                });
           //一次发送logData
            for (final LogData logData : dataList) {
                logDataStreamObserver.onNext(logData);
            }
            logDataStreamObserver.onCompleted();
            status.wait4Finish();
        }
    }
```

到此`LogData`便通过GRPC被发送到AOP后端。

## Logs接收

TODO：
