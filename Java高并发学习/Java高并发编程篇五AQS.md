# Java高并发编程篇五AQS

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及道格李的AQS论文。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是对AQS进行研究学习。

> 解决CAS恶性空自旋的有效方式之一是以空间换时间，较为常见的方案有两种：分散操作热点和使用队列削峰。JUC并发包使用的是队列削峰的方案解决CAS的性能问题，并提供了一个基于双向队列的削峰基类——抽象基础类AbstractQueuedSynchronizer（抽象同步器类，简称为AQS）。

# 六、AQS抽象同步器的核心原理

AQS建立在CAS原子操作和volatile可见性变量的基础之上，为上层的显式锁、同步工具类、阻塞队列、线程池、并发容器、Future异步工具提供线程之间同步的基础设施。所以，AQS在JUC框架中的使用是非常广泛的。

## 6.1 队列和锁的关系

无论是单体服务应用内部的锁，还是分布式环境下多体服务应用所使用的分布式锁，为了减少由于无效争夺导致的资源浪费和性能恶化，一般都基于队列进行排队与削峰。

**CLH锁的内部队列**

CLH自旋锁使用的CLH（Craig,Landin,and Hagersten Lock Queue）是一个单向队列，也是一个FIFO队列。在独占锁中，竞争资源在一个时间点只能被一个线程锁访问，队列头部的节点表示占有锁的节点，新加入的抢锁线程则需要等待，会插入队列的尾部。结构如下（图片来自于原书Java高并发编程卷2）：

![image-20220427124929931](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427124929931.png)

**分布式锁的内部队列**

在分布式锁的实现中，比较常见的是基于队列的方式进行不同节点中“等锁线程”的统一调度和管理。以基于ZooKeeper的分布式锁为例，其等待队列的结构大致如图所示。（图片来自于原书Java高并发编程卷2）

![image-20220427125008626](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427125008626.png)

**AQS的内部队列**

AQS是JUC提供的一个用于构建锁和同步容器的基础类。JUC包内许多类都是基于AQS构建的，例如ReentrantLock、Semaphore、CountDownLatch、ReentrantReadWriteLock、FutureTask等。AQS解决了在实现同步容器时设计的大量细节问题。AQS是CLH队列的一个变种，主要原理和CLH队列差不多。AQS队列内部维护的是一个FIFO的双向链表，这种结构的特点是每个数据结构都有两个指针，分别指向直接的前驱节点和直接的后继节点。所以双向链表可以从任意一个节点开始很方便地访问前驱节点和后继节点。每个节点其实是由线程封装的，当线程争抢锁失败后会封装成节点加入AQS队列中；当获取锁的线程释放锁以后，会从队列中唤醒一个阻塞的节点（线程）。内部结构如下（图片来自于原书Java高并发编程卷2）：

![image-20220427125101458](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427125101458.png)

## 6.2 AQS内部成员

AQS出于“分离变与不变”的原则，基于模板模式实现。AQS为锁获取、锁释放的排队和出队过程提供了一系列的模板方法。由于JUC的显式锁种类丰富，因此AQS将不同锁的具体操作抽取为钩子方法，供各种锁的子类（或者其内部类）去实现。

AbstractQueuedSynchronizer继承了AbstractOwnableSynchronizer，这个基类只有一个变量叫exclusiveOwnerThread，表示当前占用该锁的线程，并且提供了相应的get()和set()方法，具体如下：

![image-20220427130423812](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427130423812.png)

源码如下：

```java
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}

protected final Thread getExclusiveOwnerThread() {
    return exclusiveOwnerThread;
}
```

下面我们来学习一下AQS的内部成员

### 6.2.1 状态标志位

AQS中维持了一个单一的volatile修饰的状态信息state，AQS使用int类型的state标示锁的状态，可以理解为锁的同步状态。源码如下：

```java
/**
 * The synchronization state.同步状态。
 */
private volatile int state;
```

state因为使用volatile保证了操作的可见性，所以任何线程通过getState()获得状态都可以得到最新值。AQS提供了getState()、setState()来获取和设置同步状态，源码如下：

