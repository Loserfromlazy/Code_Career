# Disruptor学习笔记

> 初步笔记并未做细致整理
>
> 参考资料：
>
> - https://www.cnblogs.com/crazymakercircle/p/13909235.html
> - 《Disruptor 红宝书》尼恩
> - https://jitwxs.cn/a7ed43af.html
> - https://lmax-exchange.github.io/disruptor/user-guide/index.html#_getting_started
> - https://www.jianshu.com/p/d2d50cf6d887

# 一、概述

Disruptor是英国外汇公司LMAX开发的一款高性能队列，初衷是解决内存队列的延迟问题。基于Disruptor开发的系统单线程支撑每秒600w订单，2010年后在QCon演讲后，获得了业界关注。2011年，企业应用软件专家Martin Fowler专门撰写长文介绍Disruptor。

## 1.1 高性能原因

Disruptor主要是用在对性能要求高，低延迟的场景中，它通过榨干机器的性能换取处理的高性能。

它的高性能主要体现在：

- 内部是环形数据结构，且采用数组结构，因为数组不会被回收，可以避免频繁的GC。同时数组对处理器的缓存机制更加友好。

  如下图，通过long类型做index，组成环形队列，不断增长。序号是不断向前填充的，以下图为例，当生产者走到7后，下一个序号会变为8覆盖掉0。生产者向前占位进行填充，消费者在后面不断消费，消费者的序号是比生产者小的，当生产者序号减消费者序号大于队列长度时，说明当前队列还可以继续生产；当生产者序号减消费者序号等于队列长度时，说明当前队列已满（比如下图如果消费者未消费一直停留在1上，当生产者转了一圈序号会来到9，9-1=8等于队列长度，说明队列已满，生产者不在继续生产）。

  ![image-20221030193227058](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221030193227058.png)

- 无锁设计，通过CAS无锁的方式保证线程安全，CAS的性能比Java隐式锁等性能高。

- 元素位置属性进行了缓存行数据填充，避免了伪共享问题（伪共享问题后续会介绍）

  > Java8提供了`@sun.misc.Contended`，此注解可以解决伪共享问题，在变量前后增加128字节的填充数据，使用时需要使用JVM参数`-XX:-RestrictContended`

- 通过位运算定位元素在数组的位置，数组长度为2的幂，通过位运算可以加快定位的速度。

## 1.2 使用场景

Disruptor作为一个高性能队列，主要用于：

- 发布订阅场景
- 消费者生产者问题的场景

## 1.3 入门案例

Disruptor的使用方式很简单，首先我们需要自定义封装事件或数据，这里演示时仅简单使用：

**定义事件**

```java
public class LongEvent {
    private long value;

    public long getValue() {
        return value;
    }

    public void setValue(long value) {
        this.value = value;
    }
}
```

然后实现Disruptor提供的事件工厂接口：

**事件工厂**

```java
public class LongEventFactory implements EventFactory<LongEvent> {
    @Override
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```

然后我们继承Disruptor提供的EventHandler接口编写事件处理器：

**事件处理器（事件消费者）**

```java
public class LongEventHandler implements EventHandler<LongEvent> {

    @Override
    public void onEvent(LongEvent longEvent, long l, boolean b) throws Exception {
        System.out.println(longEvent.getValue());
    }
}
```

然后为了方便我们封装一个事件生产者：

**事件生产者**

```java
public class LongEventProducer {

    private final RingBuffer<LongEvent> ringBuffer;

    public LongEventProducer(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    public void onData(long data){
        //通过环形队列获取序号
        long sequence = ringBuffer.next();
        try {
            //通过序号获取对应的事件对象，将数据填充到事件对象
            LongEvent longEvent = ringBuffer.get(sequence);
            longEvent.setValue(data);
        }finally {
            //通过序号将事件对象发布出现
            ringBuffer.publish(sequence);
        }
    }
}
```

最后编写测试类：

**测试类：**

```java
@Test
    public void testSimpleDisruptor() throws InterruptedException {
        //消费者线程池，一般来说不使用自己的线程池，而是使用Disruptor自带的线程池，而我们提供线程工厂
        Executor executor = Executors.newCachedThreadPool();
        //事件工厂
        LongEventFactory eventFactory = new LongEventFactory();
        //环形队列大小
        int bufferSize = 1024;
        //构造分裂者，即事件分发器。PS：此构造函数已不被推荐使用
        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(eventFactory,bufferSize,executor);
        //连接消费者处理器
        disruptor.handleEventsWith(new LongEventHandler());
        //事件分发
        disruptor.start();
        //获取环形队列生产事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
        LongEventProducer producer = new LongEventProducer(ringBuffer);

        for (int i = 0;true ; i++) {
            //发布事件
            producer.onData(i);
            Thread.sleep(1000);
        }
    }
```

