# Java高并发编程篇三CAS原子类以及有序性和可见性

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷2尼恩编著、以及菜鸟等互联网资源
>
> 部分jdk源码不会全部展示，请自行去查阅源代码或[JavaAPI文档](https://docs.oracle.com/javase/8/docs/api/)

本文主要是Java内置锁的核心原理，主要就是对CAS的原理和弊端，和基于CAS实现的JUC原子类同时对有序性和可见性进行研究学习。

# 三、CAS及原子类

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

> 使用时需要注意，通过反射获取的实例不受JVM堆外内存参数设置的限制

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

我们下面用一个LIFO（后进先出）的单向链表来解释ABA问题，结构如下：

![CASABA120220424](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/CASABA120220424.png)

现在有AB两个线程，A想将V1弹出，B想弹出V1和V2，然后再压入V1和V3。现在A想执行此操作，但是B却抢先一步获得了CPU时间片，因此B先执行，B执行完后链表变成了这样：

![CASABA220220424](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/CASABA220220424.png)

然后这时A在B执行完后获得了时间片开始执行，A判断V1的值没发生变化所以将V1弹出，链表现在变成了这样：

![CASABA320220424](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/CASABA320220424.png)

为什么是这样呢？因为线程A将V1弹出的原子操作是CAS(V1,V2),即将head指针指向从V1变成V2，但因为E2.next为NULL，此时的链表只有1个元素，V3就变成游离状态了，就会造成链表无故缺少数据，这就是ABA问题引发的不正常状态。

**如果想解决问题，也有解决方案，那就是增加版本号，很多乐观锁的实现版本都是使用版本号（Version）方式来解决ABA问题。每次在执行数据的修改操作时都会带上一个版本号，版本号和数据的版本号一致就可以执行修改操作并对版本号执行加1操作，否则执行失败。**

### 3.3.2 AtomicStampedReference解决ABA问题

参考乐观锁的版本号，JDK提供了一个AtomicStampedReference类来解决ABA问题。AtomicStampReference在CAS的基础上增加了一个Stamp（印戳或标记），使用这个印戳可以用来觉察数据是否发生变化，给数据带上了一种实效性的检验。

我们先来看一下构造函数源码：

```java
//initialRef原始数据 ， initialStamp表示版本号
public AtomicStampedReference(V initialRef, int initialStamp) {
    pair = Pair.of(initialRef, initialStamp);
}
```

常用的几个方法源码：

```java
/**
 * Returns the current value of the reference.
 * 获取被封装的数据
 * @return the current value of the reference
 */
public V getReference() {
    return pair.reference;
}

/**
 * Returns the current value of the stamp.
 * 获取版本号
 * @return the current value of the stamp
 */
public int getStamp() {
    return pair.stamp;
}
/**
     * Atomically sets the value of both the reference and stamp
     * to the given update values if the
     * current reference is {@code ==} to the expected reference
     * and the current stamp is equal to the expected stamp.
     *
     * @param expectedReference the expected value of the reference 预期值
     * @param newReference the new value for the reference 更新的值
     * @param expectedStamp the expected value of the stamp 预期版本号
     * @param newStamp the new value for the stamp 更新后的版本号
     * @return {@code true} if successful
     */
public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```

我们下面给个例子体会用法通过两个线程分别带上版本号更新同一个AtomicStampedReference实例的值，第一个线程会更新成功，而第二个线程会更新失败：

```java
public class Test {
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(2);
        AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<>(1,0);

        ExecutorService service = Executors.newFixedThreadPool(2);
        service.submit(() -> {
            int stamp = stampedReference.getStamp();
            System.out.println("[500] value="+stampedReference.getReference()+" stamp="+stampedReference.getStamp());
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean success = stampedReference.compareAndSet(1, 10, stamp, stamp + 1);
            System.out.println("[500] isSuccess="+success+" value="+stampedReference.getReference()+" stamp="+stampedReference.getStamp());
            stamp++;
            success = stampedReference.compareAndSet(10, 1, stamp, stamp + 1);
            System.out.println("[500] isSuccess="+success+" value="+stampedReference.getReference()+" stamp="+stampedReference.getStamp());
            latch.countDown();
        });
        service.submit(() -> {
            int stamp = stampedReference.getStamp();
            System.out.println("[1000] value="+stampedReference.getReference()+" stamp="+stampedReference.getStamp());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean success = stampedReference.compareAndSet(1, 20, stamp, stamp + 1);
            System.out.println("[1000] isSuccess="+success+" value="+stampedReference.getReference()+" stamp="+stampedReference.getStamp());
            latch.countDown();
        });
        latch.await();
    }
}
```

运行结果：

[500] value=1 stamp=0
[1000] value=1 stamp=0
[500] isSuccess=true value=10 stamp=1
[500] isSuccess=true value=1 stamp=2
[1000] isSuccess=false value=1 stamp=2

### 3.3.3 AtomicMarkableReference解决ABA问题

AtomicMarkableReference是AtomicStampedReference的简化版，不关心修改过几次，只关心是否修改过。因此，其标记属性mark是boolean类型，而不是数字类型，标记属性mark仅记录值是否修改过。

其标记属性mark是boolean类型，而不是数字类型，标记属性mark仅记录值是否修改过。AtomicMarkableReference适用于只要知道对象是否被修改过，而不适用于对象被反复修改的场景。

我们下面将上面例子修改为AtomicMarkableReference并体会用法：

```java
public class Test {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(2);
        AtomicMarkableReference<Integer> markableReference = new AtomicMarkableReference<>(1,false);

        ExecutorService service = Executors.newFixedThreadPool(2);
        service.submit(() -> {
            boolean mark = getMark(markableReference);
            int value = markableReference.getReference();
            System.out.println("[500] value="+value+" mark="+mark);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            boolean success = markableReference.compareAndSet(1, 10, mark, !mark);
            System.out.println("[500] isSuccess="+success+" value="+value+" mark="+mark);
            latch.countDown();
        });
        service.submit(() -> {
            int value = markableReference.getReference();
            boolean mark = getMark(markableReference);
            System.out.println("[1000] value="+value+" mark="+mark);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("[1000] +mark="+getMark(markableReference));
            boolean success = markableReference.compareAndSet(1, 20, mark, !mark);
            System.out.println("[1000] isSuccess="+success+" value="+value+" mark="+mark);
            latch.countDown();
        });
        latch.await();
    }

    private static boolean getMark(AtomicMarkableReference<Integer> markableReference){
        boolean[] markHolder = {false};
        //此方法需要传入一个boolean的数组markHolder，然后在get内部如果发现当前值被修改了就会将传入的markHolder进行修改因此可以用此方法获取mark标记
        //可以自行进入源码查看，这里就不展示此方法的源码了
        markableReference.get(markHolder);
        return markHolder[0];
    }
}
```

运行结果：

[500] value=1 mark=false
[1000] value=1 mark=false
[500] isSuccess=true value=1 mark=false
[1000] +mark=true
[1000] isSuccess=false value=1 mark=false



## 3.4 高并发下的CAS性能

在争用激烈的场景下，会导致大量的CAS空自旋。大量的CAS空自旋会浪费大量的CPU资源，大大降低了程序的性能。

### 3.4.1 LongAdder

Java 8提供了一个新的类LongAdder，以空间换时间的方式提升高并发场景下CAS操作的性能。

LongAdder的核心思想是热点分离，与ConcurrentHashMap的设计思想类似：将value值分离成一个数组，当多线程访问时，通过Hash算法将线程映射到数组的一个元素进行操作；而获取最终的value结果时，则将数组的元素求和。

最终，通过LongAdder将内部操作对象从单个value值“演变”成一系列的数组元素，从而减小了内部竞争的粒度。如下图：

![longadder20220424](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/longadder20220424.png)

我们下面举个例子，对比一下LongAdder和AtomicLong：（PS本例中的线程池是[第一篇多线程篇](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%B8%80%E5%A4%9A%E7%BA%BF%E7%A8%8B.md)中的CPU密集型线程池示例中的线程池）

```java
@SpringBootTest
public class TestLongAdder {
    //测试轮数
    final int TURNS = 1000;

    @Test
    public void testAtomicLong(){
        //并发数
        final int TASK_COUNT = 10;
        //线程池
        ThreadPoolExecutor cpuExecutor = IOThreadUtil.getCPUExecutor();
        AtomicLong atomicLong = new AtomicLong(0);
        CountDownLatch latch = new CountDownLatch(TASK_COUNT);
        long start = System.currentTimeMillis();
        for (int i = 0; i < TASK_COUNT; i++) {
            cpuExecutor.submit(()->{
               try {
                   for (int j = 0; j < TURNS; j++) {
                       atomicLong.incrementAndGet();
                   }
               }catch (Exception e){
                   e.printStackTrace();
               }
               latch.countDown();
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) / 1000f;
        System.out.println("运行时长："+time);
        System.out.println("结果："+atomicLong.get());
    }

    @Test
    public void testLongAdder(){
        //并发数
        final int TASK_COUNT = 10;
        //线程池
        ThreadPoolExecutor cpuExecutor = IOThreadUtil.getCPUExecutor();
        LongAdder longAdder = new LongAdder();
        CountDownLatch latch = new CountDownLatch(TASK_COUNT);
        long start = System.currentTimeMillis();
        for (int i = 0; i < TASK_COUNT; i++) {
            cpuExecutor.submit(()->{
                try {
                    for (int j = 0; j < TURNS; j++) {
                        longAdder.add(1);
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
                latch.countDown();
            });
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        float time = (System.currentTimeMillis() - start) / 1000f;
        System.out.println("运行时长："+time);
        System.out.println("结果："+longAdder.longValue());
    }
}
```

testAtomicLong()运行结果：

运行时长：0.002
结果：10000

testLongAdder()运行结果：

运行时长：0.002
结果：10000

我们逐渐增大测试轮数，查看结果：

| 测试轮数 | AtomicLong运行时长 | AtomicLong结果 |  LongAdder运行时长    |  LongAdder结果 |
| -------- | ------------------ | -------------- | ---- | ------------------------------------------------ |
| 1000 | 0.002 | 10000 | 0.002 | 10000 |
| 10000 | 0.004              | 100000 | 0.009 | 100000 |
| 100000 | 0.053 | 1000000 | 0.073 | 1000000 |
| 1000000 | 0.158 | 10000000 | 0.052 | 10000000 |
| 10000000 | 1.068 | 10000000 | 0.149 | 100000000 |

随着累加次数的增加，CAS操作的次数急剧增多，AtomicLong的性能急剧下降。从对比实验的结果可以看出，在CAS争用最为激烈的场景下，LongAdder的性能明显优于AtomicLong。

### 3.4.2 LongAdder原理

我们分析一下源码，首先是类的结构：

```java
public class LongAdder extends Striped64 implements Serializable
```

可以看到LongAdder继承了Striped64类，而在Striped64类中包含了几个重要的属性：

```java
/**
 * Table of cells. When non-null, size is a power of 2.
 * 存放Cell的表，大小为2的幂
 */
transient volatile Cell[] cells;

/**
 * Base value, used mainly when there is no contention, but also as
 * a fallback during table initialization races. Updated via CAS.
 * 基础值，在没有竞争时会更新这个值，在cells初始化时cells不可用，也会通过CAS操作累加到base
 */
transient volatile long base;

/**
 * Spinlock (locked via CAS) used when resizing and/or creating Cells.
 * 自旋锁，通过CAS加锁，为0表示cells没有处于创建或扩容状态，为1表示正在创建或扩容cells数组，不能进行新cell元素的设置操作
 * 当cellsBusy成员值为1时，表示cells数组正在被某个线程执行初始化或扩容操作，其他线程不能进行以下操作：（1）对cells数组执行初始化。（2）对cells数组执行扩容。（3）如果cells数组中某个元素为null，就为该元素创建新的Cell对象。因为数组的结构正在修改，所以其他线程不能创建新的Cell对象。
 */
transient volatile int cellsBusy;
```

Striped64内部包含一个base和一个Cell[]类型的cells数组，cells数组又叫哈希表。在没有竞争的情况下，要累加的数通过CAS累加到base上；如果有竞争的话，会将要累加的数累加到cells数组中的某个Cell元素里面。所以Striped64的整体值value为base+∑[0~n]cells。

然后我们回到LongAdder。在LongAdder类中还包含了几个操作值的重要方法，源码如下：

```java
/**
 * 获取值，本质调用了sum，累加cell数组
 */
public long longValue() {
    return sum();
}
//累加cell数组
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}

public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //第一个条件：(as = cells) != null
    //cells数组不为null，说明存在争用；在不存在争用的时候，cells数组一定为null，一旦对base的cas操作失败，才会初始化cells数组。
    //第二个条件：!casBase(b = base, b + x)
    //如果cells数组为null，表示之前不存在争用，并且此次casBase执行成功，表示基于base成员累加成功，add方法直接返回；如果casBase方法执行失败，说明产生了第一次争用冲突，需要对cells数组初始化，此时即将进入内层if块。
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 || //表示cells没有初始化
            (a = as[getProbe() & m]) == null || // 指当前线程的hash在cells数组映射位置的cell对象为空，意思是还没有其他线程在同一个位置做过累加操作。
            !(uncontended = a.cas(v = a.value, v + x)))//指当前线程的哈希值在cells数组映射位置的Cell对象不为空，然后在该Cell对象上进行CAS操作，设置其值为v+x（x为该Cell需要累加的值），但是CAS操作失败，表示存在争用。
            //以上三个条件有一个为真就进入下面的longAccumulate方法
            longAccumulate(x, null, uncontended);
    }
}

//自增
public void increment() {
    add(1L);
}
//自减
public void decrement() {
    add(-1L);
}
final boolean casBase(long cmp, long val) {
    return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```

从上面源码可以看到Striped64的设计核心思路是通过内部的分散计算来避免竞争，以空间换时间。LongAdder的base类似于AtomicInteger里面的value，在没有竞争的情况下，cells数组为null，这时只使用base进行累加；而一旦发生竞争，cells数组就上场了。

然后我们分析一下add方法中的longAccumulate，源码如下：

~~~java
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;//当前线程的探测值。
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    //扩容意向，表示是否可以进行扩容
    boolean collide = false;                // True if last slot nonempty
    //自旋，直到操作成功
    for (;;) {
        //as cells数组，a 表示当前线程命中的Cell，n 表示cells数组长度，v表示期望值
        Cell[] as; Cell a; int n; long v;
        //分支1： 表示cells已经初始化了，当前线程应该将数据写入cell中
        if ((as = cells) != null && (n = as.length) > 0) {
            //分支1.1：表示下标位置的Cell为空，需要创建新 Cell
            if ((a = as[(n - 1) & h]) == null) {
                //等于0表示cells数组没有处于创建扩容阶段
                if (cellsBusy == 0) {       // Try to attach new Cell
                    Cell r = new Cell(x);   // Optimistically create
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            //分支1.2：当前线程竞争修改失败.
            //这个wasUncontended是add方法传进来的，如果为false表示cells已经被初始化了，且该位置的Cell也被初始化了，但是当前线程却修改失败了
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            //分支1.3：当前线程rehash过哈希值，CAS更新Cell。在分支1的最后执行rehash，命中新的Cell，如果新的Cell不为空则在此进行CAS操作
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            //分支1.4：调整扩容意向，进入下一轮循环。
            //如果cells数组大小大于CPU核数，修改为false不再进行扩容
            //如果cells不等于as也表示其他线程已经进行了扩容，表示当前循环不再进行扩容了
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale达到最大值或者as过期
            //分支1.5：调整扩容意向为true，不一定真的扩容
            //这里的意思是判断扩容意向是否不满足，如果不满足就设置为扩容意见已经满足，然后rehash进入下一轮循环
            else if (!collide)
                collide = true;
            //分支1.6：真正的扩容逻辑。
            //cellsBusy == 0表示无锁状态
            //casCellsBusy()表示获取锁成功，可以进行扩容
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      // Expand table unless stale
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;//释放锁
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = advanceProbe(h);//重置（rehash）当前线程Hash值
        }
        //分支2：cells未初始化，本分支准备初始化cells，执行加锁方法并且要求cellsBusy加锁成功
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        //分支3：当前线程加锁失败，表示其他线程正在初始化cells，将当前线程值累加到base
        //注意add方法调用此方法时fn为null
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base 在base操作成功时跳出自旋
    }
}
~~~

## 3.5 总结

### 3.5.1 CAS的弊端

CAS操作的弊端主要有以下三点：

1. ABA问题

   JDK提供了两个类AtomicStampedReference和AtomicMarkableReference来解决ABA问题。

2. 只能保证一个共享变量之间的原子性操作

   JDK提供了AtomicReference类来保证引用对象之间的原子性，可以把多个变量放在一个AtomicReference实例后再进行CAS操作。

3. 开销问题

   解决CAS恶性空自旋的有效方式之一是以空间换时间，较为常见的方案为：

   - 分散操作热点，使用LongAdder替代基础原子类AtomicLong，LongAdder将单个CAS热点（value值）分散到一个cells数组中。
   - 使用队列削峰，将发生CAS争用的线程加入一个队列中排队，降低CAS争用的激烈程度。JUC中非常重要的基础类AQS（抽象队列同步器）就是这么做的。

### 3.5.2 在JDK中的应用

CAS在java.util.concurrent.atomic包中的原子类、Java AQS以及显式锁、CurrentHashMap等重要并发容器类的实现都有非常广泛的应用。

在java.util.concurrent.atomic包的原子类（如AtomicXXX）中都使用了CAS来保障对数字成员进行操作的原子性。

java.util.concurrent的大多数类（包括显式锁、并发容器）都是基于AQS和AtomicXXX来实现的，其中AQS通过CAS保障它内部双向队列头部、尾部操作的原子性。

# 四、有序性和可见性

原子性、可见性、有序性是并发编程所面临的三大问题。Java通过CAS操作解决了并发编程中的原子性问题，所以我们接下来学习有序性和可见性。

## 4.1 背景CPU物理缓存结构

由于CPU的运算速度比主存（物理内存）的存取速度快很多，为了提高处理速度，现代CPU不直接和主存进行通信，而是在CPU和主存之间设计了多层的Cache（高速缓存），越靠近CPU的高速缓存越快，容量也越小。

> CPU高速缓存结构：图片来自于Java高并发编程第二卷
>
> ![image-20220424142450067](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220424142450067.png)
>
> Cache的基本结构：图片来自于计算机组成原理唐朔飞版
>
> ![image-20220424142541472](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220424142541472.png)

我们以三级缓存架构为例：CPU内核读取数据时，先从L1高速缓存中读取，如果没有命中，再到L2、L3高速缓存中读取，假如这些高速缓存都没有命中，它就会到主存中读取所需要的数据。

- L1高速缓存最接近CPU，容量最小（如32KB、64KB等）、存取速度最快，每个核上都有一个L1高速缓存。
- L2高速缓存容量更大（如256KB）、速度低些，在一般情况下，每个内核上都有一个独立的L2高速缓存。
- L3高速缓存最接近主存，容量最大（如12MB）、速度最低，由在同一个CPU芯片板上的不同CPU内核共享。

> Cache读操作流程图：图片来自于计算机组成原理唐朔飞版
>
> ![image-20220424143143239](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220424143143239.png)

## 4.2 并发编程的三大问题

由于需要尽可能释放CPU的能力，因此在CPU上不断增加内核和缓存。内核是越加越多，从之前的单核演变成8核、32核甚至更多。缓存也不止一层，可能是2层、3层甚至更多。随着CPU内核和缓存的增加，导致了并发编程的可见性和有序性问题。我们这里学习一下三大问题。

### 4.2.1 原子性问题

所谓原子操作，就是“不可中断的一个或一系列操作”，是指不会被线程调度机制打断的操作。这种操作一旦开始，就一直运行到结束，中间不会有任何线程的切换。

我们举个例子：

```java
public class TestPlusPlus {
    int sum = 0;
    public void increase(){
        sum++;
    }
}
```

我们用javap工具查看该类的反编译

![image-20220424143930948](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220424143930948.png)

从图中我们可以看到一个`sum++`操作主要有四步：

首先`getfield`表示获取当前sum变量的值，并且放入栈顶。然后`iconst_1`表示将常量1放入栈顶。再然后`iadd`表示将当前栈顶的两个值相加，并且将结果放入栈顶。最后`putfield`表示将栈顶的结果再赋值给sum变量。

通过以上4个关键性的汇编指令可以看出，在汇编代码的层面，++操作实质上是4个操作。这4个操作之间是可以发生线程切换的，或者说是可以被其他线程中断的。所以，++操作不是原子操作，在并行场景会发生原子性问题。

### 4.2.2 可见性问题

**一个线程对共享变量的修改，另一个线程能够立刻可见，我们称该共享变量具备内存可见性。**

谈到内存可见性，要先引出JMM（Java Memory Model，Java内存模型）的概念。JMM规定，将所有的变量都存放在公共主存中，当线程使用变量时会把主存中的变量复制到自己的工作空间（或者叫私有内存）中，线程对变量的读写操作，是自己工作内存中的变量副本。

如果两个线程同时操作一个共享变量，就可能发生可见性问题。举一个例子：

1. 主存中有变量sum，初始值为0。
2. 线程A计划将sum加1，先将sum=0复制到自己的私有内存中，然后更新sum的值。线程A操作完成之后其私有内存中sum的值为1，然而线程A将更新后的sum值回刷到主存的时间是不固定的。
3. 在线程A没有回刷sum到主存前，刚好线程B同样从主存中读取sum，此时值为0，和线程A进行同样的操作，最后期盼的sum=2目标没有达成，最终sum=1。线程B没有将sum变成2的原因是：线程A的修改还在其工作内存中，对线程B不可见，因为线程A的修改还没有刷入主存。这就发生了典型的内存可见性问题。

要想解决多线程的内存可见性问题，所有线程都必须将共享变量刷新到主存，一种简单的方案是：**使用Java提供的关键字volatile修饰共享变量。**

> 为什么Java局部变量、方法参数不存在内存可见性问题？在Java中，所有的局部变量、方法定义参数都不会在线程之间共享，所以也就不会有内存可见性问题。所有的Object实例、Class实例和数组元素都存储在JVM堆内存中，堆内存在线程之间共享，所以存在可见性问题。

### 4.2.3 有序性问题

所谓程序的有序性，是指程序按照代码的先后顺序执行。如果程序执行的顺序与代码的先后顺序不同，并导致了错误的结果，即发生了有序性问题。

```java
public class TestYouXuXing {
    private static  int x = 0,y =0;
    private static int a= 0,b=0;
    public static void main(String[] args) throws InterruptedException {
        int i = 0;
        for (;;) {
            i++;
            x=0;
            y=0;
            a=0;
            b=0;
            Thread thread1 = new Thread(() -> {
                a=1;
                x=b;
            });
            Thread thread2 = new Thread(() -> {
                b=1;
                y=a;
            });
            thread1.start();
            thread2.start();
            thread1.join();
            thread2.join();
            if (x==0&&y==0){
                System.out.println("第"+i+"次x="+x+",y="+y);
            }
        }
    }
}
```

第418040次x=0,y=0
第1048954次x=0,y=0

> 以上结果是随机的，取决于个人的电脑CPU配置，可能要多等一会才会出结果，我等了很久重启了好几次主函数才看到这两个结果。

由于并发执行的无序性，赋值之后x、y的值可能为(1,0)、(0,1)或(1,1)。因为线程1可能在线程2开始之前就执行完了，也可能线程2在线程1开始之前就执行完了，甚至有可能二者的指令是同时或交替执行的。

然而，执行以上代码时会发现执行结果也可能是(0,0)。

为什么会出现(0,0)结果呢？可能在程序的执行过程中发生了指令重排序（Reordering）。下面解释一下什么是指令重排序。一般来说，CPU为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行顺序同代码中的先后顺序一致，但是它会保证程序最终的执行结果和代码顺序执行的结果是一致的。

得到了(0,0)结果的语句执行过程，对于线程1来说，可能a=1和x=b这两个语句的赋值操作顺序被颠倒了，对于线程2来说，可能b=1和y=a这两个语句的赋值操作顺序被颠倒了，从而出现了(x,y)值为(0,0)的错误结果。

> 事实上，输出了乱序的结果，并不代表一定发生了指令重排序，内存可见性问题也会导致这样的输出。但是，指令重排序也是导致乱序的原因之一。

## 4.3 硬件层的MESI原理

为了缓解内存速度和CPU内核速度差的问题，现代计算机会在CPU上增加高速缓存，每个CPU内核都只有自己的一级、二级高速缓存，CPU芯片板上的CPU内核之间共享一个三级高速缓存。每个CPU的处理过程为：先将计算需要用到的数据缓存在CPU的高速缓存中，在CPU进行计算时，直接从高速缓存中读取数据并且在计算完成之后写回高速缓存中。在整个运算过程完成后，再把高速缓存中的数据同步到主存。由于每个线程可能会运行在不同的CPU内核中，因此每个线程拥有自己的高速缓存。同一份数据可能会被缓存到多个CPU内核中，在不同CPU内核中运行的线程看到同一个变量的缓存值就会不一样，就可能发生内存的可见性问题。

**硬件层的MESI协议是一种用于解决内存的可见性问题的手段。**

### 4.3.1 总线锁和缓存锁

为了解决内存的可见性问题，CPU主要提供了两种解决办法：总线锁和缓存锁。

**总线锁**

操作系统提供了总线锁机制。前端总线（也叫CPU总线）是所有CPU与芯片组连接的主干道，负责CPU与外界所有部件的通信，包括高速缓存、内存、北桥，其控制总线向各个部件发送控制信号，通过地址总线发送地址信号指定其要访问的部件，通过数据总线实现双向传输。在CPU内核1要执行i++操作的时候，将在总线上发出一个LOCK#信号锁住缓存（具体来说是变量所在的缓存行），这样其他CPU内核就不能操作缓存了，从而阻塞其他CPU内核，使CPU内核1可以独享此共享内存。

在多CPU的系统中，当其中一个CPU要对共享主存进行操作时，在总线上发出一个LOCK#信号，这个信号使得其他CPU无法通过总线来访问共享主存中的数据，总线锁把CPU和主存之间的通信锁住了，这使得锁定期间，其他CPU不能操作其他主存地址的数据，总线锁的开销比较大，这种机制显然是不合适的。

总线锁的粒度太大了，最好的方法就是控制锁的保护粒度，只需要保证被多个CPU缓存的同一份数据一致即可。所以引入了缓存锁（如缓存一致性机制），后来的CPU都提供了缓存一致性机制，Intel486之后的处理器就提供了这种优化。

**缓存锁**

相比总线锁，缓存锁降低了锁的粒度。为了达到数据访问的一致，需要各个CPU在访问高速缓存时遵循一些协议，在存取数据时根据协议来操作，常见的协议有MSI、MESI、MOSI等。最常见的就是MESI协议。

就整体而言，缓存一致性机制就是当某CPU对高速缓存中的数据进行操作之后，通知其他CPU放弃存储在它们内部的缓存数据，或者从主存中重新读取。

在多CPU的系统中，为了保证各个CPU的高速缓存中数据的一致性，会实现缓存一致性协议，每个CPU通过嗅探在总线上传播的数据来检查自己的高速缓存中的值是否过期，当CPU发现自己缓存行对应的主存地址被修改时，就会将当前CPU的缓存行设置成无效状态，当CPU对这个数据执行修改操作时，会重新从系统主存中把数据读到CPU的高速缓存中。

主要的缓存一致性协议有MSI协议、MESI协议等。

> 关于MESI协议，这里不过多赘述。想了解的更详细可以去自己百度谷歌，不学硬件层的话了解一下就行。
>
> 我在学习的时候就是简单学习了MESI这四个状态，并且学习了它们之间的状态切换和工作原理，方便自己理解。

### 4.3.2 volatile的原理

为了解决CPU访问主存时主存读写性能的短板，在CPU中增加了高速缓存，但这带来了可见性问题。而Java的volatile关键字可以保证共享变量的主存可见性，也就是将共享变量的改动值立即刷新回主存。在正常情况下，系统操作并不会校验共享变量的缓存一致性，只有当共享变量用volatile关键字修饰了，该变量所在的缓存行才被要求进行缓存一致性的校验。

举个例子：

```java
public class TestVolatile {
    volatile int value = 0;

    public void setValue(int value){
        System.out.println("value="+value);
        this.value =value;
    }

    public static void main(String[] args) {
        TestVolatile test =new TestVolatile();
        test.setValue(1);
    }
}
```

将上面代码使用下面的命令将VolatileVar的汇编代码输出到001.log文件

~~~bash
java -server -Xcomp -XX:-Inline -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly TestVolatile 001.log
~~~

> 使用-XX:+PrintAssembly需要hsdis-amd64.dll并且需要将其安装到jre的`jdk安装目录\jre\bin\server`目录下，且这个包的在外网的地址打不开，别人分享的下载包倒是有但是输出过程实在是太长了。
>
> 且汇编指令特别长找变量属实费劲，我这里就不折磨自己了，感兴趣可以自己输出试一试。所以这里就直接通过原书进行了解，简单来说就是这里没有自己手动尝试~(￣▽￣)~*
>

![image-20220424161653621](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220424161653621.png)

该lock前缀指令有三个功能

（1）将当前CPU缓存行的数据立即写回系统内存在对volatile修饰的共享变量进行写操作时，其汇编指令前用lock前缀修饰。lock前缀指令使得在执行指令期间，CPU可以独占共享内存（即主存）。对共享内存的独占，老的CPU（如Intel 486）通过总线锁方式实现。由于总线锁开销比较大，因此新版CPU（如IA-32、Intel 64）通过缓存锁实现对共享内存的独占性访问，缓存锁（缓存一致性协议）会阻止两个CPU同时修改共享内存的数据。

（2）lock前缀指令会引起在其他CPU中缓存了该内存地址的数据无效写回操作时要经过总线传播数据，而每个CPU通过嗅探在总线上传播的数据来检查自己缓存的值是否过期，当CPU发现自己缓存行对应的内存地址被修改时，就会将当前CPU的缓存行设置为无效状态，当CPU要对这个值进行修改的时候，会强制重新从系统内存中把数据读到CPU缓存。

（3）lock前缀指令禁止指令重排lock前缀指令的最后一个作用是作为内存屏障（Memory Barrier）使用，可以禁止指令重排序，从而避免多线程环境下程序出现乱序执行的现象。

其实也就是说，volatile是Java有序性和可见性的解决方案。

## 4.4 有序性和内存屏障

有序性是与可见性完全不同的概念，虽然二者都是CPU不断迭代升级的产物。由于CPU技术不断发展，为了重复释放硬件的高性能，编译器、CPU会优化待执行的指令序列，包括调整某些指令的顺序执行。优化的结果，指令执行顺序会与代码顺序略有不同，可能会导致代码执行出现有序性问题。

内存屏障又称内存栅栏（Memory Fences），是一系列的CPU指令，它的作用主要是保证特定操作的执行顺序，保障并发执行的有序性。在编译器和CPU都进行指令的重排优化时，可以通过在指令间插入一个内存屏障指令，告诉编译器和CPU，禁止在内存屏障指令前（或后）执行指令重排序。

### 4.4.1 重排序

为了提高性能，编译器和CPU常常会对指令进行重排序。重排序主要分为两类：编译器重排序和CPU重排序，

<img src="https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/ReSort20220425.png" alt="ReSort20220425" style="zoom: 67%;" />

**编译器重排序**指的是在代码编译阶段进行指令重排，不改变程序执行结果的情况下，为了提升效率，编译器对指令进行乱序（Out-of-Order）的编译。举个栗子：在代码中，A操作需要获取其他资源而进入等待的状态，而A操作后面的代码跟A操作没有数据依赖关系，如果编译器一直等待A操作完成再往下执行的话，效率要慢得多，所以可以先编译后面的代码，这样的乱序可以提升编译速度。

编译器重排序目的是：与其等待阻塞指令（如等待缓存刷入）完成，不如先去执行其他指令。与CPU乱序执行相比，编译器重排序能够完成更大范围、效果更好的乱序优化。

**CPU重排序**：流水线（Pipeline）和乱序执行（Out-of-Order Execution）是现代CPU基本都具有的特性。机器指令在流水线中经历取指令、译码、执行、访存、写回等操作。为了CPU的执行效率，流水线都是并行处理的，在不影响语义的情况下。处理次序（Process Ordering，机器指令在CPU实际执行时的顺序）和程序次序（Program Ordering，程序代码的逻辑执行顺序）是允许不一致的，只要满足**As-if-Serial规则**即可。显然，这里的不影响语义依旧只能保证指令间的显式因果关系，无法保证隐式因果关系，即无法保证语义上不相关但是在程序逻辑上相关的操作序列按序执行。

> 所谓“乱序”，仅仅是被称为“乱序”，实际上也遵循着一定规则：只要两个指令之间不存在“数据依赖”，就可以对这两个指令乱序。
>
> CPU重排序包括两类：指令级重排序和内存系统重排序。
>
> - 指令级重排序。在不影响程序执行结果的情况下，CPU内核采用ILP（Instruction-LevelParallelism，指令级并行运算）技术来将多条指令重叠执行，主要是为了提升效率。如果指令之间不存在数据依赖性，CPU就可以改变语句的对应机器指令的执行顺序，叫作指令级重排序。
> - 内存系统重排序：对于现代的CPU来说，在CPU内核和主存之间都具备一个高速缓存，高速缓存的作用主要是减少CPU内核和主存的交互，在CPU内核进行读操作时，如果缓存没有的话就从主存取，而对于写操作都是先写在缓存中，最后再一次性写入主存，原因是减少跟主存交互时CPU内核的短暂卡顿，从而提升性能。但是，内存系统重排序可能会导致一个问题——数据不一致。内存系统重排序和指令级重排序不同，内存系统重排序为伪重排序，也就是说只是看起来像在乱序执行而已。

### 4.4.2 As-if-Serial规则

在单核CPU的场景下，当指令被重排序之后，如何保障运行的正确性呢？

其实很简单，编译器和CPU都需要遵守As-if-Serial规则。As-if-Serial规则的具体内容为：无论如何重排序，都必须保证代码在单线程下运行正确。

为了遵守As-if-Serial规则，编译器和CPU不会对存在数据依赖关系的操作进行重排序，因为这种重排序会改变执行结果。但是，如果指令之间不存在数据依赖关系，这些指令可能被编译器和CPU重排序。举个栗子：

~~~java
public static void main(String[] args){
	int a = 1;//1
	int b = 1;//2
    int c = a + b;//3
}
~~~

在上面代码中1和3有数据依赖关系，2和3也有数据依赖关系，但是1和2没有数据依赖关系，所以编译器和CPU可以重排序1和2之间的执行顺序。

为了保证As-if-Serial规则，Java异常处理机制也会为指令重排序做一些特殊处理，举个栗子：

~~~java
public static void main(String[] args){
	int x,y;
        x=1;
        try {
            x=2;  //1
            y=0/0; //2
        }catch (Exception e){ //3

        }finally {
            System.out.println(x);
        }
}
~~~

在上面的语句中，语句1和2之间没有依赖关系，所以语句2可能会重排序到1之前，这时2抛出异常，被捕获，会导致1没有执行，会出现错误。为了保证最终不会出现x=1的错误，JIT在重排序时会在catch语句中插入错误补偿代码，补偿执行语句1。

这种做法会将异常捕获和处理底层逻辑变得很复杂，但是JIT的优化原则就是，尽力保障正确的运行逻辑，哪怕以catch块逻辑变得复杂为代价。

> JIT是Just In Time的缩写，也就是“即时编译器”。JVM读入“.class”文件的字节码后，默认情况下是解释执行的。但是对于运行频率很高（如大于5000次）的字节码，JVM采用了JIT技术，将直接编译为机器指令，以提高性能。

As-if-Serial规则只能保障单内核指令重排序之后的执行结果正确，不能保障多内核以及跨CPU指令重排序之后的执行结果正确。

### 4.4.3 硬件层面的内存屏障

多核情况下，所有的CPU操作都会涉及缓存一致性协议（MESI协议）校验，该协议用于保障内存可见性。但是，缓存一致性协议仅仅保障内存弱可见（高速缓存失效），没有保障共享变量的强可见，而且缓存一致性协议更不能禁止CPU重排序，也就是不能确保跨CPU指令的有序执行。

那么如何保证跨CPU指令重排序之后的程序结果正确呢？需要用到内存屏障。

**内存屏障又称内存栅栏，是让一个CPU高速缓存的内存状态对其他CPU内核可见的一项技术，也是一项保障跨CPU内核有序执行指令的技术。**

硬件层常用的内存屏障分为三种：读屏障（Load Barrier）、写屏障（Store Barrier）和全屏障（Full Barrier）。

- 读屏障

  读屏障让高速缓存中相应的数据失效。在指令前插入读屏障，可以让高速缓存中的数据失效，强制重新从主存加载数据。并且，读屏障会告诉CPU和编译器，先于这个屏障的指令必须先执行。

  读屏障对应着X86处理器上的lfence指令，将强制所有在该指令之后的读操作都在lfence指令执行之后被执行，并且强制本地高速缓冲区的值全部失效，以便从主存中重新读取共享变量的值。

  读屏障既使得当前CPU内核对共享变量的更改对所有CPU内核可见，又阻止了一些可能导致读取无效数据的指令重排。

- 写屏障

  在指令后插入写屏障指令能让高速缓存中的最新数据更新到主存，让其他线程可见。并且，写屏障会告诉CPU和编译器，后于这个屏障的指令必须后执行。

  写屏障对应X86处理器上的sfence指令，sfence指令会保证所有写操作都在该指令执行之前被完成，并把高速缓冲区的数据都刷新到主存中，使得当前CPU对共享变量的更改对所有CPU可见。

- 全屏障

  全屏障是一种全能型的屏障，具备读屏障和写屏障的能力。Full Barrier又称为StoreLoad Barriers，对应X86处理器上的mfence指令。

  在X86处理器平台上mfence指令综合了sfence指令与lfence指令的作用。X86处理器强制所有在mfence之前的store/load指令都在mfence执行之前被执行；所有在mfence之后的store/load指令都在该mfence执行之后被执行。简单来说，X86处理器禁止对mfence指令前后的store/load指令进行重排序。

那么内存屏障有什么作用呢？

1. 阻止屏障两侧的指令重排序。编译器和CPU可能为了使性能得到优化而对指令重排序，但是插入一个硬件层的内存屏障相当于告诉CPU和编译器先于这个屏障的指令必须先执行，后于这个屏障的指令必须后执行。
2. 强制让高速缓存的数据失效。硬件层的内存屏障强制把高速缓存中的最新数据写回主存，让高速缓存中相应的脏数据失效。一旦完成写入，任何访问这个变量的线程将会得到最新的值。

下面举个例子：

```java
public class TestVolatile {
    private int x = 0;
    private Boolean flag = false;

    public void update() {
        x = 8;//1
        flag = true;//2
    }

    public void show() {
        if (flag) {//3
            System.out.println(x);
        }
    }
}
```

TestVolatile并发运行之后，控制台所输出的x值可能是0或8。为什么x可能会输出0呢？update()和show()方法可能在两个CPU内核并发执行，语句1和语句2如果发生了重排序，那么show()方法输出的x就可能为0。如果输出的x结果是0，显然不是程序的正常结果。

我们现在使用含有JMM内存屏障语义的Java关键字，这类关键字的典型为volatile。

~~~java
public class TestVolatile {
    private volatile int x = 0;
    private Boolean flag = false;

    public void update() {
        x = 8;//1
        //要求编译器在这里插入Store Barrier写屏障
        flag = true;//2
    }

    public void show() {
        if (flag) {//3
            System.out.println(x);
        }
    }
}
~~~

修改后的TestVolatile代码使用volatile关键字对成员变量x进行修饰，volatile含有JMM全屏障的语义，要求JVM编译器在语句1的后面插入全屏障指令。该全屏障确保x的最新值对所有的后序操作是可见的（含跨CPU场景），并且禁止编译器和处理器对语句1和语句2进行重排序。

volatile在X86处理器上被JVM编译之后，它的汇编代码中会被插入一条lock前缀指令（lock ADD），从而实现全屏障目的。

> 由于不同的物理CPU硬件所提供的内存屏障指令的差异非常大，因此JMM定义了自己的一套相对独立的内存屏障指令，用于屏蔽不同硬件的差异性。很多Java关键字（如volatile）在语义中包含JMM内存屏障指令，在不同的硬件平台上，这些JMM内存屏障指令会要求JVM为不同的平台生成相应的硬件层的内存屏障指令。

## 4.5 JMM

### 4.5.1 Java内存模型

JMM最初由JSR-133（Java Memory Model and Thread Specification）文档描述，JMM定义了一组规则或规范，该规范定义了一个线程对共享变量写入时，如何确保对另一个线程是可见的。

实际上，JMM提供了合理的禁用缓存以及禁止重排序的方法，所以其核心的价值在于解决可见性和有序性。

JMM的另一大价值在于能屏蔽各种硬件和操作系统的访问差异，保证Java程序在各种平台下对内存的访问最终都是一致的。

Java内存模型规定所有的变量都存储在主存中，JMM的主存类似于物理内存，但有区别，还能包含部分共享缓存。每个Java线程都有自己的工作内存（类似于CPU高速缓存，但也有区别）。

Java内存模型定义的两个概念：

1. 主存：主要存储的是Java实例对象，所有线程创建的实例对象都存放在主存中，无论该实例对象是成员变量还是方法中的本地变量（也称局部变量），当然也包括共享的类信息、常量、静态变量。由于是共享数据区域，因此多条线程对同一个变量进行访问可能会发现线程安全问题。
2. 工作内存：主要存储当前方法的所有本地变量信息（工作内存中存储着主存中的变量副本），每个线程只能访问自己的工作内存，即线程中的本地变量对其他线程是不可见的，即使两个线程执行的是同一段代码，它们也会各自在自己的工作内存中创建属于当前线程的本地变量，当然也包括字节码行号指示器、相关Native方法的信息。注意，由于工作内存是每个线程的私有数据，线程间无法相互访问工作内存，因此存储在工作内存的数据不存在线程安全问题。

JMM将所有的变量都存放在公共主存中，当线程使用变量时，会把公共主存中的变量复制到自己的工作内存（或者叫作私有内存）中，线程对变量的读写操作是自己的工作内存中的变量副本。因此，JMM模型也需要解决代码重排序和缓存可见性问题。JMM提供了一套自己的方案去禁用缓存以及禁止重排序来解决这些可见性和有序性问题。JMM提供的方案包括大家都很熟悉的volatile、synchronized、final等。JMM定义了一些内存操作的抽象指令集，然后将这些抽象指令包含到Java的volatile、synchronized等关键字的语义中，并要求JVM在实现这些关键字时必须具备其包含的JMM抽象指令的能力。

Java线程、工作内存、主存的关系如下图所示：

![Java主存工作内存关系20220425](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Java%E4%B8%BB%E5%AD%98%E5%B7%A5%E4%BD%9C%E5%86%85%E5%AD%98%E5%85%B3%E7%B3%BB20220425.png)

### 4.5.2 JMM和JVM物理内存的关系

JMM（Java内存模型）看上去和JVM（Java内存结构）差不多，很多人会误以为两者是一回事，这也就导致面试过程中经常答非所问。JMM属于语言级别的内存模型，它确保了在不同的编译器和不同的CPU平台上为Java程序员提供一致的内存可见性保证和指令并发执行的有序性。

> 以Java为例，一个i++方法编译成字节码后，在JVM中是分成以下三个步骤运行的：
>
> 1. 从主存中复制i的值并复制到CPU的工作内存中。
> 2. CPU取工作内存中的值，然后执行i++操作，完成后刷新到工作内存。
> 3. 将工作内存中的值更新到主存。当多个线程同时访问该共享变量i时，每个线程都会将变量i复制到工作内存中进行修改，如果线程A读取变量i的值时，线程B正在修改i的值，问题就来了：线程B对变量i的修改对线程A而言就是不可见的。

多线程并发访问共享变量所造成的结果不一致问题，该问题属于JMM需要解决的问题。

JMM属于概念和规范维度的模型，是一个参考性质的模型。JVM模型定义了一个指令集、一个虚拟计算机架构和一个执行模型。具体的JVM实现需要遵循JVM的模型，它能够运行根据JVM模型指令集编写的代码，就像真机可以运行机器代码一样。虽然JVM也是一个概念和规范维度的模型，但是大家常常将JVM理解为实体的、实现维度的虚拟机，通常是指HotSpot VM。

> 通过对硬件缓存架构、Java内存模型以及Java多线程原理的了解，大家应该已经意识到，多线程的执行最终都会映射到CPU上执行，但是Java内存模型和硬件内存架构并不完全一致。
>
> **JMM与硬件内存架构是什么样的关系呢？**
>
> 对于硬件内存来说只有寄存器、缓存内存、主存的概念，并没有工作内存（线程私有数据区域）和主存（堆内存）之分。也就是说Java内存模型对内存的划分对硬件内存并没有任何影响，因为JMM只是一种抽象的概念，是一组规则，并不实际存在，无论是工作内存的数据还是主存的数据，对于计算机硬件来说都会存储在计算机主存中，当然也有可能存储到CPU高速缓存或者寄存器中，因此总体上来说，Java内存模型和计算机硬件内存架构是相互交叉的关系，是一种抽象概念划分与真实物理硬件的交叉。如下图（图片来自于原书Java高并发编程第二卷）：
>
> ![image-20220425152158594](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220425152158594.png)

### 4.5.3 JMM的八个操作

JMM定义了一套自己的主存与工作内存之间的交互协议，即一个变量如何从主存拷贝到工作内存，又如何从工作内存写入主存，该协议包含8种操作，并且要求JVM具体实现必须保证其中每一种操作都是原子的、不可再分的。

| 操作   | 作用对象 | 说明                                                         |
| ------ | -------- | ------------------------------------------------------------ |
| Read   | 主存     | 作用于主存变量。Read操作把一个变量的值从主存传输到工作内存中，以便随后的Load操作使用 |
| Load   | 工作内存 | Load操作把Read操作从主存中得到的值载入工作内存的变量副本中。变量副本可以简单理解为CPU的高速缓存 |
| Use    | 工作内存 | Use操作把工作内存中的一个变量的值传递给执行引擎。每当JVM遇到一个需要使用变量值的字节码指令时，执行Use操作 |
| Assign | 工作内存 | 执行引擎通过Assign操作给工作内存的变量赋值。每当JVM遇到一个给变量赋值的字节码指令时，执行Assign操作。 |
| Store  | 工作内存 | Store把工作内存的一个变量的值传递到主存中，以便随后的Write操作使用 |
| Write  | 主存     | Write操作把Store操作从工作内存得到的变量值放入主存的变量中   |
| Lock   | 主存     | 作用于主存的变量，把一个变量标识为某个线程独占状态           |
| Unlock | 主存     | 作用于主存的变量，把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定 |

如果要把一个变量从主存复制到工作内存，就要按顺序执行Read和Load操作；如果要把变量从工作内存同步回主存，就要按顺序执行Store和Write操作。JMM要求Read和Load、Store和Write必须按顺序执行，但不要求连续执行。也就是说，Read和Load之间、Store和Write之间可插入其他指令。

JMM主存与工作内存之间的交互协议的8个操作之间的关系如图所示，图片来自于原书Java高并发编程第二卷：

![image-20220425154128109](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220425154128109.png)

Java内存模型还规定了执行上述8种基本操作时必须满足如下规则：

1. 不允许read和load、store和write操作之一单独出现，以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。

   不允许read和load、store和write操作之一单独出现，意味着有read就有load，不能读取了变量值而不予加载到工作内存中；

   有store就有write，也不能存储了变量值而不写到主存中。

2. 不允许一个线程丢弃它最近的assign操作，也就是说当线程使用assign操作对私有内存的变量副本进行变更时，它必须使用write操作将其同步到主存中。

3. 不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主存中。

4. 一个新的变量只能从主存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说，就是对一个变量实施use和store操作之前，必须先执行assign和load操作。

5. 一个变量在同一个时刻只允许一个线程对其执行lock操作，但lock操作可以被同一个个线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。

6. 如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。

7. 如果一个变量实现没有被lock操作锁定，就不允许对它执行unlock操作，也不允许unlock一个被其他线程锁定的变量。

8. 对一个变量执行unlock操作之前，必须先把此变量同步回主存（执行store和write操作）。

以上JMM的8大操作规范定义相当严谨，也极为烦琐，JVM实现起来也非常复杂。Java设计团队大概也意识到了这个问题，新的JMM版本不断地对这些操作进行简化，比如将8个操作简化为Read、Write、Lock和Unlock四个操作。虽然进行了简化，但是JMM的基础设计并未改变。

> *JMM的规范细节是JVM开发人员需要掌握的内容，对于普通的Java应用工程师、应用架构师来说，只需要了解其基本的原理即可。*

### 4.5.4 JMM如何解决有序性问题

JMM如何解决顺序一致性问题？**JMM提供了自己的内存屏障指令，要求JVM编译器实现这些指令，禁止特定类型的编译器和CPU重排序（不是所有的编译器重排序都要禁止）。**

JMM内存屏障：由于不同CPU硬件实现内存屏障的方式不同，JMM屏蔽了这种底层CPU硬件平台的差异，定义了不对应任何CPU的JMM逻辑层内存屏障，由JVM在不同的硬件平台生成对应的内存屏障机器码。JMM内存屏障主要有Load和Store两类，具体如下：

- Load Barrier（读屏障）在读指令前插入读屏障，可以让高速缓存中的数据失效，重新从主存加载数据。
- Store Barrier（写屏障）在写指令之后插入写屏障，能让写入缓存的最新数据写回主存。

在实际使用时，会对以上JMM的Load Barrier和Store Barrier两类屏障进行组合，组合成LoadLoad（LL）、StoreStore（SS）、LoadStore（LS）、StoreLoad（SL）四个屏障，用于禁止特定类型的CPU重排序。

1. LL `Load1 LoadLoad Load2`在Load2要读取的数据被访问前，使用LoadLoad屏障保证Load1要读取的数据被读取完毕。

2. SS `Store1 StoreStore Store2`在Store2及后续写入操作执行前，使StoreStore屏障保证Store1的写入结果对其他CPU可见。

3. LS `Load1 LoadStore Store2`在Store2及后续写入操作执行前，使LoadStore屏障保证Load1要读取的数据被读取完毕。

4. SL `Store1 StoreLoad Load2`在Load2及后续所有读取操作执行前，使StoreLoad屏障保证Store1的写入对所有CPU可见。

   > StoreLoad（SL）屏障的开销是4种屏障中最大的，但是此屏障是一个“全能型”的屏障，兼具其他3个屏障的效果，现代的多核CPU大多支持该屏障。
   >
   > 目前的主要处理器对JMM四个内存屏障的支持感兴趣的可以自行百度

### 4.5.5 volatile中的内存屏障

在Java代码中，volatile关键字主要有两层语义：

- 不同线程对volatile变量的值具有内存可见性，即一个线程修改了某个volatile变量的值，该值对其他线程立即可见。
- 禁止进行指令重排序。

volatile语义中的有序性是通过内存屏障指令来确保的。为了实现volatile关键字语义的有序性，JVM编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。JMM建议JVM采取保守策略对重排序进行严格禁止。下面是基于保守策略的volatile操作的内存屏障插入策略。

- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。
- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。

volatile写操作的内存屏障插入策略为：在每个volatile写操作前插入StoreStore（SS）屏障，在写操作后面插入StoreLoad屏障。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/SSSL20220425.png)

volatile读操作的内存屏障插入策略为：在每个volatile读操作后插入LoadLoad（LL）屏障和LoadStore屏障，禁止后面的普通读、普通写和前面的volatile读操作发生重排序，

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/LLLS120220425.png)

