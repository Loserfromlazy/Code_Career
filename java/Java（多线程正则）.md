# Java多线程

## 一、概述

多线程听上去是非常专业的概念，其实非常简单——单线程的程序（前面介绍的绝大部分程序）只有一个顺序执行流，多线程的程序则可以包括多个顺序执行流，多个顺序流之间互不干扰。

一般而言，进程包含如下3个特征。

独立性：进程是系统中独立存在的实体，它可以拥有自己独立的资源，每一个进程都拥有自己私有的地址空间。在没有经过进程本身允许的情况下，一个用户进程不可以直接访问其他进程的地址空间。

动态性：进程与程序的区别在于，程序只是一个静态的指令集合，而进程是一个正在系统中活动的指令集合。在进程中加入了时间的概念。进程具有自己的生命周期和各种不同的状态，这些概念在程序中都是不具备的。

并发性：多个进程可以在单个处理器上并发执行，多个进程之间不会互相影响。

> 并发性（concurrency）和并行性（parallel）是两个概念，并行指在同一时刻，有多条指令在多个处理器上同时执行；并发指在同一时刻只能有一条指令执行，但多个进程指令被快速轮换执行，使得在宏观上具有多个进程同时执行的效果。

多线程则扩展了多进程的概念，使得同一个进程可以同时并发处理多个任务。线程（Thread）也被称作轻量级进程（Lightweight Process），线程是进程的执行单元。就像进程在操作系统中的地位一样，线程在程序中是独立的、并发的执行流。当进程被初始化后，主线程就被创建了。对于绝大多数的应用程序来说，通常仅要求有一个主线程，但也可以在该进程内创建多条顺序执行流，这些顺序执行流就是线程，每个线程也是互相独立的。

线程是进程的组成部分，一个进程可以拥有多个线程，一个线程必须有一个父进程。线程可以**拥有自己的堆栈、自己的程序计数器和自己的局部变量，但不拥有系统资源**，它与父进程的其他线程共享该进程所拥有的全部资源。因为多个线程共享父进程里的全部资源，因此编程更加方便；但必须更加小心，我们必须确保线程不会妨碍同一进程里的其他线程。

**多线程的优势**

进程之间不能共享内存，但线程之间共享内存非常容易。

系统创建进程时需要为该进程重新分配系统资源，但创建线程则代价小得多，因此使用多线程来实现多任务并发比多进程的效率高。

Java语言内置了多线程功能支持，而不是单纯地作为底层操作系统的调度方式，从而简化了Java的多线程编程。

## 二、线程的创建与使用

### **通过Thread类创建线程类**

> 使用继承Thread类的方法来创建线程类时，多个线程之间无法共享线程类的实例变量。

~~~java
public class test extends Thread {
    private int i;
    public void run(){
        for (;i<100;i++){
            //Thread
            // Thread对象课直接调用getName()方法获取当i去哪线程名字
            System.out.println(getName()+" "+i);
        }
    }
    public static void main(String[] args) throws Exception{
        for (int i = 0; i <100 ; i++) {
            System.out.println(Thread.currentThread().getName()+" "+i);
            if (i==20){
                test t=new test();
                new test().start();
                new test().start();
            }
        }
    }
}
~~~

虽然上面程序只显式地创建并启动了2个线程，但实际上程序有3个线程，即程序显式创建的2个子线程和主线程。

### 实现Runnable接口创建线程类

Runnable对象仅仅作为Thread对象的target，Runnable实现类里包含的run()方法仅作为线程执行体。而实际的线程对象依然是Thread实例，只是该Thread线程负责执行其target的run()方法

