---
description: Analysis of Java ide debugging code principle
---

# JAVA IDE代码调试原理浅析

{% hint style="info" %}
当你用IDA进行代码调试的时候发生了什么？为什么打个断点代码运行就能在这里停住？条件断点是怎么实现的？本文将从宏观说明其实现方式，点到为止。
{% endhint %}

TODO：

可以看一下点击debug之后IDE的启动命定

```
java -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:61353,suspend=y,server=n -javaagent:/Users/zlb/Library/Caches/JetBrains/IntelliJIdea2021.2/captureAgent/debugger-agent.jar -Dfile.encoding=UTF-8 -classpath /Users/zlb/IdeaProjects/learn/out/production/learn:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar wk
Connected to the target VM, address: '127.0.0.1:61353', transport: 'socket'
```

首先要明白什么是 `jdwp`(Java Debug Wire Protocol) java调试线协议，用来：

> The Java Debug Wire Protocol (JDWP) is the protocol used for communication between a debugger and the Java virtual machine (VM) which it debugs (hereafter called the target VM). JDWP is optional; it might not be available in some implementations of the JDK. The existence of JDWP can allow the same debugger to work
>
> * in a different process on the same computer, or
> * on a remote computer,

jdwp是调试器和java虚拟机之间用来调试交流的一种协议。可以用来对远程主机或者一台电脑上的两个进程之间进行交互。你想啊，通过IDE进行debug肯定是需要IDE与JVM进行交互，来设置断点啊，获取变量信息啊等等。下面是官方的文档，包括介绍和协议的格式等。

* [Java Debug Wire Protocol](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/jdwp-spec.html)

然后IDE还启动了一个java agent: `debugger-agent.jar`，主要的工作是通过agent来完成的，这里不介绍java agent相关的东西。