> 上述JMM建议的volatile写和volatile读的内存屏障插入策略是针对任意处理器平台的，所以非常保守。不同的处理器有不同“松紧度”的处理器内存模型，只要不改变volatile读写操作的内存语义，不同JVM编译器可以根据具体情况省略不必要的JMM屏障。以X86处理器为例，该平台的JVM实现仅仅在volatile写操作后面插入一个StoreLoad屏障，其他的JMM屏障都会被省略。由于StoreLoad屏障的开销大，因此在X86处理器中，volatile写操作比volatile读操作的开销会大很多。

## 4.6 Happens-Before规则

JMM的内存屏障指令对Java工程师是透明的，是JMM对JVM实现的一种规范和要求。那么，作为Java工程师，如何确保自己设计和开发的Java代码不存在内存可见性问题或者有序性问题？

JMM定义了一套自己的规则：Happens-Before（先行发生）规则，并且确保只要两个Java语句之间必须存在Happens-Before关系，JMM尽量确保这两个Java语句之间的内存可见性和指令有序性。



Happens-Before规则的主要内容包括以下几个方面：

1. 程序顺序执行规则（as-if-serial规则）在同一个线程中，有依赖关系的操作按照先后顺序，前一个操作必须先行发生于后一个操作（Happens-Before）。换句话说，单个线程中的代码顺序无论怎么重排序，对于结果来说是不变的。
2. volatile变量规则对volatile（修饰的）变量的写操作必须先行发生于对volatile变量的读操作。
3. 传递性规则如果A操作先于B操作，而B操作又先行发生于C操作，那么A操作先行发生于C操作。
4. 监视锁规则（Monitor Lock Rule）对一个监视锁的解锁操作先行发生于后续对这个监视锁的加锁操作。
5. start规则对线程的start操作先行于这个线程内部的其他任何操作。具体来说，如果线程A执行B.start()启动线程B，那么线程A的B.start()操作先行发生于线程B中的任意操作。
6. join规则如果线程A执行了B.join()操作并成功返回，那么线程B中的任意操作先行发生于线程A所执行的ThreadB.join()操作。