~~~java
public class test implements Runnable {
    private int i;
    public void run(){
        for (;i<100;i++){
            //当时先Runnable接口时获取当前线程时只能用Thread.currentThread().getName()方法
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
    }
    public static void main(String[] args) throws Exception{
        for (int i = 0; i <100 ; i++) {
            System.out.println(Thread.currentThread().getName()+" "+i);
            if (i==20){
                test t=new test();
                new Thread(t,"新线程1").start();
                new Thread(t,"新线程2").start();
            }
        }
    }
}
~~~

通过继承Thread类来获得当前线程对象比较简单，直接使用this就可以了；但通过实现Runnable接口来获得当前线程对象，则必须使用Thread.currentThread()方法。

程序所创建的Runnable对象只是线程的target，而多个线程可以共享同一个target，所以多个线程可以共享同一个线程类（实际上应该是线程的target类）的实例属性。

### 使用Callable和Future创建线程

实现Runnable接口创建多线程时，Thread类的作用就是把run()方法包装成线程执行体。那么是否可以直接把任意方法都包装成线程执行体呢？Java目前不行！但C#可以（C#可以把任意方法包装成线程执行体，包括有返回值的方法）。

从Java 5开始，Java提供了Callable接口，该接口怎么看都像是Runnable接口的增强版，Callable接口提供了一个call()方法可以作为线程执行体，但call()方法比run()方法功能更强大。

call()方法可以有返回值。 call()方法可以声明抛出异常。

Java 5提供了Future接口来代表Callable接口里call()方法的返回值，并为Future接口提供了一个FutureTask实现类，该实现类实现了Future接口，并实现了Runnable接口——可以作为Thread类的target。

在Future接口里定义了如下几个公共方法来控制它关联的Callable任务。

> boolean cancel(boolean mayInterruptIfRunning)：试图取消该Future里关联的Callable任务。
>
> V get()：返回Callable任务里call()方法的返回值。调用该方法将导致程序阻塞，必须等到子线程结束后才会得到返回值。
>
> V get(long timeout, TimeUnit unit)：返回Callable任务里call()方法的返回值。该方法让程序最多阻塞timeout和unit指定的时间，如果经过指定时间后Callable任务依然没有返回值，将会抛出TimeoutException异常。
>
> boolean isCancelled()：如果在Callable任务正常完成前被取消，则返回true。
>
> boolean isDone()：如果Callable任务已完成，则返回true。

Callable接口有泛型限制，Callable接口里的泛型形参类型与call()方法返回值类型相同。

~~~java
public class test implements Callable<Integer> {
    //实现call()方法，作为线程执行体
    public Integer call() throws Exception {
        int i=0;
        for (;i<100;i++){
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
        return i;
    }
    public static void main(String[] args) throws Exception{
        //创建Callable对象
        test t=new test();
        //使用FutureTask来包装Callable对象
        FutureTask<Integer> task=new FutureTask<>(t);
        for (int i = 0; i <100 ; i++) {
            System.out.println(Thread.currentThread().getName()+" "+i);
            if (i==20){
                //实质还是以Callable对象来创建并启动线程
                new Thread(task,"有返回值的线程").start();
            }
        }
        try {
            System.out.println("子线程的返回值："+task.get());
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
~~~

### 创建线程三种方式对比

采用实现Runnable、Callable接口的方式创建多线程——线程类只是实现了Runnable接口或Callable接口，还可以继承其他类。

在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情况，从而可以将CPU、代码和数据分开，形成清晰的模型，较好地体现了面向对象的思想。

 劣势是：编程稍稍复杂，如果需要访问当前线程，则必须使用Thread.currentThread()方法。

采用继承Thread类的方式创建多线程——

劣势是：因为线程类已经继承了Thread类，所以不能再继承其他父类。

优势是：编写简单，如果需要访问当前线程，则无须使用Thread.currentThread()方法，直接使用this即可获得当前线程。

## 三、线程的生命周期

当线程被创建并启动以后，它既不是一启动就进入了执行状态，也不是一直处于执行状态，在线程的生命周期中，它要经过新建（New）、就绪（Runnable）、运行（Running）、阻塞（Blocked）和死亡（Dead） 5种状态。

### 新建和就绪

当程序使用new关键字创建了一个线程之后，该线程就处于**新建状态**，此时它和其他的Java对象一样，仅仅由Java虚拟机为其分配内存，并初始化其成员变量的值。此时的线程对象没有表现出任何线程的动态特征，程序也不会执行线程的线程执行体。

当线程对象调用了start()方法之后，该线程处于**就绪状态**，Java虚拟机会为其创建方法调用栈和程序计数器，处于这个状态中的线程并没有开始运行，只是表示该线程可以运行了。至于该线程何时开始运行，取决于JVM里线程调度器的调度。

> **PS：启动线程调用的是start()方法，而不是run()方法，永远不要调用run方法，调用start方法来启动线程，系统会将run方法当成线程执行体来处理，如果直接调用run方法，系统会将线程对象当作普通对象，run也是一个普通放法，而不是线程执行体。**
>
> 只能对处于新建状态的线程调用start()方法，否则将引发IllegalThreadStateException异常。

如果希望调用子线程的start方法后，子线程立即执行，可以使用`Thread.sleep(1)`来让当前线程（主线程）休眠一毫秒，因为cpu不会空闲，所以即可立即运行。

### 运行和阻塞

如果处于就绪状态的线程获得了CPU，开始执行run()方法的线程执行体，则该线程处于**运行状态**，如果计算机只有一个CPU，那么在任何时刻只有一个线程处于运行状态。当然，在一个多处理器的机器上，将会有多个线程并行（注意是并行：parallel）执行；当线程数大于处理器数时，依然会存在多个线程在同一个CPU上轮换的现象。

当一个线程开始运行时，他不可能一直处于运行状态，线程在运行过程中需要被中断让其他线程获得执行的机会，线程调节的细节取决于底层平台的策略。

采用抢占式策略的系统，会给每一个可执行的线程一个时间段来处理任务，时间段用完后，系统就会剥夺该线程所占用的资源，让其他线程获得执行的机会。在选择下一个线程时，系统会考虑线程的优先级。现代桌面和服务器操作系统都采用抢占式调度策略。

一些小型设备如手机可能会采用协作式调度策略，只有当一个线程调用了它的sleep()或yield()方法后才会放弃所占用的资源——也就是必须由该线程主动放弃所占用的资源。当发生如下情况时，线程将会进入**阻塞状态**。

当前正在执行的线程被阻塞之后，其他线程就可以获得执行的机会。被阻塞的线程会在合适的时候重新进入就绪状态，（注意是就绪状态而不是运行状态）。也就是说，被阻塞线程的阻塞解除后，必须重新等待线程调度器再次调度它。

进入阻塞状态：

1. 调用了sleep方法主动放弃占用的处理器资源
2. 调用了一个阻塞式IO方法，返回前线程被阻塞
3. 线程试图获得同步监视器，但正在被其他线程所持有
4. 线程正在等待通知
5. 程序调用了线程的suspend()方法将该线程挂起。但这个方法容易导致死锁，所以应该尽量避免使用该方法。

重新进入就绪状态：

1. sleep方法过了指定时间
2. 阻塞式IO方法已经返回
3. 线程成功的获得了同步监视器
4. 线程正在等待某通知，其他线程发出了一个通知
5. 处于挂起状态的线程调用了resume恢复方法

> **PS：线程从阻塞状态只能进入就绪状态，无法直接进入运行状态。而就绪和运行状态之间的转换通常不受程序控制，而是由系统线程调度所决定，当处于就绪状态的线程获得处理器资源时，该线程进入运行状态；当处于运行状态的线程失去处理器资源时，该线程进入就绪状态。但有一个方法例外，调用yield()方法可以让运行状态的线程转入就绪状态。**

### 线程死亡

线程以以下三种方式结束，结束后进入死亡状态

run()或call()方法执行完成，线程正常结束

线程抛出一个未捕获的Exception或Error

直接调用该线程的stop()方法来结束线程。该方法容易导致死锁，通常不推荐使用。

**当主线程结束时，其他线程不受任何影响，并不会随之结束。一旦子线程启动起来后，它就拥有和主线程相同的地位，它不会受主线程的影响。不要试图对一个已经死亡的线程调用start()方法使它重新启动，死亡就是死亡，该线程将不可再次作为线程执行。**

为了测试某个线程是否已经死亡，可以调用线程对象的isAlive()方法，当线程处于就绪、运行、阻塞3种状态时，该方法将返回true；当线程处于新建、死亡2种状态时，该方法将返回false。

~~~java
public class StartDead extends Thread{
    private int i;
    public void run(){//线程执行体
        for(;i<100;i++){
            System.out.println(getName()+" "+i);
        }
    }
    public static void main(String [] args){
        //创建线程
        StartDead sd = new StartDead();
        for(int i =0;i<300;i++){
             System.out.println(Thread.currentThread().getName()+" "+i);
            if(i==20){
                sd.start();//启动线程
                 System.out.println(sd.isAlive());//true
            }
            if(i>20 && !sd.isAlive()){
                //再次启动该线程
                sd.start();//死亡状态下线程无法再次运行
            }
        }
    }
}
~~~



## 四、线程的控制

Java的线程支持提供了一些便捷的工具方法，通过这些便捷的工具方法可以很好地控制线程的执行。

### join线程

Thread提供了让一个线程等待另一个线程完成的方法——join()方法。当在某个程序执行流中调用其他线程的join()方法时，调用线程将被阻塞，直到被join()方法加入的join线程执行完为止。join()方法通常由使用线程的程序调用，以将大问题划分成许多小问题，每个小问题分配一个线程。当所有的小问题都得到处理后，再调用主线程来进一步操作。

join有以下三种形式重载：

join()

join(long millis):等待被join的线程的时间最长为millis毫秒。如果在millis毫秒内被join的线程还没有执行结束，则不再等待。

join(long millis , int nanos):等待被join的线程的时间最长为millis毫秒加nanos毫微秒。

~~~java
public class JoibThread extends Thread{
    //提供一个有参数的构造器，用于设置线程名字
    public JoinThread(String name){
        super(name);
    }
    //重写run，定义线程执行体
    public void run(){
        for(int i=0;i<100;i++){
            System.out.println(getName()+" "+i);
        }
    }
    public static void main(String [] args) throws Exception{
        new JoinThread("新线程").start();
        for(int i=0;i<100;i++){
            if(i==20){
                JoinThread jt =new JoinThread("被join的线程");
                jt.start();
                //main线程调用了jt线程的join方法，
                //main线程必须等待jt执行结束后才能向下执行
                jt.join();
            }
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
    }
}
~~~

上面程序中一共有3个线程，主方法开始时就启动了名为“新线程”的子线程，该子线程将会和main线程并发执行。当主线程的循环变量i等于20时，启动了名为“被Join的线程”的线程，该线程不会和main线程并发执行，main线程必须等该线程执行结束后才可以向下执行。在名为“被Join的线程”的线程执行时，实际上只有2个子线程并发执行，而主线程处于等待状态。

### 后台线程

有一种线程，它是在后台运行的，它的任务是为其他的线程提供服务，这种线程被称为“后台线程（Daemon Thread）”，又称为“守护线程”或“精灵线程”。JVM的垃圾回收线程就是典型的后台线程。

后台线程有个特征：如果所有的前台线程都死亡，后台线程会自动死亡。

调用Thread对象的setDaemon(true)方法可将指定线程设置成后台线程。setDaemon(true)必须在start()方法之前调用，否则会引发IllegalThreadStateException异常。

Thread类还提供了一个isDaemon()方法，用于判断指定线程是否为后台线程。

~~~java
public class DaemonThread extends Thread{
    public void run(){
        for(int i=0;i<100;i++){
            System.out.println(getName()+" "+i);
        }
    }
    public static void main(String [] args){
        DaemonThread t =new DaemonThread();
        t.setDaemon(true);//设置为后台进程
        t.start();
        for(int i=0;i<10;i++){
            System.out.println(Thread.currentThread().getName()+" "+i);
        }
        //程序到此处前台线程结束（main）
        //后台进程也随之结束
    }
}
~~~

当所有的前台线程死亡时，后台线程随之死亡。当整个虚拟机中只剩下后台线程时，程序就没有继续运行的必要了，所以虚拟机也就退出了。

本来该线程应该执行到i等于999时才会结束，但运行程序时不难发现该后台线程无法运行到999，因为当主线程也就是程序中唯一的前台线程运行结束后，JVM会主动退出，因而后台线程也就被结束了。

### sleep线程睡眠

如果想让当前正在执行的线程暂停一段时间进入阻塞状态，则可以使用sleep()方法来实现。sleep有两种重载。

static void sleep(long millis)：让当前正在执行的线程暂停millis毫秒，并进入阻塞状态，该方法收到系统计时器和线程调度器的精度与准确度的影响。

static void sleep(long millis,int nanos)：让当前正在执行的线程暂停millis毫秒+nanos微秒，并进入阻塞状态，该方法收到系统计时器和线程调度器的精度与准确度的影响

~~~java
public class SleepTest{
    public static void main(String args) throws Exception{
        for(int i=0;i<10;i++){
            System.out.println("当前时间"+new Date());
        	Thread.sleep(1000);//暂停1s
        }
    }
}
~~~

当前线程调用sleep()方法进入阻塞状态后，在其睡眠时间段内，该线程不会获得执行的机会，即使系统中没有其他可执行的线程，处于sleep()中的线程也不会执行，因此sleep()方法常用来暂停程序的执行。

### yield线程让步

yield()方法是一个和sleep()方法有点相似的方法，它也是Thread类提供的一个静态方法，它也可以让当前正在执行的线程暂停，但它不会阻塞该线程，它只是将该线程转入就绪状态。yield()只是让当前线程暂停一下，让系统的线程调度器重新调度一次，完全可能的情况是：当某个线程调用了yield()方法暂停之后，线程调度器又将其调度出来重新执行。

~~~java
public class YieldTest extends Thread{
    //提供一个有参数的构造器，用于设置线程名字
    public YieldTest(String name){
        super(name);
    }
    //重写run，定义线程执行体
    public void run(){
        for(int i=0;i<100;i++){
            System.out.println(getName()+" "+i);
            //i=10时，使用yield方法让当前线程让步
            if(i==10){
                Thread.yield();
            }
        }
    }
    public static void main(String [] args) throws Exception{
        //将yt设置为最高优先级
        YieldTest yt=new YieldTest("高级别");
        yt.setPriority(Thread.MAX_PRIORITY);//1
        yt.start();
        //将yt设置为最低优先级
        YieldTest yt1=new YieldTest("低级别");
        yt1.setPriority(Thread.MIN_PRIORITY);//2
        yt1.start();
        }
    }
}
~~~

上面程序中的调用yield()静态方法让当前正在执行的线程暂停，让系统线程调度器重新调度。将程序中1、2代码加上注态——使两个线程的优先级完全一样，所以当一个线程使用yield()方法暂停后，另一个线程就会开始执行。如果将1、2处代码的注释取消，也就是为两个线程分别设置不同的优先级，那么高优先级调用yield方法暂停后，没有与之相同的优先级，所以继续执行。

**sleep和yeild方法的区别**

1. sleep()方法暂停当前线程后，会给其他线程执行机会，不会理会其他线程的优先级；但yield()方法只会给优先级相同，或优先级更高的线程执行机会。
2. sleep()方法会将线程转入阻塞状态，直到经过阻塞时间才会转入就绪状态；而yield()不会将线程转入阻塞状态，它只是强制当前线程进入就绪状态。因此完全有可能某个线程调用yield()方法暂停之后，立即再次获得处理器资源被执行。
3. sleep()方法声明抛出了InterruptedException异常，所以调用sleep()方法时要么捕捉该异常，要么显式声明抛出该异常；而yield()方法则没有声明抛出任何异常。
4. sleep()方法比yield()方法有更好的可移植性，通常不建议使用yield()方法来控制并发线程的执行。

### 改变线程优先级

每个线程执行时都具有一定的优先级，优先级高的线程获得较多的执行机会，而优先级低的线程则获得较少的执行机会。

每个线程默认的优先级都与创建它的父线程的优先级相同，在默认情况下，main线程具有普通优先级，由main线程创建的子线程也具有普通优先级。

Thread类提供了setPriority(int newPriority)、getPriority()方法来设置和返回指定线程的优先级，其中setPriority()方法的参数可以是一个整数，范围是1～10之间，也可以使用Thread类的如下3个静态常量:

- MAX_PRIORITY:值为10
- MIN_PRIORITY:值为1
- NORM_PRIORITY:值为5

> 值得指出的是，虽然Java提供了10个优先级级别，但这些优先级级别需要操作系统的支持。遗憾的是，不同操作系统上的优先级并不相同，而且也不能很好地和Java的10个优先级对应，例如Windows 2000仅提供了7个优先级。在这种情况下，我们应该尽量避免直接为线程指定优先级，而应该使用MAX_PRIORITY、MIN_PRIORITY和NORM_PRIORITY三个静态常量来设置优先级，这样才可以保证程序具有最好的可移植性。

## 五、线程的同步

当使用多个线程来访问同一个数据时，很容易“偶然”出现线程安全问题

### 线程安全问题

关于线程安全问题，有一个经典的问题——银行取钱的问题。银行取钱的基本流程基本上可以分为如下几个步骤。

（1）用户输入账户、密码，系统判断用户的账户、密码是否匹配。

（2）用户输入取款金额。

（3）系统判断账户余额是否大于取款金额。

（4）如果余额大于取款金额，则取款成功；

如果余额小于取款金额，则取款失败。这个流程没有任何问题。但一旦将这个流程放在多线程并发的场景下，就有可能出现问题。注意此处说的是有可能，并不是说一定。也许你的程序运行了一百万次都没有出现问题，但没有出现问题并不等于没有问题！

下面来模拟两个人使用同一个账户并发取钱的问题：

首先定义账户类，封装属性

~~~java
 public class Account{
    //封装账户编号、账户余额
    private String accountNo;
    private double balance;
    //构造函数
    public Account(String accountNo,double balance){
        this.accountNo=accountNo;
        this.balance=balance;
    }
    //省略getter和setter
    //。。。
    //根据accountNo重写hashCode()和equals()方法
    public int hasCode(){
        return accountNo.hasCode();
    }
    public boolean equals(Object obj){
        if(this==obj)return true;
        if(obj!=null&&obj,getClass()==Account.class){
            Account target=(Account)obj;
            return target.getAccountNo.equals(AccountNo);
        }
        return false;
    }
}
~~~

接下来提供一个取钱的线程类，该线程类根据执行账户、取钱数量进行取钱操作，取钱的逻辑是当其余额不足时无法提取现金，当余额足够时系统吐出钞票，余额减少。

~~~java
public class DrawThread extends Thread{
    //模拟用户账户
    private Account account;
    //当前取钱线程希望取得钱数
    private double drawAmount;
    public DrawThread(String name,Account account,double drawAmount){
        super(name);
        this.account=account;
        this.drawAmount=drawAmount;
    }
    //当多个线程修改同一个共享数据时，将涉及数据安全问题
    public void run(){
        //账户余额大于所希望取得的钱数
        if(account.getBalance()>=drawAmount){
            //吐出钞票
            System.out.println(getName+"取钱成功："+drawAmount);
           /*//造成错误
            try{
                Thread.sleep(1);
            }catch(Exception e){
                e.printStackTrace();
            }
            */
            //修改余额
            account.setBalance(account.getBalance()-drawAmount);
            System.out.println("\t余额为："+account.getBalance());    
        }
        else{
            System.out.println(getName()+"取钱失败余额不足");
        }
    }
}
~~~

上面程序是一个非常简单的取钱逻辑，这个取钱逻辑与实际的取钱操作也很相似。程序的主程序非常简单，仅仅是创建一个账户，并启动两个线程从该账户中取钱。

~~~java
public class DrawTest{
    public static void main(String args){
        //创建一个账户
        Account account = new Account("1234567",1000);
        new DrawThread("甲",account,800).start();
        new DrawThread("乙",account,800).start();
    }
}
~~~

多次运行上面程序，很有可能会出现如下结果：

> 甲取钱成功：800.0
>
> 乙取钱成功：800.0
>
> 余额为：200
>
> 余额为：-600

如果将上面//造成错误处的代码取消注释，也必定会出现上面的错误

账户余额只有1000时取出了1600，而且账户余额出现了负值，这不是银行希望的结果。虽然上面程序是人为地使用Thread.sleep(1)来强制线程调度切换，但这种切换也是完全可能发生的——100000次操作只要有1次出现了错误，那就是编程错误引起的。

### 同步代码块

之所以出现上面的问题，是因为run方法的方法体不具有同步安全性，程序中有两个并发线程在修改Account对象，且系统恰好在被注释的代码的地方执行线程切换，所以出现了问题。

为了解决这个问题，Java的多线程支持引入了同步监视器来解决这个问题，使用同步监视器的通用方法就是同步代码块。同步代码块的语法格式如下：

~~~java
synchronized(obj){
    ...//此处代码就是同步代码块
}
~~~

上面语法格式中synchronized后括号里的obj就是同步监视器，上面代码的含义是：线程开始执行同步代码块之前，必须先获得对同步监视器的锁定。

> **任何时刻只能有一个线程可以获得对同步监视器的锁定，当同步代码块执行完成后，该线程会释放对该同步监视器的锁定。**

虽然Java程序允许使用任何对象作为同步监视器，但同步监视器的目的是阻止两个线程对同一个共享资源进行并发访问，因此通常推荐使用可能被并发访问的共享资源充当同步监视器。对于上面的取钱模拟程序，我们应该考虑使用账户（account）作为同步监视器。我们把程序修改成如下形式。

~~~java
public class DrawThread extends Thread{
    private Account account;
    private double drawAmount;
    public DrawThread(String name,Account account ,double drawAmount){
        super(name);
        this.account=account;
        this.drawAmount=drawAmount;
    }
    //当多个线程修改同一个共享数据时，涉及到数据安全问题
    public void run(){
        /*使用account作为同步监视器，任何线程进入下面同步代码块之前必须鲜活的对account账户的锁定——其他线程无法获得锁那么就无法修改它。这种做法符合加锁-修改-释放锁的逻辑*/
        synchronized(account){
            if(account.getBalance()>=drawAmount){
            //吐出钞票
            System.out.println(getName+"取钱成功："+drawAmount);
           //造成错误
            try{
                Thread.sleep(1);
            }catch(Exception e){
                e.printStackTrace();
            }
            //修改余额
            account.setBalance(account.getBalance()-drawAmount);
            System.out.println("\t余额为："+account.getBalance());    
        }
        else{
            System.out.println(getName()+"取钱失败余额不足");
        }
        }//同步代码块结束，释放同步锁
    }
}
~~~

上面程序使用synchronized将run()方法里的方法体修改成同步代码块，该同步代码块的同步监视器是account对象，这样的做法符合“加锁→修改→释放锁”的逻辑，任何线程在修改指定资源之前，首先对该资源加锁，在加锁期间其他线程无法修改该资源，当该线程修改完成后，该线程释放对该资源的锁定。通过这种方式就可以保证并发线程在任一时刻只有一个线程可以进入修改共享资源的代码区（也被称为临界区），所以同一时刻最多只有一个线程处于临界区内，从而保证了线程的安全性。

### 同步方法

与同步代码块对应，Java的多线程安全支持还提供了同步方法，同步方法就是使用synchronized关键字来修饰某个方法，则该方法称为同步方法。对于同步方法而言，无须显式指定同步监视器，同步方法的同步监视器是this，也就是该对象本身。

使用同步方法的可以很方便实现线程安全的类，线程安全类有如下特征：

- 该类的对象可以被多个线程安全的访问
- 每个线程调用该对象的任意方法之后都能得到正确结果
- 每个线程调用该对象的任意方法之后，该对象依旧保持合理状态。

不可变类总是线程安全的，因为它的对象状态不可改变；但可变对象需要额外的方法来保证其线程安全。例如上面的Account就是一个可变类，它的account和balance两个Field都可变，当两个线程同时修改Account对象的balance Field时，程序就出现了异常。下面我们将Account类对balance的访问设置成线程安全的，那么只要把balance的方法修改成同步方法即可。

~~~java
 public class Account{
    //封装账户编号、账户余额
    private String accountNo;
    private double balance;
    //构造函数
    public Account(String accountNo,double balance){
        this.accountNo=accountNo;
        this.balance=balance;
    }
    //省略accountNo的getter和setter
    //。。。
    //省略hashCode()和equals()方法
    //。。。
    //因为账户不允许随便修改余额，所以balance提供getter方法
     public double getBalance(){return this.balance}
     //提供一个线程安全的方法draw()来完成取钱操作
     public synchronized void draw(double drawAmount){
         //账户余额大于取钱数目
         if(account.getBalance()>=drawAmount){
            //吐出钞票
            System.out.println(getName+"取钱成功："+drawAmount);
           //造成错误
            try{
                Thread.sleep(1);
            }catch(Exception e){
                e.printStackTrace();
            }
            //修改余额
            account.setBalance(account.getBalance()-drawAmount);
            System.out.println("\t余额为："+account.getBalance());    
        }
        else{
            System.out.println(getName()+"取钱失败余额不足");
        }
             
         }
     }
}
~~~

上面程序中增加了一个代表取钱的draw()方法，并使用了synchronized关键字修饰该方法，把该方法变成同步方法。同步方法的同步监视器是this，因此对于同一个Account账户而言，任意时刻只能有一个线程获得对Account对象的锁定，然后进入draw ()方法执行取钱操作——这样也可以保证多个线程并发取钱的线程安全。

因为Account类中已经提供了draw()方法，而且取消了setBalance()方法，DrawThread线程类需要改写，该线程类的run()方法只要调用Account对象的draw()方法即可执行取钱操作。

> **synchronized关键字可以修饰方法，可以修饰代码块，但不能修饰构造器、属性等。**
>
> **在Account里定义draw()方法，而不是直接在run()方法中实现取钱逻辑，这种做法更符合面向对象规则。在面向对象里有一种流行的设计方式：Domain DrivenDesign（领域驱动设计，DDD），这种方式认为每个类都应该是完备的领域对象，例如Account代表用户账户，应该提供用户账户的相关方法；通过draw()方法来执行取钱操作（实际上还应该提供transfer()等方法来完成转账等操作），而不是直接将setBalance()方法暴露出来任人操作，这样才可以更好地保证Account对象的完整性和一致性。**

上面的DrawThread类无须自己实现取钱操作，而是直接调用account的draw()方法来执行取钱操作。由于已经使用synchronized关键字修饰了draw()方法，同步方法的同步监视器是this，而this总代表调用该方法的对象——在上面示例中，调用draw()方法的对象是account，因此多个线程并发修改同一份account之前，必须先对account对象加锁。这也符合了“加锁 → 修改→ 释放锁”的逻辑。

可变类的线程安全是以降低程序的运行效率作为代价的，为了减少线程安全所带来的负面影响，程序可以采用如下策略:

- 不要对线程安全类的所有方法都进行同步，只对那些共享资源的方法进行同步。例如上面Account类中的accountNo属性就无须同步，所以程序只对draw()方法进行了同步控制。
- 如果可变类有两种运行环境：单线程环境和多线程环境，则应该为该可变类提供两种版本，即线程不安全版本和线程安全版本。在单线程环境中使用线程不安全版本以保证性能，在多线程环境中使用线程安全版本。

> **JDK所提供的StringBuilder、StringBuffer就是为了照顾单线程环境和多线程环境所提供的类，在单线程环境下应该使用StringBuilder来保证较好的性能；当需要保证多线程安全时，就应该使用StringBuffer。**

### 释放同步监视器的锁定

任何线程进入同步代码块、同步方法之前，必须先获得对同步监视器的锁定，那么何时会释放对同步监视器的锁定呢？程序无法显式释放对同步监视器的锁定，线程会在如下几种情况下**释放对同步监视器的锁定**：

- 当前线程的同步方法、同步代码块执行结束时释放同步监视器
- 当前线程在同步代码块、同步方法中遇到了break、return终止了该代码块或方法时获释放同步监视器
- 当前线程在同步代码块、同步方法中出现了未处理的Error或Exception，导致了该代码块、该方法异常结束时会释放同步监视器。
- 当前线程执行同步代码块或同步方法时，程序执行了同步监视器对象的wait()方法，则当前线程暂停，并释放同步监视器。

在如下所示的情况下，线程**不会释放同步监视器**：

- 线程执行同步代码块或同步方法时，程序调用Thread.sleep()、Thread.yield()方法来暂停当前线程的执行，当前线程不会释放同步监视器。
- 线程执行同步代码块时，其他线程调用了该线程的suspend()方法将该线程挂起，该线程不会释放同步监视器。当然，我们应该尽量避免使用suspend()和resume()方法来控制线程。

### 同步锁

从Java 5开始，Java提供了一种功能更强大的线程同步机制——通过显式定义同步锁对象来实现同步，在这种机制下，同步锁使用Lock对象充当。

Lock提供了比synchronized方法和synchronized代码块更广泛的锁定操作，Lock实现允许更灵活的结构，可以具有差别很大的属性，并且支持多个相关的Condition对象。

Lock是控制多个线程对共享资源进行访问的工具。通常，锁提供了对共享资源的独占访问，每次只能有一个线程对Lock对象加锁，线程开始访问共享资源之前应先获得Lock对象。

某些锁可能允许对共享资源并发访问，如ReadWriteLock（读写锁）。Lock、ReadWriteLock是Java5新提供的两个根接口，并为Lock提供了ReentrantLock（可重入锁）实现类；为ReadWriteLock提供了ReentrantReadWriteLock实现类。在实现线程安全的控制中，比较常用的是ReentrantLock（可重入锁）。

> 读写锁就是再多线程的条件下，读读不加锁，读写和写读和写写都需要加锁。
>
> 可重入锁就是外层函数获取锁之后，内层代码仍然可以获取该锁的代码，在同一线程在外层方法获取锁的时候，在进入内层方法会自动获取锁。线程可以进入任何一个它所拥有锁的同步代码块。比如线程A获取锁，进入到 A() 方法，然后调用B方法，B方法也加了锁，但是可以直接获取到锁。不需要重新申请。

可重入锁ReentrantLock的代码格式如下：

~~~java
class X{
    //定义锁对象
    private final ReentrantLock lock = new ReentrantLock();
    //...
    //定义需要保证线程安全的方法
    public void func(){
        //加锁
        lock.lock();
        try{
            //需要保证线程安全的代码
            //。。。
        }
        //使用finally保证释放锁
        finally{
            lock.unlock();
        }
    }
}
~~~

使用ReentrantLock对象来进行同步，加锁和释放锁出现在不同的作用范围内时，通常建议使用finally块来确保在必要时释放锁。通过使用ReentrantLock对象，我们可以把Account类改为如下形式，它依然是线程安全的。

~~~java
public class Account{
	//定义锁对象
    private final ReentrantLock lock = new ReentrantLock();
    //其余地方与同步方法中一样，以下为不一样的地方对draw方法的修改
    public void draw(){
        //加锁
        lock.lock();
        try{
             //账户余额大于取钱数目
             if(account.getBalance()>=drawAmount){
                //吐出钞票
                System.out.println(getName+"取钱成功："+drawAmount);
               //造成错误
                try{
                    Thread.sleep(1);
                }catch(Exception e){
                    e.printStackTrace();
                }
                //修改余额
                balance-=drawAmount;
                //account.setBalance(account.getBalance()-drawAmount);
                System.out.println("\t余额为："+account.getBalance());    
            }
            else{
                System.out.println(getName()+"取钱失败余额不足");
            }
        }
        finally{
            //释放锁
            lock.unlock();
        }
    }
}
~~~

> 使用Lock与使用同步方法有点相似，只是使用Lock时显式使用Lock对象作为同步锁，而使用同步方法时系统隐式使用当前对象作为同步监视器，同样都符合“加锁→修改→释放锁”的操作模式，而且使用Lock对象时每个Lock对象对应一个Account对象，一样可以保证对于同一个Account对象，同一时刻只能有一个线程能进入临界区。

**同步方法/代码块与锁**

虽然同步方法和同步代码块的范围机制使得多线程安全编程非常方便，而且还可以避免很多涉及锁的常见编程错误，但有时也需要以更为灵活的方式使用锁。Lock提供了同步方法和同步代码块所没有的其他功能，包括用于非块结构的tryLock()方法，以及试图获取可中断锁的lockInterruptibly()方法，还有获取超时失效锁的tryLock(long,TimeUnit)方法。

## 六、线程的通信

### 传统的线程通信

假设现在系统中有两个线程，这两个线程分别代表存款者和取钱者——现在假设系统有一种特殊的要求，系统要求存款者和取钱者不断地重复存款、取钱的动作，而且要求每当存款者将钱存入指定账户后，取钱者就立即取出该笔钱。不允许存款者连续两次存钱，也不允许取钱者连续两次取钱。为了实现这种功能，可以借助于Object类提供的wait()、notify()和notifyAll() 3个方法，这3个方法并不属于Thread类，而是属于Object类。但这3个方法必须由同步监视器对象来调用，这可分成以下两种情况:

- 对于使用synchronized修饰的同步方法，因为该类的默认实例（this）就是同步监视器，所以可以在同步方法中直接调用这3个方法。
-  对于使用synchronized修饰的同步代码块，同步监视器是synchronized后括号里的对象，所以必须使用该对象调用这3个方法。

> 三个方法：
>
> wait()：导致当前线程等待，直到其他线程调用该同步监视器的notify()方法或notifyAll()方法来唤醒该线程。
>
> notify()：唤醒在此同步监视器上等待的单个线程。如果所有线程都在此同步监视器上等待，则会选择唤醒其中一个线程。选择是任意性的。只有当前线程放弃对该同步监视器的锁定后（使用wait()方法），才可以执行被唤醒的线程。
>
> notifyAll()：唤醒在此同步监视器上等待的所有线程。只有当前线程放弃对该同步监视器的锁定后，才可以执行被唤醒的线程。

~~~java
 public class Account{
    //封装账户编号、账户余额
    private String accountNo;
    private double balance;
    //标记账户中是否有存款
    private boolean flag=false;
    //构造函数
    public Account(String accountNo,double balance){
        this.accountNo=accountNo;
        this.balance=balance;
    }
    //省略accountNo的getter和setter
    //。。。
    //省略hashCode()和equals()方法
    //。。。
    //因为账户不允许随便修改余额，所以balance提供getter方法
     public double getBalance(){return this.balance}
     //提供一个线程安全的方法draw()来完成取钱操作
     public synchronized void draw(double drawAmount){
         try{
             //如果flag为假，表明还没人存钱，取钱方法阻塞
             if(!flag){
                 wait();
             }else{
                 System.out.println(Thread.currentThread().getName()+"取钱"+drawAmount);
                 balance-=drawAmount;
                 System.out.println("账户余额"+balance);
                 flag=false;//已有存款标为false
                 notifyAll();//唤醒其他线程
             }
         }
         catch(Exception e){e.printStackTrace();}
     }
     public synchronized void despoit(double despositAmount){
         try{
             //如果flag为真，表明已经存钱，存钱方法阻塞
             if(flag){
                 wait();
             }else{
                 System.out.println(Thread.currentThread().getName()+"存钱"+despositAmount);
                 balance+=despositAmount;
                 System.out.println("账户余额"+balance);
                 flag=true;//已有存款标为true
                 notifyAll();//唤醒其他线程
             }
         }
         catch(Exception e){e.printStackTrace();}
     }
}
~~~

上面程序中使用wait()和notifyAll()进行了控制，对存款者线程而言，当程序进入deposit()方法后，如果flag为true，则表明账户中已有存款，程序调用wait()方法阻塞；否则程序向下执行存款操作，当存款操作执行完成后，系统将flag设为true，然后调用notifyAll()来唤醒其他被阻塞的线程。

### 使用Condition控制线程通信

如果程序不使用synchronized关键字来保证同步，而是直接使用Lock对象来保证同步，则系统中不存在隐式的同步监视器，也就不能使用wait()、notify()、notifyAll()方法进行线程通信了。

当使用Lock对象来保证同步时，Java提供了一个Condition类来保持协调，使用Condition可以让那些已经得到Lock对象却无法继续执行的线程释放Lock对象，Condition对象也可以唤醒其他处于等待的线程。Condition将同步监视器方法（wait()、notify()和notifyAll()）分解成截然不同的对象，以便通过将这些对象与Lock对象组合使用，为每个对象提供多个等待集（wait-set）。在这种情况下，Lock替代了同步方法或同步代码块，Condition替代了同步监视器的功能。

Condition实例被绑定在一个Lock对象上。要获得特定Lock实例的Condition实例，调用Lock对象的**newCondition()方法**即可。Condition类提供了如下3个方法：

- await():类似于wait()方法，导致当前线程等待，知道其他线程调用Condition的signal或signalAll方法来唤醒该线程。
- signal()：唤醒在此Lock对象上等待的单个线程。如果所有线程都在该Lock对象上等待，则会选择唤醒其中一个线程。选择是任意性的。只有当前线程放弃对该Lock对象的锁定后（使用await()方法），才可以执行被唤醒的线程。
- signalAll()：唤醒在此Lock对象上等待的所有线程。只有当前线程放弃对该Lock对象的锁定后，才可以执行被唤醒的线程。

以下是对上面传统通信中Account.java的修改

~~~java
 public class Account{
     private final Lock lock=new ReentrantLock();
     private final Condition cond=lock.newCondition();
    //封装账户编号、账户余额
    private String accountNo;
    private double balance;
    //标记账户中是否有存款
    private boolean flag=false;
    //构造函数
    public Account(String accountNo,double balance){
        this.accountNo=accountNo;
        this.balance=balance;
    }
    //省略accountNo的getter和setter
    //。。。
    //省略hashCode()和equals()方法
    //。。。
    //因为账户不允许随便修改余额，所以balance提供getter方法
     public double getBalance(){return this.balance}
     //提供一个线程安全的方法draw()来完成取钱操作
     public void draw(double drawAmount){
         lock.lock();//加锁
         try{
             //如果flag为假，表明还没人存钱，取钱方法阻塞
             if(!flag){
                 cond.await();
             }else{
                 System.out.println(Thread.currentThread().getName()+"取钱"+drawAmount);
                 balance-=drawAmount;
                 System.out.println("账户余额"+balance);
                 flag=false;//已有存款标为false
                 cond.singalAll();//唤醒其他线程
             }
         }
         catch(Exception e){e.printStackTrace();}
         finally{lock.unlock();}
     }
     public synchronized void despoit(double despositAmount){
         lock.lock();
         try{
             //如果flag为真，表明已经存钱，存钱方法阻塞
             if(flag){
                 cond.await();
             }else{
                 System.out.println(Thread.currentThread().getName()+"存钱"+despositAmount);
                 balance+=despositAmount;
                 System.out.println("账户余额"+balance);
                 flag=true;//已有存款标为true
                 cond.singalAll();//唤醒其他线程
             }
         }
         catch(Exception e){e.printStackTrace();}
         finally{lock.unlock();}
     }
}
~~~

这两个程序的逻辑基本相似，只是现在显式地使用Lock对象来充当同步监视器，则需要使用Condition对象来暂停、唤醒指定线程。

### 使用阻塞队列（BlockingQueue）控制线程通信

Java 5提供了一个BlockingQueue接口，虽然BlockingQueue也是Queue的子接口，但它的主要用途并不是作为容器，而是作为线程同步的工具。BlockingQueue具有一个特征：当生产者线程试图向BlockingQueue中放入元素时，如果该队列已满，则该线程被阻塞；当消费者线程试图从BlockingQueue中取出元素时，如果该队列已空，则该线程被阻塞。

BlockingQueue提供如下两个支持阻塞的方法:

- put(E e)：尝试把E元素放入BlockingQueue中，如果该队列的元素已满，则阻塞该线程。
- take()：尝试从BlockingQueue的头部取出元素，如果该队列的元素已空，则阻塞该线程。

BlockingQueue继承了Queue接口，当然也可使用Queue接口中的方法。这些方法归纳起来可分为如下3组:

- 在队列尾部插入元素。包括add(E e)、offer(E e)和put(E e)方法，当该队列已满时，这3个方法分别会抛出异常、返回false、阻塞队列。
- 在队列头部删除并返回删除的元素。包括remove()、poll()和take()方法。当该队列已空时，这3个方法分别会抛出异常、返回false、阻塞队列。
- 在队列头部取出但不删除元素。包括element()和peek()方法，当队列已空时，这两个方法分别抛出异常、返回false。

BlockingQueue包含如下5个实现类:

- ArrayBlockingQueue：基于数组实现的BlockingQueue队列。
- LinkedBlockingQueue：基于链表实现的BlockingQueue队列。
- PriorityBlockingQueue：它并不是标准的阻塞队列。与PriorityQueue类似，该队列调用remove()、poll()、take()等方法取出元素时，并不是取出队列中存在时间最长的元素，而是队列中最小的元素。PriorityBlockingQueue判断元素的大小即可根据元素（实现Comparable接口）的本身大小来自然排序，也可使用Comparator进行定制排序。
- SynchronousQueue：同步队列。对该队列的存、取操作必须交替进行。
- DelayQueue：它是一个特殊的BlockingQueue，底层基于PriorityBlockingQueue实现。不过， DelayQueue要求集合元素都实现Delay接口（该接口里只有一个long getDelay()方法）， DelayQueue根据集合元素的getDalay()方法的返回值进行排序。

下面以ArratBlockingQueue为例介绍阻塞队列的功能和用法

~~~java
public class BlockingQueueTest{
    public static void main(String [] args)throws Exception{
        //定义长度为2的阻塞队列
        BlockingQueue<String> bq=new ArrayBlockingQueue<>(2);
        bq.put("Java");//与bq.add("Java")、bq.offer("Java")相同
        bq.put("Java");
        bq.put("Java")//阻塞队列
    }
}
~~~

上面程序先定义一个大小为2的BlockingQueue，程序先向该队列中放入2个元素，此时队列还没有满，两个元素都可以放入，因此使用put()、add()和offer()方法效果完全一样。**当程序试图放入第三个元素时，如果使用put()方法尝试放入元素将会阻塞线程。如果使用add()方法尝试放入元素将会引发异常；如果使用offer()方法尝试放入元素则会返回false，元素不会被放入。**

与此类似的是，在BlockingQueue已空的情况下，**程序使用take()方法尝试取出元素将会阻塞线程；使用remove()方法尝试取出元素将引发异常；使用poll()方法尝试取出元素将返回false，元素不会被删除。**

下面程序利用BlockingQueue来实现线程通信：

~~~java
class Producer extends Thread{
    private BlockingQueue<String> bq;
    public Producer(BlockingQueue<String> bq){
        this.bq=bq;
    }
    public void run(){
        String [] strArr = new String[]{"Java","Mysql","Web"};
        for(int i=0;i<99999999;i++){
            System.out.pringln(getName()+"生产者生产集合元素");
            try{
                Thread.sleep(200);
                bq.put(strArr[i%3]);//放入元素如果队列已满则线程被阻塞
            }
            catch(Exception e){e.printStackTrace();}
            System.out.pringln(getName()+"生产者完成");
        }
    }
}
class Consumer extends Thread{
    private BlockingQueue<String> bq;
    public Consumer(BlockingQueue<String> bq){
        this.bq=bq;
    }
    public void run(){
        while(true){
            System.out.println(getName()+"消费者消费元素");
            try{
                Thread.sleep(200);
                bq.take();//取出元素如果队列已空则线程被阻塞
            }
            catch(Exception e){e.printStackTrace();}
            System.out.pringln(getName()+"消费完成"+bq);
        }
    }
}
public class BlockingQueueTest{
	public static void main(String[] args){
        BlockingQueue<String> bq=new ArrayBlockingQueue<>(1);
        //启动3个生产者线程
        new Producer(bq).start();
        new Producer(bq).start();
        new Producer(bq).start();
        //启动一个消费者线程
        new Consumer(bq).start();
    }
}
~~~

上面程序启动了3个生产者线程向BlockingQueue集合放入元素，启动了1个消费者线程从Blocking Queue集合取出元素。本程序的BlockingQueue集合容量为1，因此3个生产者线程无法连续放入元素，必须等待消费者线程取出一个元素后，3个生产者线程的其中之一才能放入一个元素。

3个生产者线程都想向BlockingQueue中放入元素，但只要其中一个线程向该队列中放入元素之后，其他生产者线程就必须等待，等待消费者线程取出BlockingQueue队列里的元素。

## 七、线程池

系统启动一个新线程的成本是比较高的，因为它涉及与操作系统交互。在这种情形下，使用线程池可以很好地提高性能，尤其是当程序中需要创建大量生存期很短暂的线程时，更应该考虑使用线程池。

与数据库连接池类似的是，线程池在系统启动时即创建大量空闲的线程，程序将一个Runnable对象或Callable对象传给线程池，线程池就会启动一个线程来执行它们的run()或call()方法，当run()或call()方法执行结束后，该线程并不会死亡，而是再次返回线程池中成为空闲状态，等待执行下一个Runnable对象的run()或call()方法。

### Java5的线程池

从Java 5开始，Java内建支持线程池。Java 5新增了一个Executors工厂类来产生线程池，该工厂类包含如下几个静态工厂方法来创建线程池：

- newCachedThreadPool()：创建一个具有缓存功能的线程池，系统根据需要创建线程，这些线程将会被缓存在线程池中。
- newFixedThreadPool(int nThreads)：创建一个可重用的、具有固定线程数的线程池。
- newSingleThreadExecutor()：创建一个只有单线程的线程池，它相当于调用newFixedThread Pool()方法时传入参数为1。
- newScheduledThreadPool(int corePoolSize)：创建具有指定线程数的线程池，它可以在指定延迟后执行线程任务。corePoolSize指池中所保存的线程数，即使线程是空闲的也被保存在线程池内。
- newSingleThreadScheduledExecutor()：创建只有一个线程的线程池，它可以在指定延迟后执行线程任务。

上面5个方法中的前3个方法返回一个**ExecutorService对象，该对象代表一个线程池**，它可以执行Runnable对象或Callable对象所代表的线程；而后2个方法返回一个**ScheduledExecutorService线程池，它是ExecutorService的子类**，它可以在指定延迟后执行线程任务。

ExecutorService代表尽快执行线程的线程池（只要线程池中有空闲线程，就立即执行线程任务），程序只要将一个Runnable对象或Callable对象（代表线程任务）提交给该线程池，该线程池就会尽快执行该任务。**ExecutorService里提供了如下3个方法**：

1. Future<?> submit(Runnable task)：将一个Runnable对象提交给指定的线程池，线程池将在有空闲线程时执行Runnable对象代表的任务。其中Future对象代表Runnable任务的返回值——但run()方法没有返回值，所以Future对象将在run()方法执行结束后返回null。但可以调用Future的isDone()、isCancelled()方法来获得Runnable对象的执行状态。
2. < T> Future< T> submit(Runnable task, T result)：将一个Runnable对象提交给指定的线程池，线程池将在有空闲线程时执行Runnable对象代表的任务。其中result显式指定线程执行结束后的返回值，所以Future对象将在run()方法执行结束后返回result。
3. < T> Future< T> submit(Callable< T> task)：将一个Callable对象提交给指定的线程池，线程池将在有空闲线程时执行Callable对象代表的任务。其中Future代表Callable对象里call()方法的返回值。

ScheduledExecutorService代表可在指定延迟后或周期性地执行线程任务的线程池，**ScheduledExecutorService提供了如下4个方法：**

1. ScheduledFuture<V> schedule(Callable<V> callable, long delay,TimeUnit unit)：指定callable任务将在delay延迟后执行。
2.  ScheduledFuture<?> schedule(Runnable command, long delay,TimeUnit unit)：指定command任务将在delay延迟后执行。
3. ScheduledFuture<?> scheduleAtFixedRate(Runnable command, longinitialDelay, long period, TimeUnit unit)：指定command任务将在delay延迟后执行，而且以设定频率重复执行。也就是说，在initialDelay后开始执行，依次在initialDelay+period、initialDelay+2 * period…处重复执行，依此类推。
4. ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay, long delay,TimeUnit unit)：创建并执行一个在给定初始延迟后首次启用的定期操作，随后在每一次执行终止和下一次执行开始之间都存在给定的延迟。如果任务在任一次执行时遇到异常，就会取消后续执行；否则，只能通过程序来显式取消或终止该任务。

当用完一个线程池后，应该调用该线程池的**shutdown()方法**，该方法将启动线程池的关闭序列，调用shutdown()方法后的线程池不再接收新任务，但会将以前所有已提交任务执行完成。当线程池中的所有任务都执行完成后，池中的所有线程都会死亡；另外也可以调用线程池的**shutdownNow()方法**来关闭线程池，该方法试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表。

使用线程池来执行线程任务的步骤如下。

（1）调用Executors类的静态工厂方法创建一个ExecutorService对象，该对象代表一个线程池。

（2）创建Runnable实现类或Callable实现类的实例，作为线程执行任务。

（3）调用ExecutorService对象的submit()方法来提交Runnable实例或Callable实例。

（4）当不想提交任何任务时，调用ExecutorService对象的shutdown()方法来关闭线程池。

~~~java
class MyThread implements Runnable{
    public void run(){
        for(int i=0;i<100;i++){
            System.out.pringln(Thread.currentThread().getName()+"的i值："+i);
        }
    }
}
public class ThreadPoolTest{
    public static void main(String[] args){
        ExecutorService pool= Executors.newFixedThreadPool(6);
        pool.submit(new MyThread());
        pool.submit(new MyThread());
        pool.shutdown();
    }
}
~~~

上面程序中创建Runnable实现类与最开始创建线程池并没有太大差别，创建了Runnable实现类之后程序没有直接创建线程、启动线程来执行该Runnable任务，而是通过线程池来执行该任务

### Java7新增的线程池

为了充分利用多CPU、多核CPU的性能优势，计算机软件系统应该可以充分“挖掘”每个CPU的计算能力，绝不能让某个CPU处于“空闲”状态。为了充分利用多CPU、多核CPU的优势，可以考虑把一个任务拆分成多个“小任务”，把多个“小任务”放到多个处理器核心上并行执行；当多个“小任务”执行完成之后，再将这些执行结果合并起来即可。

Java7提供了ForkJoinPool来支持将一个任务拆分成多个“小任务”并行计算，再把多个“小任务”的结果合并成总的计算结果。ForkJoinPool是ExecutorService的实现类，因此是一种特殊的线程池。ForkJoinPool提供了如下两个常用的构造器：

- ForkJoinPool(int parallelism)：创建一个包含parallelism个并行线程的ForkJoinPool。
- ForkJoinPool()：以Runtime.availableProcessors()方法的返回值作为parallelism参数来创建Fork JoinPool。

创建了ForkJoinPool实例之后，就可调用ForkJoinPool的submit(ForkJoinTask task)或invoke (ForkJoinTask task)方法来执行指定任务了。其中ForkJoinTask代表一个可以并行、合并的任务。ForkJoinTask是一个抽象类，它还有两个**抽象子类：RecursiveAction和RecursiveTask**。其中RecursiveTask代表有返回值的任务，而RecursiveAction代表没有返回值的任务。

## 八、线程相关类

### ThreadLocal类

通过使用ThreadLocal类可以简化多线程编程时的并发访问，使用这个工具类可以很简洁地隔离多线程程序的竞争资源。

ThreadLocal，是Thread Local Variable（线程局部变量）的意思，也许将它命名为ThreadLocalVar更加合适。线程局部变量（ThreadLocal）的功用其实非常简单，就是为每一个使用该变量的线程都提供一个变量值的副本，使每一个线程都可以独立地改变自己的副本，而不会和其他线程的副本冲突。从线程的角度看，就好像每一个线程都完全拥有该变量一样。

ThreadLocal类的用法非常简单，它只提供了如下3个public方法：

- T get():返回此线程局部变量中当前线程副本中的值
- void remove():删除此线程局部变量中当前线程的值
- void set(T value):设置此线程局部变量中当前线程副本中的值

~~~java
public class ThreadLocalTest{
	public static void main(){
        Account ac =new Account("初始名");
        //启动两个线程两个线程共用一个Account
        /*虽然两个线程共享同一个账户，但由于账户名是ThreadLocal类型的，所以每个线程都拥有各自的账户名副本，因此在i==6后将看到两个线程访问同一个账户时出现不同的账户名*/
        new MyTest(ac,"线程1").start();
        new MyTest(ac,"线程2").start();
    }
}
class Account{
    /*定义一个ThreadLocal类型的变量，这是一个局部线程变量*/
    private ThreadLocal<String> name =new ThreadLocal<>();
    public Account(String str){
        this.name.set(str);
        System.out.println("---"+this.name.get());
    }
    //省略setter和getter方法
}
class MyTest extends Thread{
    private Account account;
    public MyTest(Account account,String name){
        super(name);
        this.account=account;
    }
    public void run(){
        for(int i=0;i<10;i++){
            if(i==6){//当i=6时将账户名替换成当前线程名
                account.setName(getName());
            }
            System.out.println(account.getName()+"账户i的值"+i);
        }
    }
}
~~~

从上面程序可以看出，实际上账户名有3个副本，主线程一个，另外启动的两个线程各一个，它们的值互不干扰，每个线程完全拥有自己的ThreadLocal变量，这就是ThreadLocal的用途。

> ThreadLocal从另一个角度来解决多线程的并发访问，ThreadLocal将需要并发访问的资源复制多份，每个线程拥有一份资源，每个线程都拥有自己的资源副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的共享对象，在编写多线程代码时，可以把不安全的整个变量封装进ThreadLocal，或者把该对象与线程相关的状态使用ThreadLocal保存。
>
> ThreadLocal**并不能替代同步机制**，两者面向的问题领域不同。同步机制是为了同步多个线程对相同资源的并发访问，是多个线程之间进行通信的有效方式；而ThreadLocal是为了隔离多个线程的数据共享，从根本上避免多个线程之间对共享资源（变量）的竞争，也就不需要对多个线程进行同步了。

### 包装线程不安全的集合

Java集合的ArrayList、LinkedList、HashSet、TreeSet、HashMap、TreeMap等都是线程不安全的，也就是说，当多个并发线程向这些集合中存、取元素时，就可能会破坏这些集合的数据完整性。

如果程序中有多个线程可能访问以上这些集合，那么我们可以使用Collections提供的静态方法把这些集合包装成线程安全的集合。Collections提供了如下几个静态方法：

- < T> Collection< T> synchronizedCollection(Collection< T> c)：返回指定collection对应的线程安全的collection。
- static < T> List< T> synchronizedList(List< T> list)：返回指定List对象对应的线程安全的List对象。
- static <K,V> Map<K,V> synchronizedMap(Map<K,V> m)：返回指定Map对象对应的线程安全的Map对象。
- static < T> Set< T> synchronizedSet(Set< T> s)：返回指定Set对象对应的线程安全的Set对象。

### 线程安全的集合

在java.util.concurrent包下提供了大量支持高效并发访问的集合接口和实现类，这些线程安全的集合类可分为如下两类：

- 以Concurrent开头的集合类，如ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet、ConcurrentLinkedQueue和ConcurrentLinkedDeque。
- 以CopyOnWrite开头的集合类，如CopyOnWriteArrayList、CopyOnWriteArraySet。

其中以Concurrent开头的集合类代表了支持并发访问的集合，它们可以支持多个线程并发写入访问，这些写入线程的所有操作都是线程安全的，但读取操作不必锁定。

## 九、volatile关键字

Java语言提供了一种稍弱的同步机制，即volatile关键之，它具备两种特性，变量可见性和和禁止重排序。

## 十、Java锁

Java中往往是按照是否含有某一特性来定义锁，通过特性将锁进行分组归类：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/Java%E9%94%81%281.8%29.png)

### 乐观锁与悲观锁

**乐观锁**是一种乐观思想，即认为读多写少，遇到并发的可能性低，使用数据时认为不会被其他线程修改，所以不会上锁，在更新数据时判断之前是否有其他线程更新数据，在写入时先读取版本号，比较上一次版本号，一样则更新，失败则重复读——比较——写的过程。

乐观锁基本是采用CAS算法实现的，CAS是一种更新的原子操作，比较当前值是否一样，一样则更新否则失败。

**悲观锁**就是悲观思想，认为在使用数据时一定会被别人修改，所以操作时先上锁，保证数据不会被别人修改。

Java中synchronized就是悲观锁，AQS框架下的锁是先尝试乐观锁cas，获取不到才会转化为悲观锁，具体流程如下图：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/java_lock001.png)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/java_lock002.png)

