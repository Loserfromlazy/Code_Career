# Java高并发编程学习笔记第二部分Java内置锁篇

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第二部分Java内置锁篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java内置锁的核心原理，主要就是对synchronized进行研究学习。

Java内置锁是一个互斥锁，这就意味着最多只有一个线程能够获得该锁，当线程B尝试去获得线程A持有的内置锁时，线程B必须等待或者阻塞，直到线程A释放这个锁，如果线程A不释放这个锁，那么线程B将永远等待下去。Java中每个对象都可以用作锁，这些锁称为内置锁。线程进入同步代码块或方法时会自动获得该锁，在退出同步代码块或方法时会释放该锁。获得内置锁的唯一途径就是进入这个锁保护的同步代码块或方法。

## 2.1 线程安全问题

什么是线程安全呢？当多个线程并发访问某个Java对象（Object）时，无论系统如何调度这些线程，也无论这些线程将如何交替操作，这个对象都能表现出一致的、正确的行为，那么对这个对象的操作是线程安全的。

我们可以以自增操作为例，这个操作就不是线程安全的，我们可以举个例子：

```java
public class MyPlus {
    private Integer count = 0;

    public Integer getCount() {
        return count;
    }

    public void selfPlus(){
        count++;
    }

    /**
     * 测试方法：设置一个临界资源然后开10个线程访问临界资源，制造多线程环境
     */
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10);
        MyPlus myPlus = new MyPlus();
        for (int i = 0; i < 10; i++) {
            new Thread(()->{
                for (int j = 0; j < 1000; j++) {
                    myPlus.selfPlus();
                }
                latch.countDown();
            }).start();
        }
        latch.await();
        System.out.println("应该达到的数值"+10*1000);
        System.out.println("实际达到的数据"+myPlus.getCount());
        System.out.println("差了"+(10*1000- myPlus.getCount()));
    }
}
```

运行结果：

应该达到的数值10000
实际达到的数据4894
差了5106

> CountDownLatch是在java1.5被引入，存在于java.util.cucurrent包下。
>
> CountDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。它是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。
>
> 它其中最主要使用的方法源码如下：
>
> 总的来说就是调用await进行等待，直到count降为0，而countDown可以使count减一。源码和官方文档的翻译如下：
>
> ```java
> //导致当前线程等待，直到锁存器计数为零，除非线程被中断或指定的等待时间已经过。
> //如果当前计数为零，则该方法立即返回值为true。
> //如果当前计数大于0，则当前线程将被禁用，并处于休眠状态，直到发生以下三种情况之一:
> //调用了countDown方法，计数为零
> //其他线程中断当前线程
> //指定的等待时间过去
> public void await() throws InterruptedException {
>     sync.acquireSharedInterruptibly(1);
> }
> //减少锁存器的计数，如果计数为0，则释放所有等待的线程。
> //如果当前计数大于零，则计数递减。如果新计数为零，那么所有等待的线程都将重新启用，用于线程调度目的。
> public void countDown() {
>     sync.releaseShared(1);
> }
> ```

实际上，一个自增运算符是一个复合操作，至少包括三个JVM指令：“内存取值”“寄存器增加1”和“存值到内存”。这三个指令在JVM内部是独立进行的，中间完全可能会出现多个线程并发进行。

在我们日常开发过程中，我们会倾向于当前代码是串行、线性的方式执行的，但是这样会忽视多个线程并行执行，从而导致意想不到的结果。例如上面的例子，单线程操作`count++;`这样的临界资源是不会有问题的，但是在多线程情况下，比如说有三个线程同时读到了count=100，但是执行完后本应该是103，但是在每一个线程中却只自增了1，所以会写回101，这样就会出现问题。尤其是在电商这样的高并发环境下，一个不小心就会造成超卖等多线程问题，给商家造成损失。

我们说上面这个MyPlus的count是临界资源，那么什么是临界资源呢？其实临界区资源表示一种可以被多个线程使用的公共资源或共享数据，但是每一次只能有一个线程使用它。一旦临界区资源被占用，想使用该资源的其他线程则必须等待。上面的自增例子中count就是临界资源，selfPlus就是临界区代码段。

在并发情况下，临界区资源是受保护的对象。临界区代码段（Critical Section）是每个线程中访问临界资源的那段代码，多个线程必须互斥地对临界区资源进行访问。线程进入临界区代码段之前，必须在进入区申请资源，申请成功之后执行临界区代码段，执行完成之后释放资源。

在Java中，我们可以使用synchronized关键字同步代码块，对临界区代码段进行排他性保护，当然我们也可以使用Lock显示锁或者原子变量对临界区代码段进行排他性保护，当然本文只对内置锁进行学习介绍，显示锁和原子变量将在后面的学习笔记中介绍。

## 2.2 synchronized基本使用

在Java中，线程同步使用最多的方法是使用synchronized关键字。每个Java对象都隐含有一把锁，这里称为Java内置锁（或者对象锁、隐式锁）。使用synchronized（syncObject）调用相当于获取syncObject的内置锁，所以可以使用内置锁对临界区代码段进行排他性保护。

> synchronized同步锁的释放有两种方式：一种场景是synchronized块（代码块或者方法）正确执行完毕，监视锁自动释放；另一种场景是程序出现异常，非正常退出synchronized块，监视锁也会自动释放。所以，使用synchronized块时不必担心监视锁的释放问题。

### 2.2.1 同步方法

synchronized可以对方法进行同步，当使用synchronized关键字修饰一个方法的时候，该方法被声明为同步方法。用法：

~~~
修饰符 synchronized 返回值 function(){  
}
~~~

以上面2.1中的的例子为例，我们给自增方法增加synchronized对方法进行同步，这时同一时间就只能有一个线程进入同步方法（临界区代码段），如果其他线程需要执行同一个方法，那么只能等待和排队。然后在试一下看看结果，这里就只贴一下关键方法的改动，其余代码与上面例子中一致：

```java
public synchronized void selfPlus(){
    count++;
}
```

运行结果：

应该达到的数值10000
实际达到的数据10000
差了0

### 2.2.2 同步代码块

对于小的临界区，我们直接在方法声明中设置synchronized同步关键字，可以避免竞态条件的问题。但是对于较大的临界区代码段，为了执行效率，最好将同步方法分为小的临界区代码段。

我们举个例子：

```java
public class TestCompute {
    private int multiply1 = 0;
    private int multiply2 = 0;
    public synchronized void compute(int value1,int value2){
        this.multiply1 *= value1;
        this.multiply2 *= value2;
    }
}
```

当线程进入compute方法时，实际上是获取了multiply和divide两个资源的控制权，这时在操作multiply时，divide变量就会被闲置。（PS：这里仅是为了举例子）

