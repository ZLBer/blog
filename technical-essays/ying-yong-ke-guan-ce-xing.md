---
description: Application observability
---

# 应用可观测性

最近在看Skywalking的源码，感觉有必要了解一些OpenTracing和OpenTelemetry。

首先可以了解一下什么是OpenTracing，包括Trace、Span、Span之间怎么联系、Span如何跨进程传播，下面是OpenTracing的官方文档，正在尝试翻译：

[OpenTracing的规范](https://github.com/opentracing/specification/blob/master/specification.md)

[翻译版：OpenTracing的规范](../article-translation/opentracing-yu-yi-gui-fan.md)

看完之后，就可以动手进行实战了。

[OpenTracing教程](https://github.com/yurishkuro/opentracing-tutorial)

有大佬翻译的java版本：[OpenTracing教程-java版](http://niyanchun.com/opentracing-introduction.html)

很遗憾，OpenTracing的项目已经被废弃了(DEPRECATED)，出现了新的规范OpenTelemetry，OpenTracing更关注的是Traces，而OpenTelemetry包含Traces、Metrics、Logs等，前者是后者的子集，且OpenTelemetry在云原生架构下更加适应。

[OpenTelemetry的规范](https://github.com/open-telemetry/opentelemetry-specification)

[OpenTelemetry 简析](https://mp.weixin.qq.com/s/n4eVf2KZRIp2yKACk88qJA)
