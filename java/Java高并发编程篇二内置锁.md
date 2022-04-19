# Java高并发编程学习笔记第二部分Java内置锁篇

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第二部分Java内置锁篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java内置锁的核心原理。

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

