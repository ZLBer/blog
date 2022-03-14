---
description: Analysis of Java ide debugging code principle
---

# JAVA IDE代码调试原理浅析

{% hint style="info" %}
当你用IDA进行代码调试的时候发生了什么？为什么打个断点代码运行就能在这里停住？条件断点是怎么实现的？本文将从宏观说明其实现方式，点到为止。
{% endhint %}

## 点击Debug之后发生了什么？

可以看一下点击debug之后IDE的启动命令：

```
java -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:61353,suspend=y,server=n -javaagent:/Users/zlb/Library/Caches/JetBrains/IntelliJIdea2021.2/captureAgent/debugger-agent.jar -Dfile.encoding=UTF-8 -classpath /Users/zlb/IdeaProjects/learn/out/production/learn:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar wk
Connected to the target VM, address: '127.0.0.1:61353', transport: 'socket'
```

* transport ： 传输方式，有 socket (dt\_socket)和 shared memory(dt\_shmem)
* server（y/n）： 是主动连接调试器还是作为服务器等待调试器连接
* address： 调试服务器的端口号，客户端用来连接服务器的端口号
* suspend（y/n）：值是 y 或者 n，若为 y，启动时候自己程序的 VM 将会暂停（挂起），直到客户端进行连接，若为 n，自己程序的 VM 不会挂起&#x20;