- 悲观锁适合写操作多的场景，先加锁可以保证写操作时的数据正确
- 乐观锁适合读操作多的场景，不加锁能保证读操作性能大幅度提升

~~~java
//————————悲观锁调用——————————
    public synchronized void testMethod(){
        
    }
    //ReentrantLock
    private ReentrantLock lock = new ReentrantLock();
    
    public void function(){
        lock.lock();
        lock.unlock();
    }
    
    //————————乐观锁调用————————————
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger();
        atomicInteger.incrementAndGet();//自增1
    }
~~~

悲观锁是在显式的锁定之后再操作同步资源，而乐观锁则直接去操作同步资源。乐观锁的主要实现方式 是“CAS” 的技术

CAS全称 Compare And Swap（比较与交换），是一种无锁算法。在不使用锁（没有线程被阻塞）的情况下实现多线程之间的变量同步。java.util.concurrent包中的原子类就是通过CAS来实现了乐观锁。

CAS操作包含三个操作数——内存位置、预期原值及新值。执行CAS操作的时候，将内存位置的值与预期原值比较，如果相匹配，那么处理器会自动将该位置值更新为新值，否则，处理器不做任何操作。

AtomicInteger和unsafe源码（Java8）：

~~~java
- unsafe： 获取并操作内存的数据。
- valueOffset： 存储value在AtomicInteger中的偏移量。
- value： 存储AtomicInteger的int值，该属性需要借助volatile关键字保证其在线程间是可见的。
public class AtomicInteger extends Number implements Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    private volatile int value;
    static {
        try {
            valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception var1) {
            throw new Error(var1);
        }
    }
    //自增方法
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
}
//-----------------Unsafe.java------------
/**
	*  CAS
  * @param o         包含要修改field的对象
  * @param offset    对象中某field的偏移量
  * @param expected  期望值
  * @param update    更新值
  * @return          true | false
  * 以下三个native方法主要是CAS相关操作
  */