为了方便，我们可以在我们自己封装的事件生产者中使用事件转换器省略部分步骤：

**事件转换器**

使用事件转换器，可以省略从环形队列获取序号，然后拿到事件填充数据。

```java
public class LongEventProducerWithTranslator {

    private final RingBuffer<LongEvent> ringBuffer;

    public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    public static final EventTranslatorOneArg<LongEvent,Long> TRANSLATOR = new EventTranslatorOneArg<LongEvent, Long>() {
        @Override
        public void translateTo(LongEvent longEvent, long sequence, Long data) {
            longEvent.setValue(data);
        }
    };

    public void onData(long data){
        //通过序号将事件对象发布出现
        System.out.println(Thread.currentThread().getName()+" 生产者生产数据 "+data);
        ringBuffer.publishEvent(TRANSLATOR,data);
    }
}
```

Disruptor提供了不同的接口的转换器：

- EventTranslator
- EventTranslatorOneArg
- EventTranslatorTwoArg

Translator中方法的参数是通过RingBuffer传递的，事件转换器的测试类如下：

PS：下面的测试类与上面有些许不同，这里通过Lambda的方式使用Disruptor，并使用了事件转换器：

```java
@Test
public void testSimpleDisruptorWithLambda() throws InterruptedException {
    //消费者线程池
    Executor executor = Executors.newCachedThreadPool();
    //环形队列大小
    int bufferSize = 1024;
    //构造分裂者，及事件分发器
    Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent::new,bufferSize,executor);
    //连接消费者处理器，
    //这里也可以将function抽取出来比如抽取成Test类的静态方法，然后使用Test::handler可以更加简便
    disruptor.handleEventsWith((longEvent, sequence, b) -> System.out.println(longEvent.getValue()));
    //事件分发
    disruptor.start();
    //获取环形队列生产事件
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);

    for (int i = 0;true ; i++) {
        //发布事件
        producer.onData(i);
        Thread.sleep(1000);
    }
}
```

## 1.4 Disruptor注意点

1. 在高并发场景下，可以使用对象池对Event进行复用，减少GC

2. 在创建Disruptor时，一般使用线程工厂而不是我们自己的线程池，因为我们自带的线程池无法查看错误，因为如果我们使用我们自带的线程池，那么在创建消费者线程时，有可能会进入到等待队列中，因为在Disruptor中线程基本与每一个消费者绑定，因此可能永远会无法创建线程，而因为进入等待队列所以新的线程可能永远无法创建也无从进行排查。下面是disruptor的源码，它使用线程工厂创建线程，并进行判断，保证线程的创建，可以避免此问题，因此一般使用线程工厂进行创建。

   ```java
   //com.lmax.disruptor.dsl.BasicExecutor#execute
   @Override
   public void execute(Runnable command)
   {
       final Thread thread = factory.newThread(command);
       if (null == thread)
       {
           throw new RuntimeException("Failed to create thread to run: " + command);
       }
   
       thread.start();
   
       threads.add(thread);
   }
   ```

3. 指定合适的等待策略

## 1.5 等待策略WaitStrategy

在构造Disruptor时可以指定WaitStrategy，Disruptor提供了多个策略，每种策略都有不同的性能和优缺点，根据实际运行环境的CPU硬件特点选择适当的策略，并配合特定的JVM的配置参数，能实现不同的性能提升。常用的有以下四个：

**BlockingWaitStrategy**

最低效的策略，使用锁和阻塞实现等待策略，对CPU消耗最小，并且在各种不同的部署环境中能提供更加一致的性能表现。

**SleepingWaitStrategy**

休眠策略，先使用自旋，然后使用Thread.yield()线程让步，最后使用sleep让线程休眠。

此策略是性能和CPU的很好的折衷。性能与BlockingWaitStrategy差不多，CPU消耗也相似，但是对生产者线程的影响较小，适合高延迟，比如异步日志的场景。

**YieldingWaitStrategy**

