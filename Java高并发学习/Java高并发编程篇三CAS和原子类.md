# Java高并发编程篇三CAS和原子类

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷2一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第三部分CAS和原子类篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java内置锁的核心原理，主要就是对CAS的原理和弊端，和基于CAS实现的JUC原子类进行研究学习。

## 3.1 CAS

JDK 5所增加的JUC（java.util.concurrent）并发包对操作系统的底层CAS原子操作进行了封装，为上层Java程序提供了CAS操作的API。

### 3.1.1 CAS简介

CAS是一种无锁算法，该算法关键依赖两个值——期望值（旧值）和新值，底层CPU利用原子操作判断内存原值与期望值是否相等，如果相等就给内存地址赋新值，否则不做任何操作。

使用CAS进行无锁编程的步骤大致如下：

1. 获得字段的期望值（oldValue）。
2. 计算出需要替换的新值（newValue）。
3. 通过CAS将新值（newValue）放在字段的内存地址上，如果CAS失败就重复第1步到第2步，一直到CAS成功，这种重复俗称CAS自旋。

当CAS将内存地址的值与预期值进行比较时，如果相等，就证明内存地址的值没有被修改，可以替换成新值，然后继续往下运行；如果不相等，就说明内存地址的值已经被修改，放弃替换操作，然后重新自旋。

### 3.1.2 Unsafe操作CAS

Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全的底层操作，如直接访问系统内存资源、自主管理内存资源等。Unsafe大量的方法都是native方法，基于C++语言实现，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。Unsafe类的全限定名为sun.misc.Unsafe，从名字中可以看出这个类对普通程序员来说是“危险”的，一般的应用开发都不会涉及此类，Java官方也不建议直接在应用程序中使用这些类。

操作系统层面的CAS是一条CPU的原子指令（cmpxchg指令），正是由于该指令具备原子性，因此使用CAS操作数据时不会造成数据不一致的问题，Unsafe提供的CAS方法直接通过native方式（封装C++代码）调用了底层的CPU指令cmpxchg。

完成Java应用层的CAS操作主要涉及Unsafe方法的调用，具体如下：

- 获取Unsafe实例。
- 调用Unsafe提供的CAS方法，这些方法主要封装了底层CPU的CAS原子操作。
- 调用Unsafe提供的字段偏移量方法，这些方法用于获取对象中的字段（属性）偏移量，此偏移量值需要作为参数提供给CAS操作。

**获取Unsafe实例**

我们先看一下Unsafe类的源码：

```java
public final class Unsafe {
    private static final Unsafe theUnsafe;
    private static native void registerNatives();
    private Unsafe() {
    }
    //其余代码略
}
```

这个类是final的且构造函数是私有化的，所以我们需要通过反射来获取实例，代码如下：

```java
public class JVMUtil {

    public static Unsafe getUnsafe(){
        try {
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            return (Unsafe) theUnsafe.get(null);
        } catch (Exception e) {
           throw new AssertionError(e);
        }
    }
}
```

**调用unsafe的CAS方法**

我们先来看一下unsafe关于CAS的方法：

```java
/**
* Unsafe提供的CAS方法包含4个操作数——字段所在的对象、字段内存位置、预期原值及新值。在执行Unsafe的CAS方法时，这些方法首先将内存位置的值与预期值（旧的值）比较，如果相匹配，那么CPU会* 自动将该内存位置的值更新为新值，并返回true；如果不匹配，CPU不做任何操作，并返回false。Unsafe的CAS操作会将第一个参数（对象的指针、地址）与第二个参数（字段偏移量）组合在一起，计算* 出最终的内存操作地址。
* @param o 需要操作的字段所在的对象
* @param offset 需要操作的字段的偏移量
* @param expected 旧的值
* @param update 新的值
* @return true 成功 false失败
*/
public final native boolean compareAndSwapObject(Object o, long offset, Object expected, Object update);

public final native boolean compareAndSwapInt(Object o, long offset, int expected, int update);

public final native boolean compareAndSwapLong(Object o, long offset, long expected, long update);
```

**调用unsafe的偏移量操作**

Unsafe提供的获取字段（属性）偏移量的源码主要如下：

