---
description: Skywalking common components - DataCarrier
---

# skywalking基础组件-DataCarrier

## DataCarrier

`DataCarrier`负责skywalking数据的存储和消费，比如Trace和log数据都是由`DataCarrier`负责。其数据结构如下所示，有存储数据的`Channels`和消费数据驱动器`IDriver`。

```
public class DataCarrier<T> {
    //真正存取数据的容器
    private Channels<T> channels;
    //驱动线程去消费数据的驱动器
    private IDriver driver;
    private String name;
}
```

`DataCarrier`重要的就是`produce生产`和`consume消费`两个方法

### `produce`方法：

```
    public boolean produce(T data) {
        if (driver != null) {
            if (!driver.isRunning(channels)) {
                return false;
            }
        }
        //调用channels去存数据
        return this.channels.save(data);
    }
```

### consume方法：

```
    public DataCarrier consume(IConsumer<T> consumer, int num, long consumeCycle) {
        if (driver != null) {
            driver.close(channels);
        }
        driver = new ConsumeDriver<T>(this.name, this.channels, consumer, num, consumeCycle);
        driver.begin(channels);
        return this;
    }
```

## Channels

`Channels`的数据结构如下，它封装了具体的数据容器，分区逻辑(就是哪个线程负责哪个`QueueBuffer`），存数满了的时候逻辑。

```
public class Channels<T> {
    //真正存数据的容器，有阻塞队列 和 循环队列
    private final QueueBuffer<T>[] bufferChannels;
    //分区逻辑   1.threadid % total 2.简单循环
    private IDataPartitioner<T> dataPartitioner;
    //当容量满了的时候的添加逻辑，有阻塞和尽可能存储(尝试指定次数后返回)
    private final BufferStrategy strategy;
    private final long size;
    }
```

`QueueBuffer`有`ArrayBlockBuffer`和`Buffer`两种实现，前一种是封装了JDK的`ArrayBlockingQueue`，支持阻塞。后一种是自己实现的循环队列。支持存数据`save`，获取全部数据`obtain`等方法。

```
public interface QueueBuffer<T> {
    /**
     * Save data into the queue;
     *
     * @param data to add.
     * @return true if saved
     */
    boolean save(T data);

    /**
     * Set different strategy when queue is full.
     */
    void setStrategy(BufferStrategy strategy);

    /**
     * Obtain the existing data from the queue
     */
    void obtain(List<T> consumeList);

    int getBufferSize();
}
```

### Channels的save方法：

```
    public boolean save(T data) {
        //先根据存数策略选取哪个QueueBuffer
        int index = dataPartitioner.partition(bufferChannels.length, data);
        int retryCountDown = 1;
        //IF_POSSIBLE策略 获取maxRetryCount
        if (BufferStrategy.IF_POSSIBLE.equals(strategy)) {
            int maxRetryCount = dataPartitioner.maxRetryCount();
            if (maxRetryCount > 1) {
                retryCountDown = maxRetryCount;
            }
        }
        //尝试maxRetryCount次，不行就返回false
        for (; retryCountDown > 0; retryCountDown--) {
            if (bufferChannels[index].save(data)) {
                return true;
            }
        }
        return false;
    }
```

## IDriver

`IDriver`是线程驱动器，有`ConsumeDriver`和`BulkConsumePool`两种实现，内部只使用了`ConsumeDriver`。先看下`ConsumeDriver`的实现。其数据结构如下

### ConsumeDriver

```
//一推消费者线程消费一堆buffer
public class ConsumeDriver<T> implements IDriver {
    private boolean running;
    //自己实现的消费者线程
    private ConsumerThread[] consumerThreads;
    //数据容器
    private Channels<T> channels;
    private ReentrantLock lock;
}
```

#### ConsumeDriver构造方法:

```
  public ConsumeDriver(String name,
                         Channels<T> channels, Class<? extends IConsumer<T>> consumerClass,
                         int num,
                         long consumeCycle,
                         Properties properties) {
        this(channels, num);
        for (int i = 0; i < num; i++) {
        //将
            consumerThreads[i] = new ConsumerThread(
                "DataCarrier." + name + ".Consumer." + i + ".Thread", getNewConsumerInstance(consumerClass, properties),
                consumeCycle
            );
            consumerThreads[i].setDaemon(true);
        }
    }
```

#### `ConsumeDriver`的begin方法

`begin`只能调用一次，表示驱动线程开始消费数据。

```
    public void begin(Channels channels) {
        //防止重复开始
        if (running) {
            return;
        }
        lock.lock();
        try {
            //将QueueBuffer分配给每个线程
            this.allocateBuffer2Thread();
            //启动每个线程
            for (ConsumerThread consumerThread : consumerThreads) {
                consumerThread.start();
            }
            running = true;
        } finally {
            lock.unlock();
        }
    }
```

#### `allocateBuffer2Thread`&#x20;

将QueueBuffer分配给每个线程。

```
   private void allocateBuffer2Thread() {
        int channelSize = this.channels.getChannelSize();
        /**
         * if consumerThreads.length < channelSize
         * each consumer will process several channels.
         *
         * if consumerThreads.length == channelSize
         * each consumer will process one channel.
         *
         * if consumerThreads.length > channelSize
         * there will be some threads do nothing.
         */
        //看上面的注释，就是分配原则，很好理解
        for (int channelIndex = 0; channelIndex < channelSize; channelIndex++) {
            int consumerIndex = channelIndex % consumerThreads.length;
            consumerThreads[consumerIndex].addDataSource(channels.getBuffer(channelIndex));
        }

    }
```

### BulkConsumePool

除了上面的默认实现，`IDriver`还有`BulkConsumePool`的实现,其数据结构如下：

```
public class BulkConsumePool implements ConsumerPool {
    private List<MultipleChannelsConsumer> allConsumers;
    private volatile boolean isStarted = false;
}
```

`MultipleChannelsConsumer`是对线程的封装，看下其数据结构，可以看出是一个线程对应多个`<channels,consumer>`的组合，而之前的`ConsumerThread`是一个线程对应一个consumer和多个`Queuebuffer`。

```
//一个线程多个<channels,consumer>
public class MultipleChannelsConsumer extends Thread {
    private volatile boolean running;
    private volatile ArrayList<Group> consumeTargets;
    @SuppressWarnings("NonAtomicVolatileUpdate")
    //buffer数目
    private volatile long size;
    private final long consumeCycle;
}
```

## ConsumerThread

先看下`ConsumerThread`的实现逻辑，封装了我们传入的`consume`逻辑：

```
public class ConsumerThread<T> extends Thread {
    //是否在运行
    private volatile boolean running;
    //消费逻辑
    private IConsumer<T> consumer;
    //数据源
    private List<DataSource> dataSources;
    //本次消费没有取到数据  线程sleep时间
    private long consumeCycle;
```

ConsumerThread的run方法，

```
    public void run() {
        running = true;

        final List<T> consumeList = new ArrayList<T>(1500);
        while (running) {
            //如果没有消费到数据，线程就sleep调，然后继续循环
            if (!consume(consumeList)) {
                try {
                    Thread.sleep(consumeCycle);
                } catch (InterruptedException e) {
                }
            }
        }

        // consumer thread is going to stop
        // consume the last time
        //只有running==false的时候才会退出循环，最后执行一次消费，保证数据都被消费掉
        consume(consumeList);

        consumer.onExit();
    }
```

不断的调用内部的`consume`，就是不断的从分配给此线程`QueueBuffer`里获取数据进行消费。

```
   private boolean consume(List<T> consumeList) {
        //将dataSources中的数据都倾倒出来
        for (DataSource dataSource : dataSources) {
            dataSource.obtain(consumeList);
        }

        if (!consumeList.isEmpty()) {
            try {
                //调用consumer消费数据
                consumer.consume(consumeList);
            } catch (Throwable t) {
                //调用consumer的报错
                consumer.onError(consumeList, t);
            } finally {
                //清空list
                consumeList.clear();
            }
            return true;
        }
        //没有可消费的数据
        consumer.nothingToConsume();
        return false;
    }
```

DataCarrier的分析到此结束，可以看出只要没加业务逻辑，单纯看数据结构还是比较简单的，唯一复杂的地方就是层层嵌套的封装。