### 4.6.1 顺序执行规则

一个线程内，按照代码顺序，书写在前面的操作先行发生（Happens-Before）于书写在后面的操作。一段程序的执行，在单个线程中看起来是有序的。程序次序规则看起来是按顺序执行的，因为虚拟机可能会对程序指令进行重排序。虽然进行了重排序，但是最终执行的结果与程序顺序执行的结果是一致的。它只会对不存在数据依赖行的指令进行重排序。

*该规则就是前面介绍的As-if-Serial规则，仅仅用来保证程序在单线程执行结果的正确性，但是无法保证程序在多线程执行结果的正确性。*

### 4.6.2 volatile规则

volatile规则的具体内容：对一个volatile变量的写先行发生（Happens-Before）于任意后续对这个volatile变量的读。

概念的意思就是下图（图片来自于Java高并发编程）内容

![image-20220425162320009](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220425162320009.png)

举个栗子：

```java
public class TestVolatile {
    int x =10;
    int sum =0;
    boolean flag = false;
    public void update(){
        x=100;//1
        flag =true;//2
    }
    public void plus(){
        if (flag){//3
            sum = x+x;//4
        }
    }
}
```

假设线程A执行update()方法，线程B执行plus()方法，因为代码1和2没有数据依赖关系，所以1和2可能被重排序，它们在重排序后的次序为：先执行2在执行1.