所以，将synchronized加在方法上，如果其保护的临界区代码段包含的临界区资源（要求是相互独立的）多于一个，就会造成临界区资源的闲置等待，进而会影响临界区代码段的吞吐量。为了提升吞吐量，可以将synchronized关键字放在函数体内，同步一个代码块。用法如下：

~~~java
synchronized(syncObject){
    //代码块
}
~~~

在synchronized同步块后边的括号中是一个syncObject对象，代表着进入临界区代码段需要获取syncObject对象的监视锁，或者说将syncObject对象监视锁作为临界区代码段的同步锁。由于每一个Java对象都有一把监视锁，因此任何Java对象都能作为synchronized的同步锁。

我们改造上面的例子：

```java
public class TestCompute {
    private int multiply1 = 0;
    private int multiply2 = 0;
    //这两个属性没有参与业务处理，仅仅利用了这两个属性的内置锁功能。
    private final Integer multiply1Lock = 1;
    private final Integer multiply2Lock = 2;
    public synchronized void compute(int value1,int value2){
        synchronized (this.multiply1Lock){
            this.multiply1 *= value1;
        }
        synchronized (this.multiply2Lock){
            this.multiply2 *= value2;
        }
    }
}
```

> synchronized方法和synchronized同步块有什么区别呢？总体来说，synchronized方法是一种粗粒度的并发控制，某一时刻只能有一个线程执行该synchronized方法；而synchronized代码块是一种细粒度的并发控制，处于synchronized块之外的其他代码是可以被多个线程并发访问的。在一个方法中，并不一定所有代码都是临界区代码段，可能只有几行代码会涉及线程同步问题。所以synchronized代码块比synchronized方法更加细粒度地控制了多个线程的同步访问。
>
> synchronized方法和synchronized代码块有什么联系呢？在Java的内部实现上，synchronized方法实际上等同于用一个synchronized代码块，这个代码块包含同步方法中的所有语句，然后在synchronized代码块的括号中传入this关键字，使用this对象锁作为进入临界区的同步锁。

### 2.2.3 静态方法的同步

在Java世界里一切皆对象。**Java有两种对象：Object实例对象和Class对象。**每个类运行时的类型信息用Class对象表示，它包含与类名称、继承关系、字段、方法有关的信息。JVM将一个类加载入自己的方法区内存时，会为其创建一个Class对象，对于一个类来说其Class对象是唯一的。Class类没有公共的构造方法，Class对象是在类加载的时候由Java虚拟机调用类加载器中的defineClass方法自动构造的，因此不能显式地声明一个Class对象。

静态方法属于Class实例而不是单个Object实例，在静态方法内部是不可以访问Object实例的this引用（也叫指针、句柄）的。所以，修饰static方法的synchronized关键字就没有办法获得Object实例的this对象的监视锁。那如果某个synchronized方法是static（静态）方法，而不是普通的对象实例方法，其同步锁又是什么呢？

实际上使用synchronized关键字修饰static方法时，synchronized的同步锁并不是普通Object对象的监视锁，而是类所对应的Class对象的监视锁。为了区分，这里将Object对象的监视锁叫作对象锁，将Class对象的监视锁叫作类锁。当synchronized关键字修饰static方法时，同步锁为类锁；当synchronized关键字修饰普通的成员方法（非静态方法）时，同步锁为对象锁。

类锁是加载类上的，而类信息是存在 JVM 方法区的，并且整个 JVM 只有一份，方法区又是所有线程共享的，所以类锁是所有线程共享的，因此使用类锁作为synchronized的同步锁时会造成同一个JVM内的所有线程只能互斥地进入临界区段。所以使用使用synchronized关键字修饰static方法是非常粗粒度的同步机制。

## 2.3 生产者-消费者问题

生产者-消费者问题（Producer-Consumer Problem）也称有限缓冲问题（Bounded-BufferProblem），是一个多线程同步问题的经典案例。

在生产者-消费者模式中，通常有两类线程，即生产者线程（若干个）和消费者线程（若干个）。生产者线程向临界资源中加入数据，消费者线程则从数据缓冲区消耗数据。

在生产者-消费者模式中，需要注意以下几点

- 生产者与生产者之间、消费者与消费者之间，对数据缓冲区的操作是并发进行的。
- 数据缓冲区是有容量上限的。数据缓冲区满后，生产者不能再加入数据；数据缓冲区空时，消费者不能再取出数据。
- 数据缓冲区是线程安全的。在并发操作数据缓冲区的过程中，不能出现数据不一致的情况；或者在多个线程并发更改共享数据后，不会造成出现脏数据的情况。
- 生产者或者消费者线程在空闲时需要尽可能阻塞而不是执行无效的空操作，尽量节约CPU资源。

我们下面就以这个经典问题给出代码案例，更深刻的了解线程安全问题。

### 2.3.1 线程不安全示例

我们先定义临界资源：

```java
public class DataNoSafe<T> {
    public static final int MAX_COUNT = 10;
    private List<T> dataList = new LinkedList<>();

    /**
     * 保存数量
     */
    private AtomicInteger count = new AtomicInteger(0);

    /**
     * 增加元素
     */
    public void add(T element) throws Exception{
        if (count.get()>MAX_COUNT){
            System.out.println("队列已经满了");
            return;
        }
        dataList.add(element);
        System.out.println(element);
        count.incrementAndGet();
        if (count.get()!=dataList.size()){
            throw new RuntimeException("count和dataList队列数量不一致，count："+count+"队列数量"+dataList.size());
        }
    }
    public T get() throws Exception{
        if (count.get()<=0){
            System.out.println("队列已经空了");
            return null;
        }
        T element = dataList.remove(0);
        System.out.println(element);
        count.decrementAndGet();
        if (count.get()!=dataList.size()){
            throw new RuntimeException("count和dataList队列数量不一致，count："+count+"队列数量"+dataList.size());
        }
        return element;
    }
}
```

然后定义生产者和消费者：