public final native boolean compareAndSwapObject(Object o, long offset,  Object expected, Object update);

public final native boolean compareAndSwapInt(Object o, long offset, int expected,int update);
  
public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);

 public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = this.getIntVolatile(o, offset);//getIntVolatile方法获取对象中offset偏移地址对应的整型field的值,
        } while(!this.compareAndSwapInt(o, offset, v, v + delta));

        return v;
    }
~~~

> getAndAddInt()循环获取给定对象o中的偏移量处的值v，然后判断内存值是否等于v。如果相等则将内存值设置为 v + delta，否则返回false，继续循环进行重试，直到设置成功才能退出循环，并且将旧值返回。整个“比较+更新”操作封装在compareAndSwapInt()中，在JNI里是借助于一个CPU指令完成的，属于原子操作，可以保证多个线程都能够看到同一个变量的修改值。
>
> JDK通过CPU的cmpxchg指令，去比较寄存器中的 A 和 内存中的值 V。如果相等，就把要写入的新值 B 存入内存中。如果不相等，就将内存值 V 赋值给寄存器中的值 A。然后通过Java代码中的while循环再次调用cmpxchg指令进行重试，直到设置成功为止。

**CAS的问题**

1. ABA问题：

   如果内存值原来是A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。ABA问题的解决思路就是在变量前面添加版本号，每次变量更新的时候都把版本号加一，这样变化过程就从“A－B－A”变成了“1A－2B－3A”