线程A执行重排之后的代码，在完成语句2但没有开始语句1时，假设线程B开始执行plus()方法，将两个x（此时值仍然为10）累加，得到的sum为20。

为了避免这种错误的情况，为以上代码的flag成员属性增加volatile修饰。

这时如果第二个操作为volatile写，无论第一个操作是什么都不能重排序。拿上面的代码来说，由于代码2为写入flag（volatile变量）操作，因此代码1不会被重排序到代码2的后面。

从前面的规则可以知道：如果第一个操作为volatile读，无论第二个操作是什么都不能重排序。拿上面的代码来说，由于代码3为读取flag（volatile变量），因此代码4不会被重排序到代码3之前。

### 4.6.3 传递性规则

如果A操作先行发生于B操作，且B操作先行发生于C操作，那么A操作先行发生于C操作。

在上一节的例子中代码1处`x=100`先行发生于代码2处`flag=true`，这是规则1。

写变量`flag=true`先行发生于`if（flag）`读变量，这是规则2。

所以，根据规则3（传递性规则），`x=100`先行发生于读变量`if（flag）`。

### 4.6.4 监视锁规则

对一个锁的unlock操作先行发生于后面对同一个锁的lock操作，即无论在单线程还是多线程中，同一个锁如果处于被锁定状态，那么必须先对锁进行释放操作，后面才能继续执行lock操作。

