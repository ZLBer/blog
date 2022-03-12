---
description: Application observability
---

# 应用可观测性

最近在看Skywalking的源码，感觉有必要学习和整理一些OpenTracing、OpenTelemetry和OpenMetrics的知识。

### OpenTracing

首先可以了解一下什么是OpenTracing，包括Trace、Span、Span之间怎么联系、Span如何跨进程传播，下面是OpenTracing的官方文档以及翻译版本。

* [OpenTracing规范](https://github.com/opentracing/specification/blob/master/specification.md)
* [翻译版：OpenTracing规范](https://github.com/opentracing-contrib/opentracing-specification-zh/blob/master/specification.md)

看完之后，就可以动手进行实战了，如何create一个span，如何把不同span联系起来，如何通过http传播span，都可以在这个教程里看到。

* [OpenTracing实战教程](https://github.com/yurishkuro/opentracing-tutorial)
* [翻译java版本：OpenTracing实战教程-java版](http://niyanchun.com/opentracing-introduction.html)

### OpenTelemetry

很遗憾，OpenTracing的项目已经被废弃了(DEPRECATED)，出现了新的规范OpenTelemetry，OpenTracing更关注的是Traces，而OpenTelemetry包含Traces、Metrics、Logs等，前者是后者的子集，且OpenTelemetry在云原生架构下更加适应。

* [OpenTelemetry的规范](https://github.com/open-telemetry/opentelemetry-specification)
* [OpenTelemetry 简析](https://mp.weixin.qq.com/s/n4eVf2KZRIp2yKACk88qJA)

了解之后，可以看一下官方文档的实战：

* [Manual Instrumentation](https://opentelemetry.io/docs/instrumentation/java/manual/)

除了手动上报之外，还支持许多组件的自动上报，比如[java侧支持的组件](https://opentelemetry.io/docs/instrumentation/java/automatic/)。感觉java这边支持的组件非常之多，可能是因为java生态的繁荣和语言agent机制的原因。

OpenTelemetry的一个核心概念是Collector，可以和应用集成部署，也可以单独部署，Collector主要负责接收(receiver)、处理(processor)、导出(exporter)观测数据，比如可以接收Skywalking上报的数据，并导出到Skywalking的OAP，只需要适配相应的接口即可，可以看到[其github的库](https://github.com/open-telemetry/opentelemetry-collector-contrib)已经有许多组件进行适配了。

### OpenMetrics

OpenTelemetry其实相当于包含了OpenMetrics，至少是符合OpenMetrics的模型。OpenMetrics定了个指标的采集格式:Gauge、Counter、Histogram、Summary等，可以去了解下这些格式。

* [OpenMetrics规范](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md)

提到OpenMetrics就不得不提**Prometheus，**OpenMetrics定义了规范，Prometheus则提供了一套完整的实现，报错exporter收集数据，server汇总分析存储数据，其更适合云原生架构。

* 一本不错的教程：[prometheus-book](https://yunlzheng.gitbook.io/prometheus-book/)

![Prometheus架构](<../.gitbook/assets/image (1).png>)