性能最好，适合低延迟系统，一般在事件处理线程数小于CPU逻辑核心数的场景中和性能要求极高的场景中。YieldingWaitStrategy是可以被用在低延迟系统的两个策略之一，这种策略在降低系统延迟的时候也会增加CPU运算量，甚至到100%，此策略会循环等待sequence增加到合适的值。循环中会调用Thread.yield()方法允许其他准备好的线程执行。如果需要高性能且消费者线程比核心逻辑数少的时候适合此策略，比如在开启超线程的情况。

**BusySpinWaitStrategy**

忙碌自旋策略，为事件处理开启一个忙碌自旋循环，当线程可以绑定到特定CPU内核时，可以使用。这是性能最高的等待策略，同时也是对部署环境要求最高的策略，这个性能最好用在消费者线程比物理内核数目还要小的时候，比如禁用超线程技术时。

# 二、Disruptor的使用场景

Disruptor是一个优秀的并发框架，可以使用在生产者消费者的多个场景，主要如下：

- 单生产者多消费者并行
- 多生产者单消费者
- 单生产者多消费者竞争
- 单生产者多消费者串行
- 菱形方式执行
- 链式并行执行
- 多组消费者相互隔离
- 多组消费者航道执行
- 六边形场景

## 单生产者多消费者并行

并发系统中提升性能最好的方式就是单一写者原则，对Disruptor也适用。

- 如果在需求中只有一个事件生产者，那么可设置为单一生产者模式来提高系统的性能，在Disruptor的构造函数中指定ProducerType为SINGLE即可。
- 如果在需求中需要多消费者并行，那么需要在handleEventsWith方法中传入需要并行的消费者

例子如下：

```java
@Test
public void SingleProducer(){
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent,TestScene::handleEvent,TestScene::handleEvent);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int i = 0; true; i++) {
            producer.onData(i);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
//TestScene::handleEvent
public static void handleEvent(LongEvent longEvent, long l, boolean b){
    System.out.println(Thread.currentThread().getName()+" "+longEvent.getValue());
}
```

运行结果：

~~~
producer 生产者生产数据 0
pool-1-thread-1 0
pool-1-thread-2 0
pool-1-thread-3 0
producer 生产者生产数据 1
pool-1-thread-1 1
pool-1-thread-2 1
pool-1-thread-3 1
producer 生产者生产数据 2
pool-1-thread-1 2
pool-1-thread-2 2
pool-1-thread-3 2
producer 生产者生产数据 3
pool-1-thread-1 3
pool-1-thread-3 3
pool-1-thread-2 3
producer 生产者生产数据 4
pool-1-thread-1 4
pool-1-thread-3 4
pool-1-thread-2 4
~~~

这里我们可以清晰地看到，多个消费者并行的消费单个生产者生产的数据，运行流程如下：

![image-20221031135551500](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031135551500.png)

## 多生产者单消费者

Disruptor中的生产者是不受Disruptor框架控制的，而是我们用户自己控制，因此如果想使用多生产者，需要将ProducerType设置为MULTI，然后自己多线程生产即可。例子如下：

```java
@Test
public void SingleConsumer(){
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.MULTI, new BlockingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    for (int i = 0; i < 3; i++) {
        Thread thread = new Thread(() -> {
            for (int j= 0; true; j++) {
                producer.onData(j);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.setName("producer"+i);
        thread.start();
    }
    try {

        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果：

~~~
producer0 生产者生产数据 0
producer2 生产者生产数据 0
producer1 生产者生产数据 0
pool-1-thread-1 0
pool-1-thread-1 0
pool-1-thread-1 0
producer1 生产者生产数据 1
producer0 生产者生产数据 1
producer2 生产者生产数据 1
pool-1-thread-1 1
pool-1-thread-1 1
pool-1-thread-1 1
producer1 生产者生产数据 2
producer0 生产者生产数据 2
producer2 生产者生产数据 2
pool-1-thread-1 2
pool-1-thread-1 2
pool-1-thread-1 2
producer0 生产者生产数据 3
producer2 生产者生产数据 3
producer1 生产者生产数据 3
pool-1-thread-1 3
pool-1-thread-1 3
pool-1-thread-1 3
producer0 生产者生产数据 4
producer1 生产者生产数据 4
producer2 生产者生产数据 4
pool-1-thread-1 4
pool-1-thread-1 4
pool-1-thread-1 4
~~~

这里我们可以看到，有多个生产者生产数据，有单个消费者对数据进行消费，运行流程如下：
![image-20221031135712495](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031135712495.png)

## 单生产者多消费者竞争

多消费者竞争的意思是多个消费者竞争同一个事件，最终只有一个消费者可以执行事件。想要做到这种地步，事件处理器需要继承`WorkHandler`接口而不是`EventHandler`接口。然后调用disruptor的handleEventsWithWorkerPool方法绑定处理器。例子如下：

```java
@Test
public void MultiConsumerContend() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWithWorkerPool(TestScene::workerHandlerEvent, TestScene::workerHandlerEvent, TestScene::workerHandlerEvent);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}

