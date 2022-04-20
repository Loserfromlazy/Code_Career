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

### 2.3.2 线程安全改造









