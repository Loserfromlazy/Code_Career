# Java高并发编程篇四显式锁

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java显式锁进行研究学习。

# 五、Java显式锁

JUC显式锁是一种非常灵活的、使用纯Java语言实现的锁，这种锁的使用非常灵活，可以进行无条件的、可轮询的、定时的、可中断的锁获取和释放操作。由于JUC锁加锁和解锁的方法都是通过Java API显式进行的，因此也叫显式锁。

## 5.1 显式锁



使用Java内置锁时，不需要通过Java代码显式地对同步对象的监视器进行抢占和释放，这些工作由JVM底层完成，而且任何一个Java对象都能作为一个内置锁使用，所以Java的对象锁使用起来非常方便。但是，Java内置锁的功能相对单一，不具备一些比较高级的锁功能。Java显式锁就是为了解决这些Java对象锁的功能问题、性能问题而生的。JDK 5版本引入了Lock接口，Lock是Java代码级别的锁。为了与Java对象锁相区分，Lock接口叫作显式锁接口，其对象实例叫作显式锁对象。

> JUC出自并发大师Doug Lea之手，Doug Lea对Java并发性能的提升做出了巨大的贡献。除了实现JUC包外，Doug Lea还提供了高并发IO模式——Reactor模式多个版本的参考实现。Reactor模式是Java高并发服务端编程的一个至关重要的模式。

### 5.1.1 Lock接口

老规矩，我们先看一下Lock接口的结构，源码太长就不贴了，这里通过IDEA的Structure可以直接看到整个接口的结构：

![image-20220425171343647](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220425171343647.png)

与synchronized关键字不同，显式锁不再作为Java内置特性来实现，而是作为Java语言可编程特性来实现。这就为多种不同功能的锁实现留下了空间，各种锁实现可能有不同的调度算法、性能特性或者锁定语义。

### 5.1.2 可重入锁

ReentrantLock是JUC包提供的显式锁的一个基础实现类，ReentrantLock类实现了Lock接口，它拥有与synchronized相同的并发性和内存语义，但是拥有了限时抢占、可中断抢占等一些高级锁特性。此外，ReentrantLock基于内置的抽象队列同步器（Abstract Queued Synchronized，AQS）实现，在争用激烈的场景下，能表现出表内置锁更佳的性能。

> 抽象队列同步器AQS是JUC包同步机制的基础设施，更是JUC锁框架的基础，会在下一篇进行学习

ReentrantLock是一个可重入的独占（或互斥）锁，其中

- 可重入表示能对一个线程进行重复加锁，比如，同一线程在外层函数获得锁后，在内层函数能再次获取该锁
- 独占表示在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能等待，只有拥有锁的线程释放了锁后，其他的线程才能够获取锁。

我们举个例子：

```java
public class TestReentrantLock {
    public static void main(String[] args) {
        final int TURNS = 1000;
        final int THREADS = 10;

        ExecutorService service = Executors.newFixedThreadPool(THREADS);
        Lock lock = new ReentrantLock();
        CountDownLatch latch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREADS; i++) {
            service.submit(() -> {
                for (int i1 = 0; i1 < TURNS; i1++) {
                    DataPlus.lockAndPlus(lock);
                }
                latch.countDown();
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) /1000F;
        System.out.println("运行时长"+time);
        System.out.println("结果"+DataPlus.sum);
    }

}
```

这里的临界资源DataPlus代码如下：

```java
public class DataPlus {
    public static int sum =0;
    public static void lockAndPlus(Lock lock){
        lock.lock();
        try {
            sum++;
        }finally {
            lock.unlock();
        }
    }
}
```

### 5.1.3 显式锁的使用方法

**使用lock方法阻塞式抢锁**

~~~java
//获取锁的实现类
Lock lock = new XXXLock();
lock.lock();//抢锁
try {
    //抢锁成功，执行临界区代码
    sum++;
}finally {
    //释放锁
    lock.unlock();
}
~~~

PS：

- 一定不能将lock方法放入try块中去执行，因为这样如果抢锁失败后再执行unlock方法会抛异常
- 释放锁操作unlock()必须在try-catch结构的finally块中执行，否则，如果临界区代码抛出异常，锁就有可能永远得不到释放。
- 在抢占锁操作lock()和try语句之间不要插入任何代码，避免抛出异常而导致释放锁操作unlock()执行不到，导致锁无法被释放。

**使用tryLock方法抢锁**

~~~java
//获取锁的实现类
Lock lock = new XXXLock();
if(lock.tryLock()){//抢锁
    try {
    //抢锁成功，执行临界区代码
    sum++;
    }finally {
        //释放锁
        lock.unlock();
    }
}else{
    //抢锁失败，执行后续代码
}
~~~

**使用tryLock方法限时阻塞式抢锁**

~~~java
//获取锁的实现类
Lock lock = new XXXLock();
if(lock.tryLock(1,TimeUnit.SECONDS)){//限时阻塞抢锁
    try {
    //抢锁成功，执行临界区代码
    sum++;
    }finally {
        //释放锁
        lock.unlock();
    }
}else{
    //抢锁失败，执行后续代码
}
~~~

> 1. lock()方法用于阻塞抢锁，抢不到锁时线程会一直阻塞。
> 2. tryLock()方法用于尝试抢锁，该方法有返回值，如果成功就返回true，如果失败（锁已被其他线程获取）就返回false。此方法无论如何都会立即返回，在抢不到锁时，线程不会像调用lock()方法那样一直被阻塞。
> 3. tryLock(long time,TimeUnit unit)方法和tryLock()方法类似，只不过这个方法在抢不到锁时会阻塞一段时间。如果在阻塞期间获取到锁就立即返回true，超时则返回false。

### 5.1.4 显式锁进行等待-通知通信