~~~java
 public synchronized void update(){
     x=100;//1
     flag =true;//2
 }
public synchronized void plus(){
    if (flag){//3
        sum = x+x;//4
    }
}
~~~

比如这段代码，先获取锁的线程，对x赋值之后释放锁，另一个再获取锁，一定能看到对x赋值的改动。

> 监视锁规则不会对临界区内的代码进行约束，临界区内的代码可以重排序（但JMM不允许临界区内的代码“逸出”到临界区之外，那样会破坏监视器的语义）。JMM会在退出临界区和进入临界区这两个关键时间点做一些特别处理，虽然线程A在临界区内进行了重排序，但由于监视器互斥执行的特性，这里的线程B根本无法“观察”到线程A在临界区内的重排序。

### 4.6.5 start()规则

如果线程A执行ThreadB.start()操作启动线程B，那么线程A的ThreadB.start()操作先行发生于线程B中的任意操作。反过来说，**如果主线程A启动子线程B后，线程B能看到线程A在启动操作前的任何操作。**比如启动前对变量进行赋值，那么B启动后一定能看到这个赋值后的变量。

### 4.6.6 join()规则

如果线程A执行threadB.join()操作并成功返回，那么线程B中的任意操作先行发生于线程A的ThreadB.join()操作。

join()规则和start()规则刚好相反，线程A等待子线程B完成后，当前线程B的赋值操作，线程A都能够看到。