```java
public class Producer implements Runnable{
    /**
     * 生产一次的等待时间默认200ms
     */
    public static final int PRODUCE_WAIT =200;
    /**
     * 生产总次数
     */
    static final AtomicInteger TOTAL_NUM = new AtomicInteger(0);
    /**
     * 生产者编号
     */
    static final AtomicInteger PNO = new AtomicInteger(1);
    /**
     * 生产者名称
     */
    String name =null;
    /**
     * 生产的动作
     */
    Callable action =null;
    int wait = PRODUCE_WAIT;
    public Producer(Callable action, int wait) {
        this.action = action;
        this.wait = wait;
        this.name = "生产者-"+ PNO.incrementAndGet();
    }
    @Override
    public void run() {
        while (true){
            try {
                Object product = action.call();
                if (product!=null){
                    System.out.println("第"+TOTAL_NUM.get()+"轮生产 "+product);
                }
                Thread.sleep(wait);
                TOTAL_NUM.incrementAndGet();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

public class Consumer implements Runnable{
    /**
     * 消费一次的等待时间默认200ms
     */
    public static final int CONSUME_WAIT =100;
    /**
     * 消费总次数
     */
    static final AtomicInteger TOTAL_NUM = new AtomicInteger(0);
    /**
     * 消费者编号
     */
    static final AtomicInteger CNO = new AtomicInteger(1);
    /**
     * 消费者名称
     */
    String name =null;
    /**
     * 消费的动作
     */
    Callable action =null;

    int wait = CONSUME_WAIT;

    public Consumer(Callable action, int wait) {
        this.action = action;
        this.wait = wait;
        this.name = "生产者-"+ CNO.incrementAndGet();
    }
    @Override
    public void run() {
        while (true){
            TOTAL_NUM.incrementAndGet();
            try {
                Object product = action.call();
                if (product!=null){
                    System.out.println("第"+TOTAL_NUM.get()+"轮消费 "+product);
                }
                Thread.sleep(wait);

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

然后定义测试类：

```java
public class ActionAndData {
    private static DataNoSafe<Goods> data = new DataNoSafe<>();

    static Callable<Goods> producerAction = () -> {
        Goods one = Goods.getOne();
        data.add(one);
        return one;
    };

    static Callable<Goods> consumerAction = () -> {
        Goods goods = data.get();
        return goods;
    };

    public static void main(String[] args) {
        final int THREAD_NUM = 20;
        ExecutorService service = Executors.newFixedThreadPool(THREAD_NUM);
        for (int i = 0; i < 5; i++) {
            service.submit(new Producer(producerAction,500));
            service.submit(new Consumer(consumerAction,1500));
        }
    }
}
//商品类代码：
@Data
public class Goods {

    private static AtomicInteger no = new AtomicInteger(0);
    private Integer ID;
    private String name;
    private Integer price;


