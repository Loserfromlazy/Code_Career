# Java高并发编程学习笔记第一部分多线程篇

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第一部分多线程篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java多线程原理与基本操作相关的知识

## 1.1 线程和进程简介

什么是进程呢？简单来说，进程是程序的一次启动执行。什么是程序呢？程序是存放在硬盘中的可执行文件，主要包括代码指令和数据。一个进程是一个程序的一次启动和执行，是操作系统将程序装入内存，给程序分配必要的系统资源，并且开始运行程序的指令。

进程与程序是什么关系呢？同一个程序可以多次启动，对应多个进程。

进程的定义一直以来没有完美的标准。一般来说，一个进程由程序段、数据段和进程控制块三部分组成：

程序段一般也被称为代码段。代码段是进程的程序指令在内存中的位置，包含需要执行的指令集合；数据段是进程的操作数据在内存中的位置，包含需要操作的数据集合；程序控制块（Program Control Block，PCB）包含进程的描述信息和控制信息，是进程存在的唯一标志。

线程是指“进程代码段”的一次顺序执行流程。线程是CPU调度的最小单位。一个进程可以有一个或多个线程，各个线程之间共享进程的内存空间、系统资源，进程仍然是操作系统资源分配的最小单位。

一个标准的线程主要由三部分组成，即线程描述信息、程序计数器（Program Counter，PC）和栈内存，如下图：

![线程20211213](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo%E7%BA%BF%E7%A8%8B20211213.png)

在Java中，执行程序流程的重要单位是“方法”，而栈内存的分配单位是“栈帧”（或者叫“方法帧”）。方法的每一次执行都需要为其分配一个栈帧（方法帧），栈帧主要保存该方法中的局部变量、方法的返回地址以及其他方法的相关信息。当线程的执行流程进入方法时，JVM就会为方法分配一个对应的栈帧压入栈内存；当线程的执行流程跳出方法时，JVM就从栈内存弹出该方法的栈帧，此时方法帧的局部变量的内存空间就会被回收。

```java
public class StackDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("当前线程名称"+Thread.currentThread().getName());
        System.out.println("当前线程ID"+Thread.currentThread().getId());
        System.out.println("当前线程状态"+Thread.currentThread().getState());
        System.out.println("当前线程优先级"+Thread.currentThread().getPriority());
        int a =1,b=1;
        int c = a/b;
        fun();
        anotherFun();
        Thread.sleep(10000);
    }

    public static void fun(){
        int a =1,b=1;
        int c = a/b;
    }

    public static void anotherFun(){
        int a =1,b=1;
        int c = a/b;
    }
}
```

如上面的程序，当执行main()方法时，JVM为其分配一个栈帧保存三个局部变量，然后将栈帧压入main线程的栈内存，接着进入fun()方法，流程一致，JVM为其分配一个栈帧保存三个局部变量，然后将栈帧压入main线程的栈内存，然后执行anotherFun()方法，跟之前一致。

三个方法的栈帧弹出的过程与f压入的过程刚好相反。anotherFun()方法执行完成后，其栈帧从main线程的栈内存首先弹出，执行流程回到fun()方法。fun()方法执行完成后，其栈帧从main线程的栈内存弹出之后，执行流程回到main()方法。main()方法执行完成后，其栈帧弹出，此时main线程的栈内存已经全部弹空，没有剩余的栈帧。至此，main线程结束。

## 1.2 创建线程的方法

### 1.2.1 继承Thread类

需要继承Thread类，创建一个新的线程类。同时重写run()方法，将需要并发执行的业务代码编写在run()方法中。

```java
public class CreateThreadDemo {

    private static final Integer MAX = 5;

    public static void main(String[] args) {
        Thread thread = null;
        for (int i = 0; i < 2; i++) {
            thread = new DemoThread();
            thread.start();
        }
    }

    static class DemoThread extends Thread{
        @Override
        public void run() {
            for (int i = 0; i < MAX; i++) {
                System.out.println(getName()+",i="+i);
            }
            System.out.println(getName()+"结束");
        }
    }
}
```

### 1.2.2 实现Runnable接口

我们进入到Thread类中查看代码：

```java
public class Thread implements Runnable {
    private Runnable target;
    public void run() {
        if (this.target != null) {
            this.target.run();
        }
    }
     public Thread(Runnable target) {
        this.init(null, target, "Thread-" + nextThreadNum(), 0L);
    }
}
```

在Thread类的run()方法中，如果target（执行目标）不为空，就执行target属性的run()方法。而target属性是Thread类的一个实例属性，并且target属性的类型为Runnable。所以只要Thread构造方法传入Runnable类型的值，target就不为空。

创建线程的流程如下：

1. 定义一个新类实现Runnable接口。
2. 实现Runnable接口中的run()抽象方法，将线程代码逻辑存放在该run()方法中。
3. 通过Thread类创建线程对象，将Runnable实例作为实际参数传递给Thread类的构造器，由Thread构造器将该Runnable实例赋值给自己的target执行目标属性。
4. 调用Thread实例的start()方法启动线程。
5. 线程启动之后，线程的run()方法将被JVM执行，该run()方法将调用target属性的run()方法，从而完成Runnable实现类中业务代码逻辑的并发执行。

```java
public class CreateThreadDemo2 {

    private static final Integer MAX = 5;

    public static void main(String[] args) {
        Thread thread =null;
        for (int i = 0; i < 2; i++) {
            Runnable target = new DemoThread();
            thread = new Thread(target);
            thread.start();
        }
    }
    static class DemoThread implements Runnable{

        @Override
        public void run() {
            for (int i = 0; i < MAX; i++) {
                System.out.println(Thread.currentThread().getName()+",i="+i);
            }
            System.out.println(Thread.currentThread().getName()+"结束");
        }
    }
}
```

除了直接实现Runnable接口外，我们还可以通过匿名类优雅地创建Runnbale线程目标类或者使用Lambda表达式优雅地创建Runnbale线程目标类。代码如下：

```java
public class CreateThreadDemo2 {

    private static final Integer MAX = 5;

    public static void main(String[] args) {
        Thread thread =null;
        //通过匿名内部类实现
        for (int i = 0; i < 2; i++) {
            thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < MAX; i++) {
                        System.out.println(Thread.currentThread().getName()+",i="+i);
                    }
                    System.out.println(Thread.currentThread().getName()+"结束");
                }
            });
            thread.start();
        }
        //通过Lambda实现
        for (int i = 0; i < 2; i++) {
            thread = new Thread(() -> {
                for (int j = 0; j < MAX; j++) {
                    System.out.println(Thread.currentThread().getName()+",j="+ j);
                }
                System.out.println(Thread.currentThread().getName()+"结束");
            });
            thread.start();
        }
    }
}
```

> Runnable缺点：
>
> 1. 所创建的类并不是线程类，而是线程的target执行目标类，需要将其实例作为参数传入线程类的构造器，才能创建真正的线程。
> 2. 如果访问当前线程的属性（甚至控制当前线程），不能直接访问Thread的实例方法，必须通过Thread.currentThread()获取当前线程实例，才能访问和控制当前线程。
>
> Runnable优点：
>
> 1. 逻辑和数据更好分离
> 2. 可以避免由于Java单继承带来的局限性

### 1.2.3 使用Callable和FutureTask

上面两种方式不能异步的获取线程的执行结果，为了解决异步执行的结果问题，Java语言在1.5版本之后提供了一种新的多线程创建方法：通过Callable接口和FutureTask类相结合创建线程。

首先我们看Callable源码：

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

这个接口带泛型，其call()方法带返回值，call()抽象方法还有一个Exception的异常声明，容许方法的实现版本的内部异常直接抛出，并且可以不予捕获。

Callable接口类似于Runnable。不同的是，Runnable的唯一抽象方法run()没有返回值，也没有受检异常的异常声明。比较而言，Callable接口的call()有返回值，并且声明了受检异常，其功能更强大一些。

但是Callable不像Runnable方法一样，Thread的target实例是Runnable类型的，这就说明Callable不能作为Thread线程的target实例使用。所以需要中间搭桥的接口来在Callable接口与Thread线程之间起到连接作用这个接口就是RunnableFuture，RunnableFuture接口实现了两个目标：一是可以作为Thread线程实例的target实例，二是可以获取异步执行的结果。源码如下：

```java
package java.util.concurrent;

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

RunnableFuture继承了Runnable接口，从而保证了其实例可以作为Thread线程实例的target目标；同时，RunnableFuture通过继承Future接口，保证了可以获取未来的异步执行结果。

Future接口源码如下：

```java
package java.util.concurrent;

public interface Future<V> {
    //取消异步执行
    boolean cancel(boolean var1);
	
