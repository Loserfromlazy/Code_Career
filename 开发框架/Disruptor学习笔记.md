# Disruptor学习笔记

> 初步笔记并未做细致整理
>
> 参考资料：
>
> - https://www.cnblogs.com/crazymakercircle/p/13909235.html
> - 《Disruptor 红宝书》》尼恩
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

- 元素位置属性进行了缓存行数据填充，避免了伪共享问题

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
	//事件转换器
    public static final EventTranslatorOneArg<LongEvent,Long> TRANSLATOR = new EventTranslatorOneArg<LongEvent, Long>() {
        @Override
        public void translateTo(LongEvent longEvent, long sequence, Long data) {
            longEvent.setValue(data);      
        }
    };

    public void onData(long data){
        //通过序号将事件对象发布出现
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
    //连接消费者处理器
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
- 多生产者多消费者
- 多消费者单生产者
- 菱形方式执行
- 链式并行执行
- 多组消费者相互隔离
- 多组消费者航道执行

### 单生产者多消费者并行

并发系统中提升性能最好的方式就是单一写者原则，对Disruptor也适用。如果在需求中只有一个事件生产者，那么可设置为单一生产者模式来提高系统的性能，在Disruptor的构造函数中指定ProducerType即可，例子如下

```java
public static void handleEvent(LongEvent longEvent, long l, boolean b){
    System.out.println(longEvent.getValue());
}

@Test
public void SingleProducer(){
    Executor executor = Executors.newCachedThreadPool();
    int bufferSize = 1024;
    //构造事件分发器，参数：事件，队列长度，线程池，生产者类型（多生产者还是单生产者），等待策略
    Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, executor, ProducerType.SINGLE, new YieldingWaitStrategy());
    disruptor.handleEventsWith(TestScene::handleEvent,TestScene::handleEvent,TestScene::handleEvent);
    //开启事件分发
    disruptor.start();
    RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
    //事件生产者
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
    thread.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

### 多生产者单消费者