与Object对象的wait、notify两类方法相类似，基于Lock显式锁，JUC也为大家提供了一个用于线程间进行“等待-通知”方式通信的接口——java.util.concurrent.locks.Condition。

我们看一下Condition的结构：

![image-20220426090003483](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426090003483.png)

可以看到，功能与我们之前学习的wait和notify相似。

Condition对象是基于显式锁的，所以不能独立创建一个Condition对象，而是需要借助于显式锁实例去获取其绑定的Condition对象。不过，每一个Lock显式锁实例都可以有任意数量的Condition对象。具体来说，可以通过lock.newCondition()方法去获取一个与当前显式锁绑定的Condition实例，然后通过该Condition实例进行“等待-通知”方式的线程间通信。

我们下面拿之前学习的生产者消费者模型进行改造，之前的代码在[篇二-内置锁篇](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%BA%8C%E5%86%85%E7%BD%AE%E9%94%81.md)

```java
public class DataAwaitSignal<T> {
    public static final int MAX_COUNT = 10;
    private List<T> dataList = new LinkedList<>();
    private Integer count = 0;

    private final Lock lock = new ReentrantLock();
    private final Condition  NOT_FULL_LOCK =  lock.newCondition();
    private final Condition NOT_EMPTY_LOCK = lock.newCondition();


    public void add(T element) throws Exception{
        lock.lock();
        try {
            while (count>MAX_COUNT){
                System.out.println("队列已经满了");
                NOT_FULL_LOCK.await();
            }
        }finally {
            lock.unlock();
        }
        lock.lock();
        try {
            dataList.add(element);
            count++;
            NOT_EMPTY_LOCK.signal();
        }finally {
            lock.unlock();
        }

    }

    public T get() throws InterruptedException {
        lock.lock();
        try {
            while (count<=0){
                System.out.println("队列已经空了");
                NOT_EMPTY_LOCK.await();
            }
        }finally {
            lock.unlock();
        }

        T element;
        lock.lock();
        try {
            element = dataList.remove(0);
            count--;
            NOT_FULL_LOCK.signal();
        }finally {
            lock.unlock();
        }
        return element;
    }

    //直接在此类进行测试，就不新建测试类了
    public static void main(String[] args) {
        DataWaitNotify<Goods> data =new DataWaitNotify<>();
        Callable<Goods> productAction = ()->{
            Goods goods =Goods.getOne();
            data.add(goods);
            return goods;
        };
        Callable<Goods> consumeAction = ()->{
            Goods goods = data.get();
            return goods;
        };
        final int ALL_THREAD = 20;
        ExecutorService service = Executors.newFixedThreadPool(ALL_THREAD);
        final int CONSUME_TOTAL = 10;
        final int PRODUCE_TOTAL = 1;
        for (int i = 0; i < PRODUCE_TOTAL; i++) {
            service.submit(new Producer(productAction,50));
        }
        for (int i = 0; i < CONSUME_TOTAL; i++) {
            service.submit(new Consumer(consumeAction,100));
        }

    }
}
```

在使用Condition时有几点需要注意：

- 在调用await()和signal()方法前，等待线程必须获得显式锁,await()方法会让当前线程加入Condition对象等待队列中。signal()方法会从Condition对象等待队列中唤醒一个线程。
- 通知线程在调用signal()方法后，一定要记得释放当前占用的显式锁，只有这样，被唤醒的等待线程才能有获得锁的机会，才能继续执行。
- 在调用await()方法后，线程会释放当前占用的显式锁，以便通知线程能够抢到锁。通知线程抢到锁之后，才可以进入临界区发送通知。

> 由于Lock有公平锁和非公平锁之分，而Condition是与Lock绑定的，因此就有与Lock一样的公平特性：如果是公平锁，等待线程按照FIFO（先进先出）顺序从Condition对象的等待队列中唤醒；如果是非公平锁，后续的唤醒次序就不保证FIFO顺序了。

### 5.1.5 LockSupport

LockSupport是JUC提供的一个线程阻塞与唤醒的工具类，该工具类可以让线程在任意位置阻塞和唤醒，其所有的方法都是静态方法。

我们这里展示下结构，源码请自行翻阅。（再次推荐IDEA的结构，可以展示当前类的完整结构）

![image-20220426093008468](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426093008468.png)

LockSupport的方法主要有两类：park和unpark。park的英文意思为停车，如果把Thread看成一辆车的话，park()方法就是让车停下，其作用是将调用park()方法的当前线程阻塞；而unpark()方法是让车启动，然后跑起来，其作用是将指定线程Thread唤醒。

举个例子：

```java
public class TestLockSupport {
    static class Demo implements Runnable{

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+"无限休眠");
            LockSupport.park();//此方法会让线程在这一直被休眠
            if (Thread.currentThread().isInterrupted()){
                System.out.println(Thread.currentThread().getName()+"线程被打断了");
            }else {
                System.out.println(Thread.currentThread().getName()+"重新唤醒");
            }
        }
    }

    public static void main(String[] args) {
        Thread thread1 = new Thread(new Demo(),"111");
        Thread thread2 = new Thread(new Demo(),"222");
        thread1.start();
        thread2.start();
        thread1.interrupt();
        LockSupport.unpark(thread2);
    }
}
```

执行结果：

~~~
111无限休眠
222无限休眠
111线程被打断了
222重新唤醒
~~~

