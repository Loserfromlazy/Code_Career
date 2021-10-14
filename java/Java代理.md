# java代理

# 一、代理模式

代理模式给某一个对象提供一个**代理对象**，并由代理对象控制对原对象的引用。同时代理对象可以调用被代理对象的方法，并对其进行增强。可以总结为**代理对象 = 增强代码 + 目标对象（原对象）**

比如张三去买房子，大致的步骤是找房子，商量价钱，然后交钱办手续，但是张三每天要上班没有那么多时间和精力，所以他找了一个中介，代理他去办这件事，这个中介不仅做了上面的那些事，而且

# 二、Java静态代理

静态代理的步骤：

- 定义接口
  - 代理和被代理类都需要实现此接口
- 定义被代理类
  - 被代理类需要实现接口
- 定义代理类
  - 代理类需要实现接口
  - 创建被代理对象，调用方法时调用被代理对象的方法，同时可以对方法进行增加增强

实现上面的例子：

```java
public interface BuyHouse {
    //找房子
    void findHouse();
    //商议价格
    void discussPrice();
    //交钱办房产证
    void payAndHaveHousePropertyCard();
}
```

```java
public class ZhangSan implements BuyHouse{
    @Override
    public void findHouse() {
        System.out.println("找房子");
    }

    @Override
    public void discussPrice() {
        System.out.println("商议价格");
    }

    @Override
    public void payAndHaveHousePropertyCard() {
        System.out.println("交钱办房产证");
    }
}
```

```java
public class IntermediaryProxy implements BuyHouse{
    public IntermediaryProxy(ZhangSan zhangSan) {
        this.zhangSan = zhangSan;
    }

    private ZhangSan zhangSan;

    @Override
    public void findHouse() {
        System.out.println("中介公司听取了张三的对房子的需求");
        zhangSan.findHouse();
        System.out.println("中介公司开始找房子");
    }

    @Override
    public void discussPrice() {
        zhangSan.discussPrice();
        System.out.println("中介公司协调价钱");
    }

    @Override
    public void payAndHaveHousePropertyCard() {
        zhangSan.payAndHaveHousePropertyCard();
        System.out.println("中介公司帮忙处理相关手续");
    }
}
```

测试类

```java
public class Test {
    public static void main(String[] args) {
        ZhangSan zhangSan = new ZhangSan();
        IntermediaryProxy proxy = new IntermediaryProxy(zhangSan);
        proxy.findHouse();
        proxy.discussPrice();
        proxy.payAndHaveHousePropertyCard();
    }
}
```

执行结果：

> 中介公司听取了张三的对房子的需求
> 找房子
> 中介公司开始找房子
> 商议价格
> 中介公司协调价钱
> 交钱办房产证
> 中介公司帮忙处理相关手续

如果一个类需要被代理，就得去创建一个代理类。如果被代理的类过多，这样就需要手动创建很多代理类。为了解决这个问题，便有了动态代理

# 三、Java动态代理

在了解动态代理之前需要了解什么是类加载和反射，这里是我的博客地址[Java类加载与反射](https://www.cnblogs.com/yhr520/p/12719529.html)

利用反射实现动态代理，继续实现上述例子。

在继续了解前首先复习一下对象的创建过程，如下图：

![createobject](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/createobject.png)

由上图可见，创建一个对象最主要的是得到对应的Class对象。Class对象包含了一个类的所有信息，而代理类和被代理类（也就是接口实现类）实现类同一个接口，接口拥有代理对象和目标对象共同的类信息。所以，我们可以从接口那得到理应由代理类提供的信息。

JDK提供了java.lang.reflect.InvocationHandler接口和 java.lang.reflect.Proxy类，这两个类相互配合，完成jdk的动态代理。

Proxy有个静态方法：getProxyClass(ClassLoader, interfaces)，只要你给它传入类加载器和一组接口，它就给你返回代理Class对象。根据代理Class的构造器创建对象时，需要传入InvocationHandler。每次调用代理对象的方法，最终都会调用InvocationHandler的invoke()方法。



以下是源码中关于Proxy的例子：

~~~
To create a proxy for some interface Foo:
       InvocationHandler handler = new MyInvocationHandler(...);
       Class<?> proxyClass = Proxy.getProxyClass(Foo.class.getClassLoader(), Foo.class);
       Foo f = (Foo) proxyClass.getConstructor(InvocationHandler.class).
                       newInstance(handler);
   
or more simply:
       Foo f = (Foo) Proxy.newProxyInstance(Foo.class.getClassLoader(),
                                            new Class<?>[] { Foo.class },
                                            handler);
~~~

**由源码给的示例Proxy类主要有两个静态方法**

- getProxyClass

  - 这个方法， 会从你传入的接口Class中，“拷贝”类结构信息到一个新的Class对象中，但**新的Class对象带有构造器，是可以创建对象的**

- newProxyInstance

  （一般直接用这个）

  - 封装了得到代理类Class对象、构造函数等细节，直接返回了代理对象

下面进行动态代理的改变，接口和被代理类不变

```java
public interface BuyHouse {
    //找房子
    void findHouse();
    //商议价格
    void discussPrice();
    //交钱办房产证
    void payAndHaveHousePropertyCard();
}
```

```java
public class ZhangSan implements BuyHouse{
    @Override
    public void findHouse() {
        System.out.println("找房子");
    }

    @Override
    public void discussPrice() {
        System.out.println("商议价格");
    }

    @Override
    public void payAndHaveHousePropertyCard() {
        System.out.println("交钱办房产证");
    }
}
```

**通过反射获得代理对象**

**自己封装获得代理对象**

**通过newProxyInstance()获得代理对象**







# 四、基于CGLib的动态代理