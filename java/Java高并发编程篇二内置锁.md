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
    <version>0.11</version>
</dependency>
```

然后编写一个等待进行对象布局分析的Java类：