> **LockSupport.park()和Thread.sleep()的区别**
>
> 从功能上说，LockSupport.park()与Thread.sleep()方法类似，都是让线程阻塞且都不会释放持有的锁，二者的区别如下：
>
> - Thread.sleep()没法从外部唤醒，只能自己醒过来；而被LockSupport.park()方法阻塞的线程可以通过调用LockSupport.unpark()方法去唤醒。
> - Thread.sleep()方法声明了InterruptedException中断异常，这是一个受检异常，调用者需要捕获这个异常或者再抛出；而调用LockSupport.park()方法时不需要捕获中断异常。
> - 被LockSupport.park()方法、Thread.sleep()方法所阻塞的线程有一个特点，当被阻塞线程的Thread.interrupt()方法被调用时，被阻塞线程的中断标志将被设置，该线程将被唤醒。不同的是，二者对中断信号的响应方式不同：LockSupport.park()方法不会抛出InterruptedException异常，仅仅设置了线程的中断标志；而Thread.sleep()方法会抛出InterruptedException异常。
> - 与Thread.sleep()相比，调用LockSupport.park()能更精准、更加灵活地阻塞、唤醒指定线程。
> - Thread.sleep()本身就是一个Native方法；LockSupport.park()并不是一个Native方法，只是调用了一个Unsafe类的Native方法（名字也叫park）去实现。
> - LockSupport.park()方法还允许设置一个Blocker对象，主要用来供监视工具或诊断工具确定线程受阻塞的原因。
>
> **LockSupport.park()与Object.wait()的区别**
>
> - Object.wait()方法需要在synchronized块中执行，而LockSupport.park()可以在任意地方执行。
>
> - 当被阻塞线程被中断时，Object.wait()方法抛出了中断异常，调用者需要捕获或者再抛出；当被阻塞线程被中断时，LockSupport.park()不会抛出异常，调用时不需要处理中断异常。
>
> - 如果线程在没有被Object.wait()阻塞之前被Object.notify()唤醒，也就是说在Object.wait()执行之前去执行Object.notify()，就会抛出IllegalMonitorStateException异常，是不被允许的；
>
>   而线程在没有被LockSupport.park()阻塞之前被LockSupport.unPark()唤醒，也就是说在LockSupport.park()执行之前去执行LockSupport.unPark()，不会抛出任何异常，是被允许的。

### 5.1.6 显式锁的分类