```java
//参数是需要操作字段的反射，返回字段的偏移量
//staticFieldOffset()方法用于获取静态属性Field在Class对象中的偏移量，在CAS中操作静态属性时会用到这个偏移量。
public native long staticFieldOffset(Field var1);
//objectFieldOffset()方法用于获取非静态Field（非静态属性）在Object实例中的偏移量，在CAS中操作对象的非静态属性时会用到这个偏移量。
public native long objectFieldOffset(Field var1);
```

### 3.1.3 Unsafe示例：实现线程安全自增

需求：总计10个线程并行运行，每个线程通过CAS自旋对一个共享数据进行自增运算，并且每个线程需要成功自增运算1000次。

```java
public class UnsafeLearn {
    private static final int THREAD_COUNT = 10;
    static class PlusNoLock {
        //內部值，使用volatile保证可见性
        private volatile int value;
        //不安全类
        private static final Unsafe unsafe = JVMUtil.getUnsafe();
        //value的内存偏移
        private static long valueOffset;

        private static final AtomicInteger failNum = new AtomicInteger(0);

        static {
            try {
                valueOffset = unsafe.objectFieldOffset(PlusNoLock.class.getDeclaredField("value"));
                System.out.println("valueOffset:" + valueOffset);
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
        }

        //CAS原子操作
        public final boolean unSafeCompareAndSet(int oldValue, int newValue) {
            return unsafe.compareAndSwapInt(this, valueOffset, oldValue, newValue);
        }

        public void selfPlus() {
            int oldValue = value;
            while (!unSafeCompareAndSet(oldValue, oldValue + 1)){
                oldValue = value;
                failNum.incrementAndGet();
            }
        }

    }
	//测试
    public static void main(String[] args) throws InterruptedException {
        final PlusNoLock cas = new PlusNoLock();
        CountDownLatch latch = new CountDownLatch(THREAD_COUNT);
        for (int i = 0; i < THREAD_COUNT; i++) {
            ExecutorService service = Executors.newFixedThreadPool(10);
            service.submit(()->{
                for (int j = 0; j < 1000; j++) {
                    cas.selfPlus();
                }
                latch.countDown();
            });
        }
        latch.await();
        System.out.println("结果："+cas.value);
        System.out.println("失败次数"+ PlusNoLock.failNum.get());
    }
}
```

运行结果：

valueOffset:12
结果：10000
失败次数23856

### 3.1.4 字段偏移量

我们在上面的例子中最后运行的偏移量结果是12，这个是怎么算的呢？我们下面来分析，首先看一下我们的PlusNoLock的属性

```java
//內部值，使用volatile保证可见性
private volatile int value;
//不安全类
private static final Unsafe unsafe = JVMUtil.getUnsafe();
//value的内存偏移
private static long valueOffset;

private static final AtomicInteger failNum = new AtomicInteger(0);
```

我们可以看到除了value以外其余全部是类的静态成员，属于类的成员而不是对象的成员，真正属于对象的字段只有其中的value字段，那么当前的对象结构就是这样的：

![CASLEARN20220422](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/CASLEARN20220422.png)

在64位的JVM堆区中一个PlusNoLock对象的Object Header（头部）占用了12字节，其中Mark Word占用了8字节（64位），压缩过的Class Pointer占用了4字节。接在Object Header之后的就是成员属性value的内存区域，所以value属性相对于Object Header的偏移量为12。

> 调用Unsafe.objectFieldOffset(…)方法获取到的Object字段（也叫Object成员属性）的偏移量值是字段相对于Object头部的偏移量，是一个相对的内存地址值，不是绝对的内存地址值。

当然其实也可以用JOL来验证，代码如下：

```java
PlusNoLock plusNoLock = new PlusNoLock();
plusNoLock.value =10;
System.out.println(ClassLayout.parseInstance(plusNoLock).toPrintable());
```

结果如下：

~~~
com.learn.testnginx.casLearn.UnsafeLearn$PlusNoLock object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c143
 12   4    int PlusNoLock.value          10
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
~~~

JOL输出的结果可以看出，一个PlusNoLock对象的Object Header占用了12字节，而value属性的内存位置紧挨在Object Header之后，所以value属性的相对偏移量值为12。

## 3.2 JUC原子类

在多线程并发执行时，诸如“++”或“--”类的运算不具备原子性，不是线程安全的操作。通常情况下，大家会使用synchronized将这些线程不安全的操作变成同步操作，但是这样会降低并发程序的性能。所以，JDK为这些类型不安全的操作提供了一些原子类，与synchronized同步机制相比，JDK原子类是基于CAS轻量级原子操作的实现，使得程序运行效率变得更高。