```java
/**
 * Returns the current value of synchronization state.返回同步状态的当前值。
 * This operation has memory semantics of a {@code volatile} read.
 * @return current state value
 */
protected final int getState() {
    return state;
}

/**
 * Sets the value of synchronization state.设置同步状态的值。
 * This operation has memory semantics of a {@code volatile} write.
 * @param newState the new state value
 */
protected final void setState(int newState) {
    state = newState;
}

/**
* 由于setState()无法保证原子性，因此AQS给我们提供了compareAndSetState()方法利用底层UnSafe的CAS机制来实现原子性。
* 
* Atomically sets synchronization state to the given updated
* value if the current state value equals the expected value.
* This operation has memory semantics of a {@code volatile} read
* and write.
* 翻译：如果当前状态值等于预期值，则自动将同步状态设置为给定的更新值。该操作具有volatile读写的内存语义。
*
*/
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

> 以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程执行该锁的lock()操作时，会调用tryAcquire()独占该锁并将state加1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（释放锁）为止，其他线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。流程如下：
>
> ![image-20220427131703783](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220427131703783.png)

### 6.2.2 队列节点类

AQS是一个虚拟队列，不存在队列实例，仅存在节点之间的前后关系。节点类型通过内部类Node定义，其核心的成员源码如下：

```java
static final class Node {
    //标记节点是独占模式还是共享模式
    
    /** Marker to indicate a node is waiting in shared mode */
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    static final Node EXCLUSIVE = null;

    /** 节点等待状态值：取消状态 waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** 节点等待状态值：表示后继线程处于等待状态 waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** 节点等待状态值：表示当前线程正在进行条件等待 waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * 节点等待状态值：表示下一次共享锁的acquireShared操作需要无条件传播
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int CONDITION = -3;

    /**
     *（文档略）
     * 节点状态，值为：SIGNAL、CANCELLED、CONDITION、CONDITION、0
     * 普通节点初始值为0，条件等待节点初始值为CONDITION（-2）
     */
    volatile int waitStatus;

    /**
     * 前驱结点（文档略）
     */
    volatile Node prev;

    /**
     * 后继结点（文档略）
     */
    volatile Node next;

    /**
     * 节点对应线程，为抢锁线程或条件等待线程（文档略）
     */
    volatile Thread thread;