图片来自于我自己的博客的文章[Java多线程](https://www.cnblogs.com/yhr520/p/13273534.html)，图片是以前自己画的。各个锁的区别也可以阅读下我这篇博文的第十章。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/Java%E9%94%81%281.8%29.png)

## 5.2 乐观锁和悲观锁

独占锁其实就是一种悲观锁，Java的synchronized是悲观锁。悲观锁可以确保无论哪个线程持有锁，都能独占式访问临界区。悲观锁总是假设会发生最坏的情况，每次线程读取数据时，也会上锁。这样其他线程在读取数据时就会被阻塞，直到它拿到锁。传统的关系型数据库用到了很多悲观锁，比如行锁、表锁、读锁、写锁等。

悲观锁的问题如下：

- 性能问题
- 一个线程获得锁后，抢锁的其余线程都会挂起
- 如果优先级高的锁等优先级低的锁释放锁，会导致优先级倒置

有效方式是使用乐观锁去替代悲观锁。与之类似，数据库操作中的带版本号数据更新、JUC包的原子类，都使用了乐观锁的方式提升性能。

### 5.2.1 CAS简单实现乐观锁

乐观锁一种比较典型的就是CAS原子操作，JUC强大的高并发性能是建立在CAS原子之上的。CAS操作中包含三个操作数：需要操作的内存位置（V）、进行比较的预期原值（A）和拟写入的新值（B）。如果内存位置V的值与预期原值A相匹配，那么处理器会自动将该位置的值更新为新值B；否则处理器不做任何操作。

自旋锁的基本含义为：当一个线程在获取锁的时候，如果锁已经被其他线程获取，调用者就一直在那里循环检查该锁是否已经被释放，一直到获取到锁才会退出循环。

CAS自旋锁的实现原理为：抢锁线程不断进行CAS自旋操作去更新锁的owner（拥有者），如果更新成功，就表明已经抢锁成功，退出抢锁方法。如果锁已经被其他线程获取（也就是owner为其他线程），调用者就一直在那里循环进行owner的CAS更新操作，一直到成功才会退出循环。

下面进行简单实现：

```java
public class SpinLock implements Lock{
    //当前拥有锁的线程
    private AtomicReference<Thread> atomicReference = new AtomicReference<>();
    @Override
    public void lock() {
        Thread thread = Thread.currentThread();
        while (!atomicReference.compareAndSet(null, thread)){
            //Thread.yield();
        }
    }
    @Override
    public void unlock() {
        Thread thread = Thread.currentThread();
        if (thread == atomicReference.get()){
            atomicReference.set(null);
        }
    }
    //其余方法不实现
    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public Condition newCondition() {
        return null;
    }

    public static void main(String[] args) {
        SpinLock spinLock = new SpinLock();
        final int TURNS = 1000;
        final int THREADS = 10;

        ExecutorService service = Executors.newFixedThreadPool(THREADS);
        CountDownLatch latch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREADS; i++) {
            service.submit(() -> {
                for (int j = 0; j < TURNS; j++) {
                    //这个DataPlus跟上面5.1.2例子相同
                    DataPlus.lockAndPlus(spinLock);
                }
                latch.countDown();
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) /1000F;
        System.out.println("运行时长"+time);
        System.out.println("结果"+DataPlus.sum);
    }

}
```

当然这个代码不可以重入，思考一下就能知道，当一个线程第一次已经获取到了该锁，在锁没有被释放之前，如果又一次重新获取该锁，第二次将不能成功获取到。

所以如果想实现可重入的版本，可以通过增加计数器的方式如下：

```java
//当前拥有锁的线程
private AtomicReference<Thread> atomicReference = new AtomicReference<>();
//记录同一个线程重复获取锁的次数
private int count = 0;
@Override
public void lock() {
    Thread thread = Thread.currentThread();
    if (thread==atomicReference.get()){
        count++;
        return;
    }
    while (!atomicReference.compareAndSet(null, thread)){

    }
}
@Override
public void unlock() {
    Thread thread = Thread.currentThread();
    if (thread == atomicReference.get()){
        if (count>0){
            count--;
        }else {
            atomicReference.set(null);
        }

    }
}
```

自旋锁的特点：线程获取锁的时候，如果锁被其他线程持有，当前线程将循环等待，直到获取到锁。线程抢锁期间状态不会改变，一直是运行状态（RUNNABLE），在操作系统层面线程处于用户态。

自旋锁的问题：在争用激烈的场景下，如果某个线程持有锁的时间太长，就会导致其他空自旋的线程耗尽CPU资源。另外，如果大量的线程进行空自旋，还可能导致硬件层面的“总线风暴”。

### 5.2.2 CAS可能会导致总线风暴

> 个人感觉本小节了解即可

CAS有关的sun.misc.Unsafe类的compareAndSwapInt()方法是一个Native方法调用，该本地方法在JDK中依次调用的C++代码为：（图片来自于原书）

![image-20220426131010353](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426131010353.png)

以上程序会根据当前CPU的类型是否为多核CPU来决定是否为cmpxchg指令添加lock前缀。如果程序在多核CPU上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序在单核CPU上运行，就省略lock前缀，因为单核CPU不需要lock前缀提供的内存屏障效果。

在SMP架构的CPU平台上，所有的Core（内核）会共享一条总线（BUS），靠此总线连接主存。每个核都有自己的高速缓存，各核相对于BUS对称分布。因此，这种结构称为“对称多处理器”。如下图（图片来自于原书）：

![image-20220426130756904](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426130756904.png)

假设Core 1和Core 2可同时把某个变量加载到自己的高速缓存中，当Core 1在自己的高速缓存中修改这个位置的值时，会通过总线使Core 2中L1高速缓存对应的值“失效”，而Core 2一旦发现自己缓存中的值失效，就会通过总线从内存中读取最新的值，使得Core 2和Core 1中的值再次一致，这样CPU就保障了变量的“缓存一致性”。

CPU会通过MESI协议保障变量的缓存一致性。为了保障“缓存一致性”，不同的内核需要通过总线来回通信，因而所产生的流量一般被称为“缓存一致性流量”。因为总线被设计为固定的“通信能力”，如果缓存一致性流量过大，总线将成为瓶颈，这就是所谓的“总线风暴”。

> 总线风暴当然与CPU的架构和设计有关，并不是所有的CPU都会产生总线风暴。
>
> Java轻量级锁会快速膨胀为重量级锁，其本质上一是为了减少CAS空自旋，二是为了避免同一时间大量CAS操作所导致的总线风暴。
>
> JUC基于CAS实现的轻量级锁如何避免总线风暴呢？答案是：**使用队列对抢锁线程进行排队，最大程度上减少了CAS操作数量。**

### 5.2.3 CLH自旋锁

CLH锁其实就是一种基于队列（具体为单向链表）排队的自旋锁，这个名字是作者三人的名字组合的，所以叫CLH自旋锁。

简单的CLH锁可以基于单向链表实现，申请加锁的线程首先会通过CAS操作在单向链表的尾部增加一个节点，之后该线程只需要在其前驱节点上进行普通自旋，等待前驱节点释放锁即可。由于CLH锁只有在节点入队时进行一下CAS的操作，在节点加入队列之后，抢锁线程不需要进行CAS自旋，只需普通自旋即可。因此，在争用激烈的场景下，CLH锁能大大减少CAS操作的数量，以避免CPU的总线风暴。

> JUC中显式锁基于AQS抽象队列同步器，而AQS是CLH锁的一个变种
>

我们实现一个简单的CLH自旋锁版本（这只是个简单版本，性能很差）：

```java
public class CLHLock implements Lock {
    //虚拟等待队列的节点
    @Data
    static class Node{
        //true当前线程正在抢占或已经占有锁
        //false 当前线程已经释放锁
        volatile boolean locked;
        //上一个节点
        Node prevNode;
        public Node(boolean locked, Node prevNode) {
            this.locked = locked;
            this.prevNode = prevNode;
        }
        //空节点
        public static final Node EMPTY = new Node(false,null);
    }
    //当前节点的线程本地变量
    private static ThreadLocal<Node> nowNodeLocal = new ThreadLocal<>();

    //CLS队列的队尾指针
    private AtomicReference<Node> tail = new AtomicReference<>(null);

    public CLHLock() {
        tail.getAndSet(Node.EMPTY);
    }

    //加锁操作，节点加入队列的尾部
    @Override
    public void lock(){
        Node nowNode = new Node(true,null);
        Node preNode = tail.get();
        //自旋将当前节点插入队列尾部
        while (!tail.compareAndSet(preNode,nowNode)){
            preNode = tail.get();
        }
        nowNode.setPrevNode(preNode);
        //自旋监听前驱节点的locked变量，直到值为false
        while (nowNode.getPrevNode().isLocked()){
            //让出时间片，提高性能
            Thread.yield();
        }
        //能执行到这里说明获取到了锁
        System.out.println("获取到了锁");
        //将当前节点存在线程本地变量中
        nowNodeLocal.set(nowNode);
    }

    @Override
    public void unlock(){
        Node nowNode = nowNodeLocal.get();
        nowNode.setLocked(false);
        nowNode.setPrevNode(null);
        nowNodeLocal.set(null);
    }
	//测试实例
    public static void main(String[] args) {
        CLHLock clhLock = new CLHLock();
        final int TURNS = 100000;
        final int THREADS = 10;

        ExecutorService service = Executors.newFixedThreadPool(THREADS);
        CountDownLatch latch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREADS; i++) {
            service.submit(() -> {
                for (int j = 0; j < TURNS; j++) {
                    DataPlus.lockAndPlus(clhLock);
                }
                latch.countDown();
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) /1000F;
        System.out.println("运行时长"+time);
        System.out.println("结果"+DataPlus.sum);
    }
	//方法未实现
    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}
```

运行结果：

~~~
......
获取到了锁
获取到了锁
运行时长5.311
结果1000000
~~~

简单回顾一下CLH的算法：抢锁线程在队列尾部加入一个节点，然后仅在前驱节点上进行普通自旋，它不断轮询前一个节点状态，如果发现前一个节点释放锁，当前节点抢锁成功。

CLH的算法有以下几个要点：

- 初始状态队列尾部属性（tail）指向一个EMPTY节点。tail属性使用AtomicReference类型是为了使得多个线程并发操作tail时不会发生线程安全问题。
- Thread在抢锁时会创建一个新的Node加入等待队列尾部：tail指向新的Node，同时新的Node的preNode属性指向tail之前指向的节点，并且以上操作通过CAS自旋完成，以确保操作成功。
- Thread加入抢锁队列之后，会在前驱节点上自旋：循环判断前驱节点的locked属性是否为false，如果为false就表示前驱节点释放了锁，当前线程抢占到锁。
- Thread抢到锁之后，它的locked属性一直为true，一直到临界区代码执行完，然后调用unlock()方法释放锁，释放之后其locked属性才为false。线程从本地变量curNodeLocal中获取当前节点curNode，将其状态设置为false，以便其后继节点能获得锁。线程在设置当前节点curNode的locked状态为false前，为了GC能回收前驱节点，需要将curNode前驱节点引用设置为空。另外，为了使得线程下一次抢锁不会出错，需要将线程本地变量curNodeLocal中的节点引用设置为空。

> CLH锁是一种队列锁，其优点是空间复杂度低。如果有N个线程、L个锁，每个线程每次只获取一个锁，那么需要的存储空间是O(L+N)，N个线程有N个Node，L个锁有L个Tail。
>
> CLH队列锁的一个显著缺点是它在NUMA架构的CPU平台上性能很差。CLH队列锁在NUMA架构的CPU平台上，每个CPU内核有自己的内存，如果前驱节点在不同的CPU内核上，它的内存位置比较远，在自旋判断前驱节点的locked属性时，性能将大打折扣。然而，CLH锁在SMP架构的CPU平台上则不存在这个问题，性能还是挺高的。

## 5.3 公平与非公平锁

synchronized内置锁是一种非公平锁，默认情况下ReentrantLock锁也是非公平锁。

公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。

虽然ReentrantLock锁默认是非公平锁，但可以通过构造器指定该锁为公平锁，具体的代码如下：

```java
Lock lock = new ReentrantLock(true);
```

> 源码中可以看到ReentrantLock里面有一个内部类Sync，他又FairSync公平锁和NonfairSync非公平锁两个子类。默认使用非公平锁。

## 5.4 可中断和不可中断锁

可中断锁是指抢占过程可以被中断的锁，JUC的显式锁（如ReentrantLock）是一个可中断锁。不可中断锁是指抢占过程不可以被中断的锁，如Java的synchronized内置锁就是一个不可中断锁。

### 5.4.1 锁的可中断抢占

在JUC的显式锁Lock接口中，有以下两个方法可以用于可中断抢占：

1. lockInterruptibly()可中断抢占锁抢占过程中会处理Thread.interrupt()中断信号，如果线程被中断，就会终止抢占并抛出InterruptedException异常。
2. tryLock(long timeout,TimeUnit unit)阻塞式“限时抢占”（在timeout时间内）锁抢占过程中会处理Thread.interrupt()中断信号，如果线程被中断，就会终止抢占并抛出InterruptedException异常。

举个例子：

我们先编写临界资源：

```java
public class DataPlus {
    public static int sum =0;

    //可中断抢锁
    public static void lockInterruptAndPlus(Lock lock){
        System.out.println(Thread.currentThread().getName()+" 开始抢锁");
        try {
            lock.lockInterruptibly();
        }catch (Exception e){
            System.out.println(Thread.currentThread().getName()+"抢占被中断，抢锁失败");
            return;
        }
        try {
            System.out.println(Thread.currentThread().getName()+"抢到了锁");
            Thread.sleep(1000);
            sum++;
            if (Thread.currentThread().isInterrupted()){
                System.out.println(Thread.currentThread().getName()+"执行被中断");
            }
        }catch (Exception e){
            System.out.println(Thread.currentThread().getName()+" 抛出异常");
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
```

然后编写测试用例：

```java
public class TestInterruptLock {
    public static void main(String[] args) throws InterruptedException {
        Lock lock = new ReentrantLock();
        Runnable runnable = () -> DataPlus.lockInterruptAndPlus(lock);
        Thread thread1 = new Thread(runnable);
        Thread thread2 = new Thread(runnable);

        thread1.start();
        thread2.start();
        Thread.sleep(100);
        thread1.interrupt();
        thread2.interrupt();
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

运行结果：

![image-20220426142824307](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426142824307.png)

可以看到在执行过程中，线程0抢到了锁，然后把两个线程都中断，然后会发现，线程0在执行过程中（正在sleep(1000)）所以抛出异常，而线程1因为没抢到锁还在抢锁中，然后直接被中断了。

### 5.4.2 死锁的检测和中断

死锁是指两个或两个以上线程因抢占锁而造成的相互等待的现象。多个线程通过AB-BA模式抢占两个锁是造成多线程死锁比较普遍的原因。AB-BA模式的死锁具体表现为：线程X按照先后次序去抢占锁A与锁B，线程Y按照先后次序去抢占锁B与锁A，当线程X抢到锁A再去抢占锁B时，发现已经被其他线程拿走，然而线程Y拿到锁B后再去抢占锁A时，发现已经被其他线程拿走，于是线程X等待其他线程释放锁B，线程Y等待其他线程释放锁A，两个线程互相等待从而造成死锁。

JDK 8中包含的ThreadMXBean接口提供了多种监视线程的方法，其中包括两个死锁监测的方法，具体如下：

- findDeadlockedThreads用于检测由于抢占JUC显式锁、Java内置锁引起死锁的线程。
- findMonitorDeadlockedThreads仅仅用于检测由于抢占Java内置锁引起死锁的线程。

ThreadMXBean的实例可以通过JVM管理工厂ManagementFactory去获取，具体的获取代码如下：

~~~
public static ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
~~~

> JVM管理工厂ManagementFactory类提供静态方法，返回各种获取JVM信息的Bean实例。我们通过这些Bean实例能获取大量的JVM运行时信息，比如JVM堆的使用情况、GC情况、线程信息等

我们举个死锁的例子：

```java
public class DeadLockDemo {

    public static ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();

    //死锁
    public static void useLockInterruptLock(Lock lock1,Lock lock2){
        final String lockAName = lock1.toString().replace("java.util.concurrent.locks", "");
        final String lockBName = lock1.toString().replace("java.util.concurrent.locks", "");
        System.out.println(Thread.currentThread().getName()+" 开始抢第一把锁："+lockAName);
        try {
            lock1.lockInterruptibly();
        } catch (InterruptedException e) {
            System.out.println(Thread.currentThread().getName()+" 被中断，抢第一把锁失败："+lockAName);
            return;
        }
        try {
            System.out.println(Thread.currentThread().getName()+" 抢到了第一把锁："+lockAName);
            System.out.println(Thread.currentThread().getName()+" 开始抢第二把锁："+lockBName);
            try {
                lock2.lockInterruptibly();
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName()+" 被中断，抢第二把锁失败："+lockBName);
                return;
            }
            try {
                System.out.println(Thread.currentThread().getName()+" 抢到了第二把锁："+lockBName);
                System.out.println("do Something");
                Thread.sleep(1000);
            }catch (Exception e){
                e.printStackTrace();
            }finally {
                lock2.unlock();
                System.out.println("释放了第二把锁"+lockBName);
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock1.unlock();
            System.out.println("释放了第一把锁"+lockAName);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Lock lockA = new ReentrantLock();
        Lock lockB = new ReentrantLock();
        Runnable runnable1 = ()-> DeadLockDemo.useLockInterruptLock(lockA,lockB);
        Runnable runnable2 = () ->DeadLockDemo.useLockInterruptLock(lockB,lockA);
        Thread thread1 = new Thread(runnable1,"thread1");
        Thread thread2 = new Thread(runnable2,"thread2");
        thread1.start();
        thread2.start();
        //等执行一段时间在执行死锁检测
        Thread.sleep(2000);
        System.out.println("开始死锁检测");
        long[] deadlockedThreads = mxBean.findDeadlockedThreads();
        if (deadlockedThreads.length>0){
            System.out.println("发生了死锁");
            for (long thread : deadlockedThreads) {
                //此方法用于获取不带有堆栈跟踪信息的线程数据
                //ThreadInfo threadInfo = mxBean.getThreadInfo(thread);
                ThreadInfo threadInfo = mxBean.getThreadInfo(thread, Integer.MAX_VALUE);
                System.out.println(threadInfo);
            }
            System.out.println("中断一个死锁线程"+thread1.getName());
            thread1.interrupt();//中断一个死锁线程
        }
    }
}
```

执行结果：

![image-20220426172054023](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220426172054023.png)

## 5.5 共享和独占锁

在访问共享资源之前进行加锁操作，在访问完成之后进行解锁操作。按照“是否允许在同一时刻被多个线程持有”来区分，锁可以分为共享锁与独占锁。

### 5.5.1 独占锁

独占锁也叫排他锁、互斥锁、独享锁，是指锁在同一时刻只能被一个线程所持有。一个线程加锁后，任何其他试图再次加锁的线程都会被阻塞，直到持有锁线程解锁。通俗来说，就是共享资源某一时刻只能有一个线程访问，其余线程阻塞等待。如果是公平地独占锁，在持有锁线程解锁时，如果有一个以上的线程在阻塞等待，那么最先抢锁的线程被唤醒变为就绪状态去执行加锁操作，其他的线程仍然阻塞等待。Java中的Synchronized内置锁和ReentrantLock显式锁都是独占锁。

### 5.5.2 共享锁

共享锁就是在同一时刻允许多个线程持有的锁。当然，获得共享锁的线程只能读取临界区的数据，不能修改临界区的数据。JUC中的共享锁包括Semaphore（信号量）、ReadLock（读写锁）中的读锁、CountDownLatch倒数闩。

**Semaphore**

Semaphore**可以用来控制在同一时刻访问共享资源的线程数量**，通过协调各个线程以保证共享资源的合理使用。Semaphore维护了一组虚拟许可，它的数量可以通过构造器的参数指定。线程在访问共享资源前必须调用Semaphore的acquire()方法获得许可，如果许可数量为0，该线程就一直阻塞。线程访问完资源后，必须调用Semaphore的release()方法释放许可。更形象的说法是：**Semaphore是一个许可管理器**。

我们看一下类的结构和主要方法：

![image-20220427085205211](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427085205211.png)

下面举个例子：

```java
public class TestSemaphore {
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch latch = new CountDownLatch(10);
        final Semaphore semaphore = new Semaphore(2);
        AtomicInteger integer = new AtomicInteger(0);

        Runnable runnable = ()->{
          try {
              //阻塞开始获取请求
              semaphore.acquire(1);
              System.out.println(new Date()+",受理中，服务号"+integer.incrementAndGet());
              Thread.sleep(1000);
              semaphore.release(1);
          }catch (Exception e){
              e.printStackTrace();
          }
          latch.countDown();
        };
        Thread [] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(runnable,"线程"+i);
        }
        for (int i = 0; i < 10; i++) {
            threads[i].start();
        }
        latch.await();

    }
}
```

运行结果：每一秒中只有两个线程进入临界区。

~~~
Tue Apr 26 21:39:21 GMT+08:00 2022,受理中，服务号1
Tue Apr 26 21:39:21 GMT+08:00 2022,受理中，服务号2
Tue Apr 26 21:39:22 GMT+08:00 2022,受理中，服务号4
Tue Apr 26 21:39:22 GMT+08:00 2022,受理中，服务号3
Tue Apr 26 21:39:23 GMT+08:00 2022,受理中，服务号5
Tue Apr 26 21:39:23 GMT+08:00 2022,受理中，服务号6
Tue Apr 26 21:39:24 GMT+08:00 2022,受理中，服务号7
Tue Apr 26 21:39:24 GMT+08:00 2022,受理中，服务号8
Tue Apr 26 21:39:25 GMT+08:00 2022,受理中，服务号9
Tue Apr 26 21:39:25 GMT+08:00 2022,受理中，服务号10
~~~

**CountDownLatch**

CountDownLatch是一个常用的共享锁，其功能相当于一个多线程环境下的倒数门闩。CountDownLatch可以指定一个计数值，在并发环境下由线程进行减一操作，当计数值变为0之后，被await方法阻塞的线程将会唤醒。通过CountDownLatch可以实现线程间的计数同步。

在之前我们一直都在使用这个共享锁，所以这里就不再举例。

## 5.6 读写锁

读写锁的内部包含两把锁：一把是读（操作）锁，是一种共享锁；另一把是写（操作）锁，是一种独占锁。在没有写锁的时候，读锁可以被多个线程同时持有。写锁是具有排他性的：如果写锁被一个线程持有，其他的线程不能再持有写锁，抢占写锁会阻塞；进一步来说，如果写锁被一个线程持有，其他的线程不能再持有读锁，抢占读锁也会阻塞。

读写锁规则如下：

- 读操作、读操作能共存，是相容的。
- 读操作、写操作不能共存，是互斥的。
- 写操作、写操作不能共存，是互斥的。

JUC包中的读写锁接口为ReadWriteLock，主要有两个方法：

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```