2. 循环时间长开销大。CAS操作如果长时间不成功，会导致其一直自旋，给CPU带来非常大的开销。

3. 能保证一个共享变量的原子操作。对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量操作时，CAS是无法保证操作的原子性的。

### 自旋锁与适应性自旋锁

自旋锁原理非常简单，如果持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起状态，它们只需要等一等（自旋），而为了让当前线程“稍等一下”，我们需让当前线程进行自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

以下为自旋锁流程：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E8%87%AA%E6%97%8B%E9%94%81%E6%B5%81%E7%A8%8B.png)

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

自旋锁的实现原理同样也是CAS，AtomicInteger中调用unsafe进行自增操作的源码中的do-while循环就是一个自旋操作（源码见乐观锁），如果修改数值失败则通过循环来执行自旋，直至修改成功。

自旋锁在JDK1.4.2中引入，使用-XX:+UseSpinning来开启。JDK 6中变为默认开启，并且引入了自适应的自旋锁（适应性自旋锁）。

自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

### 无锁、偏向锁、轻量级锁与重量级锁

这四种锁是指锁的状态，专门针对synchronized的。

在这之前，首先synchronized为什么能实现线程同步？

这需要了解Java对象头和Monitor。

**Java对象头**

> synchronized是悲观锁，在操作同步资源之前需要加锁，这把锁就是加在对象头里的。以Hotspot为例，hotspot的对象头主要包括两部分数据：mark word（标记字段）、class pointer（类型指针）
>
> mark word：默认存储对象的hashcode分代年龄和锁标志位信息。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。
>
> class point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

