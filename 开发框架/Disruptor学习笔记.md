# Disruptor学习笔记

> 初步笔记并未做细致整理
>
> 参考资料：https://www.cnblogs.com/crazymakercircle/p/13909235.html

## 使用

### 入门

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

**事件工厂**

```java
public class LongEventFactory implements EventFactory<LongEvent> {
    @Override
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```

**事件处理器（事件消费者）**

```java
public class LongEventHandler implements EventHandler<LongEvent> {

    @Override
    public void onEvent(LongEvent longEvent, long l, boolean b) throws Exception {
        System.out.println(longEvent.getValue());
    }
}
```

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

**测试类：**

```java
@Test
    public void testSimpleDisruptor() throws InterruptedException {
        //消费者线程池
        Executor executor = Executors.newCachedThreadPool();
        //事件工厂
        LongEventFactory eventFactory = new LongEventFactory();
        //环形队列大小
        int bufferSize = 1024;
        //构造分裂者，及事件分发器
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
        ringBuffer.publishEvent(TRANSLATOR,data);
    }
}
```

Disruptor提供了不同的接口的转换器：

- EventTranslator
- EventTranslatorOneArg
- EventTranslatorTwoArg

Translator中方法的参数是通过RingBuffer传递的，事件转换器的测试类：

**通过Lambda使用Disruptor**

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

### WaitStrategy

在构造Disruptor时可以指定WaitStrategy，Disruptor提供了多个策略，每种策略都有不同的性能和优缺点，根据实际运行环境的CPU硬件特点选择适当的策略，并配合特定的JVM的配置参数，能实现不同的性能提升。

**BlockingWaitStrategy**

最低效的策略，对CPU消耗最小，并且在各种不同的部署环境中能提供更加一致的性能表现

**SleepingWaitStrategy**

性能与BlockingWaitStrategy差不多，CPU消耗也相似，但是对生产者线程的影响较小，适合高延迟，比如异步日志的场景。

**YieldingWaitStrategy**

性能最好，适合低延迟系统，一般在事件处理线程数小于CPU逻辑核心数的场景中和性能要求极高的场景中。YieldingWaitStrategy是可以被用在低延迟系统的两个策略之一，这种策略在降低系统延迟的时候也会增加CPU运算量，此策略会循环等待sequence增加到合适的值。循环中会调用Thread.yield()方法允许其他准备好的线程执行。如果需要高性能且消费者线程比核心逻辑数少的时候适合此策略，比如在开启超线程的情况。

**BusySpinWaitStrategy**

这是性能最高的等待策略，同时也是对部署环境要求最高的策略，这个性能最好用在消费者线程比物理内核数目还要小的时候，比如禁用超线程技术时。

## 使用场景

Disruptor是一个优秀的并发框架，可以使用在生产者消费者的多个场景：

- 单生产者多消费者
- 多生产者多消费者
- 多消费者单生产者
- 菱形方式执行
- 链式并行执行
- 多组消费者相互隔离
- 多组消费者航道执行

### 单生产者多消费者

并发系统中提升性能最好的方式就是单一写者原则，对Disruptor也适用。如果在需求中只有一个事件生产者，那么可设置为单一生产者模式来提高系统的性能。

```java
public static void handleEvent(LongEvent longEvent, long l, boolean b){
    System.out.println(longEvent.getValue());
}

@Test
public void SingleProducer(){
    Executor executor = Executors.newCachedThreadPool();
    int bufferSize = 1024;
    //构造事件分发器
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





