# Java高并发编程学习笔记

# 一、Java多线程

## 1.1 线程和进程

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

FutureTask类是Future接口的实现类，提供了对异步任务的操作的具体实现。但是，FutureTask类不仅实现了Future接口，还实现了Runnable接口，或者更加准确地说，FutureTask类实现了RunnableFuture接口。继承类图如下：

![image-20211213212843822](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgoimage-20211213212843822.png)

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
            System.out.println("做自己的事情");
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

现代操作系统（如Windows、Linux、Solaris）提供了强大的线程管理能力，Java不需要再进行独立的线程管理和调度，而是将线程调度工作委托给操作系统的调度进程去完成。在某些系统（比如Solaris操作系统）上，JVM甚至将每个Java线程一对一地对应到操作系统的本地线程，彻底将线程调度委托给操作系统。

### 1.3.1 线程的调度与时间片

由于CPU的计算频率非常高，每秒钟现代操作系统（如Windows、Linux、Solaris）提供了强大的线程管理能力，Java不需要再进行独立的线程管理和调度，而是将线程调度工作委托给操作系统的调度进程去完成。在某些系统（比如Solaris操作系统）上，JVM甚至将每个Java线程一对一地对应到操作系统的本地线程，彻底将线程调度委托给操作系统。

目前操作系统中**主流的线程调度方式是：基于CPU时间片方式进行线程调度**。线程只有得到CPU时间片才能执行指令，处于执行状态，没有得到时间片的线程处于就绪状态，等待系统分配下一个CPU时间片。由于时间片非常短，在各个线程之间快速地切换，因此表现出来的特征是很多个线程在“同时执行”或者“并发执行”。**线程的调度模型主要分为两种：分时调度和抢占式调度。**分时调度就是系统平均分配CPU的时间片，所有线程轮流占用CPU；抢占式调度模型就是系统按照线程优先级分配时间片，优先级高的线程优先分配CPU时间片，如果所有就绪线程的优先级相同，那么会随机选择一个。

### 1.3.2 线程的优先级

在Thread类中有一个实例属性和两个实例方法，专门用于进行线程优先级相关的操作。与线程优先级相关的成员属性为：`priority`,方法为

~~~java
public final int getPriority();//获取线程优先级。
public final void setPriority(int priority);//设置线程优先级。
~~~

Thread实例的priority属性默认是级别5，对应的类常量是NORM_PRIORITY。优先级最大值为10，最小值为1.