若有异议，可以直接看官方文档那个，非常详细：[官方文档-Connection and Invocation Details](https://docs.oracle.com/en/java/javase/14/docs/specs/jpda/conninv.html)

* &#x20;  \-agentlib:jwp ：用jwp的lib包 ，那jwp是什么？

## Java Platform Debug Architecture

> * **Java Debug Interface** (**JDI**) - a high-level API used by a debugger to debug Java applications in a target JVM
>   * **JDI Connectors and Transports** - an auxiliary API for attaching to a target JVM in different ways using different transport channels from the debugger side
> * **Java Debug Wire Protocol** (**JDWP**) - a specification of a protocol used for communication between the debugger and the JDWP agent loaded into the JVM process
>   * **JDWP Transport Interface** - an auxiliary API that specifies way to use different transport channels from the JDWP agent side
> * **Java Virtual Machine Tool Interface** (**JVMTI**) - a low-level API exposed by a JVM to a JDWP agent (or any other agent) to control execution of a running Java[\*](https://svn.apache.org/repos/asf/harmony/enhanced/java/trunk/jdktools/modules/jpda/doc/JDWP\_agent.htm#\*) application and provide access to its data

java平台调试架构包括：java调试接口(JDI)、java调试协议(JDWP)、JDWP代理等。看下面的架构图，调试器(Debugger)，通过JDWP协议与目标JVM进行交互。

![JPDA](<../.gitbook/assets/image (1).png>)

### Java Debug Wire Protocol

首先要明白什么是 `JDWP`(Java Debug Wire Protocol) java调试线协议，用来：

> The Java Debug Wire Protocol (JDWP) is the protocol used for communication between a debugger and the Java virtual machine (VM) which it debugs (hereafter called the target VM). JDWP is optional; it might not be available in some implementations of the JDK. The existence of JDWP can allow the same debugger to work
>
> * in a different process on the same computer, or
> * on a remote computer,

JDWP是调试器和java虚拟机之间用来调试交流的一种协议。可以用来对远程主机或者一台电脑上的两个进程之间进行交互。你想啊，通过IDE进行debug肯定是需要IDE与JVM进行交互，来设置断点啊，获取变量信息啊等等。下面是官方的文档，包括介绍和协议的格式等。

* [Java Debug Wire Protocol](https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/jdwp-spec.html)

### **Java Debug Interface**

我们直接去操作的是JDI(Java Debug Interface),用来屏蔽底层复杂的细节。`com.sun.jdi`是官方提供的debug工具包,下面两篇文章是用该工具包进行debug的示例，比较易懂：

[Java 调试接口 API (JDI) – Hello World示例|适合初学者的编程调试](https://itsallbinary.com/java-debug-interface-api-jdi-hello-world-example-programmatic-debugging-for-beginners/)

[Java 调试接口 API (JDI) – Hello World 示例 |单步调试执行代码行](https://itsallbinary.com/java-debug-interface-api-jdi-hello-world-example-programmatic-stepping-through-the-code-lines/)

### **JDWP Agent**&#x20;

> **The JDWP agent is a JPDA component responsible for executing debugger commands sent to a target JVM.** The debugger establishes a connection with the JDWP agent using **any available transport** and then interacts with the JDWP agent according to the **JDWP specification** . This way, the debugger uses the connection to send commands and get replies with requested data and to set requests for specific events and receive asynchronous event packets with requested data when these events are triggered.
>
> **In the current implementation, the JDWP agent is a `.dll` library loaded into the JVM process on JVM start according to command-line options.** The agent accesses JVM and application data via the JVMTI and JNI interfaces exposed by JVM. The agent receives JDWP commands from the debugger and triggers their execution using corresponding calls to JVMTI and JNI functions. The JDWP agent works as a usual JVMTI agent and follows the JVMTI specification.

通过上面的介绍可以看到agent的实现方式是.dll，而不是以java代码的形式存在，所以在jdk的包中找不到。JDWP Agent就是负责解析JDWP协议的代理，然后调用JNI接口去操作JVM。下面的文章很详细的介绍了Agent的实现。

* [JDWP Agent实现描述](https://svn.apache.org/repos/asf/harmony/enhanced/java/trunk/jdktools/modules/jpda/doc/JDWP\_agent.htm)

## IDE需要做的事

IDE会启动一些后台进程与VM进行交互，里面会封装IDE如果通过JDI操作debug。比如我们要实现断点，IDE收到断点要求之后，封装请求与目标JVM进行通信。然后监听目标JVM的Event，根据不同的事件进行不同的处理。

### 条件断点(Condition Breakpoint)是怎么实现的？

java的debug sdk是不支持条件断点的，那IDE的条件断点是怎么做的呢？我也是查了很久的资料，直到找到了eclipse调试器的源码，才大概明白了其中的原理。之前主要的疑惑是：当你设置了条件断点之后，是直接将这个条件修改到字节码中吗？如果是，又是怎么做的？

在查阅代码的过程中，其实并不是修改了代码，而是在agent里进行判断。大概的实现是这样的：当我们向目标JVM发出断点请求的时候，目标JVM会返回给我们中断的位置集合，比如说在for循环中。然后将条件断掉的表达式封装成Expression，将Location里的变量值与Expression 的值进行比较，然后判断哪些Location需要真的暂停，返回给IDE即可。

* [eclipse调试器源码 ](https://git.eclipse.org/c/ajdt/org.eclipse.ajdt.git/)

也有看IDEA社区版的调试器的代码，这里也贴出来：

* [intellij-community 调试器源码](https://github.com/JetBrains/intellij-community/tree/master/java/debugger)

为了讲清楚条件变量，我们改一下上面的例子,改成for循环。

被调试程序，一个简单的for循环：

```
public class HelloWorld {
	public static void main(String[] args) {
           for(int i=0;i<10;i++){
             System.out.print(i);
       }		
	}

}
```

调试器程序，我们在第4行for循环里打断点，重点代码我都有注释：

```
import java.util.Map;

import com.sun.jdi.Bootstrap;
import com.sun.jdi.ClassType;
import com.sun.jdi.LocalVariable;
import com.sun.jdi.Location;
import com.sun.jdi.StackFrame;
import com.sun.jdi.VMDisconnectedException;
import com.sun.jdi.Value;
import com.sun.jdi.VirtualMachine;
import com.sun.jdi.connect.Connector;
import com.sun.jdi.connect.LaunchingConnector;
import com.sun.jdi.event.BreakpointEvent;
import com.sun.jdi.event.ClassPrepareEvent;
import com.sun.jdi.event.Event;
import com.sun.jdi.event.EventSet;
import com.sun.jdi.request.BreakpointRequest;
import com.sun.jdi.request.ClassPrepareRequest;

/**
 * Hello world example for Java Debugging API i.e. JDI. Very basic & simple
 * example.
 * 
 * @author ravik
 *
 */
public class HelloWorldJDIExample {

	public static void main(String[] args) throws Exception {

		Class classToDebug = HelloWorld.class;
		int lineNumberToPutBreakpoint = 4;

		/*
		 * Prepare connector, set class to debug & launch VM.
		 */
		LaunchingConnector launchingConnector = Bootstrap.virtualMachineManager().defaultConnector();
		Map<String, Connector.Argument> env = launchingConnector.defaultArguments();
		env.get("main").setValue(classToDebug.getName());
		VirtualMachine vm = launchingConnector.launch(env);

		/*
		 * Request VM to trigger event when HelloWorld class is prepared.
		 */
		ClassPrepareRequest classPrepareRequest = vm.eventRequestManager().createClassPrepareRequest();
		classPrepareRequest.addClassFilter(classToDebug.getName());
		classPrepareRequest.enable();

		EventSet eventSet = null;

		try {
			while ((eventSet = vm.eventQueue().remove(100)) != null) {

				for (Event event : eventSet) {

					/*
					 * If this is ClassPrepareEvent, then set breakpoint
					 */
					if (event instanceof ClassPrepareEvent) {
						ClassPrepareEvent evt = (ClassPrepareEvent) event;
						ClassType classType = (ClassType) evt.referenceType();
						//设置行号，发出中断请求
						Location location = classType.locationsOfLine(lineNumberToPutBreakpoint).get(0);
						BreakpointRequest bpReq = vm.eventRequestManager().createBreakpointRequest(location);
						bpReq.enable();

					}

					/*
					 * If this is BreakpointEvent, then read & print variables.
					 */
					//因为我们的行号设置在循环里，VM会返回多次BreakpointEvent，比如i==1时、i==2时等等
					if (event instanceof BreakpointEvent) {
						// disable the breakpoint event
						//将这里的disbable去掉，因为我们的断点事件会返回多次
						//event.request().disable();

						// Get values of all variables that are visible and print
						//这里我们会获取到临时变量的name和value
						//比如我们设置的条件变量是i==5,那么就需要对event事件进行过滤筛选，直到i==5时通知IDE
						//条件表达式解析是很复杂的，这里我们不讲复杂的表达式。
						StackFrame stackFrame = ((BreakpointEvent) event).thread().frame(0);
						Map<LocalVariable, Value> visibleVariables = (Map<LocalVariable, Value>) stackFrame
								.getValues(stackFrame.visibleVariables());
						System.out.println("Local Variables =");
						for (Map.Entry<LocalVariable, Value> entry : visibleVariables.entrySet()) {
							System.out.println("	" + entry.getKey().name() + " = " + entry.getValue());
						}

					}
					vm.resume();

				}

			}
		} catch (VMDisconnectedException e) {
			System.out.println("VM is now disconnected.");
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

}
```

至此，IDE整个debug的机制我们已经了解的差不多了，看起来简单的debug实现起来也如此复杂，可能这就是程序的魅力，将复杂的功能封装成简单的使用方式提供给用户。我们没有深入到VM内部了解断点具体是怎么实现的，以我的能力估计是不能了，点到为止吧。