### 3.2.1 Atomic原子类

Atomic操作翻译成中文是指一个不可中断的操作，即使在多个线程一起执行Atomic类型操作的时候，一个操作一旦开始，就不会被其他线程中断。所谓Atomic类，指的是具有原子操作特征的类。

JUC并发包中的原子类都存放在java.util.concurrent.atomic类路径下，如下图：

![image-20220422135731956](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220422135731956.png)

根据操作的目标数据类型，可以将JUC包中的原子类分为4类：基本原子类、数组原子类、原子引用类和字段更新原子类。

1. 基本原子类的功能是通过原子方式更新Java基础类型变量的值。基本原子类主要包括以下三个：

   - AtomicInteger：整型原子类。
   - AtomicLong：长整型原子类。
   - AtomicBoolean：布尔型原子类。

2. 数组原子类数组原子类的功能是通过原子方式更数组中的某个元素的值。数组原子类主要包括以下三个：

   - AtomicIntegerArray：整型数组原子类。
   - AtomicLongArray：长整型数组原子类。
   - AtomicReferenceArray：引用类型数组原子类。

3. 引用原子类引用原子类主要包括以下三个：

   - AtomicReference：引用类型原子类。
   - AtomicMarkableReference：带有更新标记位的原子引用类型。
   - AtomicStampedReference：带有更新版本号的原子引用类型。
   
     AtomicMarkableReference类将boolean标记与引用关联起来，可以解决使用AtomicBoolean进行原子更新时可能出现的ABA问题。AtomicStampedReference类将整数值与引用关联起来，可以解决使用AtomicInteger进行原子更新时可能出现的ABA问题。
   
4. 字段更新原子类字段更新原子类主要包括以下三个：

   - AtomicIntegerFieldUpdater：原子更新整型字段的更新器。
   - AtomicLongFieldUpdater：原子更新长整型字段的更新器。
   - AtomicReferenceFieldUpdater：原子更新引用类型中的字段。

### 3.2.2 基本原子类

我们以AtomicInteger入手了解基本原子类，其余的基本原子类用法都差不多。

此类的常用方法源码如下:

```java
//获取当前值
public final int get() {
    return value;
}
//获取当前值，然后设置新值
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
//获取当前值，然后自增
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
//获取当前值，然后自减
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}
//以原子方式将给定值添加到当前值。
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}
//如果当前值==期望值，则自动将该值设置为给定的更新值。
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

我们举个例子：

```java
public class AtomicIntegerLearn {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10);
        AtomicInteger integer = new AtomicInteger(0);
        for (int i = 0; i < 10; i++) {
            ExecutorService service = Executors.newFixedThreadPool(10);
            service.submit(()->{
                for (int j = 0; j < 1000; j++) {
                    integer.incrementAndGet();
                }
                latch.countDown();
            });
        }
        latch.await();
        System.out.println("结果"+integer.get());
    }

}
```

通过结果可以看出：10个线程每个线程累加1000次，结果为10000，该结果与预期结果相同。所以，对基础原子类实例的并发操作是线程安全的。

我们可以看一下AtomicInteger的源码，其实AtomicInteger源码中的主要方法都是通过CAS自旋实现的，而且其实和我们上面3.1.3的例子有点像：

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;
    // setup to use Unsafe.compareAndSwapInt for updates
    //获取unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //偏移量
    private static final long valueOffset;

    //设置偏移量
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	//value成员是一个使用关键字volatile修饰的内部成员。关键字volatile的原理比较复杂，简单地说，该关键字可以保证任何线程在任何时刻总能拿到该变量的最新值，其目的在于保障变量值的线程可见性。
    private volatile int value;
    
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }
   
    public AtomicInteger() {
    }

    public final int get() {
        return value;
    }

    public final void set(int newValue) {
        value = newValue;
    }

    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
   
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }
    //其余方法略
}
```

### 3.2.3 数组原子类

跟上面同理我们这里以AtomicIntegerArray为例来学习。

此类的常用方法源码如下:

```java
//获取i位置元素的值
public final int get(int i) {
    return getRaw(checkedByteOffset(i));
}

private int getRaw(long offset) {
    return unsafe.getIntVolatile(array, offset);
}
//原子地将位置i的元素设置为给定值并返回旧值。
public final int getAndSet(int i, int newValue) {
    return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
}
//获取在索引i处的元素并且自增1。
public final int getAndIncrement(int i) {
    return getAndAdd(i, 1);
}
//获取在索引i处的元素并且自减1
public final int getAndDecrement(int i) {
    return getAndAdd(i, -1);
}
//获取i位置的值，并将给定的值原子地加到索引为i的元素上。
public final int getAndAdd(int i, int delta) {
    return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
}
//如果当前值==期望值，则原子地将位置i的元素设置为给定的更新值。
public final boolean compareAndSet(int i, int expect, int update) {
    return compareAndSetRaw(checkedByteOffset(i), expect, update);
}
//最终将i位置的元素设置为给定的值。
//此方法可能导致其他线程在之后的一小段时间内还可以读取到旧值
public final void lazySet(int i, int newValue) {
    unsafe.putOrderedInt(array, checkedByteOffset(i), newValue);
}
```

我们还是举个例子：

```java
public class AtomicIntegerArrayLearn {
    public static void main(String[] args) {
        int temp = 0;
        int [] array ={1,2,3,4,5,6};
        AtomicIntegerArray integerArray = new AtomicIntegerArray(array);
        temp = integerArray.getAndSet(0,2);
        System.out.println("temp="+temp+" 数组"+integerArray);
        temp = integerArray.getAndIncrement(0);
        System.out.println("temp="+temp+" 数组"+integerArray);
        temp = integerArray.getAndAdd(0,9);
        System.out.println("temp="+temp+" 数组"+integerArray);
    }
}
```

运行结果：

temp=1 数组[2, 2, 3, 4, 5, 6]
temp=2 数组[3, 2, 3, 4, 5, 6]
temp=3 数组[12, 2, 3, 4, 5, 6]

### 3.2.4 引用类型原子类

基础的原子类型只能保证一个变量的原子操作，当需要对多个变量进行操作时，CAS无法保证原子性操作，这时可以用AtomicReference（原子引用类型）保证对象引用的原子性。

跟上面同理，我们以AtomicReference为例来进行学习。

我们通过简单的例子来学习一下用法：

首先搞个实体类：

```java
@Data
public class User {
    String uid;
    String nickName;
    public volatile int age;//为属性更新原子类的例子使用

    public User(String uid, String nickName) {
        this.uid = uid;
        this.nickName = nickName;
    }
}
```

然后编写测试类：

```java
public class LearnAtomicReference {
    public static void main(String[] args) {
        AtomicReference<User> userRef = new AtomicReference<>();
        User user = new User("1","zhansan");
        //为原子对象设置值
        userRef.set(user);
        System.out.println("userRef:: "+userRef.get());
        User newUser = new User("2","lisi");
        boolean isSuccess = userRef.compareAndSet(user, newUser);
        System.out.println("isSuccess:"+isSuccess);
        System.out.println("cas userRef："+userRef.get());
    }
}
```

> 使用原子引用类型AtomicReference包装了User对象之后，只能保障User引用的原子操作，对被包装的User对象的字段值修改时不能保证原子性，这点要切记。

### 3.2.5 属性更新原子类

跟上面同理，我们以AtomicIntegerFieldUpdater为例来进行学习。

使用属性更新原子类保障属性安全更新的流程大致需要两步：

1. 更新的对象属性必须使用public volatile修饰符。
2. 因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须调用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。

我们通过简单的例子来学习一下用法：



```java
public class LearnAtomicIntegerFieldUpdater {
    public static void main(String[] args) {
        AtomicIntegerFieldUpdater<User> updater = AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
        User user = new User("1","zhangsan");
        System.out.println(updater.getAndIncrement(user));
        System.out.println(updater.getAndAdd(user,100));
        System.out.println(updater.get(user));
    }
}
```

运行结果：

0
1
101

## 3.3 CAS的ABA问题

### 3.3.1 ABA介绍

什么是ABA问题？举一个例子来说明。比如一个线程A从内存位置M中取出1，另一个线程B也取出1。现在假设线程B进行了一些操作之后将M位置的数据1变成了2，然后又在一些操作之后将2变成了1。之后，线程A进行CAS操作，但是线程A发现M位置的数据仍然是1，然后线程A操作成功。尽管线程A的CAS操作成功，但是不代表这个过程是没有问题的，线程A操作的数据1可能已经不是之前的1，而是被线程B替换过的1，这就是ABA问题。

我们下面用一个LIFO（后进先出）堆栈来解释ABA问题。
