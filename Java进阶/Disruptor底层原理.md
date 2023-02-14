# Disruptor底层原理

> 参考资料：
>
> - https://zhuanlan.zhihu.com/p/458926355
> - https://www.cnblogs.com/crazymakercircle/p/13909235.html
> - 《Disruptor 红宝书》尼恩
> - https://zhuanlan.zhihu.com/p/458926355
> - https://blog.csdn.net/w57685321/article/details/111350424
> - https://blog.csdn.net/daocaokafei/article/details/114707040
> - http://events.jianshu.io/p/25bee95a9296

## 一、伪共享问题

### 1.1 CPU缓存结构

CPU的缓存架构如下：

一般来说，现代的 CPU 里通常会有多个 CPU 核心，并且每个 CPU 核心都有自己的 L1 Cache 和 L2 Cache，而 L1 Cache 通常分为 dCache（数据缓存） 和 iCache（指令缓存），L1缓存很小很快紧靠在CPU内核中，L2缓存更大一些，但依然只被一个CPU核心使用。L3缓存更大更慢，但是是被单个插槽的多个核心共享的。最后才是主存，由全部插槽的所有CPU核心共享。这就是 CPU 典型的缓存层次。一般来说越靠近CPU的缓存速度越快、容量也越小。

![image-20221105222048153](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105222048153.png)

为了提高效率，CPU每次从内存中不是一个字节一个字节读取的，而是一批一批的去读取的，这一批数据也叫做缓存行。一般来说CPU读取一行数据时，会将需要的数据周围的数据连续的一次性全部读取到缓存中（如下图），一般一行缓存行为64字节，目前主流的CPU缓存行都是64字节。

> 在Linux中可以使用如下命令查看缓存行大小
>
> `cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size`

![image-20221105222250478](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105222250478.png)

### 1.2 伪共享问题的原因

在多线程执行的过程中，可能会存在这样一种情况，就是多个需要频繁修改的变量存在同一缓存行中，比如现在有a和b两个变量，且他们在同一缓存行中，这时假如有两个线程同时修改，线程1在CPU核心1中，线程2在CPU核心2中，这时线程1先获取变量值a，但由于a和b在同一缓存行，所以 A 和 B 的数据都会被加载到 Cache，如下图：

![image-20221105223012108](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105223012108.png)

这时线程2要修改变量b的值那么这一缓存行又会被加载到CPU核心2的缓存，此时 1 号和 2 号核心的 Cache Line 状态变为共享状态，如下图：

![image-20221105222250478](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105222250478.png)

这时假如CPU核心1需要修改变量 a，发现此 Cache Line 的状态是共享状态，所以先需要通过总线发送消息给 CPU核心2，通知CPU核心2把 Cache 中对应的 Cache Line 标记为已失效状态，然后 1 号核心对应的 Cache Line 状态变成已修改状态，并且修改变量 a：

![image-20221105223421096](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105223421096.png)

这时CPU2号核心想修改变量b，但是发现缓存行已经失效，这时它就需要重新去内存中加载。如果CPU1修改了变量a时，缓存行数据就失效了，也就是CPU2中的缓存行数据也失效了，CPU2需要的y需要重新从内存加载。如果，CPU2修改了变量y时，缓存行数据就失效了，也就是CPU1中的缓存行数据也失效了，CPU1需要的x需要重新从内存加载。也就是说a和b两个变量在两个CPU中本来是不影响的，但是由于缓存行在一起因此互相受到了影响。

出现伪共享问题的本质原因是一行缓存行可以缓存多个变量，而CPU对于缓存的修改又是以缓存行为最小单位的，对缓存行中的单个变量进行修改，导致整个缓存行失效，需要从内存中重新加载。

### 1.3 Java中解决伪共享的手段

一般来说解决伪共享的方式通常是以空间换时间，通过占位字符将变量的缓存行填满。在Java8中提供了一个`@sun.misc.Contended`注解，被此注解修饰会自动在缓冲行中进行填充，我们可以用JOL工具查看：

```java
public class ContendedDemo {

    public volatile  long testNoPadding;

    @Contended
    public volatile long testPadding;

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new ContendedDemo()).toPrintable());
    }

}
```

运行结果，注意运行时需要加JVM参数`-XX:-RestrictContended`

![image-20221105231810716](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221105231810716.png)

我们可以看到，再添加了注解变量我们发现其会在两个填充128字节的填充数据。但是虽然这可以解决为共享问题，但是也会占用了缓存，使用时需谨慎。