    /**
     * 当前node不是普通节点，而是条件等待节点，则表示节点正处于某个条件等待队列上
     * 此属性指向下一个条件等待节点，即条件队列上的后继结点（文档略）
     */
    Node nextWaiter;
    //成员方法略
}
```

1. 抢占类型常量标识

   SHARED表示线程是因为获取共享资源时阻塞而被添加到队列中的；EXCLUSIVE表示线程是因为获取独占资源时阻塞而被添加到队列中的。

2. thread成员变量

   Node的thread成员用来存放进入AQS队列中的线程引用；Node的nextWaiter成员用来指向自己的后继等待节点，此成员只有线程处于条件等待队列中的时候使用。

3. waitStatus属性

   每个节点与等待线程关联，每个节点维护一个状态waitStatus。

   - CANCELLED=1

     waitStatus值为1时表示该线程节点已释放（超时、中断），已取消的节点不会再阻塞，表示线程因为中断或者等待超时，需要从等待队列中取消等待。由于该节点线程等待超时或者被中断，需要从同步队列中取消等待，因此该线程被置1。节点进入了取消状态，该类型节点不会参与竞争，且会一直保持取消状态。

   - SIGNAL=-1

     waitStatus为SIGNAL时表示其后继的节点处于等待状态，当前节点对应的线程如果释放了同步状态或者被取消，就会通知后继节点，使后继节点的线程得以运行。

   - CONDITION=-2

     waitStatus为CONDITION时，表示该线程在条件队列中阻塞（Condition有使用），表示节点在等待队列中（这里指的是等待在某个锁的CONDITION上），当持有锁的线程调用了CONDITION的signal()方法之后，节点会从该CONDITION的等待队列转移到该锁的同步队列上，去竞争锁（注意：这里的同步队列就是我们讲的AQS维护的FIFO队列，等待队列则是每个CONDITION关联的队列）。节点处于等待队列中，节点线程等待在CONDITION上，当其他线程对CONDITION调用了signal()方法后，该节点从等待队列中转移到同步队列中，加入对同步状态的获取中。
   
   - PROPAGATE=-3
   
     waitStatus为－3时，表示下一个线程获取共享锁后，自己的共享状态会被无条件地传播下去，因为共享锁可能出现同时有N个锁可以用，这时直接让后面的N个节点都来工作。这种状态在CountDownLatch中使用到了。
   
     为什么当一个节点的线程获取共享锁后，要唤醒后继共享节点？共享锁是可以多个线程共有的，当一个节点的线程获取共享锁后，必然要通知后继共享节点的线程也可以获取锁了，这样就不会让其他等待的线程等很久，这种向后通知（传播）的目的也是尽快通知其他等待的线程尽快获取锁。
   
   - waitStatus为0
   
     waitStatus为0时，表示当前节点处于初始状态。

### 6.2.3 AQS和JUC的关系

AQS是java.util.concurrent包的一个同步器，它实现了锁的基本抽象功能，支持独占锁与共享锁两种方式。该类是使用模板模式来实现的，成为构建锁和同步器的框架，使用该类可以简单且高效地构造出应用广泛的同步器（或者等待队列）。java.util.concurrent.locks包中的显式锁如ReentrantLock、ReentrantReadWriteLock，线程同步工具如Semaphore，异步回调工具如FutureTask等，内部都使用了AQS作为等待队列。通过IDEA进行AQS的子类导航会发现大量的AQS子类以内部类的形式使用，如下图：

![image-20220428124922506](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220428124922506.png)

### 6.2.4 AQS与ReentranLock的关系

下面我们以ReentranLock为例，来了解二者的关系。

ReentrantLock是一个可重入的互斥锁，又称为“可重入独占锁”。ReentrantLock锁在同一个时间点只能被一个线程锁持有，而可重入的意思是，ReentrantLock锁可以被单个线程多次获取。

实际上ReentrantLock把所有Lock接口的操作都委派到一个Sync类上，

比如说我们以lock方法和unlock方法源码举例：

```java
public void lock() {
    sync.lock();
}
public void unlock() {
    sync.release(1);
}
```

Sync类继承了AbstractQueuedSynchronizer，简略源码如下：

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

ReentrantLock为了支持公平锁和非公平锁两种模式，为Sync又定义了两个子类，简略源码如下：

```java
static final class NonfairSync extends Sync
static final class FairSync extends Sync
```

NonfairSync为非公平（或者不公平）同步器，FairSync为公平同步器，默认是不公平的，ReentrantLock构造器源码如下：

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

由于ReentrantLock的显式锁操作是委托（或委派）给一个Sync内部类的实例来完成的。而Sync内部类只是AQS的一个子类，所以本质上ReentrantLock的显式锁操作是委托（或委派）给AQS完成的。类图如下：

![aqsreentranlock20220428](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/aqsreentranlock20220428.png)

聚合关系的特点是，整体由部分构成，但是整体和部分之间并不是强依赖的关系，而是弱依赖的关系，也就是说，即使整体不存在了，部分仍然存在。例如，一个部门由多个员工组成，如果部门撤销了，人员不会消失，人员依然存在。

组合关系的特点是，整体由部分构成，但是整体和部分之间是强依赖的关系，如果整体不存在了，部分也随之消失。例如，一个公司由多个部门组成，如果公司不存在了，部门也将不存在。可以说，组合关系是一种强依赖的、特殊的聚合关系。

由于显式锁与AQS之间是一种强依赖的关系，如果显式锁的实例销毁，其聚合的AQS子类实例也会被销毁，因此显式锁与AQS之间是组合关系。

## 6.3 AQS的模板模式

AQS同步器是基于模板模式设计的，并且是模板模式很经典的一个运用。关于模板模式可以看我的[设计模式的笔记](https://github.com/Loserfromlazy/Code_Career/blob/master/java/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.md)关于模板模式的部分

> （这篇笔记除了模板模式以外的其他部分可能不是很全，后续会陆续的补全，但是模板模式是全的）

### 6.3.1 AQS的模板流程

AQS定义了两种资源共享方式：

- Exclusive（独享锁）：只有一个线程能占有锁资源，如ReentrantLock。独享锁又可分为公平锁和非公平锁。
- Share（共享锁）：多个线程可同时占有锁资源，如Semaphore、CountDownLatch、CyclicBarrier、ReadWriteLock的Read锁。

AQS为不同的资源共享方式提供了不同的模板流程，包括共享锁、独享锁模板流程。这些模板流程完成了具体线程进出等待队列的（如获取资源失败入队/唤醒出队等）基础、通用逻辑。基于基础、通用逻辑，AQS提供了一种实现阻塞锁和依赖FIFO等待队列的同步器的框架，AQS模板为ReentedLocK、CountDownLatch、Semaphore提供了优秀的解决方案。

自定义的同步器只需要**实现共享资源state的获取与释放方式**即可，这些逻辑都编写在钩子方法中。无论是共享锁还是独享锁，AQS在执行模板流程时都会回调自定义的钩子方法。

### 6.3.2 AQS的钩子方法



自定义同步器时，AQS中需要重写的钩子方法大致如下：

- tryAcquire(int)：独占锁钩子，尝试获取资源，若成功则返回true，若失败则返回false。
- tryRelease(int)：独占锁钩子，尝试释放资源，若成功则返回true，若失败则返回false。
- tryAcquireShared(int)：共享锁钩子，尝试获取资源，负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享锁钩子，尝试释放资源，若成功则返回true，若失败则返回false。
- isHeldExclusively()：独占锁钩子，判断该线程是否正在独占资源。只有用到condition条件队列时才需要去实现它。

以上钩子方法的默认实现会抛出UnsupportedOperationException异常。除了这些钩子方法外，AQS类中的其他方法都是final类型的方法，所以无法被其他类继承，只有这几个方法可以被其他类继承。

我们拿ReentranLock的内部类Sync的钩子方法实现来简单看一下：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;//获取当前重入的锁的层数，减去需要释放的层数
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {//判断是否当前线程的所有锁都释放了
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);//设置释放后的状态
    return free;
}
```