通过ReadWriteLock接口能获取其内部的两把锁：一把是ReadLock，负责读操作；另一把是WriteLock，负责写操作。JUC中ReadWriteLock接口的实现类为ReentrantReadWriteLock。

### 5.6.1 ReentrantReadWriteLock

通过ReentrantReadWriteLock类能获取读锁和写锁，它的读锁是可以多线程共享的共享锁，而它的写锁是排他锁，在被占时不允许其他线程再抢占操作。然而其读锁和写锁之间是有关系的：同一时刻不允许读锁和写锁同时被抢占，二者之间是互斥的。

我们举个栗子(●ˇ∀ˇ●)：

```java
public class TestReadWriteLock {
    //共享数据
    final static Map<String,String> MAP = new HashMap<>();
    //读写锁
    final static ReentrantReadWriteLock LOCK = new ReentrantReadWriteLock();
    //读锁
    final static Lock READ_LOCK = LOCK.readLock();
    //写锁
    final static Lock WRITE_LOCK = LOCK.writeLock();

    public static Object put(String key,String value){
        WRITE_LOCK.lock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了写锁，执行写操作");
            Thread.sleep(1000);
            String put = MAP.put(key, value);
            return put;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            WRITE_LOCK.unlock();
        }
        return null;
    }

    public static Object get(String key){
        READ_LOCK.lock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了读锁，执行写操作");
            Thread.sleep(1000);
            String value = MAP.get(key);
            return value;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            READ_LOCK.unlock();
        }
        return null;
    }
    //测试
    public static void main(String[] args) {
        Runnable runnableWrite = ()-> put("key","value");
        Runnable runnableRead = () -> get("key");
        for (int i = 0; i < 4; i++) {
            new Thread(runnableRead,"读线程"+i).start();
        }
        for (int i = 0; i < 2; i++) {
            new Thread(runnableWrite,"写线程"+i).start();
        }
    }
}
```