public static void workerHandlerEvent(LongEvent event) throws Exception {
    System.out.println(Thread.currentThread().getName() + " " + event.getValue());
}
```

运行结果：

~~~
producer 生产者生产数据 0
pool-1-thread-1 0
producer 生产者生产数据 1
pool-1-thread-2 1
producer 生产者生产数据 2
pool-1-thread-3 2
producer 生产者生产数据 3
pool-1-thread-1 3
producer 生产者生产数据 4
pool-1-thread-2 4
~~~

这里我们可以看到生产数据每次由不同的线程或者说是竞争胜利的线程执行该事件，运行流程如下：

![image-20221031141836416](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031141836416.png)

## 单生产者多消费者串行

多消费者串行表示消费者互相依赖，当前一个消费者不消费完后一个消费者不能消费，如果想实现这样的话，需要在disruptor绑定处理器后调用then方法执行依赖。例子如下：

```java
@Test
public void MultiConsumerSerial() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent)
            .then(TestScene::handleEvent)
            .then(TestScene::handleEvent);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果：

~~~
producer 生产者生产数据 0
pool-1-thread-1 0
pool-1-thread-2 0
pool-1-thread-3 0
producer 生产者生产数据 1
pool-1-thread-1 1
pool-1-thread-2 1
pool-1-thread-3 1
producer 生产者生产数据 2
pool-1-thread-1 2
pool-1-thread-2 2
pool-1-thread-3 2
producer 生产者生产数据 3
pool-1-thread-1 3
pool-1-thread-2 3
pool-1-thread-3 3
producer 生产者生产数据 4
pool-1-thread-1 4
pool-1-thread-2 4
pool-1-thread-3 4
~~~

这里我们可以看到消费者消费的顺序是按照我们指定的依赖关系进行消费的，运行流程如下：

![image-20221031142834833](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031142834833.png)

## 菱形方式执行

菱形方式也可以说是先并行在串行。例子如下：

```java
@Test
public void MultiConsumerRhombus() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent, TestScene::handleEvent)
        //这里handleEvent1的区别就是手动设置了线程名称便于区分
        .then(TestScene::handleEvent1);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果：

~~~
producer 生产者生产数据 0
pool-1-thread-1 0
pool-1-thread-2 0
consumer1 0
producer 生产者生产数据 1
pool-1-thread-1 1
pool-1-thread-2 1
consumer1 1
producer 生产者生产数据 2
pool-1-thread-1 2
pool-1-thread-2 2
consumer1 2
producer 生产者生产数据 3
pool-1-thread-1 3
pool-1-thread-2 3
consumer1 3
producer 生产者生产数据 4
pool-1-thread-1 4
pool-1-thread-2 4
consumer1 4
~~~

这里我们可以看到我们consumer1消费者一定是在依赖的两个消费者消费完后才去消费，菱形流程如下：

![image-20221031143917835](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031143917835.png)

## 链式并行执行

链式并行流程如下：

![image-20221031145047396](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031145047396.png)

disruptor实现也比较简单，例子如下：

```java
@Test
public void MultiConsumerParallel() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent1).then(TestScene::handleEvent2);
    disruptor.handleEventsWith(TestScene::handleEvent3).then(TestScene::handleEvent4);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果如下：

~~~
producer 生产者生产数据 0
consumer1 0
consumer3 0
consumer2 0
consumer4 0
producer 生产者生产数据 1
consumer1 1
consumer3 1
consumer4 1
consumer2 1
producer 生产者生产数据 2
consumer1 2
consumer3 2
consumer4 2
consumer2 2
producer 生产者生产数据 3
consumer1 3
consumer2 3
consumer3 3
consumer4 3
producer 生产者生产数据 4
consumer1 4
consumer3 4
consumer2 4
consumer4 4
~~~