    public static Goods getOne(){
        Goods goods = new Goods();
        goods.setID(no.get());
        goods.setName("商品"+no);
        Random random = new Random(100);
        goods.setPrice(random.nextInt());
        no.incrementAndGet();
        return goods;
    }
}
```

运行结果截图：

![image-20220420112455418](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220420112455418.png)

从以上异常可以看出，在向数据缓冲区进行元素的增加或者提取时，多个线程在并发执行对count、dataList两个成员操作时次序已经混乱，导致出现数据不一致和线程安全问题。

### 2.3.2 线程安全改造

解决线程安全问题很简单，为临界区代码加上synchronized关键字即可，主要修改的是涉及操作两个临界区资源count和dataList的代码，具体为add和get方法。

我们为了对比，将DataNoSafe类复制新建一个DataSafe类，然后进行改造代码如下：

```java
public class DataSafe<T> {
   //其余代码略
    public synchronized void add(T element) throws Exception{
        //代码略
    }
    public synchronized T get() throws Exception{
        //代码略
    }
}
```

我们再次进行测试（注意测试类的data也要改成DataSafe），测试结果：

![image-20220420124324416](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220420124324416.png)

这时就不会再出现上面的报错了。

## 2.4 Java对象结构

Java内置锁的很多重要信息都存放在对象结构中。作为铺垫，我们先了解一下Java的对象结构

### 2.4.1 Java对象

Java对象（Object实例）结构包括三部分：对象头、对象体和对齐字节，如下图：

<img src="https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/javaobjectstruct20220420.png" alt="javaobjectstruct20220420" style="zoom:50%;" />

Java对象的三部分：

1. 对象头

   对象头包含三个字段，分别是Mark Word、Class Pointer、Array Length。

   `Mark Word`(标记字)用于存储自身运行时的数据，例如GC标志位、哈希码、锁状态等信息。此字段主要用来表示对象的线程锁状态，还可以用来配合GC存放该对象的hashCode。

   `Class Pointer`(类对象指针)用于存放方法区Class对象的地址，虚拟机通过这个指针来确定这个对象是哪个类的实例。

   `Array Length`(数组长度)这是一个可选字段，如果对象是一个Java数组，此字段必须有，用于记录数组长度，如果不是则此字段不存在。此字段占用32位（在32位JVM）字节，这是可选的，只有本对象是一个数组对象时才会有这个部分。

2. 对象体

   对象体包含对象的实例变量，用于成员属性值，包括父类的成员属性值，这部分内存按四字节对齐，占用内存大小取决于对象的属性数量和类型。

3. 对齐字节

   这部分的作用是用来保证Java对象所占内存字节数为8的倍数。HotSpot Vm的内存管理要求对象起始地址必须是8字节的整数倍，对象头本身是8的倍数，如果对象实例变量不是8的倍数时，则需要填充数据保证8字节的对齐。

Mark Word、Class Pointer、Array Length等字段的长度都与JVM的位数有关。在32位JVM虚拟机中，三者都是32位的；在64位JVM虚拟机中，三者都是64位的。但是如果64位JVM开启指针压缩Array Length字段的长度也将由64位压缩至32位。

对于对象指针而言，如果JVM中的对象数量过多，使用64位的指针将浪费大量内存，通过简单统计，64位JVM将会比32位JVM多耗费50%的内存。为了节约内存可以使用选项`+UseCompressedOops`开启指针压缩。UseCompressedOops中的Oop为Ordinary object pointer（普通对象指针）的缩写。

如果开启UseCompressedOops选项，以下类型的指针将从64位压缩至32位：

- Class对象的属性指针（静态变量）。
- Object对象的属性指针（成员变量）。
- 普通对象数组的元素指针。

当然，也不是所有的指针都会压缩，一些特殊类型的指针不会压缩，比如指向PermGen（永久代）的Class对象指针（JDK 8中指向元空间的Class对象指针）、本地变量、堆栈元素、入参、返回值和NULL指针等。



在堆内存小于32GB的情况下，64位虚拟机的UseCompressedOops选项是默认开启的，手动开启指令：`java -XX:+UseCompressedOops mainclass`手动关闭指令：`java -XX:-UseCompressedOops mainclass`

### 2.4.2 Mark Word字段

Java内置锁涉及很多重要信息，这些都存放在对象结构中，并且存放于对象头的Mark Word字段中。

Java内置锁的状态总共有4种，级别由低到高依次为：无锁、偏向锁、轻量级锁和重量级锁。在JDK 1.6之前，Java内置锁是一个重量级锁，在JDK 1.6之后，JVM为了提高锁的获取与释放效率，对synchronized的实现进行了优化，引入了偏向锁和轻量级锁，从此Java内置锁的状态就有了4种（无锁、偏向锁、轻量级锁和重量级锁）。并且内置锁会进行锁升级，即锁的状态不断变化，但是锁不能降级。

Mark Word字段的结构与Java内置锁的状态强相关。为了让Mark Word字段存储更多的信息，JVM将Mark Word最低两个位设置为Java内置锁状态位，不同锁状态下的32位Mark Word结构如下：

<table>
    <thead>
        <tr>
	    <th rowspan="3">内置锁状态</th>
	    <th colspan="2" >25bit</th>
	    <th rowspan="3">4bit</th> 
        <th>1bit</th>
        <th>2bit</th>
	</tr>
	<tr>
	    <th rowspan="2">23bit</th>
        <th rowspan="2">2bit</th>
        <th rowspan="2">biased偏向标志位</th>
        <th rowspan="2">lock锁状态</th>
	</tr>
    </thead>
	<tbody>
    	<tr>
	    <td>无锁</td>
        <td colspan="2" >对象的hashCode(25bit)</td>
        <td >分代年龄</td>
        <td >0</td>
        <td >01</td>
	</tr>
    <tr>
	    <td>偏向锁</td>
        <td>线程ID(23bit)</td>
        <td>epoch(2bit)</td>
        <td >分代年龄</td>
        <td >1</td>
        <td >01</td>
	</tr>
    <tr>
	    <td>轻量级锁</td>
        <td colspan="4" >ptr_to_lock_record方法栈帧中的锁记录指针(30bit)</td>
        <td >00</td>
	</tr>
    <tr>
	    <td>重量级锁</td>
        <td colspan="4" >ptr_to_heavyweight_monitor指向重量级锁监视器的指针(30bit)</td>
        <td >10</td>
	</tr>
    <tr>
	    <td>GC标志</td>
        <td colspan="4" >空(30bit)</td>
        <td >11</td>
	</tr>
    </tbody>
</table>

64位Mark Word结构如下：

<table>
    <thead>
        <tr>
	    <th rowspan="2">内置锁状态</th>
	    <th rowspan="2" colspan="3" >57bit</th>
	    <th rowspan="2">4bit</th> 
        <th>1bit</th>
        <th>2bit</th>
	</tr>
	<tr>
        <th>biased</th>
        <th>lock</th>
	</tr>
    </thead>
	<tbody>
    	<tr>
	    <td>无锁</td>
        <td >unused(25bit)</td>
        <td >对象的hashCode(31bit)</td>
        <td >unused(1bit)</td>
        <td >分代年龄</td>
        <td >0</td>
        <td >01</td>
	</tr>
    <tr>
	    <td>偏向锁</td>
        <td>线程ID(54bit)</td>
        <td>epoch(2bit)</td>
        <td >unused(1bit)</td>
        <td >分代年龄</td>
        <td >1</td>
        <td >01</td>
	</tr>
    <tr>
	    <td>轻量级锁</td>
        <td colspan="5" >ptr_to_lock_record方法栈帧中的锁记录指针(62bit)</td>
        <td >00</td>
	</tr>
    <tr>
	    <td>重量级锁</td>
        <td colspan="5" >ptr_to_heavyweight_monitor指向重量级锁监视器的指针(62bit)</td>
        <td >10</td>
	</tr>
    <tr>
	    <td>GC标志</td>
        <td colspan="5" >空(62bit)</td>
        <td >11</td>
	</tr>
    </tbody>
</table>

在64位的Mark Word中

- lock：锁状态标记位，占两个二进制位，由于希望用尽可能少的二进制位表示尽可能多的信息，因此设置了lock标记。该标记的值不同，整个Mark Word表示的含义就不同。
- biased：表示当前对象无锁还是偏向锁
- age：分代年龄，在GC中，对象在Survivor区复制一次，年龄就增加1，当对象达到设定的阈值就会晋升老年代，默认并行GC的年龄阈值为15，并发GC的年龄阈值是6.因为只有4为所以最大是15.
- identity_hashcode:对象的hashCode，31位对象标识哈希码采用延迟加载，只有调用`Object.hashCode()`方法或者`System.identityHashCode()`方法计算对象的HashCode后，其结果将被写到该对象头中。当对象被锁定时，该值会移动到Monitor（监视器）中.
- epoch:偏向时间戳
- thread：线程ID，54位是持有偏向锁的线程ID
- ptr_to_lock_record，轻量级锁状态下指向栈帧中锁记录的指针。
- ptr_to_heavyweight_monitor，重量级锁状态下指向对象监视器的指针。

### 2.4.3 JOL工具

如何通过Java程序查看Object对象头的结构呢？OpenJDK提供的JOL（Java Object Layout）包是一个非常好的工具，可以帮我们在运行时计算某个对象的大小。

JOL工具快速入门：首先引入Maven的依赖坐标：

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.16</version>
</dependency>
<!--如果看VALUE中具体的值，可以使用0.11版本。在0.16版本中直接就能看到具体的结果-->
```

首先我们可以查看一下Class类对象的布局：

```java
public class JolObject {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(Class.class).toPrintable());
    }
}
```

执行结果：

~~~
java.lang.Class object internals:
OFF  SZ                                              TYPE DESCRIPTION                    VALUE
  0   8                                                   (object header: mark)          N/A
  8   4                                                   (object header: class)         N/A
 12   4                     java.lang.reflect.Constructor Class.cachedConstructor        N/A
 16   4                                   java.lang.Class Class.newInstanceCallerCache   N/A
 20   4                                  java.lang.String Class.name                     N/A
 24   4                                                   (alignment/padding gap)        
 28   4                       java.lang.ref.SoftReference Class.reflectionData           N/A
 32   4   sun.reflect.generics.repository.ClassRepository Class.genericInfo              N/A
 36   4                                java.lang.Object[] Class.enumConstants            N/A
 40   4                                     java.util.Map Class.enumConstantDirectory    N/A
 44   4                    java.lang.Class.AnnotationData Class.annotationData           N/A
 48   4             sun.reflect.annotation.AnnotationType Class.annotationType           N/A
 52   4                java.lang.ClassValue.ClassValueMap Class.classValueMap            N/A
 56  32                                                   (alignment/padding gap)        
 88   4                                               int Class.classRedefinedCount      N/A
 92   4                                                   (object alignment gap)         
Instance size: 96 bytes
Space losses: 36 bytes internal + 4 bytes external = 40 bytes total
~~~

我们也可以查看Java对象的布局：