运行结果：

~~~
读线程2] Wed Apr 27 09:04:49 CST 2022抢占了读锁，执行写操作
读线程0] Wed Apr 27 09:04:49 CST 2022抢占了读锁，执行写操作
写线程0] Wed Apr 27 09:04:50 CST 2022抢占了写锁，执行写操作
写线程1] Wed Apr 27 09:04:51 CST 2022抢占了写锁，执行写操作
读线程1] Wed Apr 27 09:04:52 CST 2022抢占了读锁，执行写操作
读线程3] Wed Apr 27 09:04:52 CST 2022抢占了读锁，执行写操作
~~~

从输出结果可以看出：读线程0、2同时获取了读锁，说明可以同时进行共享数据的读操作。但是写线程0、1只能依次获取写锁，说明共享数据的写操作不能同时进行。同时读线程1、3必须等待写线程1释放写锁后才能获取到读锁，说明读写操作是互斥的。

### 5.6.2 锁的升级与降级

锁升级是指读锁升级为写锁，锁降级指的是写锁降级为读锁。在ReentrantReadWriteLock读写锁中，只支持写锁降级为读锁，而不支持读锁升级为写锁。

我们下面对上面的例子进行修改：

```java
public class TestReadWriteLock2 {
    //共享数据
    final static Map<String,String> MAP = new HashMap<>();
    //读写锁
    final static ReentrantReadWriteLock LOCK = new ReentrantReadWriteLock();
    //读锁
    final static Lock READ_LOCK = LOCK.readLock();
    //写锁
    final static Lock WRITE_LOCK = LOCK.writeLock();

    public static Object put(String key,String value){
        WRITE_LOCK.lock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了写锁，执行写操作");
            Thread.sleep(1000);
            String put = MAP.put(key, value);
            System.out.println(Thread.currentThread().getName()+"尝试将写锁降级为读锁");
            READ_LOCK.lock();
            System.out.println(Thread.currentThread().getName()+"写锁降级成功");
            return put;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            READ_LOCK.unlock();
            WRITE_LOCK.unlock();
        }
        return null;
    }

    public static Object get(String key){
        READ_LOCK.lock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了读锁，执行写操作");
            Thread.sleep(1000);
            String value = MAP.get(key);
            System.out.println(Thread.currentThread().getName()+"读锁尝试升级为写锁");
            WRITE_LOCK.lock();
            System.out.println(Thread.currentThread().getName()+"读锁升级写锁成功");
            return value;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            WRITE_LOCK.unlock();
            READ_LOCK.unlock();
        }
        return null;
    }
    //测试
    public static void main(String[] args) throws InterruptedException {
        Runnable runnableWrite = ()-> put("key","value");
        Runnable runnableRead = () -> get("key");
        //要先创建写线程，因为读线程不能升级，可能会导致看不到运行结果
        new Thread(runnableWrite,"写线程").start();
        new Thread(runnableRead,"读线程").start();
    }
}
```