    boolean isCancelled();
	//判断异步任务是否完成
    boolean isDone();
	//获取异步任务的执行结果
    /*获取异步任务执行的结果。注意，这个方法的调用是阻塞性的。如果异步任务没有执行完成，异步结果获取线程（调用线程）会一直被阻塞，一直阻塞到异步任务执行完成，其异步结果返回给调用线程*/
    V get() throws InterruptedException, ExecutionException;
	//设置时限获取异步任务完成的结果
    /*该方法的调用也是阻塞性的，但是结果获取线程（调用线程）会有一个阻塞时长限制，不会无限制地阻塞和等待，如果其阻塞时间超过设定的timeout时间，该方法将抛出异常，调用线程可捕获此异常*/
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

总体来说，Future是一个对异步任务进行交互、操作的接口。但是Future仅仅是一个接口，通过它没有办法直接完成对异步任务的操作，JDK提供了一个默认的实现类——FutureTask。

FutureTask类是Future接口的实现类，提供了对异步任务的操作的具体实现。但是，FutureTask类不仅实现了Future接口，还实现了Runnable接口，或者更加准确地说，FutureTask类实现了RunnableFuture接口。继承类图（IDEA快捷键crtl+alt+u）如下：

![image-20220331165448008](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220331165448008.png)

通过Callable创建线程步骤：

1. 创建一个Callable接口的实现类，并实现其call()方法，编写好异步执行的具体逻辑，可以有返回值。
2. 使用Callable实现类的实例构造一个FutureTask实例。
3. 使用FutureTask实例作为Thread构造器的target入参，构造新的Thread线程实例。
4. 调用Thread实例的start()方法启动新线程，启动新线程的run()方法并发执行。其内部的执行过程为：启动Thread实例的run()方法并发执行后，会执行FutureTask实例的run()方法，最终会并发执行Callable实现类的call()方法。
5. 调用FutureTask对象的get()方法阻塞性地获得并发线程的执行结果。

```java
public class CreateThreadDemo3 {
    private static final Integer MAX = 5;

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        Thread thread = null;
        for (int i = 0; i < 2; i++) {
            DemoThread demoThread = new DemoThread();
            FutureTask<Long> futureTask = new FutureTask<>(demoThread);
            thread = new Thread(futureTask);
            thread.start();
            Thread.sleep(500);
            System.out.println(thread.getName()+"占用时间"+futureTask.get());
        }
    }

    static class DemoThread implements Callable<Long>{

        @Override
        public Long call() throws Exception {
            long start = System.currentTimeMillis();
            for (int i = 0; i < MAX; i++) {
                System.out.println(Thread.currentThread().getName()+",i="+i);
            }
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName()+"结束");
            long used = System.currentTimeMillis() - start;
            return used;
        }
    }
}
```

在这个例子中有两个线程：一个是执行main()方法的主线程，叫作main；另一个是main线程通过thread.start()方法启动的业务线程，该线程是一个包含FutureTask任务作为target的Thread线程。两个线程的执行流程如下：

![Callable20211213](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgoCallable20211213.png)

### 1.2.4 线程池创建线程

实际上创建一个线程实例在时间成本、资源耗费上都很高（稍后会介绍），在高并发的场景中，断然不能频繁进行线程实例的创建与销毁，而是需要对已经创建好的线程实例进行复用，这就涉及线程池的技术。Java中提供了一个静态工厂来创建不同的线程池，该静态工厂为Executors工厂类。例如：

~~~java
private static ExecutorService pool = new Executors.newFixedThreadPool(3);
~~~

ExecutorService是Java提供的一个线程池接口，每次我们在异步执行target目标任务的时候，可以通过ExecutorService线程池实例去提交或者执行。ExecutorService实例负责对池中的线程进行管理和调度，并且可以有效控制最大并发线程数，提高系统资源的使用率，同时提供定时执行、定频执行、单线程、并发数控制等功能。

向ExecutorService线程池提交异步执行target目标任务的常用方法有:

~~~java
//执行一个Runnable的target实例
void execute(Runnable command);
//提交一个Callable的target实例，返回一个Future异步任务实例
<T> Future<T> submit(Callable<T> task);
//提交一个Runnable类型的target实例，返回一个Future异步任务实例
Future<?> submit(Runnable task)
~~~

> ExecutorService线程池的execute(...)与submit(...)方法的区别如下。
>
> 1. 接收的参数不一样submit()可以接收两种入参：无返回值的Runnable类型的target执行目标实例和有返回值的Callable类型的target执行目标实例。而execute()仅仅接收无返回值的target执行目标实例，或者无返回值的Thread实例。
> 2. submit()有返回值，而execute()没有submit()方法在提交异步target执行目标之后会返回Future异步任务实例，以便对target的异步执行过程进行控制，比如取消执行、获取结果等。execute()没有任何返回，target执行目标实例在执行之后没有办法对其异步执行过程进行控制，只能任其执行，直到其执行结束。
>
> **实际生产环境禁止使用Executors创建线程池，而是通过ThreadPoolExecutor的方式**

## 1.3 线程的核心原理

### 1.3.1 线程的调度与时间片

由于CPU的计算频率非常高，每秒钟现代操作系统（如Windows、Linux、Solaris）提供了强大的线程管理能力，Java不需要再进行独立的线程管理和调度，而是将线程调度工作委托给操作系统的调度进程去完成。在某些系统（比如Solaris操作系统）上，JVM甚至将每个Java线程一对一地对应到操作系统的本地线程，彻底将线程调度委托给操作系统。

目前操作系统中**主流的线程调度方式是：基于CPU时间片方式进行线程调度**。线程只有得到CPU时间片才能执行指令，处于执行状态，没有得到时间片的线程处于就绪状态，等待系统分配下一个CPU时间片。由于时间片非常短，在各个线程之间快速地切换，因此表现出来的特征是很多个线程在“同时执行”或者“并发执行”。**线程的调度模型主要分为两种：分时调度和抢占式调度。**分时调度就是系统平均分配CPU的时间片，所有线程轮流占用CPU；抢占式调度模型就是系统按照线程优先级分配时间片，优先级高的线程优先分配CPU时间片，如果所有就绪线程的优先级相同，那么会随机选择一个。

### 1.3.2 线程的优先级

在Thread类中有一个实例属性和两个实例方法，专门用于进行线程优先级相关的操作。与线程优先级相关的成员属性为：`priority`,方法为

~~~java
private int            priority;
public final int getPriority();//获取线程优先级。
public final void setPriority(int priority);//设置线程优先级。
~~~

Thread实例的priority属性默认是级别5，对应的类常量是NORM_PRIORITY。优先级最大值为10，最小值为1.

Java使用的就是抢占式调度模型，在Java中执行机会的获取具有随机性，但是从整体来看高优先级的线程获得的执行机会更多。我们可以通过代码来验证：

```java
public class PriorityDemo {
    static class PriorityThread extends Thread{
        static int noo =1;

        public PriorityThread() {
            super("thread-"+noo);
            noo++;
        }
        public long opportunity = 0;
        @Override
        public void run() {
            for (int i = 0; ; i++) {
                opportunity++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        PriorityThread [] threads = new PriorityThread[10];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new PriorityThread();
            threads[i].setPriority(i+1);
        }
        for (PriorityThread thread : threads) {
            thread.start();
        }
        Thread.sleep(500);
        for (PriorityThread thread : threads) {
            thread.stop();
        }
        for (PriorityThread thread : threads) {
            System.out.println(thread.getName()+"  优先级："+thread.getPriority()+"机会："+thread.opportunity);
        }
    }
}
```

上面我们通过opportunity来表示10个线程在CPU中获得的时间片，我们给10个线程依次给出优先级，运行结果如下：

~~~
thread-1  优先级：1机会：659209944
thread-2  优先级：2机会：689254505
thread-3  优先级：3机会：763575964
thread-4  优先级：4机会：742457001
thread-5  优先级：5机会：741392146
thread-6  优先级：6机会：748299795
thread-7  优先级：7机会：759256402
thread-8  优先级：8机会：771598037
thread-9  优先级：9机会：756852358
thread-10  优先级：10机会：750843505
~~~

可以看到，**执行机会的获取具有随机性，但是从整体来看高优先级的线程获得的执行机会更多。**

### 1.3.3 线程的生命周期

Java中线程的生命周期分为6种状态。Thread类有一个实例属性和一个实例方法专门用于保存和获取线程的状态。同时还有一个内部枚举用来描述线程的状态，源码如下:

```java
/* Java thread status for tools,
 * initialized to indicate thread 'not yet started'
 */
private volatile int threadStatus = 0;
public enum State {
        /** 新建,线程尚未启动的状态，即未调用start方法
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /** 可执行，包含操作系统的就绪和运行。Java中线程管理是通过JNI本地调用委托操作系统的线程管理API完成的
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /** 阻塞
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /** 等待状态，调用以下方法会进入此状态
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /** 限时等待，处于一种特殊的等待状态，调用以下方法会进入此状态
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /** 终止，线程中run方法执行完后就变为终止状态
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

> 线程的WAITING（等待）状态表示线程在等待被唤醒。处于WAITING状态的线程不会被分配CPU时间片。执行以下两个操作，当前线程将处于WAITING状态：
>
> - 执行没有时限（timeout）参数的thread.join()调用：在线程合并场景中，若线程A调用B.join()去合入B线程，则在B执行期间线程A处于WAITING状态，一直等线程B执行完成。
> - 执行没有时限（timeout）参数的object.wait()调用：指一个拥有object对象锁的线程，进入相应的代码临界区后，调用相应的object的wait()方法去等待其“对象锁”（Object Monitor）上的信号，若“对象锁”上没有信号，则当前线程处于WAITING状态
>
> 线程的TIMED_WAITING状态表示在等待唤醒。处于TIMED_WAITING状态的线程不会被分配CPU时间片，它们要等待被唤醒，或者直到等待的时限到期。在线程合入场景中，若线程A在调用B.join()操作时加入了时限参数，则在B执行期间线程A处于TIMED_WAITING状态。若B在等待时限内没有返回，则线程A结束等待TIMED_WAITING状态，恢复成RUNNABLE状态。
>
> 总结：其实就是在等待方法执行时有没有时间的期限。

### 1.3.4 Jstack工具

Jstack时Java虚拟机自带的堆栈跟踪工具，它用于导出JVM当前时刻的线程快照，它的语法如下：

~~~
jstack <pid> //java的进程id，可使用jsp查看
~~~

> 在实际运行中，建议使用三次快照信息，如果每次都是一个问题，才能确定问题的非偶发性。

Jstack输出的信息包含以下重要信息：

1. tid：线程实例在JVM中的id
2. nid：线程实例在操作系统中对应的底层线程的线程id
3. prio：线程实例在JVM进程中的优先级
4. os_prio：线程实例在操作系统中对应的底层线程给的优先级
5. 线程状态：如runnable等。

## 1.4 线程的基本操作

### 1.4.1 线程名称的设置和获取

在Thread类中可以通过构造器Thread(…)初始化设置线程名称，也可以通过setName(…)实例方法设置线程名称，取得线程名称可以通过getName()方法完成。

关于线程名称有以下需要注意的地方：

- 线程名称一般在启动线程前设置，但也允许为运行的线程设置名称。
- 允许两个Thread对象有相同的名称，但是应该避免。
- 如果程序没有为线程指定名称，系统会自动为线程设置名称。

举个栗子：

```java
public class ThreadNameDemo {
    static class MyThreadDemo implements Runnable{
        @Override
        public void run() {
            for (int i = 0; i < 2; i++) {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String name = Thread.currentThread().getName();
                System.out.println(name+" ---"+i);
            }
        }
    }