可以看到我们在实现钩子方法时，只需要对state的获取和释放方式进行实现即可

### 6.3.4 根据AQS自定义一把锁

我们下面模拟ReentrantLock的源码，基于AQS实现一把非常简单的独占锁。

下面来看代码，首先我们定义一个SimpleLock继承Lock接口（我们这里仅实现lock和unlock方法）

```java
public class SimpleLock implements Lock {
    @Override
    public void lock() {
        
    }

    @Override
    public void unlock() {

    }
	//其余不实现Lock接口的方法这里就略掉了
    //...
}
```

然后我们实现一个内部类Sync继承AQS

```java
public class SimpleLock implements Lock {
    private Sync sync = new Sync();

    static class Sync extends AbstractQueuedSynchronizer{
        //我们这里仅实现两个钩子方法
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0,1)){
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (Thread.currentThread()!= getExclusiveOwnerThread()){
                throw new IllegalMonitorStateException();
            }
            if (getState()==0){
                throw new IllegalMonitorStateException();
            }
            //下面操作不存在并发
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
    }
    @Override
    public void lock() {
        //委托给AQS
        sync.acquire(1);
    }

    @Override
    public void unlock() {
        //委托给AQS
        sync.release(1);
    }
   //其余不实现Lock接口的方法这里就略掉了
    //...
}
```

从代码可以看到我们的锁很简单，且不支持可重入。

SimpleLock的锁抢占和锁释放是委托给Sync实例的acquire()方法和release()方法来完成的。

SimpleLock的内部类Sync继承了AQS类，实际上acquire()、release()是AQS的两个模板方法。在抢占锁时，AQS的模板方法acquire()会调用tryAcquire(int arg)钩子方法；在释放锁时，AQS的模板方法release()会调用tryRelease(int arg)钩子方法。

我们搞个之前经常用的自增案例来测试我们的锁：

