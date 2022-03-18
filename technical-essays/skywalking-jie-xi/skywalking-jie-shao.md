---
description: Skywalking introduction
---

# skywalking介绍

Skywalking是一款比较火的APM(Application Performance Management)程序，负责应用程序的可观测性。其主要流程如下：首先是各种agent负责采集数据(Trace、Metrics、Log)，agent可以是自动注入和手动采集方式，通过grpc或kafka或http发送到后端OAP系统，OAP负责分析和展示。

![skywalking官方架构图](<../../.gitbook/assets/image (3) (1) (1).png>)



写这篇文章的目的主要是介绍Skywalking主要模块的功能，让自己对其源码更加熟悉。