这里我们可以看到在消费者消费的过程中消费者1一定在2前面，3一定在4前面，因为2依赖于1而4也依赖于3.

## 多组消费者相互隔离

这个场景可以说是组内互相竞争，组外互相并行，在Disruptor中写法如下：

```java
@Test
public void MultiConsumerIsolate () {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWithWorkerPool(TestScene::workerHandlerEvent1,TestScene::workerHandlerEvent2);
    disruptor.handleEventsWithWorkerPool(TestScene::workerHandlerEvent3,TestScene::workerHandlerEvent4);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果如下：

~~~
producer 生产者生产数据 0
consumer4 0
consumer2 0
producer 生产者生产数据 1
consumer3 1
consumer1 1
producer 生产者生产数据 2
consumer4 2
consumer2 2
producer 生产者生产数据 3
consumer3 3
consumer1 3
producer 生产者生产数据 4
consumer4 4
consumer2 4
~~~

我们可以看到结果也符合我们的预期，运行流程如下：

![image-20221031145635581](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031145635581.png)

## 多组消费者航道执行

这种场景与上面类似，即组内互相竞争，组外相互依赖，例子如下：

```java
@Test
public void MultiConsumerParallelDepend() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    disruptor.handleEventsWithWorkerPool(TestScene::workerHandlerEvent1, TestScene::workerHandlerEvent2)
            .thenHandleEventsWithWorkerPool(TestScene::workerHandlerEvent3, TestScene::workerHandlerEvent4);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果如下：

~~~
producer 生产者生产数据 0
consumer1 0
consumer3 0
producer 生产者生产数据 1
consumer2 1
consumer4 1
producer 生产者生产数据 2
consumer1 2
consumer3 2
producer 生产者生产数据 3
consumer2 3
consumer4 3
producer 生产者生产数据 4
consumer1 4
consumer3 4
~~~

流程如下：

![image-20221031150803953](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031150803953.png)

## 六边形场景

场景流程或者消费者依赖如下：

![image-20221031151021499](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221031151021499.png)

这种情况disruptor可以使用after方法进行设置，例子如下：

