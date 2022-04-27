# Java高并发编程篇五AQS

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第五部分AQS抽象同步器篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是对AQS进行研究学习。

> 解决CAS恶性空自旋的有效方式之一是以空间换时间，较为常见的方案有两种：分散操作热点和使用队列削峰。JUC并发包使用的是队列削峰的方案解决CAS的性能问题，并提供了一个基于双向队列的削峰基类——抽象基础类AbstractQueuedSynchronizer（抽象同步器类，简称为AQS）。

# AQS抽象同步器的核心原理



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

### 6.2.3 FIFO双向同步队列