~~~java
public class JolObject {
    public static void main(String[] args) {
        final JolObject a = new JolObject();
        ClassLayout layout = ClassLayout.parseInstance(a);
        System.out.println("**** 新对象");
        System.out.println(layout.toPrintable());
    }
}
~~~

执行结果

~~~
**** 新对象
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
~~~

从结果中，我们可以看到Java的对象结构，注意此时锁状态是non-biasable表示是无锁不可偏向状态。

当然我们主要是用来分析Java对象的锁状态的，所以我们编写以下代码：

```java
public class JolObject {
    public static void main(String[] args) {
        final JolObject a = new JolObject();
        ClassLayout layout = ClassLayout.parseInstance(a);
        System.out.println("**** 新对象");
        System.out.println(layout.toPrintable());
        synchronized (a) {
            System.out.println("**** 持有对象锁");
            System.out.println(layout.toPrintable());
        }
        System.out.println("**** 释放锁");
        System.out.println(layout.toPrintable());
    }
}
```

增加运行参数然后执行，执行结果：

~~~
**** 新对象
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 持有对象锁
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000001ad69e6d805 (biased: 0x000000006b5a79b6; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 释放锁
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000001ad69e6d805 (biased: 0x000000006b5a79b6; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
~~~

我们可以很清晰的看到，当对象获取锁之后，mark word由biasable（此状态表示无锁可偏向状态，无锁是指没有存线程的ID）变成了biased（偏向锁）

> JDK 9之前偏向锁在JVM启动5秒后才可以使用。因此，在JDK 8及以下版本中运行时需要增加JVM运行参数 -XX:BiasedLockingStartupDelay=0。 JDK 15 之后偏向锁不再是默认的锁，所以在JDK 15 中运行需要增加JVM运行参数： -XX:+UseBiasedLocking。
>
> 我们这里是使用的JDK8因此当设置-XX:BiasedLockingStartupDelay=0之后对象不用等5s直接就可以可偏向状态即biasable状态，但此时是无锁的。其实这是一种匿名偏向状态，是对象初始化中，JVM 帮我们做的，这样当有线程进入同步块时如果是可偏向状态：直接就 CAS 替换 ThreadID，如果成功，就可以获取偏向锁了。但如果是不可偏向状态：就会变成轻量级锁。

我们将JVM运行参数去掉再次执行，运行结果：

~~~
**** 新对象
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 持有对象锁
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00000055dabff2f0 (thin lock: 0x00000055dabff2f0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 释放锁
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
~~~

可以看到这时由non-biasable进入了thin lock状态（轻量级锁）

我们在增加一个线程获得锁，代码如下：

```java
public class JolObject {
    public static void main(String[] args) throws InterruptedException {
        final JolObject a = new JolObject();
        ClassLayout layout = ClassLayout.parseInstance(a);
        System.out.println("**** 新对象");
        System.out.println(layout.toPrintable());
        Thread t = new Thread(() -> {
            synchronized (a) {
                try {
                    TimeUnit.SECONDS.sleep(10);
                } catch (InterruptedException e) {
                    return;
                }
            }
        });
        t.start();
        TimeUnit.SECONDS.sleep(1);
        System.out.println("**** 锁之前");
        System.out.println(layout.toPrintable());
        synchronized (a) {
            System.out.println("**** 持有锁");
            System.out.println(layout.toPrintable());
        }
        System.out.println("**** 锁之后");
        System.out.println(layout.toPrintable());
        System.gc();
        System.out.println("**** System.gc()后");
        System.out.println(layout.toPrintable());
    }
}
```

结果：

~~~
**** 新对象
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 锁之前
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00000001fb0ff0f8 (thin lock: 0x00000001fb0ff0f8)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 持有锁
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000001621ad5b4ea (fat lock: 0x000001621ad5b4ea)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** 锁之后
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x000001621ad5b4ea (fat lock: 0x000001621ad5b4ea)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** System.gc()后
com.learn.testnginx.synchronizedLearn.JolObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000009 (non-biasable; age: 1)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
~~~

可以看到，主线程获得锁之前是轻量级锁（因为没开偏向锁同时还有一个线程持有锁），在主线程也去获取锁之后，锁升级成重量级锁fat lock，且在主线程释放后还会一直持有，直到gc后恢复无锁状态。

### 2.4.4 锁的四个状态

JDK 1.6版本之前，所有的Java内置锁都是重量级锁。重量级锁会造成CPU在用户态和核心态之间频繁切换，所以代价高、效率低。JDK 1.6版本为了减少获得锁和释放锁所带来的性能消耗，引入了偏向锁和轻量级锁的实现。所以，在JDK 1.6版本中内置锁一共有4种状态：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这些状态随着竞争情况逐渐升级。内置锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能再降级成偏向锁。这种能升级却不能降级的策略，其目的是提高获得锁和释放锁的效率。

在上面JOL实例中我们也见识过个这几个锁的状态，下面我们来详细了解一下。

- Java对象刚创建时还没有任何线程来竞争，说明该对象处于无锁状态（无线程竞争它），这时偏向锁标识位是0，锁状态是01。

- 偏向锁是指一段同步代码一直被同一个线程所访问，那么该线程会自动获取锁，降低获取锁的代价。如果内置锁处于偏向状态，当有一个线程来竞争锁时，先用偏向锁，表示内置锁偏爱这个线程，这个线程要执行该锁关联的同步代码时，不需要再做任何检查和切换。偏向锁在竞争不激烈的情况下效率非常高。偏向锁状态的Mark Word会记录内置锁自己偏爱的线程ID，内置锁会将该线程当作自己的熟人。

- 当有两个线程开始竞争这个锁对象时，情况就发生变化了，不再是偏向（独占）锁了，锁会升级为轻量级锁，两个线程公平竞争，哪个线程先占有锁对象，锁对象的Mark Word就指向哪个线程的栈帧中的锁记录。当锁处于偏向锁，又被另一个线程企图抢占时，偏向锁就会升级为轻量级锁。企图抢占的线程会通过自旋的形式尝试获取锁，不会阻塞抢锁线程，以便提高性能。

  > 线程自旋是需要消耗CPU的，如果一直获取不到锁，那么线程也不能一直占用CPU自旋做无用功，所以需要设定一个自旋等待的最大时间。JVM对于自旋周期的选择，JDK 1.6之后引入了适应性自旋锁，适应性自旋锁意味着自旋的时间不是固定的，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定的。线程如果自旋成功了，下次自旋的次数就会更多，如果自旋失败了，自旋的次数就会减少。如果持有锁的线程执行的时间超过自旋等待的最大时间仍没有释放锁，就会导致其他争用锁的线程在最大等待时间内还是获取不到锁，自旋不会一直持续下去，这时争用线程会停止自旋进入阻塞状态，该锁膨胀为重量级锁。

- 重量级锁会让其他申请的线程之间进入阻塞，性能降低。重量级锁也叫同步锁，这个锁对象MarkWord再次发生变化，会指向一个监视器对象，该监视器对象用集合的形式来登记和管理排队的线程。

## 2.5 偏向锁

### 2.5.1偏向锁原理

在实际场景中，如果一个同步块（或方法）没有多个线程竞争，而且总是由同一个线程多次重入获取锁，如果每次还有阻塞线程，唤醒CPU从用户态转为核心态，那么对于CPU是一种资源的浪费，为了解决这类问题，就引入了偏向锁的概念。

偏向锁的核心原理是：如果不存在线程竞争的一个线程获得了锁，那么锁就进入偏向状态，此时Mark Word的结构变为偏向锁结构，锁对象的锁标志位（lock）被改为01，偏向标志位（biased_lock）被改为1，然后线程的ID记录在锁对象的Mark Word中（使用CAS操作完成）。以后该线程获取锁时判断一下线程ID和标志位，就可以直接进入同步块，连CAS操作都不需要，这样就省去了大量有关锁申请的操作，从而也就提升了程序的性能。

偏向锁的主要作用是消除无竞争情况下的同步原语，进一步提升程序性能，所以，在没有锁竞争的场合，偏向锁有很好的优化效果。但是，一旦有第二条线程需要竞争锁，那么偏向模式立即结束，进入轻量级锁的状态。但是如果锁对象时常被多个线程竞争，偏向锁就是多余的，并且其撤销的过程会带来一些性能开销。

### 2.5.2 偏向锁示例

在上面JOL工具中我们呢简单演示了偏向锁的示例，这里我们在详细的进行分析：

```java
public class JolObject {
    private Integer count =0;
    public void increase(){
        count++;
    }
    public static void main(String[] args) throws InterruptedException {
        showBiasedLock();
    }
    public static void showBiasedLock() throws InterruptedException {
        //如果用这个test对象去锁定的话会直接升级成轻量级锁，因为当前是无锁状态
        JolObject test = new JolObject();
        System.out.println("获取新创建test对象状态: "+ClassLayout.parseInstance(test).toPrintable());
        //等待JVM的偏向锁延迟
        Thread.sleep(5000);
        //用这个对象才能升级成偏向锁因为当前是匿名偏向状态
        JolObject jolObject = new JolObject();
        System.out.println("获取5s后创建jolObject对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
        Thread.sleep(5000);
        CountDownLatch latch = new CountDownLatch(1);
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                synchronized (jolObject){
                    jolObject.increase();
                    if(i==5){
                        System.out.println("jolObject占有锁的对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
                    }
                }
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            latch.countDown();
        }).start();
        latch.await();
        Thread.sleep(5000);
        System.out.println("jolObject释放锁的对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
    }
}
```

我们通过JOL工具的0.16和0.11两个版本运行以上代码，输出结果如下：

![image-20220421131334139](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220421131334139.png)

从输出结果的最后也可以看出：虽然抢锁的线程已经结束，但是JolObject实例的对象结构仍然记录了其之前的偏向线程ID，其锁状态还是偏向锁状态101。所以偏向锁是针对一个线程而言的，线程获得锁之后就不会再有解锁等操作了，这样可以节省很多开销。

### 2.5.3 偏向锁的撤销和升级

假如有多个线程来竞争偏向锁，此对象锁已经有所偏向，其他的线程发现偏向锁并不是偏向自己，就说明存在了竞争，尝试撤销偏向锁（很可能引入安全点），然后膨胀到轻量级锁。

偏向锁撤销的开销花费还是挺大的，其大概过程如下：

1. 在一个安全点停止拥有锁的线程。
2. 遍历线程的栈帧，检查是否存在锁记录。如果存在锁记录，就需要清空锁记录，使其变成无锁状态，并修复锁记录指向的Mark Word，清除其线程ID。
3. 将当前锁升级成轻量级锁。
4. 唤醒当前线程。

所以，如果某些临界区存在两个及两个以上的线程竞争，那么偏向锁反而会降低性能。在这种情况下，可以在启动JVM时就把偏向锁的默认功能关闭。撤销偏向锁的条件：一个是多个线程竞争偏向锁。另一个是调用偏向锁对象的hashcode()方法或者System.identityHashCode()方法计算对象的HashCode之后，将哈希码放置到Mark Word中，内置锁变成无锁状态，偏向锁将被撤销。

> 因为重新计算锁对象的hash值会将哈希码放置到Mark Word中，内置锁变成无锁状态，偏向锁将被撤销，所以计算hashcode会导致偏向锁失效。

**偏向锁的升级**

如果偏向锁被占据，一旦有第二个线程争抢这个对象，因为偏向锁不会主动释放，所以第二个线程可以看到内置锁偏向状态，这时表明在这个对象锁上已经存在竞争了。JVM检查原来持有该对象锁的占有线程是否依然存活，如果挂了，就可以将对象变为无锁状态，然后进行重新偏向，偏向为抢锁线程。如果JVM检查到原来的线程依然存活，就进一步检查占有线程的调用堆栈是否通过锁记录持有偏向锁。如果存在锁记录，就表明原来的线程还在使用偏向锁，发生锁竞争，撤销原来的偏向锁，将偏向锁膨胀（INFLATING）为轻量级锁。

## 2.6 轻量级锁

### 2.6.1 轻量级锁原理

引入轻量级锁的主要目的是在多线程竞争不激烈的情况下，通过CAS机制竞争锁减少重量级锁产生的性能损耗。重量级锁使用了操作系统底层的互斥锁（Mutex Lock），会导致线程在用户态和核心态之间频繁切换，从而带来较大的性能损耗。

轻量锁存在的目的是尽可能不动用操作系统层面的互斥锁，因为其性能比较差。线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁地阻塞和唤醒对CPU来说是一件负担很重的工作。同时我们可以发现，很多对象锁的锁定状态只会持续很短的一段时间，例如整数的自加操作，在很短的时间内阻塞并唤醒线程显然不值得，为此引入了轻量级锁。轻量级锁是一种自旋锁，是一种应用层面的锁。



**轻量级锁的执行过程**：在抢锁线程进入临界区之前，如果内置锁（临界区的同步对象）没有被锁定，JVM首先将在抢锁线程的栈帧中建立一个锁记录（Lock Record），用于存储对象目前Mark Word的拷贝，这时的线程堆栈与内置锁对象头如下图：

![thinlock120220421](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/thinlock120220421.png)

然后抢锁线程将使用CAS自旋操作，尝试将内置锁对象头的Mark Word的ptr_to_lock_record（锁记录指针）更新为抢锁线程栈帧中锁记录的地址，如果这个更新执行成功了，这个线程就拥有了这个对象锁。然后JVM将Mark Word中的lock标记位改为00（轻量级锁标志），即表示该对象处于轻量级锁状态。抢锁成功之后，JVM会将Mark Word中原来的锁对象信息（如哈希码等）保存在抢锁线程锁记录的Displaced Mark Word字段中，再将抢锁线程中锁记录的owner指针指向锁对象，如下图

![thinlock220220421](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/thinlock220220421.png)

> 在创建完锁记录后，会将内置锁对象的Mark Word复制到锁记录的Displaced Mark Word字段。这是为什么呢？因为内置锁对象的MarkWord的结构会有所变化，Mark Word将会出现一个指向锁记录的指针，而不再存着无锁状态下的锁对象哈希码等信息，所以必须将这些信息暂存起来，供后面在锁释放时使用。

### 2.6.2 轻量级锁示例

本示例仅展示关键函数将此函数放到主函数中即可，其余所有代码均与上一个示例相同：

```java
public static void showThinLock() throws InterruptedException {
    Thread.sleep(5000);
    JolObject jolObject = new JolObject();
    System.out.println("抢占锁前的对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
    Thread.sleep(5000);
    CountDownLatch latch = new CountDownLatch(2);
    new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            synchronized (jolObject){
                jolObject.increase();
                if (i==1){
                    System.out.println("第一个线程占有锁，对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
                }
            }
        }
        latch.countDown();
        //让线程释放锁后死循环
        while (true){
            try {
                Thread.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }).start();
    new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            synchronized (jolObject){
                jolObject.increase();
                if (i==5){
                    System.out.println("第二个线程占有锁，对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
                }
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        latch.countDown();
    }).start();
    latch.await();
    Thread.sleep(2000);
    System.out.println("释放锁后对象状态: "+ClassLayout.parseInstance(jolObject).toPrintable());
}
```

运行结果（这回我们只用JOL0.16输出）：

![image-20220421140850497](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220421140850497.png)

### 2.6.3 轻量级锁的分类和升级

轻量级锁主要有两种：普通自旋锁和自适应自旋锁。

普通自旋锁，就是指当有线程来竞争锁时，抢锁线程会在原地循环等待，而不是被阻塞，直到那个占有锁的线程释放锁之后，这个抢锁线程才可以获得锁。说明锁在原地循环等待的时候是会消耗CPU的，就相当于在执行一个什么也不干的空循环。所以轻量级锁适用于临界区代码耗时很短的场景，这样线程在原地等待很短的时间就能够获得锁了。默认情况下，自旋的次数为10次，用户可以通过-XX:PreBlockSpin选项来进行更改。

**自适应自旋锁**，就是等待线程空循环的自旋次数并非是固定的，而是会动态地根据实际情况来改变自旋等待的次数，自旋次数由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。自适应自旋锁的大概原理是：如果抢锁线程在同一个锁对象上之前成功获得过锁，JVM就会认为这次自旋很有可能再次成功，因此允许自旋等待持续相对更长的时间。如果对于某个锁，抢锁线程很少成功获得过，那么JVM将可能减少自旋时间甚至省略自旋过程，以避免浪费处理器资源。

> JDK 1.7后，轻量级锁使用自适应自旋锁，JVM启动时自动开启，且自旋时间由JVM自动控制。

虽然大部分临界区代码的执行时间都是很短的，但是也会存在执行得很慢的临界区代码。临界区代码执行耗时较长，在其执行期间，其他线程都在原地自旋等待，会空消耗CPU。因此，如果竞争这个同步锁的线程很多，就会有多个线程在原地等待继续空循环消耗CPU（空自旋），这会带来很大的性能损耗。因此在争用激烈的场景下，轻量级锁会膨胀为基于操作系统内核互斥锁实现的重量级锁。

## 2.7 重量级锁

### 2.7.1 重量级锁原理

JVM中每个对象都会有一个监视器，监视器和对象一起创建、销毁。监视器相当于一个用来监视这些线程进入的特殊房间，其义务是保证（同一时间）只有一个线程可以访问被保护的临界区代码块。本质上，监视器是一种同步工具，也可以说是一种同步机制，主要特点是：

- 同步。监视器所保护的临界区代码是互斥地执行的。一个监视器是一个运行许可，任一线程进入临界区代码都需要获得这个许可，离开时把许可归还。
- 协作。监视器提供Signal机制，允许正持有许可的线程暂时放弃许可进入阻塞等待状态，等待其他线程发送Signal去唤醒；其他拥有许可的线程可以发送Signal，唤醒正在阻塞等待的线程，让它可以重新获得许可并启动执行。

在Hotspot虚拟机中，监视器是由C++类ObjectMonitor实现的，ObjectMonitor类定义在ObjectMonitor.hpp文件中，其构造器代码大致如下，（下图直接来自原书Java高并发编程卷二）：

![image-20220421142649193](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220421142649193.png)

`ObjectMonitor`的`Owner（_owner）`、`WaitSet（_WaitSet）`、`Cxq（_cxq）`、`EntryList（_EntryList）`这几个属性比较关键。

ObjectMonitor的WaitSet、Cxq、EntryList这三个队列存放抢夺重量级锁的线程，而ObjectMonitor的Owner所指向的线程即为获得锁的线程。

Cxq、EntryList、WaitSet这三个队列的说明如下：

- Cxq：竞争队列（Contention Queue），所有请求锁的线程首先被放在这个竞争队列中。

  Cxq并不是一个真正的队列，只是一个虚拟队列，原因在于Cxq是由Node及其next指针逻辑构成的，并不存在一个队列的数据结构。每次新加入Node会在Cxq的队头进行，通过CAS改变第一个节点的指针为新增节点，同时设置新增节点的next指向后续节点；从Cxq取得元素时，会从队尾获取。显然，Cxq结构是一个无锁结构。因为只有Owner线程才能从队尾取元素，即线程出列操作无争用，当然也就避免了CAS的ABA问题。

  > 在线程进入Cxq前，抢锁线程会先尝试通过CAS自旋获取锁，如果获取不到，就进入Cxq队列，这明显对于已经进入Cxq队列的线程是不公平的。所以，synchronized同步块所使用的重量级锁是不公平锁。

- EntryList：Cxq中那些有资格成为候选资源的线程被移动到EntryList中。

  EntryList与Cxq在逻辑上都属于等待队列。Cxq会被线程并发访问，为了降低对Cxq队尾的争用，而建立EntryList。在Owner线程释放锁时，JVM会从Cxq中迁移线程到EntryList，并会指定EntryList中的某个线程（一般为Head）为OnDeck Thread（Ready Thread）。EntryList中的线程作为候选竞争线程而存在。

  JVM不直接把锁传递给Owner Thread，而是把锁竞争的权利交给OnDeck Thread，OnDeck需要重新竞争锁。这样虽然牺牲了一些公平性，但是能极大地提升系统的吞吐量，在JVM中，也把这种选择行为称为“竞争切换”。OnDeck Thread获取到锁资源后会变为Owner Thread。无法获得锁的OnDeck Thread则会依然留在EntryList中，考虑到公平性，OnDeck Thread在EntryList中的位置不发生变化（依然在队头）。在OnDeck Thread成为Owner的过程中，还有一个不公平的事情，就是后来的新抢锁线程可能直接通过CAS自旋成为Owner而抢到锁。

- WaitSet：某个拥有ObjectMonitor的线程在调用Object.wait()方法之后将被阻塞，然后该线程将被放置在WaitSet链表中。

  如果Owner线程被Object.wait()方法阻塞，就转移到WaitSet队列中，直到某个时刻通过Object.notify()或者Object.notifyAll()唤醒，该线程就会重新进入EntryList中。

ObjectMonitor的内部抢锁过程如图：

![heightLock20220421](C:\Users\yhr\Pictures\Saved Pictures\heightLock20220421.png)

由于JVM轻量级锁使用CAS进行自旋抢锁，这些CAS操作都处于用户态下，进程不存在用户态和内核态之间的运行切换，因此JVM轻量级锁开销较小。而JVM重量级锁使用了Linux内核态下的互斥锁，这是重量级锁开销很大的原因。

> Linux系统的体系架构分为用户态（或者用户空间）和内核态（或者内核空间）。
>
> Linux系统的内核是一组特殊的软件程序，负责控制计算机的硬件资源，例如协调CPU资源、分配内存资源，并且提供稳定的环境供应用程序运行。应用程序的活动空间为用户空间，应用程序的执行必须依托于内核提供的资源，包括CPU资源、存储资源、I/O资源等。用户态与内核态有各自专用的内存空间、专用的寄存器等，进程从用户态切换至内核态需要传递许多变量、参数给内核，内核也需要保护好用户态在切换时的一些寄存器值、变量等，以便内核态调用结束后切换回用户态继续工作。
>
> 用户态的进程能够访问的资源受到了极大的控制，而运行在内核态的进程可以“为所欲为”。一个进程可以运行在用户态，也可以运行在内核态，那么肯定存在用户态和内核态切换的过程。进程从用户态到内核态切换主要包括以下三种方式：
>
> 1. 硬件中断。硬件中断也称为外设中断，当外设完成用户的请求时会向CPU发送中断信号。
> 2. 系统调用。其实系统调用本身就是中断，只不过是软件中断，跟硬件中断不同。
> 3. 异常。如果当前进程运行在用户态，这个时候发生了异常事件（例如缺页异常），就会触发切换。
>
> 用户态是应用程序运行的空间，为了能访问到内核管理的资源（例如CPU、内存、I/O），可以通过内核态所提供的访问接口实现，这些接口就叫系统调用。pthread_mutex_lock系统调用是内核态为用户态进程提供的Linux内核态下互斥锁的访问机制，所以使用pthread_mutex_lock系统调用时，进程需要从用户态切换到内核态，而这种切换是需要消耗很多时间的，有可能比用户执行代码的时间还要长。

### 2.7.2 重量级锁示例

本示例只需要将上一个轻量级锁的示例中，两个线程中间的等待时间去掉即可，如下图：

![image-20220421150631021](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220421150631021.png)

### 2.7.3 锁升级过程

1. 线程抢锁时，JVM首先检测内置锁对象是否是可偏向状态。
2. 在内置锁对象确认为可偏向状态之后，JVM检查Mark Word中的线程ID是否为抢锁线程ID，如果是，就表示抢锁线程处于偏向锁状态，抢锁线程快速获得锁，开始执行临界区代码。如果不是可偏向状态就直接升级成轻量级锁
3. 如果Mark Word中的线程ID并未指向抢锁线程，就通过CAS操作竞争锁。如果竞争成功，就将Mark Word中的线程ID设置为抢锁线程，偏向标志位设置为1，锁标志位设置为01，然后执行临界区代码，此时内置锁对象处于偏向锁状态。
4. 如果CAS操作失败，就说明发生了竞争，撤销偏向锁，进而升级为轻量级锁。
5. JVM使用CAS将锁对象的Mark Word替换为抢锁线程的锁记录指针，如果成功，抢锁线程就获得锁。如果替换失败，就表示其他线程竞争锁，JVM尝试使用CAS自旋替换抢锁线程的锁记录指针，如果自旋成功（抢锁成功），那么锁对象依然处于轻量级锁状态。
6. 如果JVM的CAS替换锁记录指针自旋失败，轻量级锁就膨胀为重量级锁，后面等待锁的线程也要进入阻塞状态。

总体来说，偏向锁是在没有发生锁争用的情况下使用的；一旦有了第二个线程争用锁，偏向锁就会升级为轻量级锁；如果锁争用很激烈，轻量级锁的CAS自旋到达阈值后，轻量级锁就会升级为重量级锁。

## 2.8 线程通信

在现实中，如果需要多个线程按照指定的规则共同完成一个任务，那么这些线程之间就需要互相协调，这个过程被称为线程的通信。

线程的通信可以被定义为：当多个线程共同操作共享的资源时，线程间通过某种方式互相告知自己的状态，以避免无效的资源争夺。线程间通信的方式可以有很多种：等待-通知、共享内存、管道流。最常用的就是等待通知模式。

### 2.8.1 剖析低效线程轮询

### 2.8.2 Java的wait和notify

Java对象中的wait()、notify()两类方法就如同信号开关，用于等待方和通知方之间的交互。

对象的wait()方法的主要作用是让当前线程阻塞并等待被唤醒。wait()方法与对象监视器紧密相关，使用wait()方法时一定要放在同步块中。Java中wait一共有三种重载，源码如下：

```java
//显示等待版本
public final native void wait(long timeout) throws InterruptedException;
//高精度x
public final void wait(long timeout, int nanos) throws InterruptedException {
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
            "nanosecond timeout value out of range");
    }

    if (nanos > 0) {
        timeout++;
    }

    wait(timeout);
}
//基础版本，当前线程调用了同步对象的wait实例方法后，导致当前线程等待，当前线程进入对象的监视器WaitSet，等待被其他线程唤醒。
public final void wait() throws InterruptedException {
	wait(0);
}
```

### 2.8.3 生产者消费者改造