**Monitor**

> Monitor可以理解为一个同步工具或一种同步机制，通常被描述为一个对象。每一个Java对象就有一把看不见的锁，称为内部锁或者Monitor锁。
>
> Monitor是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联，同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。

synchronized通过Monitor来实现线程同步，Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的线程同步。

阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态转换需要耗费处理器时间。如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长。这种方式就是synchronized最初实现同步的方式，这就是JDK 6之前synchronized效率低的原因。这种依赖于操作系统Mutex Lock所实现的锁我们称之为“重量级锁”，JDK 6中为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”。

**所以目前锁一共有4种状态，级别从低到高依次是：无锁、偏向锁、轻量级锁和重量级锁。锁状态只能升级不能降级。**

以下是四种锁状态的mark word的内容：

| 锁状态   | 存储内容                                                | 存储内容 |
| -------- | ------------------------------------------------------- | -------- |
| 无锁     | 对象的hashCode、对象分代年龄、是否是偏向锁（0）         | 01       |
| 偏向锁   | 偏向线程ID、偏向时间戳、对象分代年龄、是否是偏向锁（1） | 01       |
| 轻量级锁 | 指向栈中所记录的指针                                    | 00       |
| 重量级锁 | 指向互斥量的指针                                        | 10       |

**无锁**

无锁没有对资源进行锁定，所有线程都能访问修改同一个资源，但只有一个线程能修改成功。