## 4.7 volatile不具备原子性

对于非volatile修饰的普通变量而言，在读取变量时，JMM要求保持read、load有相对顺序即可。例如，若从主存读取i、j两个变量，可能的操作是read i=>read j=>load j=>load i，并不要求read、load操作是连续的。

对于关键字volatile修饰的内存可见变量而言，具有两个重要的语义：

1. 使用volatile修饰的变量在变量值发生改变时，会立刻同步到主存，并使其他线程的变量副本失效。
2. 禁止指令重排序：用volatile修饰的变量在硬件层面上会通过在指令前后加入内存屏障来实现，编译器级别是通过下面的规则实现的。

为了实现这些volatile内存语义，JMM对于volatile变量会有特殊的约束：

1. 使用volatile修饰的变量其read、load、use都是连续出现的，所以每次使用变量的时候都要从主存读取最新的变量值，替换私有内存的变量副本值（如果不同的话）。
2. 其对同一变量的assign、store、write操作都是连续出现的，所以每次对变量的改变都会立马同步到主存中。

举个例子：现在线程A将value变成1之后，完成了assign、store的操作，假设在执行write指令之前，线程A的CPU时间片用完，线程A被空闲，但是线程A的write操作没有到达主存。由于线程A的store指令触发了写的信号，线程B缓存过期，重新从主存读取到value值，但是线程A的写入没有最终完成，线程B读到的value值还是0。线程B执行完成所有的操作之后，将value变成1写入主存。线程A的时间片重新拿到，重新执行store操作，将过期了的1写入主存。

> volatile变量无法保障其原子性，如果要保证复合操作的原子性，就需要使用锁。并且，在高并发场景下，volatile变量一定需要使用Java的显式锁结合使用。