```java
@Test
public void MultiConsumerHexagon() {
    int bufferSize = 1024;
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());
    LongEventHandlerWithThreadName handler1 = new LongEventHandlerWithThreadName("consumer1");
    LongEventHandlerWithThreadName handler2 = new LongEventHandlerWithThreadName("consumer2");
    LongEventHandlerWithThreadName handler3 = new LongEventHandlerWithThreadName("consumer3");
    LongEventHandlerWithThreadName handler4 = new LongEventHandlerWithThreadName("consumer4");
    LongEventHandlerWithThreadName handler5 = new LongEventHandlerWithThreadName("consumer5");
    disruptor.handleEventsWith(handler1, handler2);
    disruptor.after(handler1).handleEventsWith(handler3);
    disruptor.after(handler2).handleEventsWith(handler4);
    disruptor.after(handler3,handler4).handleEventsWith(handler5);
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
    Thread thread = new Thread(() -> {
        for (int j = 0; true; j++) {
            producer.onData(j);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    thread.setName("producer");
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

运行结果如下：

~~~
producer 生产者生产数据 0
consumer1 0
consumer2 0
consumer3 0
consumer4 0
consumer5 0
producer 生产者生产数据 1
consumer1 1
consumer2 1
consumer3 1
consumer4 1
consumer5 1
producer 生产者生产数据 2
consumer2 2
consumer1 2
consumer4 2
consumer3 2
consumer5 2
producer 生产者生产数据 3
consumer2 3
consumer1 3
consumer4 3
consumer3 3
consumer5 3
producer 生产者生产数据 4
consumer2 4
consumer1 4
consumer3 4
consumer4 4
consumer5 4
~~~

从运行结果总我们也可以看到1一定在3前面，2一定在4前面，5一定在最后面。

# 三、Disruptor源码分析

> 这里使用3.4版本的disruptor源码，源码使用Gradle构建，如果构建过程中出现Gradle下载失败，请自行下载对应的版本然后将gradle地址手动指定到本地文件系统的地址上

## 3.1 核心组件

### 3.1.1 Disruptor

持有 RingBuffer、消费者线程池 Executor、消费者集合 CounsumerRepository 等引用。

### 3.1.2 RingBuffer 环形缓冲

RingBuffer 在 3.0 版本之前被认为是 Disruptor 的主要概念。但从 Diruptor 3.0 开始，RingBuffer 只负责存储和更新 Disruptor 的数据，在一些高级的使用场景中用户也可以自定义它。

RingBuffer 是基于数组的缓存实现，存储生产和消费的 Event，它实现了阻塞队列的语义，也是创建 sequencer 与定义 WaitStartegy 的入口。**如果 RingBuffer 满了，则生产者会阻塞等待；如果 RingBuffer 空了，则消费者会阻塞等待**。

### 3.1.3 Sequence 序列

Disruptor 使用 Sequence 来标记特定组件的处理进度，通过顺序递增的序号来编号，管理进行交换的 Event。一个 Sequence 用于跟踪标识某个特定的事件处理者（RingBuffer、Producer、Consumer）的处理进度。

Sequence 具有许多 AtomicLong 的特征，虽然使用 AtomicLong 也可以用于标识进度，但它可以防止不同的 Sequence 之间的 CPU 缓存伪共享(Flase Sharing)问题。

### 3.1.4 Sequencer 序列器

Sequencer 是 Disruptor 中的真正核心，主要实现生产者和消费者之间快速、正确地传递数据的并发算法。

它具有 `SingleProducerSequencer` 和 `MultiProducerSequencer` 这两个实现。

### 3.1.5 Sequence Barrier 序列屏障

Sequence Barrier 由 Sequencer 创建，它包含了来自 Sequencer 的已经发布的主要 sequence 的引用，或者包含了依赖的消费者的 sequence。

用于保持对 RingBuffer 的 Main Published Sequence(Producer) 和 Consumer 之间的平衡关系，它决定了 Consumer 是否还有可处理的 Event 的逻辑。

### 3.1.6 WaitStrategy 等待策略

WaitStrategy 决定了消费者以何种方式等待生产者将 Event 放进 Disruptor 中。

### 3.1.7 Event 事件

从生产者传到消费者的数据单元叫做 Event。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者自行定义。

### 3.1.8 Event Processor 事件处理器

持有特定的消费者的 Sequence，并且拥有一个主事件循环（main event loop）用于处理 Disruptor 的 Event。

其中 BatchEventProcessor 是其具体实现，实现了事件循环（event loop），并且会回调到实现了 EventHandler 的接口对象中。

### 3.1.9 EventHandler 事件处理逻辑

由用户实现并且代表了 Disruptor 中的一个消费者的接口，消费者相关的逻辑都需要写在这里。

### 3.1.10 Producer 生产者

生产者，泛指调用 Disruptor 发布事件的用户代码，它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者自行定义。

### 3.1.11 WorkProcessor

确保每个 sequence 只被一个 processor 消费，在同一个 WorkPool 中处理多个 WorkProcessor 不会消费同样的 sequence。

### 3.1.12 总结

以上内容来自官方文档，实际上为了方便理解，我们可以这样理解各个组件的作用：

- Disruptor可以看作是方便我们使用的工具类，有点类似Netty的Bootstrap类，便于我们使用
- ConsumerRepository类就是消费者的存储仓库，内部有三个map维护映射关系。
- ConsumerInfo维护者每一个消费者的信息，在此类中有三个类：
  - EventProcessor：可以理解为消费者的执行线程
  - EventHandler：我们用户自己的业务处理
  - SequenceBarrier：序号屏障，用于获取序号等操作
- Sequencer：序号管理器，只有生产者持有，用于生产或管理消费者序号
- Sequence：序号类
- WaitStrategy：等待策略，用于SequenceBarrier获取序号时，如无法获取时的等待策略，比如条件等待，线程让步等。

Disruptor主要就是围绕这几个核心类或者说是核心组件进行的，Disruptor的整体流程如下：



## 3.2 Sequence

Sequence是Disruptor的序号类也可以理解为id类，核心属性只有一个long类型的value属性，它是被填充过缓存行的，这么做的目的就是为了解决伪共享(false sharing)的问题。我们下面来一步步了解这个类。

首先此类的继承关系如下：

![image-20221101084622419](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221101084622419.png)

我们看到Sequence类真正的属性再Value类中，而Value类的父类和子类都填充了7个long类型的字段，最终Sequence类继承RhsPadding类完成缓存行填充。那么为什么要填充7个long类型的字节呢？因为一个long是8个字节，**现在的计算机CPU缓存行一般为64字节**，这样的话就能保证CPU的每一个缓存行我们的value数据都能被填充数据所包裹，如下图：

![cachelineproblem](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/cachelineproblem.gif)

> linux 查看缓存行大小命令
>
> `cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size`

然后我们看一下它的静态代码块，在这里会获取unsafe类和value对象便于其他方法调用：

```java
static
{
    //反射获取unsafe类
    UNSAFE = Util.getUnsafe();
    try
    {
        //获取value对象的内存偏移量
        VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
    }
    catch (final Exception e)
    {
        throw new RuntimeException(e);
    }
}
```

然后此类为value提供了各种写入方法：

```java
/**
* Perform an ordered write of this sequence.  The intent is
* a Store/Store barrier between this write and any previous
* store.
*/
public void set(final long value)
{
    //这里调用unsafe类的putOrderedLong方法进行写入数据，此方法会插入一个SS内存屏障，延迟写入数据
    UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
}
/**
* Performs a volatile write of this sequence.  The intent is
* a Store/Store barrier between this write and any previous
* write and a Store/Load barrier between this write and any
* subsequent volatile read.
*/
public void setVolatile(final long value)
{
    //此方法符合JMM规范，会在前面插入SS屏障，后面插入Sl屏障
    UNSAFE.putLongVolatile(this, VALUE_OFFSET, value);
}
public boolean compareAndSet(final long expectedValue, final long newValue)
{
    //此方法是通过CAS进行写入
    return UNSAFE.compareAndSwapLong(this, VALUE_OFFSET, expectedValue, newValue);
}
```

这里我们重点看一下`UNSAFE.putOrderedLong`延迟写入方法，因为我们的字段是volatile字段，因此根据JMM规范需要插入SS和Sl屏障，但是SL屏障开销很大，可以理解为需要保证当前写入对所有CPU可见，所以`putOrderedLong`延迟写入方法只使用了SS屏障，这样可能会造成写后结果并不会被其他线程立即看到即取消了可见性，但是可以提升性能，且延迟一般在纳秒级别，因此此方法可以提升写入性能。关于JVM内存屏障可以看我的笔记[CAS和原子类以及有序性和可见性](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%B8%89CAS%E5%92%8C%E5%8E%9F%E5%AD%90%E7%B1%BB%E4%BB%A5%E5%8F%8A%E6%9C%89%E5%BA%8F%E6%80%A7%E5%92%8C%E5%8F%AF%E8%A7%81%E6%80%A7.md#45-jmm)

## 3.3 Sequencer

继承关系如下

![image-20221101124108393](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221101124108393.png)

我们这里主要以SingleProducerSequencer为例来了解Disruptor的这个组件（因为Disruptor更适合于单生产者的场景，*MPSC*更适合多生产者场景），先来看它的父类：

![image-20221101130626547](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221101130626547.png)

![image-20221102135125039](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102135125039.png)

我们可以看到Sequencer序号管理器跟Sequence一样也进行了缓存行填充。在Sequencer中有几个重要属性：

- nextValue：序号的当前值
- cacheValue：缓存的门禁序号值
- gatingSequences：门禁序号数组，主要用于防止过多生产覆盖未消费的信息
- cursor：游标，即当前生产者生产到了哪个序号

下面我们来结合方法一起看一下上面的属性的作用，作为序号管理器最重要的就是next方法预定序号和publish方法发布序号。下面我们分别来看：

```java
@Override
public long next(int n)
{
    if (n < 1)
    {
        throw new IllegalArgumentException("n must be > 0");
    }
	//获取当前序号值
    long nextValue = this.nextValue;
	//获取要预定的序号值
    long nextSequence = nextValue + n;
    //计算wrapPoint
    long wrapPoint = nextSequence - bufferSize;
    //获取缓存的门禁序号值
    long cachedGatingSequence = this.cachedValue;
	//这句判断的意思是判断是否生产者的序号是否已经大于消费者的一环或者说一圈，下面详解
    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
    {
        //保障有序性的Volatile写
        cursor.setVolatile(nextValue);  // StoreLoad fence

        long minSequence;
        //阻塞等待 直到wrapPoint大于最小门禁序号
        while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))
        {
            //阻塞当前线程1纳秒，注意sequencer主要是生产者持有，也就是说生产者的等待并未使用disruptor的等待策略，但是源码右面有todo注释表明开发者有意在以后的版本使用等待策略去自旋等待
            LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
        }
		//缓存最小门禁序号值
        this.cachedValue = minSequence;
    }

    this.nextValue = nextSequence;

    return nextSequence;
}
```

上面方法中最重要的就是这两个判断了，我们上面介绍过disruptor是使用的环形队列，且序号不断自增不设限制，但是环形队列是有长度的，我们如何保证生产者的数据不会覆盖掉消费者未消费的数据呢？disruptor就使用了gatingSequences门禁序号来解决此问题。**门禁序号保存的是消费者的sequence，每个消费者都会维护一个自己的Sequence对象，来记录自己的已消费的位置，每增加一个消费者，都会将引用添加到gatingSequences**。在计算时，我们取消费在最末尾的消费者的序号，当生产者序号减去最末尾的消费者序号只要小于环形队列一圈或一环的长度即可，算式为`生产者序号-消费者序号<环形队列长度`，否则表示生产者已经生产到了最末尾的消费者的位置，则进行自旋等待。

那么warpPoint是什么意思呢？其实这里只是不等式变换，即`warpPoint=生产者序号-环形队列长度<消费者序号`。到这里我们已经能明白上面方法的大概意思了，当调用next方法时，我们计算是否生产者的序号是否已经大于最末端消费者的一环或者说一圈，如果大于那么就自旋阻塞等待。

然后我们看一下publis方法，此方法就很简单，先设置游标即当前生产者发布的序号然后调用等待策略唤醒消费者：

```java
@Override
public void publish(long sequence)
{
    //这里使用的是unsafe的有序写入
    cursor.set(sequence);
    waitStrategy.signalAllWhenBlocking();
}
```

## 3.4 ConsumerRepository和ConsumerInfo

ConsumerRepository可以理解为消费者仓库，主要保存消费者的映射关系，其内部有三个Map分别保存的是：

- 业务事务处理器和消费者信息
- 序号和消费者信息，这里ConsumerInfo和Sequence是一对多的关系，一个ConsumerInfo可能有多个Sequence，但是一个Sequence只属于一个ConsumerInfo
- 消费者集合

![image-20221102150303379](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102150303379.png)

当我们调用disruptor的handleEventsWith等绑定方法时，最终会调用ConsumerRepository的add方法，然后去维护这些集合，源码略可自行翻阅。

然后我们去看一下ConsumerInfo消费者信息类，其实ConsumerInfo是一个接口，实现类如下：

![image-20221102150934640](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102150934640.png)

我们先看一下EventProcessorInfo，其主要属性如下：

![image-20221102151058249](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102151058249.png)

然后我们再看一下WorkerPoolInfo：

![image-20221102151419715](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102151419715.png)

![image-20221102151426825](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102151426825.png)

这里我们可以看到EventProcessorInfo中，只有一个EventProcessor，是一个单线程的消费者，然后再WorkerPoolInfo中，WorkPool整体是一个消费者，是一个多线程消费者，每个生产者的事件只会被WorkPool中的某一个WorkProcessor消费，也就是竞争消费。

## 3.5 事件处理器

然后我们看看消费者具体的执行线程，也可以叫做事件处理器或消费者处理器，注意与Handler的处理器相区分，Handler是事务处理器，是用户自定的事务，Processor可以看作是流程，是disruptor框架的执行流程。

事件处理器或消费者处理器有很多，如下图：

![image-20221102153230999](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221102153230999.png)

我们这里只拿BatchEventProcessor来举例：

```java
@Override
public void run()
{
    if (running.compareAndSet(IDLE, RUNNING))
    {
        sequenceBarrier.clearAlert();

        notifyStart();
        try
        {
            if (running.get() == RUNNING)
            {
                processEvents();
            }
        }
        finally
        {
            notifyShutdown();
            running.set(IDLE);
        }
    }
    else
    {
        // This is a little bit of guess work.  The running state could of changed to HALTED by
        // this point.  However, Java does not have compareAndExchange which is the only way
        // to get it exactly correct.
        if (running.get() == RUNNING)
        {
            throw new IllegalStateException("Thread is already running");
        }
        else
        {
            earlyExit();
        }
    }
}
```



## 3.6 SequenceBarrier

## 3.7 RingBuffer

## 3.8 WaitStrategy

# 四、Disruptor底层核心原理