运行结果：

~~~
写线程] Wed Apr 27 09:18:47 CST 2022抢占了写锁，执行写操作
写线程尝试将写锁降级为读锁
写线程写锁降级成功
读线程] Wed Apr 27 09:18:48 CST 2022抢占了读锁，执行写操作
读线程读锁尝试升级为写锁
~~~

从结果来看，只能降级不能升级。

> ReentrantReadWriteLock不支持读锁的升级，主要是避免死锁，例如两个线程A和B都占了读锁并且都需要升级成写锁，A升级要求B释放读锁，B升级要求A释放读锁，二者就会由于相互等待形成死锁。
>
> 与ReentrantLock相比，ReentrantReadWriteLock更适合读多写少的场景

### 5.6.3 StampedLock

StampedLock（印戳锁）是对ReentrantReadWriteLock读写锁的一种改进，主要的改进为：在没有写只有读的场景下，StampedLock支持不用加读锁而是直接进行读操作，最大程度提升读的效率，只有在发生过写操作之后，再加读锁才能进行读操作。

StampedLock的三种模式如下：

- 悲观读锁：与ReadWriteLock的读锁类似，多个线程可以同时获取悲观读锁，悲观读锁是一个共享锁。
- 乐观读锁：相当于直接操作数据，不加任何锁，连读锁都不要。
- 写锁：与ReadWriteLock的写锁类似，写锁和悲观读锁是互斥的。虽然写锁与乐观读锁不会互斥，但是在数据被更新之后，之前通过乐观读锁获得的数据已经变成了脏数据。