### 1.4 Disruptor中解决伪共享的手段

在disruptor框架中也是通过填充缓存行的方式解决此问题的。我们以Sequence类为例：

```java
class LhsPadding
{
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding
{
    protected volatile long value;
}

class RhsPadding extends Value
{
    protected long p9, p10, p11, p12, p13, p14, p15;
}

/**
 * <p>Concurrent sequence class used for tracking the progress of
 * the ring buffer and event processors.  Support a number
 * of concurrent operations including CAS and order writes.
 *
 * <p>Also attempts to be more efficient with regards to false
 * sharing by adding padding around the volatile field.
 */
public class Sequence extends RhsPadding
{
//代码略
}
```

在此类中在变量value两边都填充了7个long类型的值，这样能保证value变量所在的缓存行中只有自己，其余的全是填充字节，如下图（图片来自我的disruptor学习笔记）

![cachelineproblem](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/cachelineproblem.gif)



## 二、有序写入

### 2.1 JMM和内存屏障

这部分内容可以见我的[JAVA高并发编程学习笔记第三章4.5JMM](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%B8%89CAS%E5%92%8C%E5%8E%9F%E5%AD%90%E7%B1%BB%E4%BB%A5%E5%8F%8A%E6%9C%89%E5%BA%8F%E6%80%A7%E5%92%8C%E5%8F%AF%E8%A7%81%E6%80%A7.md#45-jmm)

### 2.2 Unsafe类的有序写入

在Disruptor中，对于序号的value属性的set方法中就是使用的有序写入：

```java
/**
 * Perform an ordered write of this sequence.  The intent is
 * a Store/Store barrier between this write and any previous
 * store.
 *
 * @param value The new value for the sequence.
 */
public void set(final long value)
{
    UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
}
```

其实它的原理十分简单，在上面方法的注解中也描述了，就是通过使用SS屏障通过消除可见性达到延迟写入的目的。因为我们的字段是volatile字段，因此根据JMM规范需要插入SS和SL屏障，但是SL屏障开销很大，可以理解为需要保证当前写入对所有CPU可见，所以`putOrderedLong`延迟写入方法只使用了SS屏障，这样可能会造成写后结果并不会被其他线程立即看到即取消了可见性，但是可以提升性能，且延迟一般在纳秒级别，因此此方法可以提升写入性能。

### 2.3 伪共享和有序写入的测试

我们可以进行一个对比测试：

**正常写入**

首先我们测试一下正常的写入，实体类和测试方法如下：

```java
public class EntityNoPadding implements Entity{

    private volatile long value= 1L;

    public void setValue(long value) {
        this.value = value;
    }
}
```

```java
@org.junit.Test
public void testNoPadding() throws InterruptedException {
    EntityNoPadding[] values = new EntityNoPadding[2];
    values[0] = new EntityNoPadding();
    values[1] = new EntityNoPadding();
    execute(values);
}
```

其中execute是为了方便封装的计算方法：

```java
public void execute(Entity [] values) throws InterruptedException {
    CountDownLatch latch = new CountDownLatch(2);
    Thread thread =new Thread(new Runnable() {
        @Override
        public void run() {
            for (long i = 0; i < 100_000_000L; i++) {
                values[0].setValue(i);
            }
            latch.countDown();
        }
    });
    Thread thread1 =new Thread(new Runnable() {
        @Override
        public void run() {
            for (long i = 0; i < 100_000_000L; i++) {
                values[1].setValue(i);
            }
            latch.countDown();
        }
    });
    long start = System.nanoTime();
    thread.start();
    thread1.start();
    latch.await();
    long end = System.nanoTime();
    System.out.println("it uses "+(end-start)/1_000_000+"ms");
}
```

多次实验结果：

it uses 1681ms

it uses 3528ms

it uses 1690ms

it uses 3569ms

it uses 1728ms

从实验结果看，平均在2432ms左右

**消除伪共享写入**

我们模仿disruptor进行缓存行填充：

```java
public class EntityWithPadding implements Entity{
    private volatile long p1,p2,p3,p4,p5,p6,p7;
    private volatile long value = 1L;
    private long p9, p10, p11, p12, p13, p14, p15;

    public void setValue(long value) {
        this.value = value;
    }
}
```

```java
@org.junit.Test
public void testPadding() throws InterruptedException {
    EntityWithPadding[] values = new EntityWithPadding[2];
    values[0] = new EntityWithPadding();
    values[1] = new EntityWithPadding();
    execute(values);
}
```

实验结果：

it uses 665ms

it uses 660ms

it uses 665ms

it uses 658ms

it uses 652ms

从实验结果看，平均在660ms左右

**有序写入**

我们还是模仿Disruptor进行有序写入：

```java
public class OrderedInsert implements Entity{

    private volatile long p1,p2,p3,p4,p5,p6,p7;
    private volatile long value = 1L;
    private long p9, p10, p11, p12, p13, p14, p15;

    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;

    static {
        UNSAFE = Util.getUnsafe();
        try {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(OrderedInsert.class.getDeclaredField("value"));
        } catch (final Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void setValue(long value) {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, value);
    }

}
```

```java
@org.junit.Test
public void testPaddingOrdered() throws InterruptedException {
    OrderedInsert[] values = new OrderedInsert[2];
    values[0] = new OrderedInsert();
    values[1] = new OrderedInsert();
    execute(values);
}
```

实验结果：

it uses 79ms

it uses 79ms

it uses 74ms

it uses 78ms

it uses 78ms

从实验结果看，平均在77.6ms左右

综上，可以看出消除伪共享和有序写入确实能大幅提升性能。

## 三、位运算代替数学运算

在Disruptor中计算ringbuffer元素位置取值时是通过位运算来计算取模的（源码如下），这样可以加快处理速度：

```java
protected final E elementAt(long sequence)
{
    return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
}
```

位运算取模原理如下：

> 位运算取模有公式：
>
> X % 2^ n = X & (2^ n – 1)
> 2^ n表示2的n次方，也就是说，一个数对2^ n取模 == 一个数和(2^n – 1)做按位与运算 。
> 假设n为3，则2^ 3 = 8，表示成2进制就是1000。2^3 = 7 ，即0111。
> 此时X & (2^3 – 1) 就相当于取X的2进制的最后三位数。
> 从2进制角度来看，X / 8相当于 X >> 3，即把X右移3位，此时得到了X / 8的商，而被移掉的部分(后三位)，则是X % 8，也就是余数。
>
> 用位运算的优点就是位运算(&)效率要比代替取模运算(%)高很多，主要原因是位运算直接对内存数据进行操作，不需要转成十进制，因此处理速度非常快。

## 四、空自旋优化

### 4.1 ThreadHints.onSpinWait()

在Disruptor空自旋的过程中，其调用了ThreadHints.onSpinWait();

```java
while ((availableSequence = dependentSequence.get()) < sequence)
{
    barrier.checkAlert();
    ThreadHints.onSpinWait();
}
```

其实底层是调用了java.lang.Thread#onSpinWait方法，源码如下（注意此方法是从jdk9开始提供的）：

```java
@IntrinsicCandidate
public static void onSpinWait() {}
```

那么为什么要调用这个方法呢？

此方法的好处一个是在某些CPU（比如Intel P4）下，CPU自旋会执行的很快，会消耗大量的电力，所以此方法可以省电，还有一个就是可以避免内存顺序违规，因为内存顺序违规会在退出循环时清空缓存行。

此方法其实本质上就是一个空方法，但是在某些架构的机器上比如x86环境下，Thread.onSpinWait在被调用一定次数后，C1编译器会将其替换为使用PAUSE这个x86指令去实现，而这个指令能提高自旋等待循环的性能，原因可见[PAUSE](https://www.felixcloutier.com/x86/pause)。它会告诉处理器，执行的代码是一个不断检查状态为是否就绪的代码，这样的话CPU就会给根据这个提示避开内存序列冲突，CPU就不会将这块读取的内存进行缓存，提高执行效率。

> 如果你的CPU支持PAUSE指令，且jdk在9以后，我们也可以用此方法去优化CAS空自旋。

### 4.2 Thread.sleep(0)和Thread.yield

在disruptor中等待策略中还使用了线程让步作为等待策略。我们这里主要看一下Thread.yield和另一个与他类似的操作Thread.sleep(0)的区别。

Thread.yield()和Thread.sleep(0)语义实现取决于具体的jvm虚拟机，某些jvm可能什么都不做，而大多数虚拟机会让线程放弃剩余的cpu时间片，重新变为runnable状态，并放到同优先级线程队列的末尾等待cpu资源。 但是当我们调用Thread.yield()的那一刻，并不意味着当前线程立马释放cpu资源，这是因为获得时间片的线程从runable切换到running仍需要一定的准备时间，这段时间当前线程仍可能运行一小段时间，且Yield方法暂时暂停当前执行的线程，以便给其余相同优先级的等待线程一个执行的机会。如果没有等待的线程或者所有等待的线程的优先级都很低，那么当前线程将继续执行。

但是Thread.sleep(0)有副作用就是可能会更频繁的运行GC，或者说防止长时间的GC。[参考资料](https://www.cnblogs.com/rsapaper/p/16720725.html)

其实在RocketMQ中就使用Thread.sleep(0)阻止GC，本质是调用Thread.sleep(0)会触发操作系统立即进行一次CPU竞争，竞争的结果可能是GC获取CPU控制权，这样可能会更频繁地运行GC，但是这样可以防止单次GC时间过长，我们偶尔让GC运行一次其实比单次GC事件运行时间长效果更好。

## 五、线程核绑

核绑又叫CPU亲和性，一般为了防止缓存行清空和消息延迟抖动，我们可能会进行线程的CPU绑定。CPU 亲和性（affinity）就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。

CPU的亲和性又分为硬亲和性和软亲和性。**软亲和性（affinity）:**就是进程要在指定的 CPU 上尽量长时间地运行而不被迁移到其他处理器，Linux 内核进程调度器天生就具有被称为 软 CPU 亲和性（affinity） 的特性，这意味着进程通常不会在处理器之间频繁迁移。**硬亲和性（affinity）**简单来说就是利用linux内核提供给用户的API，强行将进程或者线程绑定到某一个指定的cpu核运行。

### 5.1 核绑的原因

我们先来看Disruptor的BusySpinWaitStrategy策略

```java
/**
 * Busy Spin strategy that uses a busy spin loop for {@link com.lmax.disruptor.EventProcessor}s waiting on a barrier.
 * <p>
 * This strategy will use CPU resource to avoid syscalls which can introduce latency jitter.  It is best
 * used when threads can be bound to specific CPU cores.
 */
public final class BusySpinWaitStrategy implements WaitStrategy{//代码略}
```

我们也可看到注释上，此策略避免延迟抖动让我们最好将线程绑定到特定的CPU。

首先我们来看一下跨核调度会导致什么问题？

1. 首先，跨核调度会导致CPU的缓存行的丢失，因为线程1在CPU Core1中已经将数据加载进缓存行了，但这是如果需要在其他CPU核心去执行，那么就会导致缓存行丢失。因此为了避免线程跨核调度上下文切换导致的缓存丢失，提升核心缓存行的命中率，我们可以进行核绑。
2. 核绑会避免跨核调用导致的处理器的延迟抖动。我们可以简单理解，比如我们正常的的消息是在每一秒中都会传过来的，但是这时跨核调用要占用时间所以就会造成延时，而在低延时的系统中就会造成影响。

其实很多情况下，为了提高性能，Linux会自动的尽量将进程或线程绑定到固定的CPU上去执行（即软亲和性），所以非必须的话是没必要显式的绑定CPU的。

在 Linux 内核中，所有的进程都有一个相关的数据结构，称为 task_struct 。这个结构非常重要，原因有很多；其中与 亲和性（affinity）相关度最高的是cpus_allowed 位掩码。这个位掩码由n位组成，与系统中的n个逻辑处理器一一对应。 具有 4 个物理 CPU 的系统可以有 4 位。如果这些 CPU 都启用了超线程，那么这个系统就有一个 8 位的位掩码。

如果为给定的进程设置了给定的位，那么这个进程就可以在相关的 CPU 上运行。因此，如果一个进程可以在任何 CPU 上运行，并且能够根据需要在处理器之间进行迁移，那么位掩码就全是 1。实际上，这就是 Linux 中进程的缺省状态。

Linux 内核 API 提供了一些方法，让用户可以修改位掩码或查看当前的位掩码（也就是绑核）：

- `sched_set_affinity()` （用来修改位掩码）
- `sched_get_affinity()` （用来查看当前的位掩码）

那么如何设置硬亲和性呢？

设置用户态进程与CPU绑定：

`int sched_setaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);`

设置用户态线程与CPU绑定：

- `int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize,const cpu_set_t *cpuset);`
- ``int pthread_getaffinity_np(pthread_t thread, size_t cpusetsize,cpu_set_t *cpuset);`

### 5.2 Java如何绑定CPU

> Java中的线程模型是一比一的。
>
> 进程控制块PCB在Linux中对应的是一个结构体`task_struct`，内核并没有线程的概念. 每一个线程在内核中都存在一个进程描述符`task_struct`对应, 所以线程也叫作轻量级进程Light Weighted Process。

在Java中可以使用开源类库`Java-Thread-Affinity`来进行Java的CPU亲和性的设置，[官方GitHub](https://github.com/OpenHFT/Java-Thread-Affinity)。这个类库底层是用JNA Java Native Access (Java 本地访问)实现的。

在学习之前我们可以使用lscpu查看系统的CPU的情况：

![image-20221106150038868](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221106150038868.png)

> Architecture:         #架构
> CPU(s):                 #逻辑cpu颗数
> Thread(s) per core:     #每个核心线程
> Core(s) per socket:     #每个cpu插槽核数/每颗物理cpu核数
> CPU socket(s):          #cpu插槽数
> Vendor ID:              #cpu厂商ID
> CPU family:             #cpu系列
> Model:                  #型号
> Stepping:               #步进
> CPU MHz:                #cpu主频
> Virtualization:         #cpu支持的虚拟化技术
> L1d cache:              #一级缓存（google了下，这具体表示表示cpu的L1数据缓存）
> L1i cache:              #一级缓存（具体为L1指令缓存）
> L2 cache:               #二级缓存

- socket就是主板上的CPU插槽的数量，就是可插入的物理CPu的个数
- core就是平时说的核，每个物理CPU可以双核、四核等。
- threads就是每个core的硬件线程数

> 几核几线程就是指有多少个“Core per Socket”和多少个“Thread per Core”,当后者比前者多时，说明启用了超线程（虚拟线程）技术

在Java-Thread-Affinity中有一个CpuLayout与之相对应，源码如下：

```java
public interface CpuLayout {
    int cpus();

    int sockets();

    int coresPerSocket();

    int threadsPerCore();

    int socketId(int var1);

    int coreId(int var1);

    int threadId(int var1);
}
```

下面我们来介绍一下这个类库的使用方式：

先导入依赖：

~~~xml
<dependency>
    <groupId>net.openhft</groupId>
    <artifactId>affinity</artifactId>
    <version>3.23.2</version>
</dependency>
~~~

这个类库首先是获得一个CPU的锁：

```java
AffinityLock affinityLock = AffinityLock.acquireLock();
try {
    //do Something locked a cpu
}finally {
    affinityLock.release();
}
//或者Java7之后这样写
try(AffinityLock aLock = AffinityLock.acquireLock();) {
    
}
```

然后我们来测试一下：

```java
public class TestCpu {
    public static void main(String[] args) {
        try(AffinityLock aLock = AffinityLock.acquireLock(3);) {
            //空自旋
            while (true){

            }
        }
    }
}
```

然后观察任务管理器发现CPU3很快就被打满了。

![image-20221106150905120](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221106150905120.png)

如果是线程池应该怎么绑定呢？

在Java-Thread-Affinity中提供了一个线程工厂，我们在使用线程池时，可以使用线程工厂去设置线程池中的线程的CPU亲和性，例子如下：

```java
public static void main(String[] args) throws InterruptedException {
    //设置CPU亲和性的线程工厂
    AffinityThreadFactory factory = new AffinityThreadFactory("affinityThread");
    ExecutorService executorService = Executors.newFixedThreadPool(4,factory);
    for (int i = 0; i < 12; i++) {
        executorService.submit(new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                Thread.sleep(100);
                return null;
            }
        });
    }
    Thread.sleep(200);
    executorService.shutdown();
    executorService.awaitTermination(1, TimeUnit.SECONDS);
}
```

### 5.3 Disruptor设置CPU亲和性

由于Java-Thread-Affinity提供了线程工厂，因此我们只需要在Disruptor创建时将核绑的线程工厂传入即可，代码如下：

```java
AffinityThreadFactory factory = new AffinityThreadFactory("affinityThread");
Disruptor<LongEvent> disruptor = new Disruptor<>(LongEvent::new, bufferSize, factory, ProducerType.SINGLE, new BusySpinWaitStrategy());
```

### 5.4 Nginx设置CPU亲和性

nginx官方文档：[affinity](http://nginx.org/en/docs/ngx_core_module.html#worker_cpu_affinity)

首先使用以下命令查看自己的linux系统的CPU核数

`cat /proc/cpuinfo | grep 'processor' |wc -l`

然后编辑配置文件，具体需要根据核数进行修改：

~~~nginx
worker_processes 2;
worker_cpu_affinity 01 10;
#worker_processes 4;
#worker_cpu_affinity 0001 0010 0100 1000;
#worker_processes 8;
#worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
~~~