无锁的特点就是修改操作在循环内进行，线程会不断的尝试修改共享资源。如果没有冲突就修改成功并退出，否则就会继续循环尝试。如果有多个线程修改同一个值，必定会有一个线程能修改成功，而其他修改失败的线程会不断重试直到修改成功。上面我们介绍的CAS原理及应用即是无锁的实现。无锁无法全面代替有锁，但无锁在某些场合下的性能是非常高的。

**偏向锁**

偏向锁是指，一个同步代码一直被一个线程所访问，该线程会自动获取锁，降低锁获取的代价。

当一个线程访问同步代码块并获取锁时，会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令即可。

偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态。撤销偏向锁后恢复到无锁（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

**轻量级锁**

轻量级锁是指锁是偏向锁的时候，被另外的线程所访问，偏向锁就会升级成轻量级锁，其他线程会通过自旋的形式尝试获取锁，不会阻塞，从而提高性能。

在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，然后拷贝对象头中的Mark Word复制到锁记录中。

拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock Record里的owner指针指向对象的Mark Word。

如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，表示此对象处于轻量级锁定状态。

如果轻量级锁的更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行，否则说明多个线程竞争锁。

若当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁。

**重量级锁**

升级为重量级锁时，锁标志的状态值变为“10”，此时Mark Word中存储的是指向重量级锁的指针，此时等待锁的线程都会进入阻塞状态。

整体的锁状态升级流程如下：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E9%94%81%E5%8D%87%E7%BA%A7.png)

偏向锁通过对比Mark Word解决加锁问题，避免执行CAS操作。而轻量级锁是通过用CAS操作和自旋来解决加锁问题，避免线程阻塞和唤醒而影响性能。重量级锁是将除了拥有锁的线程以外的线程都阻塞。

### 公平锁与非公平锁

公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体吞吐效率相对非公平锁要低，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E5%85%AC%E5%B9%B3%E9%94%81.png)

以水井为例，由管理员看守且管理员只有一把锁，如果前面有人打水，那么这个d想打水就必须排队，且必须去队尾排队，管理员只会给队伍中最前面的人锁并让你去打水。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81.png)

如果是非公平锁，即便队伍中有等待的人，但如果刚好上一个人刚打完水交还锁且管理员还没有允许下一个人去打水时，这时来了一个插队的人，这个插队的人可以直接去拿锁不需要排队。

以ReentrantLock源码（Java8）为例：

~~~java
public class ReentrantLock implements Lock, Serializable {

    private final ReentrantLock.Sync sync;

    public ReentrantLock() {
        this.sync = new ReentrantLock.NonfairSync();
    }

    public ReentrantLock(boolean var1) {
        this.sync = (ReentrantLock.Sync)(var1 ? new ReentrantLock.FairSync() : new ReentrantLock.NonfairSync());
    }
    
    static final class FairSync extends ReentrantLock.Sync {
        //。。。
    }
    static final class NonfairSync extends ReentrantLock.Sync {
        //。。。
    }
    abstract static class Sync extends AbstractQueuedSynchronizer {
    	//。。。
    }
}
~~~

由源码可知ReentrantLock里面有一个内部类Sync，他又FairSync公平锁和NonfairSync非公平锁两个子类。默认使用非公平锁。

### 可重入锁与非可重入锁

可重入锁又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的一个优点是可一定程度避免死锁。

~~~java
    public synchronized void doSomethings(){
        System.out.println("do somethings");
        doOthers();
    }

    public synchronized void doOthers(){
        System.out.println("do others");
    }
~~~

在上面的代码中,两个方法都是被synchronized修饰的，doSomething()方法中调用diOthers()方法。因为内置锁是可重入的，所以同一个线程在调用doOthers()时可以直接获得当前对象的锁，进入doOthers()操作。

如果是一个不可重入锁，那么当前线程在调用doOthers()之前需要将执行doSomething()时获取当前对象的锁释放掉，实际上该对象锁已被当前线程所持有，且无法释放。所以此时会出现死锁。

以水井为例，有多个人在排队打水，此时管理员允许锁和同一个人的多个水桶绑定。这个人用多个水桶打水时，第一个水桶和锁绑定并打完水之后，第二个水桶也可以直接和锁绑定并开始打水，所有的水桶都打完水之后打水人才会将锁还给管理员。这个人的所有打水流程都能够成功执行，后续等待的人也能够打到水。这就是可重入锁。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81.png)

如果是非可重入锁的话，此时管理员只允许锁和同一个人的一个水桶绑定。第一个水桶和锁绑定打完水之后并不会释放锁，导致第二个水桶不能和锁绑定也无法打水。当前线程出现死锁，整个等待队列中的所有线程都无法被唤醒。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_lock/%E9%9D%9E%E5%8F%AF%E9%87%8D%E5%85%A5%E9%94%81.png)

下面将展示源码：

~~~java
public class ReentrantLock implements Lock, Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    private final ReentrantLock.Sync sync;
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        
        final boolean nonfairTryAcquire(int var1) {
            Thread var2 = Thread.currentThread();
            int var3 = this.getState();
            if (var3 == 0) {
                if (this.compareAndSetState(0, var1)) {
                    this.setExclusiveOwnerThread(var2);
                    return true;
                }
            } else if (var2 == this.getExclusiveOwnerThread()) {
                /*当线程尝试获取锁时，可重入锁先尝试获取并更新status值，如果status == 0表示没有其他线程在执行同步代码，则把status置为1，当前线程开始执行。如果status != 0，则判断当前线程是否是获取到这个锁的线程，如果是的话执行status+1，且当前线程可以再次获取锁。而非可重入锁是直接去获取并尝试更新当前status的值，如果status != 0的话会导致其获取锁失败，当前线程阻塞。*/
                int var4 = var3 + var1;
                if (var4 < 0) {
                    throw new Error("Maximum lock count exceeded");
                }

                this.setState(var4);
                return true;
            }

            return false;
        }

        protected final boolean tryRelease(int var1) {
            /*释放锁时，可重入锁同样先获取当前status的值，在当前线程是持有锁的线程的前提下。如果status-1 == 0，则表示当前线程所有重复获取锁的操作都已经执行完毕，然后该线程才会真正释放锁。而非可重入锁则是在确定当前线程是持有锁的线程之后，直接将status置为0，将锁释放。*/
            int var2 = this.getState() - var1;
            if (Thread.currentThread() != this.getExclusiveOwnerThread()) {
                throw new IllegalMonitorStateException();
            } else {
                boolean var3 = false;
                if (var2 == 0) {
                    var3 = true;
                    this.setExclusiveOwnerThread((Thread)null);
                }

                this.setState(var2);
                return var3;
            }
        }
}
~~~

通过上面的公平锁已知ReentrantLock中有内部类，内部类继承AQS父类，其父类AQS中维护了一个同步状态status来计数重入次数，status初始值为0。

~~~java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements Serializable {
    
    private volatile int state;
}
~~~

### 共享锁与排他锁

共享锁也叫排他锁，是指该锁一次只能被一个线程所持有，如果线程T对数据A加上排他锁后，其他线程不能在对A加任何类型的锁。获得排他锁的线程既能读数据又能修改数据。JDK中的synchronized和JUC（java.util.concurrent）中的Lock实现类就是互斥锁。

共享锁是指该锁可以被多个线程所持有。如果线程T对数据A假声共享锁后，则其他线程只能对A再加共享锁，不能加排他锁。获得共享锁的线程只能读数据，不能修改数据。对ReentrantReadWriteLock而言，其读锁是共享锁，其写锁是独占锁。读锁的共享性可保证并发读是非常高效的，读写、写读、写写的过程都是互斥的。

下面是ReentrantReadWriteLock的部分源码（Java8）：

~~~java
public class ReentrantReadWriteLock implements ReadWriteLock, Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    private final ReentrantReadWriteLock.ReadLock readerLock;
    private final ReentrantReadWriteLock.WriteLock writerLock;
    final ReentrantReadWriteLock.Sync sync;
    private static final Unsafe UNSAFE;
    private static final long TID_OFFSET;

    public ReentrantReadWriteLock() {
        this(false);
    }

    public ReentrantReadWriteLock(boolean var1) {
        this.sync = (ReentrantReadWriteLock.Sync)(var1 ? new ReentrantReadWriteLock.FairSync() : new ReentrantReadWriteLock.NonfairSync());
        this.readerLock = new ReentrantReadWriteLock.ReadLock(this);
        this.writerLock = new ReentrantReadWriteLock.WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() {
        return this.writerLock;
    }

    public ReentrantReadWriteLock.ReadLock readLock() {
        return this.readerLock;
    }
    
    //
        public static class ReadLock implements Lock, Serializable {
        private final ReentrantReadWriteLock.Sync sync;
        protected ReadLock(ReentrantReadWriteLock var1) {
            this.sync = var1.sync;
        }
    }
    
    //
        public static class WriteLock implements Lock, Serializable {
        private final ReentrantReadWriteLock.Sync sync;
        protected WriteLock(ReentrantReadWriteLock var1) {
            this.sync = var1.sync;
        }
    }
}
~~~

我们看到ReentrantReadWriteLock有两把锁：ReadLock和WriteLock，由词知意，一个读锁一个写锁，合称“读写锁”。再进一步观察可以发现ReadLock和WriteLock是靠内部类Sync实现的锁。Sync是AQS的一个子类，这种结构在CountDownLatch、ReentrantLock、Semaphore里面也都存在。

在ReentrantReadWriteLock里面，读锁和写锁的锁主体都是Sync，但读锁和写锁的加锁方式不一样。读锁是共享锁，写锁是独享锁。读锁的共享锁可保证并发读非常高效，而读写、写读、写写的过程互斥，因为读锁和写锁是分离的。所以ReentrantReadWriteLock的并发性相比一般的互斥锁有了很大提升。

在AQS中有state字段，该字段用来描述有多少线程持有锁。

再独享锁中这个值通常为0或1（如果是重入锁就是重入的次数），在共享锁中state就是持有锁的数量。但是在ReentrantReadWriteLock中有读、写两把锁，所以需要在一个整型变量state上分别描述读锁和写锁的数量（或者也可以叫状态）。于是将state变量“按位切割”切分成了两个部分，高16位表示读锁状态（读锁个数），低16位表示写锁状态（写锁个数）。

以下是写锁的加锁代码：

~~~java
protected final boolean tryAcquire(int var1) {
            Thread var2 = Thread.currentThread();
            int var3 = this.getState();//获取当前锁的个数
            int var4 = exclusiveCount(var3);//获取写锁的个数
            if (var3 != 0) {//如果已经有线程持有了锁
                //如果写线程数var4为0，即存在读锁，或者持有锁不是当前线程就返回失败
                if (var4 != 0 && var2 == this.getExclusiveOwnerThread()) {
                    //如果写锁的数量大于最大数65535(2的16次幂-1)就抛出Error
                    if (var4 + exclusiveCount(var1) > 65535) {
                        throw new Error("Maximum lock count exceeded");
                    } else {
                        this.setState(var3 + var1);
                        return true;
                    }
                } else {
                    return false;
                }
                //如果当写线程数为0，且当前相乘需要阻塞那么就返回失败；或者如果通过CAS增加写线程数失败也返回失败
            } else if (!this.writerShouldBlock() && this.compareAndSetState(var3, var3 + var1)) {
                this.setExclusiveOwnerThread(var2);
                return true;
            } else {
                return false;
            }
        }
