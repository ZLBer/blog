---
description: The OpenTracing Semantic Specification
---

# OpenTracing语义规范

{% hint style="info" %}
原文地址:[https://github.com/opentracing/specification/blob/master/specification.md](https://github.com/opentracing/specification/blob/master/specification.md)
{% endhint %}

### The Big Picture: OpenTracing's Scope&#x20;

### 从全局看OpenTracing的范围

OpenTracing 的核心规范（即本文档）有意不了解特定下游跟踪或监控系统的细节

### The OpenTracing Data Model

OpenTracing 中的跟踪由它们的 Span 隐式定义。特别是，可以将 Trace 视为 Spans 的有向无环图（DAG），其中 Spans 之间的边称为 References。

例如，以下是由 8 个 Span 组成的示例 Trace：

```
Causal relationships between Spans in a single Trace


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C is a `ChildOf` Span A)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G `FollowsFrom` Span F)


```

有时使用时间轴更容易可视化跟踪，如下图所示：

```
Temporal relationships between Spans in a single Trace


––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

每个 Span 都封装了以下状态：

* 操作名称，An operation name
* 开始时间，A start timestamp
* 完成时间，A finish timestamp
* 0或多个键值对key:value，Span Tags。这些key必须是字符串，这些value可以是字符串、布尔值、数字类型。

A set of zero or more key:value **Span Tags**. The keys must be strings. The values may be strings, bools, or numeric types.

\


\