    public static void main(String[] args) {
        MyThreadDemo demo = new MyThreadDemo();
        new Thread(demo).start();
        new Thread(demo).start();
        new Thread(demo,"自定义名称1").start();
        new Thread(demo,"自定义名称2").start();
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```

执行结果：

自定义名称2 ---0
Thread-0 ---0
Thread-1 ---0
自定义名称1 ---0
自定义名称2 ---1
自定义名称1 ---1
Thread-1 ---1
Thread-0 ---1

### 1.4.2 sleep

Sleep()方法定义在Thread类中，是一组静态方法，有两个重载版本：

```java
public static native void sleep(long millis) throws InterruptedException;
public static void sleep(long millis, int nanos)throws InterruptedException{
    //处理纳秒，代码略
    sleep(millis);
}
```

我们这里熟练一下jstack工具，将上一节的代码修改一下：

```java
public class ThreadSleepDemo {
    static class MyThreadDemo implements Runnable{
        @Override
        public void run() {
            for (int i = 0; i < 50; i++) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String name = Thread.currentThread().getName();
                System.out.println("线程"+name+"睡眠第"+i+"次");
            }
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 3; i++) {
            MyThreadDemo demo = new MyThreadDemo();
            new Thread(demo,"sleep"+i).start();
        }
    }
}
```

然后我们运行代码，之后在终端输入jps查询当前ThreadSleepDemo的线程id

![image-20220414104007329](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220414104007329.png)

然后用jstack +id可以查看 我们的线程的状态：

![image-20220414104104353](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220414104104353.png)

### 1.4.3 interrupt

Java语言提供了stop()方法终止正在运行的线程，但是Java将Thread的stop()方法设置为过时，不建议使用。

在程序中，我们是不能随便中断一个线程的，我们无法知道这个线程正运行在什么状态，它可能持有某把锁，强行中断线程可能导致锁不能释放的问题；或者线程可能在操作数据库，强行中断线程可能导致数据不一致的问题。正是由于调用stop()方法来终止线程可能会产生不可预料的结果，因此不推荐调用stop()方法。

Java提供了Thread的interrupt()方法，此方法本质不是用来中断一个线程，而是将线程设置为中断状态。

1. 如果此线程处于阻塞状态（线程被`Object.wait()`、`Thread.join()`和`Thread.sleep()`三种方法之一阻塞），就会立马退出阻塞，并抛出InterruptedException异常，线程就可以通过捕获InterruptedException来做一定的处理，然后让线程退出。
2. 如果此线程正处于运行之中，线程就不受任何影响，继续运行，仅仅是线程的中断标记被设置为true。所以，程序可以在适当的位置通过调用isInterrupted()方法来查看自己是否被中断，并执行退出操作。

举例，我们将上面的1.4.2的demo修改一下：

```java
public class ThreadSleepDemo {
    static class MyThreadDemo implements Runnable{
        @Override
        public void run() {
            String name = Thread.currentThread().getName();
            try {
                System.out.println("线程"+name+"进入睡眠");
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                System.out.println("线程"+name+"发生异常被打断");
                return;
            }
            /*
            */
            System.out.println("线程"+name+"运行结束");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MyThreadDemo demo = new MyThreadDemo();
        Thread thread1 = new Thread(demo,"sleep1");
        thread1.start();
        Thread thread2 =new Thread(demo,"sleep2");
        thread2.start();
        Thread.sleep(2000);
        thread1.interrupt();
        Thread.sleep(5000);
        thread2.interrupt();
        Thread.sleep(1000);
        System.out.println("主程序运行结束");

    }
}
```

运行结果：

~~~
线程sleep2进入睡眠
线程sleep1进入睡眠
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.learn.testnginx.threadLearn.ThreadSleepDemo$MyThreadDemo.run(ThreadSleepDemo.java:18)
	at java.lang.Thread.run(Thread.java:748)
线程sleep1发生异常被打断
线程sleep2运行结束
主程序运行结束
~~~

我们可以发现sleep1线程在大致睡眠了两秒后，被主线程打断（或者中断）。被打断后停止睡眠，并捕获到InterruptedException异常。程序在异常处理时直接返回，其后面的执行逻辑被跳过。而sleep2线程在睡眠了7秒后，被主线程中断，但是在sleep2线程被中断的时候，已经执行结束了，所以thread2.interrupt()中断操作没有产生实质性的效果。

我们也可以通过isInterrupted来判断线程是否被终止，如下：

```java
while (true){
    System.out.println(Thread.currentThread().isInterrupted()+"");
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    if (Thread.currentThread().isInterrupted()){
        return;
    }
}
```

### 1.4.4 join

线程的合并可以这么理解，有AB两个线程，A线程在执行过程中需要依赖B线程执行的流程，线程A需要将线程B的执行流程合并到自己的执行流程中，这就是线程合并，被动方线程B可以叫作被合并线程。

join有三个重载，如下（源码为jdk8）：

```java
//把当前线程变成TIMED_WAITING，直到被合并线程执行结束，或者等待被合并线程执行millis时间
public final synchronized void join(long millis)
throws InterruptedException{
    //代码略
}
//把当前线程变成TIMED_WAITING，直到被合并线程执行结束，或者等待被合并线程执行millis+nanos时间
public final synchronized void join(long millis, int nanos)
    throws InterruptedException {
	//代码略
}
//把当前线程变成TIMED_WAITING，直到被合并线程执行结束
public final void join() throws InterruptedException {
    join(0);
}
```

我们可以通过流程图来理解：

![image-20220415095052916](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220415095052916.png)

这里举个例子：

```java
public class JoinDemo {
    static class MyThreadDemo extends Thread{
        private static int num =1;
        public MyThreadDemo() {
            super("MyThread"+num);
            num++;
        }
        @Override
        public void run() {
            String name = getName();
            try {
                System.out.println("线程"+name+"进入睡眠");
                Thread.sleep(4000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("线程"+name+"运行结束");

        }
    }
    public static void main(String[] args) {
        Thread thread1 = new MyThreadDemo();
        System.out.println("thread1启动");
        thread1.start();
        try {
            thread1.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Thread thread2 = new MyThreadDemo();
        System.out.println("thread2启动");
        thread2.start();
        try {
            thread2.join(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程运行结束");

    }
}
```

运行结果：

~~~
thread1启动
线程MyThread1进入睡眠
线程MyThread1运行结束
thread2启动
线程MyThread2进入睡眠
线程MyThread2运行结束
主线程运行结束
//如果将thread2.join(5000);修改为->thread2.join(1000);结果将会变成这样：
thread1启动
线程MyThread1进入睡眠
线程MyThread1运行结束
thread2启动
线程MyThread2进入睡眠
主线程运行结束
线程MyThread2运行结束
~~~

上面结果的不同就是因为join中等待时间的不同导致的。

### 1.4.5 yield

线程的yield（让步）操作的作用是让目前正在执行的线程放弃当前的执行，让出CPU的执行权限，使得CPU去执行其他的线程。处于让步状态的JVM层面的线程状态仍然是RUNNABLE状态，但是该线程所对应的操作系统层面的线程从状态上来说会从执行状态变成就绪状态。线程在yield时，线程放弃和重占CPU的时间是不确定的，可能是刚刚放弃CPU，马上又获得CPU执行权限，重新开始执行。

使用方式：`Thread.yield();`

### 1.4.6 daemon

Java中的线程分为两类：守护线程与用户线程。守护线程也称为后台线程，专门指在程序进程运行过程中，在后台提供某种通用服务的线程。比如，每启动一个JVM进程，都会在后台运行一系列的GC（垃圾回收）线程。如果你在上面跟着我用jstack查看信息的话，就可以看到GC的信息。

> 只要JVM实例中尚存在任何一个用户线程没有结束，守护线程就能执行自己的工作；只有当最后一个用户线程结束，守护线程随着JVM一同结束工作。

我们看一下源码，跟守护进程有关的如下：

```java
/* Whether or not the thread is a daemon thread.实例属性，默认为false */
private boolean     daemon = false;
//实例方法，设置当前线程是用户进程还是守护进程
public final void setDaemon(boolean on) {
        //代码略
}
/**
 * Tests if this thread is a daemon thread.
 * 获取状态，判断是不是守护线程
 * @see     #setDaemon(boolean)
 */
public final boolean isDaemon() {
    return daemon;
}
```

我们下面给个例子：

```java
public class DaemonDemo {
    static class MyThreadDemo extends Thread{
        public MyThreadDemo() {
            super("daemonThread");
        }
        @Override
        public void run() {
            System.out.println("daemonThread线程开始");
            //死循环一直跑
            for (int i = 1; ; i++) {
                System.out.println("第"+i+"轮");
                System.out.println("守护状态"+isDaemon());
                sleepMillSeconds(500);
            }
        }
    }

    public static void main(String[] args) {
        Thread myThreadDemo = new MyThreadDemo();
        myThreadDemo.setDaemon(true);
        myThreadDemo.start();
        Thread userThread = new Thread(()->{
            for (int i = 0; i < 4; i++) {
                System.out.println("用户线程第"+i+"轮");
                System.out.println("用户线程守护状态"+Thread.currentThread().isDaemon());
                sleepMillSeconds(500);
            }
            System.out.println("用户线程结束");
        });
        userThread.start();
        System.out.println("主线程守护状态"+Thread.currentThread().isDaemon());
        System.out.println("运行结束");
    }
    private static void sleepMillSeconds(long mills){
        try {
            Thread.sleep(mills);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果：

~~~
daemonThread线程开始
第1轮
守护状态true
主线程守护状态false
运行结束
用户线程第1轮
用户线程守护状态false
第2轮
守护状态true
用户线程第2轮
用户线程守护状态false
第3轮
守护状态true
用户线程结束
~~~

上面代码执行后，主线程在启动两个线程后就结束了，这时还有个用户线程再跑，当用户线程跑完后，此时没有用户线程了，所以后台进程也结束了。

> - 守护线程必须在启动前将其守护状态设置为true，启动之后不能再将用户线程设置为守护线程，否则JVM会抛出一个InterruptedException异常。
> - 在守护线程中创建的线程，新的线程都是守护线程。
> - 守护线程中尽量不去访问系统资源，如文件句柄、数据库连接等

## 1.5 线程池原理

Java线程的创建非常昂贵，需要JVM和OS（操作系统）配合完成大量的工作：

1. 必须为线程堆栈分配和初始化大量内存块，其中包含至少1MB的栈内存。
2. 需要进行系统调用，以便在OS（操作系统）中创建和注册本地线程。

线程池主要解决了以下两个问题：

1. 提升性能：线程池能独立负责线程的创建、维护和分配。在执行大量异步任务时，可以不需要自己创建线程，而是将任务交给线程池去调度。线程池能尽可能使用空闲的线程去执行异步任务，最大限度地对已经创建的线程进行复用，使得性能提升明显。
2. 线程管理：每个Java线程池会保持一些基本的线程统计信息，例如完成的任务数量、空闲时间等，以便对线程进行有效管理，使得能对所接收到的异步任务进行高效调度。

> 在主要大厂的编程规范中，不允许在应用中自行显式地创建线程，线程必须通过线程池提供。

### 1.5.1 JUC线程池介绍

在JUC（JUC就是java.util.concurrent工具包的简称）中线程池的大概架构如下：

![image-20220415133016157](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220415133016157.png)

下面介绍一下：

1. Executor

   是Java异步执行目标任务的接口，Executor作为执行者，提供execute方法，目的是将任务提交者和任务执行者分离。

   ```java
   void execute(Runnable command);
   ```

2. ExecutorService

   继承于Executor，是Java异步异步执行目标任务的执行者服务接口，对外提供异步任务的接收服务。它提供了接收异步任务转交给执行者的方法，如submit、invokeAll等方法。

3. AbstractExecutorService

   抽象类，实现了ExecutorService接口，为ExecutorService接口提供默认实现。

   > 官方文档：Provides default implementations of ExecutorService execution methods. 

4. ThreadPoolExecutor

   线程池实现类，继承AbstractExecutorService抽象类。ThreadPoolExecutor是JUC线程池的核心实现类。线程的创建和终止需要很大的开销，线程池中预先提供了指定数量的可重用线程，所以使用线程池会节省系统资源，并且每个线程池都维护了一些基础的数据统计，方便线程的管理和监控。

   > 官方文档：An ExecutorService that executes each submitted task using one of possibly several pooled threads, normally configured using Executors factory methods.
   >
   > Thread pools address two different problems: they usually provide improved performance when executing large numbers of asynchronous tasks, due to reduced per-task invocation overhead, and they provide a means of bounding and managing the resources, including threads, consumed when executing a collection of tasks. Each ThreadPoolExecutor also maintains some basic statistics, such as the number of completed tasks.
   > To be useful across a wide range of contexts, this class provides many adjustable parameters and extensibility hooks. 

5. ScheduledExecutorService

   ScheduledExecutorService是一个接口，它继承于ExecutorService。它是一个可以完成“延时”和“周期性”任务的调度线程池接口，其功能和Timer/TimerTask类似。

6. ScheduledThreadPoolExecutor

   继承于ThreadPoolExecutor，它提供了ScheduledExecutorService线程池接口中“延时执行”和“周期执行”等抽象调度方法的具体实现。

   > 官方文档：A ThreadPoolExecutor that can additionally schedule commands to run after a given delay, or to execute periodically. This class is preferable to Timer when multiple worker threads are needed, or when the additional flexibility or capabilities of ThreadPoolExecutor (which this class extends) are required.

7. Executors

   Executors是一个静态工厂类，它通过静态工厂方法返回ExecutorService、ScheduledExecutorService等线程池示例对象，这些静态工厂方法可以理解为一些快捷的创建线程池的方法。

### 1.5.2 Executors中快捷创建线程池的方法

Executors类中提供了快捷创建线程池的方法，我们这里主要介绍四个，因为大部分企业的开发规范都会禁止使用快捷线程池：

1. newSingleThreadExecutor创建“单线程化线程池”

   举个栗子：

   ```java
   public class ThreadPoolDemo {
       static class TargetTask implements Runnable{
           static AtomicInteger integer = new AtomicInteger(1);
           private String name;
           public TargetTask() {
               name = "task-"+integer.get();
               integer.incrementAndGet();
           }
           @Override
           public void run() {
               System.out.println(name+"正在执行");
               sleepMillsSecond(500);
               System.out.println(name+"运行结束");
           }
       }
       private static void sleepMillsSecond(long mills){
           try {
               Thread.sleep(mills);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
       public static void main(String[] args) {
           ExecutorService service = Executors.newSingleThreadExecutor();
           for (int i = 0; i < 5; i++) {
               service.execute(new TargetTask());
               service.submit(new TargetTask());
           }
           sleepMillsSecond(1000);
           service.shutdown();
       }
   }
   ```

   执行结果：

   ~~~
   task-1正在执行
   task-1运行结束
   task-2正在执行
   task-2运行结束
   task-3正在执行
   task-3运行结束
   task-4正在执行
   task-4运行结束
   task-5正在执行
   task-5运行结束
   task-6正在执行
   task-6运行结束
   task-7正在执行
   task-7运行结束
   task-8正在执行
   task-8运行结束
   task-9正在执行
   task-9运行结束
   task-10正在执行
   task-10运行结束
   ~~~

   我们可以发现该线程池有以下特点：

   （1）单线程化的线程池中的任务是按照提交的次序顺序执行的。

   （2）池中的唯一线程的存活时间是无限的。

   （3）当池中的唯一线程正繁忙时，新提交的任务实例会进入内部的阻塞队列中，并且其阻塞队列是无界的。总体来说，单线程化的线程池所适用的场景是：任务按照提交次序，一个任务一个任务地逐个执行的场景。

2. newFixedThreadPool创建“固定数量的线程池”

   该方法用于创建一个“固定数量的线程池”，其唯一的参数用于设置池中线程的“固定数量”。

   举个栗子，这里的代码与上面基本一致，只修改了主函数，因此这里只展示主函数代码：

   ```java
   public static void main(String[] args) {
           //固定大小线程池
           ExecutorService service = Executors.newFixedThreadPool(3);
           for (int i = 0; i < 5; i++) {
               service.execute(new TargetTask());
               service.submit(new TargetTask());
           }
           sleepMillsSecond(1000);
           service.shutdown();
       }
   ```

   执行结果：

   ~~~
   task-1正在执行
   task-2正在执行
   task-3正在执行
   task-2运行结束
   task-1运行结束
   task-3运行结束
   task-4正在执行
   task-6正在执行
   task-5正在执行
   task-4运行结束
   task-6运行结束
   task-7正在执行
   task-5运行结束
   task-8正在执行
   task-9正在执行
   task-7运行结束
   task-10正在执行
   task-8运行结束
   task-9运行结束
   task-10运行结束
   ~~~

   “固定数量的线程池”的特点大致如下：

   （1）如果线程数没有达到“固定数量”，每次提交一个任务线程池内就创建一个新线程，直到线程达到线程池固定的数量。

   （2）线程池的大小一旦达到“固定数量”就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

   （3）在接收异步任务的执行目标实例时，如果池中的所有线程均在繁忙状态，新任务会进入阻塞队列中（无界的阻塞队列）。

3. newCachedThreadPool创建“可缓存线程池”

   该方法用于创建一个“可缓存线程池”，如果线程池内的某些线程无事可干成为空闲线程，“可缓存线程池”可灵活回收这些空闲线程。举个栗子，这里同样只展示主函数代码：

   ```java
   //可缓存的线程池
   ExecutorService service = Executors.newCachedThreadPool();
   for (int i = 0; i < 5; i++) {
       service.execute(new TargetTask());
       service.submit(new TargetTask());
   }
   sleepMillsSecond(1000);
   service.shutdown();
   ```

   运行结果：

   ~~~
   task-1正在执行
   task-2正在执行
   task-4正在执行
   task-3正在执行
   task-5正在执行
   task-6正在执行
   task-8正在执行
   task-7正在执行
   task-10正在执行
   task-9正在执行
   task-8运行结束
   task-9运行结束
   task-2运行结束
   task-10运行结束
   task-6运行结束
   task-5运行结束
   task-4运行结束
   task-3运行结束
   task-1运行结束
   task-7运行结束
   ~~~

   此线程池特点：

   （1）在接收新的异步任务target执行目标实例时，如果池内所有线程繁忙，此线程池就会添加新线程来处理任务。

   （2）此线程池不会对线程池大小进行限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。

   （3）如果部分线程空闲，也就是存量线程的数量超过了处理任务数量，就会回收空闲（60秒不执行任务）线程。

4. newScheduledThreadPool创建“可调度线程池”

   该方法用于创建一个“可调度线程池”，即一个提供“延时”和“周期性”任务调度功能的ScheduledExecutorService类型的线程池。举个栗子，这里同样只展示主函数代码：
   
   ```java
   public static void main(String[] args) {
           ScheduledExecutorService service = Executors.newScheduledThreadPool(2);
           for (int i = 0; i < 2; i++) {
               //0是initialDelay，表示首次执行任务的延迟时间，500是period表示每次执行任务的时间间隔
               service.scheduleAtFixedRate(new TargetTask(),0,500, TimeUnit.MILLISECONDS);
           }
           sleepMillsSecond(1000);
           service.shutdown();
       }
   ```
   
   执行结果：
   
   ~~~
   task-1正在执行
   task-2正在执行
   task-2运行结束
   task-1运行结束
   task-2正在执行
   task-1正在执行
   task-2运行结束
   task-1运行结束
   ~~~
   
   ScheduledExecutorService提供了3个方法执行任务，常用的有两个一个是上面例子中的`scheduleAtFixedRate`还有一个是`scheduleWithFixedDelay`，具体参数可以自行查看API文档。

**PS:尽管Executors的工厂方法使用方便，但是在生产场景中被很多企业（尤其是大厂）的开发规范所禁用。原因如下：**

- 固定大小线程池隐患：

  我们看源码：

  ```java
  public static ExecutorService newFixedThreadPool(int nThreads) {
      return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
  }
  ```

  这个快捷线程池使用的是无界队列，如果提交线程的速度大于处理的速度，那么就会出现OOM异常

- 单线程线程池隐患

  源码：

  ```java
  public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
          (new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
  }
  ```

  以上代码首先通过调用工厂方法newFixedThreadPool(1)创建一个数量为1的“固定大小的线程池”，然后使用FinalizableDelegatedExecutorService对该“固定大小的线程池”进行包装，这一层包装的作用是防止线程池的corePoolSize被动态地修改。

  但是这个快捷线程池使用的也是是无界队列，所以会有OOM的风险。

- 可缓存线程池问题

  我们看一下源码：

  ```java
  public static ExecutorService newCachedThreadPool() {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
  }
  ```

  以上代码通过调用ThreadPoolExecutor标准构造器创建一个核心线程数为0、最大线程数不设限制的线程池。所以，理论上“可缓存线程池”可以拥有无数个工作线程，即线程数量几乎无限制。“可缓存线程池”的workQueue为SynchronousQueue同步队列，这个队列类似于一个接力棒，入队出队必须同时传递，正因为“可缓存线程池”可以无限制地创建线程，不会有任务等待，所以才使用SynchronousQueue。

  此线程池由于没有上限，所以会有OOM问题。

  > SynchronousQueue是一个比较特殊的阻塞队列实现类，SynchronousQueue没有容量，每一个插入操作都要等待对应的删除操作，反之每个删除操作都要等待对应的插入操作。也就是说，如果使用SynchronousQueue，提交的任务不会被真实地保存，而是将新任务交给空闲线程执行，如果没有空闲线程，就创建线程，如果线程数都已经大于最大线程数，就执行拒绝策略。使用这种队列需要将maximumPoolSize设置得非常大，从而使得新任务不会被拒绝。

- 可调度线程池问题

  我们看一下源码：
  
  ```java
  public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
      return new ScheduledThreadPoolExecutor(corePoolSize);
  }
  //-->
  public ScheduledThreadPoolExecutor(int corePoolSize) {
          super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                new DelayedWorkQueue());
      }
  ```
  
  我们也可以清晰的看到，这里也使用了几乎无限大的线程上限，因此会有OOM问题。

### 1.5.3 标准线程池创建方式

Executors工厂类中创建线程池的快捷工厂方法实际上是调用ThreadPoolExecutor（定时任务使用ScheduledThreadPoolExecutor）线程池的构造方法完成的。最常用的构造器如下（源码jdk8）：

```java
public ThreadPoolExecutor(int corePoolSize,//核心线程数，线程空闲也不会回收
                          int maximumPoolSize,//线程数的上限
                          long keepAliveTime,//最大空闲时长，如果超过这个时间，默认情况下Idle、非Core线程会被回收
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,//任务的排队队列，用于暂时接收到的异步任务，如果线程池的核心线程都在忙，那么所接收到的目标任务缓存在阻塞队列中。
                          ThreadFactory threadFactory,//产生方式
                          RejectedExecutionHandler handler)//拒绝策略
```

线程池执行器将会根据corePoolSize和maximumPoolSize自动维护线程池中的工作线程。大致规则为：

- 当前工作线程数少于corePoolSize，创建一个新线程来处理该请求，直到线程数达到corePoolSize
- 当前工作线程数多于corePoolSize数量，但小于maximumPoolSize数量，仅当任务排队队列已满时才会创建新线程

> 我们下面会介绍这些ThreadPoolExecutord的这些参数的

### 1.5.4 提交任务

源码中共有以下提交任务的方法：

```java
void execute(Runnable command);
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

- execute()方法只能接收Runnable类型的参数，而submit()方法可以接收Callable、Runnable两种类型的参数。Callable类型的任务是可以返回执行结果的，而Runnable类型的任务不可以返回执行结果。
- submit()提交任务后会有返回值，而execute()没有
- submit()方便Exception处理，因为能抛出异常

> ThreadPoolExecutor类的submit()方法本质上还是执行execute()方法，源码如下：
>
> ```java
> public Future<?> submit(Runnable task) {
>     if (task == null) throw new NullPointerException();
>     RunnableFuture<Void> ftask = newTaskFor(task, null);
>     execute(ftask);
>     return ftask;
> }
> ```

### 1.5.5 线程池的任务调度流程

整体流程可以见下面的流程图

<img src="https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/threadPoolcelue20220418.png" alt="threadPoolcelue20220418" style="zoom:50%;" />

在创建线程池时，如果线程池的参数（如核心线程数量、最大线程数量、BlockingQueue等）配置得不合理，就会出现任务不能被正常调度的问题。我们举个错误的例子：

```java
public static void main(String[] args) {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            1,
            100,
            100,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<Runnable>(50),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());
    for (int i = 0; i < 5; i++) {
        final int taskNo = i;
        threadPoolExecutor.execute(()->{
            System.out.println("index="+taskNo);
            try {
                //极端测试
                Thread.sleep(Long.MAX_VALUE);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    while (true){
        System.out.println("activeCount:"+threadPoolExecutor.getActiveCount()+"taskCount:"+threadPoolExecutor.getTaskCount());
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

执行结果：

~~~
activeCount:1taskCount:5
index=0
activeCount:1taskCount:5
activeCount:1taskCount:5
activeCount:1taskCount:5
activeCount:1taskCount:5
......
~~~

我们会发现只有1个任务在执行，其他的4个任务都在等待。其他任务被加入到了阻塞队列中，需要等线程执行完第一个任务后，才能依次从阻塞队列取出执行。但是，实例中的第一个任务是一个永远也没有办法完成的任务，所以其他的4个任务只能永远在阻塞队列中等待着。

### 1.5.6 线程工厂

线程工厂主要用于创建线程，线程工厂的接口十分简单，如下：

```java
public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}
```

在调用ThreadFactory的唯一方法newThread()创建新线程时，可以更改所创建的新线程的名称、线程组、优先级、守护进程状态等。如果newThread()的返回值为null，表示线程工厂未能成功创建线程，线程池可能无法执行任何任务。

如果基于自定义的ThreadFactory实例创建线程池，首先需要实现一个ThreadFactory类，实现其唯一的抽象方法newThread(Runnable)。举例：

```java
public class ThreadFactoryDemo {
    static class MyThreadFactory implements ThreadFactory{

        private AtomicInteger threadNo = new AtomicInteger(1);
        @Override
        public Thread newThread(Runnable r) {
            String name ="myThread-"+threadNo.get();
            System.out.println("create Thread,and its name is "+name);
            threadNo.incrementAndGet();
            Thread thread = new Thread(r,name);
            thread.setDaemon(false);
            return thread;
        }
    }

    public static void main(String[] args) {
        ExecutorService service = Executors.newFixedThreadPool(1, new MyThreadFactory());
        service.execute(() ->{
            System.out.println("running");
            return;
        });
    }
}
```

执行结果：

create Thread,and its name is myThread-1
running

### 1.5.7 任务阻塞队列

Java中的阻塞队列（BlockingQueue）与普通队列相比有一个重要的特点：在阻塞队列为空时会阻塞当前线程的元素获取操作。具体来说，在一个线程从一个空的阻塞队列中获取元素时线程会被阻塞，直到阻塞队列中有了元素；当队列中有元素后，被阻塞的线程会自动被唤醒（唤醒过程不需要用户程序干预）。

Java线程池使用BlockingQueue实例暂时接收到的异步任务，BlockingQueue是JUC包的一个超级接口，比较常用的实现类有：

1. ArrayBlockingQueue：是一个数组实现的有界阻塞队列（有界队列），队列中的元素按FIFO排序。ArrayBlockingQueue在创建时必须设置大小，接收的任务超出corePoolSize数量时，任务被缓存到该阻塞队列中，任务缓存的数量只能为创建时设置的大小，若该阻塞队列已满，则会为新的任务创建线程，直到线程池中的线程总数大于maximumPoolSize。
2. LinkedBlockingQueue：是一个基于链表实现的阻塞队列，按FIFO排序任务，可以设置容量（有界队列），不设置容量则默认使用Integer.Max_VALUE作为容量（无界队列）。该队列的吞吐量高于ArrayBlockingQueue。如果不设置LinkedBlockingQueue的容量（无界队列），当接收的任务数量超出corePoolSize时，则新任务可以被无限制地缓存到该阻塞队列中，直到资源耗尽。有两个快捷创建线程池的工厂方法Executors.newSingleThreadExecutor和Executors.newFixedThreadPool使用了这个队列，并且都没有设置容量（无界队列）。
3. PriorityBlockingQueue：是具有优先级的无界队列。
4. DelayQueue：这是一个无界阻塞延迟队列，底层基于PriorityBlockingQueue实现，队列中每个元素都有过期时间，当从队列获取元素（元素出队）时，只有已经过期的元素才会出队，队列头部的元素是过期最快的元素。快捷工厂方法Executors.newScheduledThreadPool所创建的线程池使用此队列。
5. SynchronousQueue：（同步队列）是一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程的调用移除操作，否则插入操作一直处于阻塞状态，其吞吐量通常高于LinkedBlockingQueue。快捷工厂方法Executors.newCachedThreadPool所创建的线程池使用此队列。与前面的队列相比，这个队列比较特殊，它不会保存提交的任务，而是直接新建一个线程来执行新来的任务。

类图如下：

![image-20220418152044671](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220418152044671.png)

### 1.5.8 线程池钩子函数

ThreadPoolExecutor线程池调度器为每个任务执行前后都提供了钩子方法。ThreadPoolExecutor类提供了三个钩子方法（空方法），这三个钩子方法一般用作被子类重写，具体如下：

```java
protected void beforeExecute(Thread t, Runnable r) { }
protected void afterExecute(Runnable r, Throwable t) { }
protected void terminated() { }//线程池终止
```

beforeExecute和afterExecute两个方法在每个任务执行前后被调用，如果钩子（回调方法）引发异常，内部工作线程可能失败并突然终止。

举个钩子函数的例子：

```java
public class TestHooks {
    public static void main(String[] args) {
        ThreadLocal<Long> time =new ThreadLocal<>();
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2,
                4,
                60, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(2),
                //这里用1.5.6中的线程工厂
                new ThreadFactoryDemo.MyThreadFactory()){
            @Override
            protected void beforeExecute(Thread t, Runnable r) {
                System.out.println("["+Thread.currentThread().getName()+"]before is running");
                time.set(System.currentTimeMillis());
                super.beforeExecute(t, r);
            }

            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                super.afterExecute(r, t);
                long tempTime = System.currentTimeMillis() - time.get();
                System.out.println("["+Thread.currentThread().getName()+"]after is running .all time is"+tempTime);
                time.remove();
            }

            @Override
            protected void terminated() {
                super.terminated();
                System.out.println("调度器终止");
            }
        };
        for (int i = 0; i < 5; i++) {
            executor.execute(() -> {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("["+Thread.currentThread().getName()+"]i am running!");
            });
        }
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("shutdown pool");
        executor.shutdown();

    }
}
```

~~~
执行结果：
create Thread,and its name is myThread-1
create Thread,and its name is myThread-2
create Thread,and its name is myThread-3
[myThread-1]before is running
[myThread-2]before is running
[myThread-3]before is running
[myThread-2]i am running!
[myThread-1]i am running!
[myThread-1]after is running .all time is3012
[myThread-3]i am running!
[myThread-1]before is running
[myThread-3]after is running .all time is3012
[myThread-3]before is running
[myThread-2]after is running .all time is3012
[myThread-1]i am running!
[myThread-1]after is running .all time is3011
[myThread-3]i am running!
[myThread-3]after is running .all time is3011
shutdown pool
调度器终止
~~~

我们可以看到，在线程执行之前和之后会调用钩子方法。上面的例子中因为核心线程有两个，队列有两个空位置，且最大线程有4个，因此我们会创建出三个线程来（核心线程塞满，队列塞满，但没超最大线程）。当关闭线程池后执行terminated钩子方法。

### 1.5.9 拒绝策略

在线程池的任务缓存队列为有界队列（有容量限制的队列）的时候，如果队列满了，提交任务到线程池的时候就会被拒绝。总体来说，任务被拒绝有两种情况：

1. 线程池已经被关闭。
2. 工作队列已满且maximumPoolSize已满。

无论以上哪种情况任务被拒绝，线程池都会调用RejectedExecutionHandler实例的rejectedExecution方法。

RejectedExecutionHandler是拒绝策略的接口，JUC为该接口提供了以下几种实现：

![image-20220418155804778](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220418155804778.png)

- 拒绝策略：使用该策略时，如果线程池队列满了，新任务就会被拒绝，并且抛出RejectedExecutionException异常。该策略是线程池默认的拒绝策略。
- 抛弃策略 DiscardPolicy：该策略是AbortPolicy的Silent（安静）版本，如果线程池队列满了，新任务就会直接被丢掉，并且不会有任何异常抛出。
- 抛弃最老任务 DiscardOldestPolicy：抛弃最老任务策略，也就是说如果队列满了，就会将最早进入队列的任务抛弃，从队列中腾出空间，再尝试加入队列。因为队列是队尾进队头出，队头元素是最老的，所以每次都是移除队头元素后再尝试入队。
- 调用者执行策略。在新任务被添加到线程池时，如果添加失败，那么提交任务线程会自己去执行该任务，不会使用线程池中的线程去执行新任务。

当然，也可以自定义策略，这里给出示例：

```java
public class MyRejectDemo {

    public static class MyRejectPolicy implements RejectedExecutionHandler {

        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            System.out.println(r + "rejected;"+"getTaskCount:"+executor.getTaskCount());
        }
    }
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(2, 4, 10, 
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(2), 
                new ThreadFactoryDemo.MyThreadFactory(), //这里用1.5.6中的线程工厂
                new MyRejectPolicy());
        //预启动核心线程
        executor.prestartAllCoreThreads();
        for (int i = 0; i < 10; i++) {
            executor.execute(()->{
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("["+Thread.currentThread().getName()+"]i am running!");
            });
        }
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("关闭线程池");
        executor.shutdown();
    }
}
```

执行结果：

~~~
create Thread,and its name is myThread-1
create Thread,and its name is myThread-2
create Thread,and its name is myThread-3
create Thread,and its name is myThread-4
com.learn.testnginx.threadpool.MyRejectDemo$$Lambda$1/1867750575@4783da3frejected;getTaskCount:6
com.learn.testnginx.threadpool.MyRejectDemo$$Lambda$1/1867750575@4783da3frejected;getTaskCount:6
com.learn.testnginx.threadpool.MyRejectDemo$$Lambda$1/1867750575@4783da3frejected;getTaskCount:6
com.learn.testnginx.threadpool.MyRejectDemo$$Lambda$1/1867750575@4783da3frejected;getTaskCount:6
[myThread-2]i am running!
[myThread-4]i am running!
[myThread-3]i am running!
[myThread-1]i am running!
[myThread-2]i am running!
[myThread-4]i am running!
关闭线程池
~~~

我们可以看到最后拒绝了四个，执行了六个。因为最大线程4个加队列中两个一共六个，最后四个就会被拒绝。而且拒绝也是执行我们自己的拒绝策略。

### 1.5.10 线程池的状态和优雅关闭

我们可以先看一下线程池的状态，如下：

```java
// runState is stored in the high-order bits
//线程池创建后的初始状态，这种状态下可以执行任务
private static final int RUNNING    = -1 << COUNT_BITS;
//该状态下线程池不再接受新任务，但是会将工作队列中的任务执行完毕。
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//该状态下线程池不再接受新任务，也不会处理工作队列中的剩余任务，并且将会中断所有工作线程。
private static final int STOP       =  1 << COUNT_BITS;
//该状态下所有任务都已终止或者处理完成，将会执行terminated()钩子方法。
private static final int TIDYING    =  2 << COUNT_BITS;
//执行完terminated()钩子方法之后的状态。
private static final int TERMINATED =  3 << COUNT_BITS;
```

线程池状态转换如下：

![threadpoolStatus20220418](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/threadpoolStatus20220418.png)

优雅地关闭线程池主要涉及的方法有3个：

1. shutdown：源码如下：

   ```java
   //启动有序关闭，在此过程中执行之前提交的任务，但不接受新任务。调用如果已经关闭，则没有其他效果。
   //此方法不等待以前提交的任务完成执行。使用awaitTermination来完成。
   public void shutdown() {
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           //检查状态
           checkShutdownAccess();
           //修改状态
           advanceRunState(SHUTDOWN);
           //中断空闲线程
           interruptIdleWorkers();
           onShutdown(); // hook for ScheduledThreadPoolExecutor
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
   }
   ```

2. shutdownNow

   源码：

   ```java
   /*尝试停止所有正在执行的任务，停止等待任务的处理，并返回等待执行的任务列表。从该方法返回时，这些任务将从任务队列中抽干(删除)。
   此方法不等待主动执行的任务终止。使用awaitTermination来完成。
   除了尽最大努力停止处理正在积极执行的任务外，没有任何保证。这个实现通过Thread.interrupt来取消任务，因此任何无法响应中断的任务都可能永远不会终止。*/
   public List<Runnable> shutdownNow() {
       List<Runnable> tasks;
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
           //检查状态
           checkShutdownAccess();
           //修改状态
           advanceRunState(STOP);
           //中断所有线程
           interruptWorkers();
           //丢弃队列中剩余任务
           tasks = drainQueue();
       } finally {
           mainLock.unlock();
       }
       tryTerminate();
       return tasks;
   }
   ```

3. awaitTermination

   调用了线程池的shutdown()与shutdownNow()方法之后，用户程序都不会主动等待线程池关闭完成，如果需要等待线程池关闭完成，需要调用awaitTermination()进行主动等待。

   此方法源码如下：

   ```java
   public boolean awaitTermination(long timeout, TimeUnit unit)
       throws InterruptedException {
       long nanos = unit.toNanos(timeout);
       final ReentrantLock mainLock = this.mainLock;
       mainLock.lock();
       try {
       //等待
           for (;;) {
           //如果线程池完成关闭，awaitTermination()方法将会返回true，
           //否则当等待时间超过指定时间后将会返回false。
               if (runStateAtLeast(ctl.get(), TERMINATED))
                   return true;
               if (nanos <= 0)
                   return false;
               nanos = termination.awaitNanos(nanos);
           }
       } finally {
           mainLock.unlock();
       }
   }
   ```

我们在关闭线程池时可以遵循以下步骤：

1. 执行shutdown()方法，拒绝新任务的提交，并等待所有任务有序地执行完毕。
2. 执行awaitTermination方法，指定超时时间，判断是否已经关闭所有任务，线程池关闭完成。
3. 如果awaitTermination()方法返回false，或者被中断，就调用shutDownNow()方法立即关闭线程池所有任务。
4. 补充执行awaitTermination方法，判断线程池是否关闭完成。如果超时，就可以进入循环关闭，循环一定的次数（如1000次），不断关闭线程池，直到其关闭或者循环结束。

给出例子：

```java
public class ShutdownPoolDemo {
    public static void shutdownThreadPoolGracefully(ExecutorService pool) {
        //已经关闭返回
        if (pool.isTerminated()||!(pool instanceof ExecutorService)){
            return;
        }
        try{
            pool.shutdown();
        }catch (SecurityException | NullPointerException e){
            return;
        }
        //等待关闭
        try {
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)){
                pool.shutdownNow();
            }
            if (!pool.awaitTermination(60, TimeUnit.SECONDS)){
                System.err.println("线程池任务为正常执行结束");
            }
        } catch (InterruptedException e) {
            pool.shutdownNow();
        }
        //还没关闭，循环关闭1000次
        try {
            for (int i = 0; i < 1000; i++) {
                if (pool.awaitTermination(10, TimeUnit.MILLISECONDS)) {
                    break;
                }
                pool.shutdownNow();
            }
        } catch (Throwable e){
            System.err.println(e.getMessage());
        }

    }
}
```

当然，我们也可以使用JVM钩子函数来关闭线程池。

```java
static {
    Runtime.getRuntime().addShutdownHook(new Thread(()-> shutdownThreadPoolGracefully(EXECUTOR)));
}
```

## 1.6 线程池的线程数参考

使用标准构造器ThreadPoolExecutor创建线程池时，会涉及线程数的配置，而线程数的配置与异步任务类型是分不开的。这里将线程池的异步任务大致分为以下三类：

1. IO密集型任务。此类任务主要是执行IO操作。由于执行IO操作的时间较长，导致CPU的利用率不高，这类任务CPU常处于空闲状态。Netty的IO读写操作为此类任务的典型例子。
2. CPU密集型任务。此类任务主要是执行计算任务。由于响应时间很快，CPU一直在运行，这种任务CPU的利用率很高。
3. 混合型任务。此类任务既要执行逻辑计算，又要进行IO操作（如RPC调用、数据库访问）。相对来说，由于执行IO操作的耗时较长（一次网络往返往往在数百毫秒级别），这类任务的CPU利用率也不是太高。Web服务器的HTTP请求处理操作为此类任务的典型例子。

### 1.6.1 IO密集型示例

由于IO密集型任务的CPU使用率较低，导致线程空余时间很多，因此通常需要开CPU核心数两倍的线程。当IO线程空闲时，可以启用其他线程继续使用CPU，以提高CPU的使用率。

```java
public class IOThreadUtil {
    /**
     * CPU核数
     */
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    /**
     * IO处理线程数
     * corePoolSize和maximumPoolSize保持一致，使得在接收到新任务时，如果没有空闲工作线程，就优先创建新的线程去执行新任务，
     * 而不是优先加入阻塞队列，等待现有工作线程空闲后再执行。
     */
    private static final int IO_MAX = Math.max(2,CPU_COUNT*2);
    /**
     * 保活时长:秒
     */
    private static final int KEEP_ALIVE_SECONDS = 30;
    /**
     * 有界队列size
     */
    private static final int QUEUE_SIZE = 128;

    private static class IoIntenseTargetThreadPoolLazyHolder{
        private static final ThreadPoolExecutor EXECUTOR = new ThreadPoolExecutor(
                IO_MAX,
                IO_MAX,
                KEEP_ALIVE_SECONDS,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(QUEUE_SIZE),
                new MyThreadFactory("io")
        );
        static {
            //keepAliveTime参数所设置的Idle超时策略也将被应用于核心线程，当池中的线程长时间空闲时，可以自行销毁。
            EXECUTOR.allowCoreThreadTimeOut(true);
            Runtime.getRuntime().addShutdownHook(new Thread(()-> shutdownThreadPoolGracefully(EXECUTOR)));
        }

    }

    private IOThreadUtil() {
    }
    //懒汉式单例模式
    public static ThreadPoolExecutor getExecutor(){
        return IoIntenseTargetThreadPoolLazyHolder.EXECUTOR;
    }

    static class MyThreadFactory implements ThreadFactory {
        private final String name;
        public MyThreadFactory(String name) {
            this.name = name;
        }
        private final AtomicInteger threadNo = new AtomicInteger(1);
        @Override
        public Thread newThread(Runnable r) {
            System.out.println("create Thread,and its name is "+name+threadNo);
            Thread thread = new Thread(r,name+threadNo);
            threadNo.incrementAndGet();
            thread.setDaemon(false);
            return thread;
        }
    }

    /**
     * 测试类
     */
    public static void main(String[] args) {
        ThreadPoolExecutor executor = IOThreadUtil.getExecutor();
    }

}
```

### 1.6.2 CPU密集型示例

CPU密集型任务也叫计算密集型任务，其特点是要进行大量计算而需要消耗CPU资源，比如计算圆周率、对视频进行高清解码等。CPU密集型任务虽然也可以并行完成，但是并行的任务越多，花在任务切换的时间就越多，CPU执行任务的效率就越低，所以要最高效地利用CPU，CPU密集型任务并行执行的数量应当等于CPU的核心数。其余代码与上面一致，这里仅给出核心代码：

```java
/**
 * CPU密集型任务处理线程数
 */
private static final int CPU_TASK_MAX = CPU_COUNT;
private static class CPUIntenseTargetThreadPoolLazyHolder{
    private static final ThreadPoolExecutor EXECUTOR = new ThreadPoolExecutor(
            CPU_TASK_MAX,
            CPU_TASK_MAX,
            KEEP_ALIVE_SECONDS,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(QUEUE_SIZE),
            new MyThreadFactory("cpu")
    );
    static {
        //keepAliveTime参数所设置的Idle超时策略也将被应用于核心线程，当池中的线程长时间空闲时，可以自行销毁。
        EXECUTOR.allowCoreThreadTimeOut(true);
        Runtime.getRuntime().addShutdownHook(new Thread(()-> shutdownThreadPoolGracefully(EXECUTOR)));
    }

}
```

### 1.6.3 混合型任务示例

混合型任务既要执行逻辑计算，又要进行大量非CPU耗时操作（如RPC调用、数据库访问、网络通信等），所以**混合型任务CPU的利用率不是太高，非CPU耗时往往是CPU耗时的数倍**。比如在Web应用中处理HTTP请求时，一次请求处理会包括DB操作、RPC操作、缓存操作等多种耗时操作。一般来说，一次Web请求的CPU计算耗时往往较少，大致在100～500毫秒，而其他耗时操作会占用500～1000毫秒，甚至更多的时间。在为混合型任务创建线程池时，如何确定线程数呢？业界有一个比较成熟的估算公式（当然公式的估算结果仅仅是理论最佳值，在生产环境中的使用也仅供参考。生产环境需要结合系统网络环境和硬件情况（CPU、内存、硬盘读写速度）不断尝试，获取一个符合实际的线程数值。），具体如下：

> 最佳线程数 = （（线程等待时间 + 线程CPU时间）/线程CPU时间）*CPU核数

**等待时间所占的比例越高，需要的线程就越多；CPU耗时所占的比例越高，需要的线程就越少。**下面举一个例子：比如在Web服务器处理HTTP请求时，假设平均线程CPU运行时间为100毫秒，而线程等待时间（比如包括DB操作、RPC操作、缓存操作等）为900毫秒，如果CPU核数为8，那么根据上面这个公式，估算如下：

> (900ms+100ms)/100ms *8 = 80

这里同样只给出核心代码：

```java
private static final int MIXED_MAX = 128;
//可以替换为配置文件等
private static final String MIXED_THREAD_AMOUNT = "mixed.thread.amount";

private static class MixedIntenseTargetThreadPoolLazyHolder{
    private static final int max = (System.getProperty(MIXED_THREAD_AMOUNT)!= null)
            ? Integer.parseInt(System.getProperty(MIXED_THREAD_AMOUNT))
            :MIXED_MAX;
    private static final ThreadPoolExecutor EXECUTOR = new ThreadPoolExecutor(
            max,
            max,
            KEEP_ALIVE_SECONDS,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(QUEUE_SIZE),
            new MyThreadFactory("mixed")
    );
    static {
        //keepAliveTime参数所设置的Idle超时策略也将被应用于核心线程，当池中的线程长时间空闲时，可以自行销毁。
        EXECUTOR.allowCoreThreadTimeOut(true);
        Runtime.getRuntime().addShutdownHook(new Thread(()-> shutdownThreadPoolGracefully(EXECUTOR)));
    }
}
 /**
  * 测试类
  */
public static void main(String[] args) {
    System.setProperty("mixed.thread.amount", String.valueOf(80));
    ThreadPoolExecutor executor = IOThreadUtil.getExecutor();
    System.out.println("size"+executor.getCorePoolSize());
    for (int i = 0; i < 100; i++) {
        try {
            Thread.sleep(10000);
            executor.execute(()->{
                try {
                    System.out.println("["+Thread.currentThread().getName()+"] is running");
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    try {
        Thread.sleep(10000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("关闭线程池");
}
```

## 1.7 ThreadLocal

### 1.7.1 基本使用

ThreadLocal位于JDK的java.lang核心包中。如果程序创建了一个ThreadLocal实例，那么在访问这个变量的值时，每个线程都会拥有一个独立的、自己的本地值。“线程本地变量”可以看成专属于线程的变量，不受其他线程干扰，保存着线程的专属数据。当线程结束后，每个线程所拥有的那个本地值会被释放。在多线程并发操作“线程本地变量”的时候，线程各自操作的是自己的本地值，从而规避了线程安全问题。

> 这个ThreadLocal可以看作是一个Map,key存放线程实例，value存放线程本地变量。“线程本地变量”中绑定的值为Value（本地值）。早期版本中的Map结构，其拥有者为ThreadLocal，每一个ThreadLocal实例拥有一个Map实例。
>
> 在JDK 8版本中，ThreadLocal的内部结构发生了变化，虽然还是使用了Map结构，但是Map结构的拥有者已经发生了变化，其拥有者为Thread（线程）实例，每一个Thread实例拥有一个Map实例。另外，Map结构也发生了变化，其中ThreadLocal实例为Key，本地数据为Value。

ThreadLocal常用有三个方法操作本地变量：

~~~java
public T get(){//略}
public void set(T value){//略}
public void remove(){//略}
~~~

这里给出一个ThreadLocal的使用示例：

```java
public class TestLocal {

    @Data
    static class MyPojo{
        static final AtomicInteger AMOUNT = new AtomicInteger(0);
        int index = 0;
        int bar =10;

        public MyPojo() {
            //总数增加并给对象编号
            index = AMOUNT.incrementAndGet();
        }
    }
    private static final ThreadLocal<MyPojo> LOCAL_MYPOJO = new ThreadLocal<>();

    public static void main(String[] args) {
        //获取自定义混合型线程池
        ThreadPoolExecutor executor = IOThreadUtil.getExecutor();
        for (int i = 0; i < 5; i++) {
            executor.execute(() -> {
                //设置当前线程绑定的值
                if (LOCAL_MYPOJO.get()==null){
                    LOCAL_MYPOJO.set(new MyPojo());
                }
                System.out.println("初始本地值"+LOCAL_MYPOJO.get());
                for (int j = 0; j < 10; j++) {
                    MyPojo myPojo = LOCAL_MYPOJO.get();
                    myPojo.setBar(myPojo.getBar()+1);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("累加10次后的本地值"+LOCAL_MYPOJO.get());
                //删除当前线程绑定的值，对线程池中的线程很重要
                LOCAL_MYPOJO.remove();
            });
        }
    }
}
```

执行结果略，本质上是了解ThreadLocal的使用方式，可以自行跑一下上面的代码。

### 1.7.2 使用场景

ThreadLocal是解决线程安全问题的一个较好的方案，它通过为每个线程提供一个独立的本地值去解决并发访问的冲突问题。ThreadLocal的使用场景大致可以分为两类：

1. 线程隔离，ThreadLocal中的数据只属于当前线程，其本地值对别的线程是不可见的，在多线程环境下，可以防止自己的变量被其他线程篡改。另外，由于各个线程之间的数据相互隔离，避免了同步加锁带来的性能损失，大大提升了并发性的性能。ThreadLocal在线程隔离的常用案例为：可以为每个线程绑定一个用户会话信息、数据库连接、HTTP请求等，这样一个线程所有调用到的处理函数都可以非常方便地访问这些资源。常见的ThreadLocal使用场景为数据库连接独享、Session数据管理等。在“线程隔离”场景中，使用ThreadLocal的典型案例为：可以为每个线程绑定一个数据库连接，使得这个数据库连接为线程所独享，从而避免数据库连接被混用而导致操作异常问题。
2. 跨函数传递数据。通常用于同一个线程内，跨类、跨方法传递数据时，如果不用ThreadLocal，那么相互之间的数据传递势必要靠返回值和参数，这样无形之中增加了这些类或者方法之间的耦合度。由于ThreadLocal的特性，同一线程在某些地方进行设置，在随后的任意地方都可以获取到。线程执行过程中所执行到的函数都能读写ThreadLocal变量的线程本地值，从而可以方便地实现跨函数的数据传递。在“跨函数传递数据”场景中使用ThreadLocal的典型案例为：可以为每个线程绑定一个Session（用户会话）信息，这样一个线程所有调用到的代码都可以非常方便地访问这个本地会话，而不需要通过参数传递。

### 1.7.3 ThreadLocal源码分析

ThreadLocal类的结构如下图：

![image-20220419131126046](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220419131126046.png)

我们先看这几个常用方法：

set方法：

```java
public void set(T value) {
    //获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    //如果map不为空，将当前ThreadLocal设为key，将值存入map
    if (map != null)
        map.set(this, value);
    //如果为空，则为该线程创建map，然后设置key-value
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

get方法：

```java
public T get() {
    //获取当前线程和当前线程的ThreadLocalMap成员
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //如果map不为空，则以当前ThreadLocal为key获取map中的Entry
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        //如果Entry不为空则返回value
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果为空调用setInitialValue方法
    return setInitialValue();
}

private T setInitialValue() {
    //调用钩子函数initialValue获取ThreadLocal初始值，并存入ThreadLocalMap
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

remove方法：移除当前线程的本地变量

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

initialValue方法：当“线程本地变量”在当前线程的ThreadLocalMap中尚未绑定值时，此方法用于获取初始值。

```java
//官方文档：Returns the current thread's "initial value" for this thread-local variable. This method will be invoked the first time a thread accesses the variable with the get method, unless the thread previously invoked the set method, in which case the initialValue method will not be invoked for the thread. Normally, this method is invoked at most once per thread, but it may be invoked again in case of subsequent invocations of remove followed by get.
//This implementation simply returns null; if the programmer desires thread-local variables to have an initial value other than null, ThreadLocal must be subclassed, and this method overridden. Typically, an anonymous inner class will be used.
//大意是这个方法一般只调用一次。且这是个钩子函数默认返回null，如果不想初始值为null就重写这个方法，通常用匿名内部类重写。
protected T initialValue() {
    return null;
}
```

> 当然，其实没有必要继承并重写此方法，JDK提供了一个静态子类SuppliedThreadLocal，并提供了个静态工厂方法，方便定义ThreadLocal时设置初始值：
>
> ```java
> public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
>     return new SuppliedThreadLocal<>(supplier);
> }
> ```
>
> 用法：
>
> ```java
> private static final ThreadLocal<MyPojo> LOCAL_MYPOJO = ThreadLocal.withInitial(MyPojo::new);
> ```

当然，ThreadLocal中最重要的还是ThreadLocalMap这个内部类,其实现了一套简单的Map结构，所有的方法基本都围绕者这个来进行，下面我们来分析这个内部类：

![image-20220419142546944](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220419142546944.png)

如上图，这个内部类属性和map类似，实现一个简单的Map结构。对于ThreadLocalMap其中的set、get等方法可以自行查看源码，因为本质和map的方法相似，比如以set方法为例：

![image-20220419143644950](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220419143644950.png)

下面我们主要分析一下，Entry的key为什么需要使用弱引用包裹，

> Java中的弱引用就是：弱引用的对象拥有更短暂的生命周期。GC线程在扫描它所管辖的内存区域的过程中，一旦发现只具有弱引用的对象，就会回收掉这些被弱引用关联的对象。也就是说，无论当前内存是否紧缺，GC都会回收被弱引用关联的对象。

我们举个栗子：

```java
public class TestLocalMapEntry {
    public static void main(String[] args) {
        test();
    }
    public static void test(){
        ThreadLocal<Integer> local = new ThreadLocal<>();
        local.set(100);
        local.get();
    }
}
```

当线程执行完test()方法后，test()的方法栈帧将被销毁，强引用local的值也就没有了，但此时线程的ThreadLocalMap中对应的Entry的Key引用还指向ThreadLocal实例。如果Entry的Key引用是强引用，就会导致Key引用指向的ThreadLocal实例及其Value值都不能被GC回收，这将造成严重的内存泄漏问题。如下图：

![ThreadLocalMap20220419](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/ThreadLocalMap20220419.png)

> 什么是内存泄漏？不再用到的内存没有及时释放（归还给系统），就叫作内存泄漏。对于持续运行的服务进程必须及时释放内存，否则内存占用率越来越高，轻则影响系统性能，重则导致进程崩溃甚至系统崩溃。

**使用ThreadLocal的规范**

ThreadLocal实例作为ThreadLocalMap的Key，针对一个线程内的所有操作是共享的，所以建议设置static修饰符，以便被所有的对象共享。由于静态变量会在类第一次被使用时装载，只会分配一次存储空间，此类的所有实例都会共享这个存储空间，所以使用static修饰ThreadLocal就会节约内存空间。另外，为了确保ThreadLocal实例的唯一性，除了使用static修饰之外，还会使用final进行加强修饰，以防止其在使用过程中发生动态变更。

但是使用static、final修饰ThreadLocal实例会使得Thread实例内部的ThreadLocalMap中Entry的Key在Thread实例的生命期内将始终保持为非null，从而导致Key所在的Entry不会被自动清空，这就会让Entry中的Value指向的对象一直存在强引用。**因此需要在使用后调用remove方法**

~~~java
private static final ThreadLocal<MyPojo> LOCAL_MYPOJO = ThreadLocal.withInitial(MyPojo::new);
~~~

> 如果使用线程池，可以在afterExecute中调用remote进行释放。
>
> ```java
> @Override
> protected void afterExecute(Runnable r, Throwable t) {
>     threadLocal.remove();
> }
> ```