~~~

tryAcquire()除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：必须确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。

因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，然后等待的读写线程才能够继续访问读写锁，同时前次写线程的修改对后续的读写线程可见。

以下是读锁的加锁代码：

~~~java
        protected final int tryAcquireShared(int var1) {
            Thread var2 = Thread.currentThread();
            int var3 = this.getState();
            if (exclusiveCount(var3) != 0 && this.getExclusiveOwnerThread() != var2) {
                return -1;//如果去他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态
            } else {
                int var4 = sharedCount(var3);
                if (!this.readerShouldBlock() && var4 < 65535 && this.compareAndSetState(var3, var3 + 65536)) {
                    if (var4 == 0) {
                        this.firstReader = var2;
                        this.firstReaderHoldCount = 1;
                    } else if (this.firstReader == var2) {
                        ++this.firstReaderHoldCount;
                    } else {
                        ReentrantReadWriteLock.Sync.HoldCounter var5 = this.cachedHoldCounter;
                        if (var5 != null && var5.tid == ReentrantReadWriteLock.getThreadId(var2)) {
                            if (var5.count == 0) {
                                this.readHolds.set(var5);
                            }
                        } else {
                            this.cachedHoldCounter = var5 = (ReentrantReadWriteLock.Sync.HoldCounter)this.readHolds.get();
                        }

                        ++var5.count;
                    }

                    return 1;
                } else {
                    return this.fullTryAcquireShared(var2);
                }
            }
        }
~~~

可以看到在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是“1<<16”。所以读写锁才能实现读读的过程共享，而读写、写读、写写的过程互斥。

## 十一、总结

### sleep与wait区别

1. sleep是属于Thread类中的，而wait方法是属于Object类中的
2. sleep方法线程不会释放对象锁
3. 调用wait方法时，线程会放弃对象锁，进入等待锁定池，只有针对此对象调用notify方法后本线程才进入对象锁定池准备获取对象所进入运行状态

# 正则表达式

### 概述

正则表达式定义了字符串的模式。正则表达式可以用来搜索、编辑或处理文本。正则表达式并不仅限于某一种语言，但是在每种语言中有细微的差别。

java.util.regex 包主要包括以下三个类：

- Pattern 类：

  pattern 对象是一个正则表达式的编译表示。Pattern 类没有公共构造方法。要创建一个 Pattern 对象，你必须首先调用其公共静态编译方法，它返回一个 Pattern 对象。该方法接受一个正则表达式作为它的第一个参数。

- Matcher 类：

  Matcher 对象是对输入字符串进行解释和匹配操作的引擎。与Pattern 类一样，Matcher 也没有公共构造方法。你需要调用 Pattern 对象的 matcher 方法来获得一个 Matcher 对象。

- PatternSyntaxException：

  PatternSyntaxException 是一个非强制异常类，它表示一个正则表达式模式中的语法错误。

### 语法

> 根据 Java Language Specification 的要求，Java 源代码的字符串中的反斜线被解释为 Unicode 转义或其他字符转义。因此必须在字符串字面值中使用两个反斜线，表示正则表达式受到保护，不被 Java 字节码编译器解释。例如，当解释为正则表达式时，字符串字面值 "\b" 与单个退格字符匹配，而 "\ \b" 与单词边界匹配。字符串字面值 `"\(hello\)" `是非法的，将导致编译时错误；要与字符串 (hello) 匹配，必须使用字符串字面值    `"\\(hello\\)"`。

#### **常见匹配符号**

1. 任意一个字符串表示匹配任意对应的字符，如a匹配a，7匹配7，-匹配-
2. ^regex正则必须匹配字符串开头
3. regex$正则必须匹配字符串结尾
4. []代表匹配中括号中的任一个字符如[abc]匹配a或b或c
5. -在中括号中不匹配自己，比如[a-z]代表匹配匹配26个小写字母中的任一个；[a-zA-Z]匹配大小写共52个字母中任一个；[0-9]匹配十个数字中任一个。
6. ^在中括号中与外面不同。在外时，就表示开头，如^7[0-9]表示匹配开头是7的，且第二位是任一数字的字符串。如果在中括号里面，表示除了这个字符之外的任意字符(包括数字，特殊字符)，如`[^abc]`表示匹配出去abc之外的其他任一字符。
7. 点.表示匹配任意的字符
8. XY表示X后面跟着Y，这里X和Y分别是正则表达式的一部分。
9. 匹配 *x* 或 *y*。例如，'z|food' 匹配"z"或"food"。'(z|f)ood' 匹配"zood"或"food"。

示例：

- `[jpg|png]` 代表匹配 `j` 或 `p` 或 `g` 或 `p` 或 `n` 或 `g` 中的任意一个字符。
- `(jpg|png)` 代表匹配 `jpg` 或 `png`。

#### **元字符**（是一个预定义的字符）

1. \d表示数字
2. \D表示非数字
3. \s表示匹配任何空白字符，包括空格、制表符、换页符等。与 [ \f\n\r\t\v] 等效。
4. \S匹配任何非空白字符。与 [ ^\f\n\r\t\v] 等效。
5. \w表示字母数字下划线匹配任何字类字符，包括下划线。与"[A-Za-z0-9_]"等效。
6. \W与任何非单词字符匹配。与[ ^A-Za-z0-9_]等效。

#### **限定符**（定义了一个元素可以发生的概率）

1. ?表示出现0次或1次。例如，"do(es)?"匹配"do"或"does"中的"do"。? 等效于 {0,1}。
2. +表示出现1次或多次。例如，"zo+"与"zo"和"zoo"匹配，但与"z"不匹配。+ 等效于 {1,}。
3. *表示出现0次、1次或多次。  如，zo *匹配“z”和“zoo”。等效于{0,}
4. {n}表示出现n次。
5. {n,m}表示出现n~m次。
6. {n,}表示出现n次或n次以上。

#### **分组和反向引用**

小括号 `()` 可以达到对正则表达式进行分组的效果。

在以正则表达式替换字符串的语法中，是通过 `$` 来引用分组的反向引用，`$0` 是匹配完整模式的字符串（注意在 JavaScript 中是用 `$&` 表示）；`$1` 是第一个分组的反向引用；`$2` 是第二个分组的反向引用，以此类推。

例如：

```java
package com.wuxianjiezh.demo.regex;
public class RegexTest {

    public static void main(String[] args) {
        // 去除单词与 , 和 . 之间的空格
        String Str = "Hello , World .";
        String pattern = "(\\w)(\\s+)([.,])";
        // $0 匹配 `(\w)(\s+)([.,])` 结果为 `o空格,` 和 `d空格.`
        // $1 匹配 `(\w)` 结果为 `o` 和 `d`
        // $2 匹配 `(\s+)` 结果为 `空格` 和 `空格`
        // $3 匹配 `([.,])` 结果为 `,` 和 `.`
        System.out.println(Str.replaceAll(pattern, "$1$3")); // Hello, World.
    }
}
```

当我们在小括号 `()` 内的模式开头加入 `?:`，那么表示这个模式仅分组，但不创建反向引用。

#### **否定先行断言**

我们可以创建否定先行断言模式的匹配，即某个字符串后面不包含另一个字符串的匹配模式。

否定先行断言模式通过 `(?!pattern)` 定义。比如，我们匹配后面不是跟着 "b" 的 "a"：a(?!b)

#### Java中的反斜杠

反斜杠 `\` 在 Java 中表示转义字符，这意味着 `\` 在 Java 拥有预定义的含义。

这里例举两个特别重要的用法：

- 在匹配 `.` 或 `{` 或 `[` 或 `(` 或 `?` 或 `$` 或 `^` 或 `*` 这些特殊字符时，需要在前面加上 `\\`，比如匹配 `.` 时，Java 中要写为 `\\.`，但对于正则表达式来说就是 `\.`。
- 在匹配 `\` 时，Java 中要写为 `\\\\`，但对于正则表达式来说就是 `\\`。

**注意**：Java 中的正则表达式字符串有两层含义，首先 Java 字符串转义出符合正则表达式语法的字符串，然后再由转义后的正则表达式进行模式匹配。

### 在字符串中使用正则表达式

在 Java 中有四个内置的运行正则表达式的方法，分别是 `matches()`、`split())`、`replaceFirst()`、`replaceAll()`。注意 `replace()` 方法不支持正则表达式。

| 方法                                  | 描述                                 |
| ------------------------------------- | ------------------------------------ |
| s.matchs("regex")                     | 当且仅当正则匹配整个字符串时返回true |
| s.split("regex")                      | 按匹配的正则表达式切片字符串         |
| s,replaceFirst("regex","replacement") | 替换首次匹配的字符串片段             |
| s.replaceAll("regex","replacement")   | 替换所有匹配的字符                   |

### 模式和匹配

Java 中使用正则表达式需要用到两个类，分别为 `java.util.regex.Pattern` 和 `java.util.regex.Matcher`。

```java
package com.wuxianjiezh.regex;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
public class RegexTest {

    public static void main(String[] args) {
        String text = "Hello Regex!";
		//第一步通过正则表达式创建模式对象Pattern
        Pattern pattern = Pattern.compile("\\w+");
        // Java 中忽略大小写，有两种写法：
        // Pattern pattern = Pattern.compile("\\w+", Pattern.CASE_INSENSITIVE);
        // Pattern pattern = Pattern.compile("(?i)\\w+"); // 推荐写法
        //第二步，通过模式对象根据自定字符串创建匹配对象
        Matcher matcher = pattern.matcher(text);
        //第三步，根据匹配对象Matcher根据正则表达式创建对象
        // 遍例所有匹配的序列
        while (matcher.find()) {
            System.out.print("Start index: " + matcher.start());
            System.out.print(" End index: " + matcher.end() + " ");
            System.out.println(matcher.group());
        }
        // 创建第两个模式，将空格替换为 tab
        Pattern replace = Pattern.compile("\\s+");
        Matcher matcher2 = replace.matcher(text);
        System.out.println(matcher2.replaceAll("\t"));
    }
}
```

### 常见示例

#### 匹配中文

`[\u4e00-\u9fa5]+` 代表匹配中文字。

#### 数字范围

正则表达式匹配数字范围时，首先要确定最大值与最小值，最后写中间值。比如，匹配 1990 到 2017。

**注意：**这里有个新手易范的错误，就是正则 `[1990-2017]`，实际这个正则只匹配 `0` 或 `1` 或 `2` 或 `7` 或 `9` 中的任一个字符。

```
"^1990$|^199[1-9]$|^20[0-1][0-6]$|^2017$" 为判断 1990-2017 正确的正则表达式
```

#### img标签的匹配

```java
// 这里考虑了一些不规范的 img 标签写法，比如：空格、引号
Pattern pattern = Pattern.compile("<img\\s+src=(?:['\"])?(?<src>\\w+.(jpg|png))(?:['\"])?\\s*/>");
```