```java
public class TestAQS {
    public static void main(String[] args) {
        final int TURNS = 100000;
        final int THREADS = 10;

        ExecutorService service = Executors.newFixedThreadPool(THREADS);
        SimpleLock lock = new SimpleLock();
        CountDownLatch latch = new CountDownLatch(THREADS);
        long start = System.currentTimeMillis();
        for (int i = 0; i < THREADS; i++) {
            service.submit(() -> {
                for (int j = 0; j < TURNS; j++) {
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

运行结果：

运行时长0.065
结果1000000

## 6.4 AQS的抢锁原理

### 6.4.1 抢锁的原理

Java并发的作者道格李专门除了一篇[论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)来阐述关于JavaAQS，我们这里根据AQS的部分原文，来简单了解一下AQS抢锁的原理：

首先Lea介绍了CLH锁的优点，比如CLH锁入队出队非常快，且是无阻塞的，在判断是否有线程等待时也只需要判断首尾节点是否相同。而且通过显式地在节点中维护前继节点字段，CLH锁可以处理超时和其他形式的取消:如果节点的前任取消了，节点可以向上滑到使用前一个节点的状态字段。关于CLH的介绍，原文图片如下：

![image-20220429111628900](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220429111628900.png)

所以Lea最终选择了CLH作为AQS同步阻塞器的实现基础（这也是上一篇简单实现CLH锁的原因）。然后其对变动进行了说明。

首先就是CLH锁没有后继节点，因为自旋锁只需要改变自己的状态，但是AQS阻塞队列需要显式的唤醒后继节点。

其次AQS阻塞队列是通过保存在每一个节点的状态来控制线程的阻塞，而不是像CLH中那样通过普通自旋来完成。而且队列节点状态字段也用于避免不必要的park and unpark操作。在调用park阻塞当前线程之前会设置一个通知状态（也就是SIGNAL），然后再次进行检查。这么做可以避免线程不必要地频繁阻塞，特别是对于锁相关的类，在这些锁类中，等待下一个符合条件的线程获得锁的时间会加剧竞争。

AQS的主要特点就是利用GC来管理节点的存储回收，从而避免了复杂性和开销。基本acquire操作(独占的、不可中断的、不定时的情况)的结果实现的一般形式如下图:

![image-20220429112817123](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220429112817123.png)

释放操作如下图：

![image-20220429112850242](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220429112850242.png)











我们下面根据上面自定义SimpleLock来分析一下AQS的抢锁原理。在SimpleLock中，加锁是委托给AQS的，代码如下：

```java
@Override
public void lock() {
    //委托给AQS
    sync.acquire(1);
}
```

### 6.4.2 模板方法acquire()

我们跟进acquire方法，会调用AQS的模板方法：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### 6.4.3 钩子实现tryAcquire()

在模板方法中首先执行一次钩子方法，而钩子方法是锁实现的，所以会执行SimpleLock的tryAcquire：

```java
@Override
protected boolean tryAcquire(int arg) {
    if (compareAndSetState(0,1)){
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

> SimpleLock的实现非常简单，是不可以重入的，仅仅为了学习AQS而编写。

### 6.4.4 直接入队addWaiter

SimpleLock的tryAcquire逻辑不再赘述，然后在acquire模板方法中，如果钩子方法tryAcquire尝试获取同步状态失败的话，就构造同步节点（独占式节点模式为Node.EXCLUSIVE），通过addWaiter(Node node,int args)方法将该节点加入同步队列的队尾。addWaiter方法源码如下：

```java
private Node addWaiter(Node mode) {
    //创建新节点，将当前线程存入节点，并表示此节点是共享的还是独占的
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //尝试将目前队列的末尾作为自己的前驱结点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //尝试CAS方式修改队尾节点为当前最新建的节点
        if (compareAndSetTail(pred, node)) {
            //如果修改成功就将新节点加到队列的尾部
            pred.next = node;
            return node;
        }
    }
    //尝试加入队列尾部失败，自旋
    enq(node);
    return node;
}
```

### 6.4.5 自旋入队enq

addWaiter()第一次尝试在尾部添加节点失败，意味着有并发抢锁发生，需要进行自旋。enq()方法通过CAS自旋将节点添加到队列尾部。enq方法源码如下：

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //t==null表示当前队列为空，需要初始化
        if (t == null) { // Must initialize
            //将头尾节点都初始化成新节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //当前队列不为空，新节点插入队列尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

### 6.4.6 自旋抢锁acquireQueued

addWaiter这个方法我们已经了解了，下面我们回到模板方法acquire(),在addWaiter将节点入队之后，启动自旋抢锁的流程（也就是acquireQueued方法）。

**acquireQueued()方法的主要逻辑**：当前Node节点线程在死循环中不断获取同步状态，并且**不断在前驱节点上自旋**，只有当前驱节点是头节点时才能尝试获取锁，原因是：

- 头节点是成功获取同步状态（锁）的节点，而头节点的线程释放了同步状态以后，将会唤醒其后继节点，后继节点的线程被唤醒后要检查自己的前驱节点是否为头节点。
- 维护同步队列的FIFO原则，节点进入同步队列之后，就进入了自旋的过程，每个节点都在不断地执行for死循环。



我们现在看一下acquireQueued方法源码：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        //自旋不断检查当前节点的前驱节点是否是头节点
        for (;;) {
            //获取节点的前驱节点
            final Node p = node.predecessor();
            //如果前驱节点是头节点，那么就执行tryAcquire进行抢锁
            if (p == head && tryAcquire(arg)) {
                //抢锁成功后将当前节点设置为头节点，移除之前的头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //检查前一个节点的状态，预判当前线程获取锁失败是否要挂起
            //如果需要挂起就执行parkAndCheckInterrupt挂起当前线程，直到被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//两个操作都为true，就设置为true
        }
    } finally {
        //如果等待过程中没有成功获取到资源（如timeout，或者可中断的情况下被中断），那么就取消节点在队列的等待
        //由于超时或中断而被取消的线程会设置它的节点状态并取消它的后继节点，这样它就可以重置链接
        if (failed)
            //取消请求，将当前节点从队列中移除
            cancelAcquire(node);
    }
}
```

此方法的关键点：

- acquireQueued()自旋过程中会阻塞线程，等待被前驱节点唤醒后才启动循环。如果成功就返回，否则执行shouldParkAfterFailedAcquire()、parkAndCheckInterrupt()来达到阻塞的效果。
- 调用acquireQueued()方法的线程一定是node所绑定的线程（由它的thread属性所引用），该线程也是最开始调用lock()方法抢锁的那个线程，在acquireQueued()的死循环中，该线程可能重复进行阻塞和被唤醒。
- AQS队列上每一个节点所绑定的线程在抢锁的过程中都会自旋执行acquireQueued()方法的死循环，也就是说，AQS队列上每个节点的线程都不断自旋
- 如果头节点获取了锁，那么该节点绑定的线程会终止acquireQueued()自旋，线程会去执行临界区代码。此时，其余的节点处于自旋状态，处于自旋状态的线程当然也不会执行无效的空循环而导致CPU资源浪费，而是被挂起（Park）进入阻塞状态。AQS队列的节点自旋不像CLH节点那样在空自旋而耗费资源。

我们下面来分析一下此方法中的shouldParkAfterFailedAcquire()、parkAndCheckInterrupt()

### 6.4.7 挂起预判shouldParkAfterFailedAcquire

acquireQueued()自旋在阻塞自己的线程之前会进行挂起预判。shouldParkAfterFailedAcquire()方法的主要功能是：将当前节点的有效前驱节点（是指有效节点不是CANCELLED类型的节点）找到，并且将有效前驱节点的状态设置为SIGNAL，如果返回true代表当前线程可以马上被阻塞了。具体可以分为三种情况，见代码。

我们看一下源码：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //pred是传入的前驱结点，node是传入的当前节点
    int ws = pred.waitStatus;//获取前驱节点的状态
    //如果是SIGNAL就直接返回
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    //如果是CANCELLED，说明前驱节点已经取消
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            //不断循环，找到有效的前驱节点（就是状态不是CANCELLED的节点）
            node.prev = pred = pred.prev;//这句话的意思就是将node的前驱变成pred的前驱
            //如果不理解可以将上面这句话分解，分解成这样
            //pred = pred.prev;//将pred变成前驱的前驱，就是pred的前驱，node的前前驱
            //node.prev = pred;//当前指针指向pred的前驱
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        //如果前驱状态不是CANCELLED也不是SIGNAL（PROPAGATE共享锁等待、CONDITION条件等待、0（初始状态））那么就设置成SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        //设置成SIGNAL此方法返回false，表示线程不可用，被阻塞
    }
    return false;
}
```

在独占锁的场景中，shouldParkAfterFailedAcquire()方法是在acquireQueued()方法的死循环中被调用的，由于此方法返回false时acquireQueued()不会阻塞当前线程，只有此方法返回true时当前线程才阻塞，**因此在一般情况下，此方法至少需要执行两次，当前线程才会被阻塞**。

> 在第一次进入此方法时，首先会进入后一个if判断的else分支，通过CAS设置pred前驱的waitStatus为SIGNAL，然后返回false。此方法返回false之后，获取独占锁的acquireQueued()方法会继续进行for循环去抢锁：（1）假设node的前驱节点是头节点，tryAcquire()抢锁成功，则获取到锁。（2）假设node的前驱节点不是头节点，或者tryAcquire()抢锁失败，仍然会再次调用此方法。第二次进入此方法时，由于上一次进入时已经将pred.waitStatus设置为?1（SIGNAL）了，因此这次会进入第一个判断条件，直接返回true，表示应该调用parkAndCheckInterrupt()阻塞当前线程，等待前一个节点执行完成之后唤醒。

**什么时候遇到前驱节点状态waitStatus等于0（初始状态）的场景呢**？分为两种情况：

1. node节点刚成为新队尾，但还没有将旧队尾的状态设置为SIGNAL。
2. node节点的前驱节点为head。

前驱节点为waitStatus等于0的情况是最常见的。比如现在AQS的等待队列中有很多节点正在等待，当前线程刚执行完毕addWaiter()（节点刚成为新队尾），然后现在开始执行获取锁的死循环（独占锁对应的是acquireQueued()里的死循环，共享锁对应的是doAcquireShared()里的死循环），此时节点的前驱（也就是旧队尾的状态）肯定还是0（也就是默认初始化的值），然后死循环执行两次，第一次执行shouldParkAfterFailedAcquire()自然会检测到前驱状态为0，然后将0设置为SIGNAL，第二次执行shouldParkAfterFailedAcquire()，由于前驱节点为SIGNAL，当前线程直接返回true，去执行自我阻塞。

**什么时候会遇到前驱节点的状态waitStatus大于0的场景呢**？当pred前驱节点的抢锁请求被取消，后期状态为CANCELLED（值为1）时，当前节点（如果被唤醒）就会循环移除所有被取消的前驱节点，直到找到未被取消的前驱。在移除所有被取消的前驱节点后，此方法将返回false，再次去执行acquireQueued()的自旋抢占。

**什么时候遇到前驱节点状态waitStatus等于-3（PROPAGATE）的场景呢**？PROPAGATE只能在使用共享锁的时候出现，并且只可能设置在head上。所以，对于非队尾节点，如果它的状态为0或PROPAGATE，那么它肯定是head。当等待队列中有多个节点时，如果head的状态为0或PROPAGATE，说明head处于一种中间状态，且此时有线程刚才释放锁了。而对于抢锁线程来说，如果检测到这种状态，说明再次执行acquire()方法是极有可能获得锁的。

### 6.4.8 线程挂起parkAndCheckInterrupt 

parkAndCheckInterrupt()的主要任务是暂停当前线程：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//阻塞当前线程
    return Thread.interrupted();//如果被唤醒看看当前线程是否被中断
}
```

AQS会把所有的等待线程构成一个阻塞等待队列，当一个线程执行完lock.unlock()时，会激活其后继节点，通过调用LockSupport.unpark(postThread)完成后继线程的唤醒。

### 6.4.9 取消请求cancelAcquire

> 关于取消请求，Doug Lea的论文中对其介绍如下
>
> Cancellation support mainly entails checking for interrupt or timeout upon each return from park inside the acquire loop. A cancelled thread due to timeout or interrupt sets its node status and unparks its successor so it may reset links. With cancellation, determining predecessors and successors and resetting status may include O(n) traversals (where n is the length of the queue). Because a thread never again blocks for a cancelled operation, links and status fields tend to restabilize quickly.
>
> 

```java
private void cancelAcquire(Node node) {
    // 当前节点不存在忽略
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    Node pred = node.prev;
    //如果前驱状态为取消CANCELLED
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;//将node的前驱变成pred的前驱，可以见6.5.7中的注释

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

## 6.5 AQS释放锁原理

根据上面自定义SimpleLock来分析一下AQS的锁释放的原理。

### 6.5.1 AQS模板方法 release()

SimpleMockLock的unlock()方法被调用时，会调用AQS的release(…)的模板方法。AQS的release(…)的模板方法代码如下：

```java
public final boolean release(int arg) {
    //如果同步状态的钩子方法执行成功
    if (tryRelease(arg)) {
        Node h = head;
        //当head指向的头节点不为null，并且该节点的状态值不为0时，执行unparkSuccessor()方法
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

### 6.5.2 AQS 钩子实现 tryRelease()

~~~java
 @Override
protected boolean tryRelease(int arg) {
    if (Thread.currentThread()!= getExclusiveOwnerThread()){
        throw new IllegalMonitorStateException();
    }
    if (getState()==0){
        throw new IllegalMonitorStateException();
    }
    //下面操作不存在并发
    setExclusiveOwnerThread(null);
    setState(0);
    return true;
}
~~~

此方法不再赘述

### 6.5.3 唤醒后继结点

release()钩子执行tryRelease()钩子成功之后，使用unparkSuccessor()唤醒后继节点，具体的代码如下：

```java
/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    //如果头节点的状态小于0，将其置为0，表示初始状态
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        //如果节点已经被取消
        s = null;
        //从队列尾部开始，往前去找前面一个waitStatus小于0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //唤醒后继结点队对应的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## 6.6 节点的入队和出队

对于上面AQS原理的一个比较重要的关键点在于掌握节点的入队和出队。

### 6.6.1 节点的入队

节点在第一次入队失败后，就会开始自旋入队，分为以下两种情况：

1. 如果AQS的队列非空，新节点入队的插入位置在队列的尾部，并且通过CAS方式插入，插入之后AQS的tail将指向新的尾节点。
2. 如果AQS的队列为空，新节点入队时，AQS通过CAS方法将新节点设置为头节点head，并且将tail指针指向新节点。

~~~java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //t==null表示当前队列为空，需要初始化
        if (t == null) { // Must initialize
            //将头尾节点都初始化成新节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //当前队列不为空，新节点插入队列尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
~~~

### 6.6.2 节点的出队

节点出队的算法在acquireQueued()方法中，这是一个非常重要的模板方法。acquireQueued()方法不断在前驱节点上自旋（for死循环），如果前驱节点是头节点并且当前线程使用钩子方法tryAcquire(arg)获得了锁，就移除头节点，将当前节点设置为头节点。

~~~java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        //自旋不断检查当前节点的前驱节点是否是头节点
        for (;;) {
            //获取节点的前驱节点
            final Node p = node.predecessor();
            //如果前驱节点是头节点，通过子类的tryAcquire进行抢锁
            if (p == head && tryAcquire(arg)) {
                //抢锁成功后将当前节点设置为头节点，移除之前的头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //检查前一个节点的状态，预判当前线程获取锁失败是否要挂起
            //如果需要挂起就执行parkAndCheckInterrupt挂起当前线程，直到被唤醒
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//两个操作都为true，就设置为true
        }
    } finally {
        //如果等待过程中没有成功获取到资源（如timeout，或者可中断的情况下被中断），那么就取消节点在队列的等待
        //由于超时或中断而被取消的线程会设置它的节点状态并取消它的后继节点，这样它就可以重置链接
        if (failed)
            //取消请求，将当前节点从队列中移除
            cancelAcquire(node);
    }
}
~~~

节点加入队列尾部后，如果其前驱节点不是头节点，通常情况下，该新节点所绑定的线程会被无限期阻塞，而不会去执行无效循环，从而导致CPU资源的浪费。

那么阻塞的线程何时唤醒？

对于公平锁而言，头节点就是占用锁的节点，在释放锁时，将会唤醒其后继节点所绑定的线程。后继节点的线程被唤醒后会重新执行以上acquireQueued()的自旋（for死循环）抢锁逻辑，检查自己的前驱节点是否为头节点，如果是，在抢锁成功之后会移除旧的头节点。

## 6.7 AQS同步队列

Condition是JUC用来替代传统Object的wait()/notify()线程间通信与协作机制的新组件，相比调用Object的wait()/notify()，调用Condition的await()/signal()这种方式实现线程间协作更加高效。

原作者道格李的关于AQS的论文中是这么描述等待和唤醒的流程：

~~~
The basic await operation is:
 create and add new node to condition queue;
 release lock;
 block until node is on lock queue;
 re-acquire lock;
And the signal operation is:
 transfer the first node from condition queue to lock queue;
~~~

### 6.7.1 Condition原理

Condition与Object的wait()/notify()作用是相似的，都是使得一个线程等待某个条件，只有当该条件具备signal()或者signalAll()方法被调用时等待线程才会被唤醒，从而重新争夺锁。不同的是，Object的wait()/notify()由JVM底层实现，而Condition接口与实现类完全使用Java代码实现。当需要进行线程间的通信时，建议结合使用ReetrantLock与Condition，通过Condition的await()和signal()方法进行线程间的阻塞与唤醒。

ConditionObject类是实现条件队列的关键，每个ConditionObject对象都维护一个单独的条件等待队列。每个ConditionObject对应一个条件队列，它记录该队列的头节点和尾节点。

ConditionObject类是AQS中Condition接口的实现类，我们看一下ConditionObject的部分源码

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    //队列头节点
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    //队列尾节点
    private transient Node lastWaiter;

    /**
     * Creates a new {@code ConditionObject} instance.
     */
    public ConditionObject() { }
    //其他方法略....
}
```

当然，在一个显式锁上，我们可以创建多个等待任务队列，这点和内置锁不同，Java内置锁上只有唯一的一个等待队列。

> 举个例子：
>
> ~~~java
> private Lock lock = new ReentrantLock();
> private Condition firstCon = lock.newCondition();
> private Condition secondCon = lock.newCondition();
> ~~~

Condition条件队列是单向的，而AQS同步队列是双向的，AQS节点会有前驱指针。一个AQS实例可以有多个条件队列，是聚合关系；但是一个AQS实例只有一个同步队列，是逻辑上的组合关系。

### 6.7.2 Condition的await()原理

当线程调用await()方法时，说明当前线程的节点为当前AQS队列的头节点，正好处于占有锁的状态，await()方法需要把该线程从AQS队列挪到Condition等待队列里。

await()方法源码如下：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //执行await()方法时，会创建一个新节点并放入Condition队列中
    Node node = addConditionWaiter();
    //然后释放锁，并唤醒AQS头节点的下一个节点
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //执行while循环，将该节点的线程阻塞，直到该节点离开等待队列，重新回到同步队列成为同步节点后，线程才退出while循环
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //退出循环后调用acquireQueued尝试拿到锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //拿到锁后会清空Condition队列中被取消的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

放入Condition队列的addConditionWaiter方法源码

```java
/**
 * Adds a new waiter to wait queue.
 * @return its new wait node
 */
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    //如果尾节点取消，重新定位尾节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //创建新Node
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    //将新Node放入等待队列
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

### 6.7.3 signal()唤醒原理

线程在某个ConditionObject对象上调用signal()方法后，等待队列中的firstWaiter会被加入同步队列中，等待节点被唤醒。

signal方法源码如下：

```java
public final void signal() {
    //如果当前线程不是锁的持有者，就抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);//唤醒头节点
}
//执行唤醒操作
private void doSignal(Node first) {
    do {
        //first出队，firstWaiter指向下一个节点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;//如果下一个节点是空的，那么尾部也是空的
        //将原来头部first的后继节点置为空，help for GC
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
//将被唤醒的节点转移到同步队列
final boolean transferForSignal(Node node) {
    /*
    * If cannot change waitStatus, the node has been cancelled.
    */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	//通过enq自旋将条件队列的头节点放入AQS同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    ////如果前驱节点的状态是取消状态，或者设置前驱节点的Signal状态失败，就唤醒当前节点的线程，否则节点在同步队列的尾部，参与排队
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        //线程被唤醒之后，表示重新获得了显式锁，然后执行Condition.await()后面的临近区代码
        LockSupport.unpark(node.thread);
    return true;
}
```