与ReentrantReadWriteLock不同，StampedLock没有继承ReadWriteLock接口，而是自己根据实现了锁。类结构有些长，我们这里看主要方法的源码：

```java
//悲观读锁

//获取普通（悲观）读锁，返回long类型的印戳值
public long readLock()
//释放普通（悲观）读锁，以获取的印戳值为参数
public void unlockRead(long stamp)

//写锁

//获取写锁，返回long类型印戳值
public long writeLock()
//释放写锁
public void unlockWrite(long stamp)

//乐观锁的印戳获取和有效性判断

//获取乐观锁，返回long印戳值，返回0表示答盎前所处于写锁模式，不能乐观读
public long tryOptimisticRead()
//判断乐观读的印戳值是否有效，
public boolean validate(long stamp)
```

我们举个栗子(●ˇ∀ˇ●)：

```java
public class TestStampedLock {

    final static Map<String,String> MAP = new HashMap<>();
    final static StampedLock STAMPED_LOCK = new StampedLock();
    //写操作
    public static Object put(String key,String value){
        long stamp = STAMPED_LOCK.writeLock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了写锁，执行写操作");
            Thread.sleep(1000);
            String put = MAP.put(key, value);
            return put;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"释放了写锁");
            STAMPED_LOCK.unlockWrite(stamp);
        }
        return null;
    }
    //悲观读
    public static Object pessimismGet(String key){
        System.out.println(Thread.currentThread().getName()+"] "+new Date()+"进入过写模式，只能悲观读");
        long stamp = STAMPED_LOCK.readLock();
        try {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"抢占了读锁");
            Thread.sleep(1000);
            String value = MAP.get(key);
            return value;
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"释放了读锁");
            STAMPED_LOCK.unlockRead(stamp);
        }
        return null;
    }
    //乐观读
    public static Object optimismGet(String key){
        String value =null;
        long stamp = STAMPED_LOCK.tryOptimisticRead();
        if (stamp!=0){
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"获取乐观读成功");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            value = MAP.get(key);
        }else {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"获取乐观读失败");
            return pessimismGet(key);
        }
        //对乐观读是否有效进行验证，判断是否进入过写模式
        if (!STAMPED_LOCK.validate(stamp)){
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"乐观读已经失效");
            return pessimismGet(key);
        }else {
            System.out.println(Thread.currentThread().getName()+"] "+new Date()+"乐观读没有失效");
            return value;
        }
    }
    //测试
    public static void main(String[] args) {
        Runnable runnableWrite = () -> put("key","value");
        Runnable runnableRead = () -> optimismGet("key");
        new Thread(runnableWrite,"写线程").start();
        new Thread(runnableRead,"读线程").start();
    }
}
```

运行结果：

~~~
写线程] Wed Apr 27 09:55:55 CST 2022抢占了写锁，执行写操作
读线程] Wed Apr 27 09:55:55 CST 2022获取乐观读失败
读线程] Wed Apr 27 09:55:55 CST 2022进入过写模式，只能悲观读
写线程] Wed Apr 27 09:55:56 CST 2022释放了写锁
读线程] Wed Apr 27 09:55:56 CST 2022抢占了读锁
读线程] Wed Apr 27 09:55:57 CST 2022释放了读锁
~~~

