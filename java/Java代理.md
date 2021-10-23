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

在继续了解前首先复习一下对象的创建过程，如下图：（Loserfromlazy是我的水印）

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

![dongtaidaili](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/dongtaidaili.png)

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

```java
public class Test2 {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        //获取代理类的Class对象
        Class<?> proxyClass = Proxy.getProxyClass(BuyHouse.class.getClassLoader(), BuyHouse.class);
        //获得构造函数
        Constructor<?> constructor = proxyClass.getConstructor(InvocationHandler.class);
        //创建代理对象
        BuyHouse proxy = (BuyHouse) constructor.newInstance(new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                //创建被代理对象，用于调用方法
                ZhangSan zhangSan = new ZhangSan();
                //通过invoke方法调用被代理对象的方法
                Object invoke = method.invoke(zhangSan, args);
                //增强
                System.out.println("中介进行处理");
                return invoke;
            }
        });

        proxy.findHouse();
        proxy.discussPrice();
        proxy.payAndHaveHousePropertyCard();
        System.out.println("代理结束");

    }
}
```

运行结果

> 找房子
> 中介进行处理
> 商议价格
> 中介进行处理
> 交钱办房产证
> 中介进行处理
> 代理结束

我们可以对上面的方法进行封装

```java
public class Test2 {
    // 该方法返回一个代理对象
    public static Object getProxy(Object target) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        // target为被代理对象，得到其代理类的Class对象
        Class<?> proxyClazz = Proxy.getProxyClass(target.getClass().getClassLoader(), target.getClass().getInterfaces());

        // 获得构造函数
        Constructor<?> constructor = proxyClazz.getConstructor(InvocationHandler.class);

        // 获得代理对象
        Object targetProxy = constructor.newInstance(new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Object invoke = method.invoke(target, args);
                System.out.println("代理类增强方法");
                return invoke;
            }
        });
        return targetProxy;
    }
    
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        ZhangSan zhangSan = new ZhangSan();
        BuyHouse proxy = (BuyHouse) getProxy(zhangSan);
        proxy.findHouse();
        proxy.discussPrice();
        proxy.payAndHaveHousePropertyCard();
        System.out.println("代理结束");

    }
}
```

**通过newProxyInstance()获得代理对象**

上面的封装方法是我们自己写的，其实通过Proxy类的**newProxyInstance()**可以直接返回代理对象

~~~java
    public static Object getProxyByProxyMethod(Object target) {
        // 直接返回代理对象
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Object invoke = method.invoke(target, args);
                System.out.println("代理类增强方法");
                return invoke;
            }
        });
    }
~~~

我们自己获取代理对象的步骤是

- 通过**getProxyClass**获得**代理对象的Class对象**（接口+对应的构造函数）
- 通过Class对象调用得到构造方法
- 构造方法去创建实例
  - 构造方法传入`InvocationHandler`实例，需要实现其invoke方法

newProxyInstance帮我们获取代理的步骤和上面类似，只不过Class对象是直接通过**getProxyClass0(loader, intfs)**来获取的。而我们自己封装的代码中，使用的是getProxyClass方法。但是该方法最终还是调用的getProxyClass0(loader, intfs)来获取的Class对象

# 四、基于`CGLib`的动态代理

- `CGLIB`（Code Generator Library）是一个强大的、高性能的代码生成库。其被广泛应用于`AOP`框架（Spring）中，用以提供方法拦截操作。
- `CGLIB`代理主要通过对字节码的操作，以控制对象的访问。`CGLIB`底层使用了`ASM`（一个短小精悍的字节码操作框架）来操作字节码生成新的类。
- `CGLIB`相比于`JDK`动态代理更加强大
  - `JDK`动态代理虽然简单易用，但只能对接口进行代理。
  - 如果要代理的类为一个普通类，没有接口，那么Java动态代理就没法使用了。
- Java动态代理使用Java原生的反射`API`进行操作（运行期），在生成类上比较高效。
- `CGLIB`使用`ASM`框架直接对字节码进行操作（编译期），在类的执行过程中比较高效



## 4.1 `CGLib`介绍

### 4.1.1 EnHancer

- Enhancer既能够代理普通的class，也能够代理接口。
- Enhancer创建一个被代理对象的子类并且拦截所有的方法调用（包括从Object中继承的toString和hashCode方法）。
- Enhancer不能够拦截final类与方法。

常用方法：

~~~java
//    用来设置父类型
Enhancer.setSuperclass(Class superclass);

//    增强
Enhancer.setCallback(Callback callback);
Enhancer.setCallback(new InvocationHandler(){});
Enhancer.setCallback(new MethodInterceptor(){});
//    方法是用来创建代理对象，其提供了很多不同参数的方法用来匹配被增强类的不同构造方法。
Enhancer.create(Class type, Callback callback);
Enhancer.create(Class superclass, Class[] interfaces, Callback callback);
Enhancer.create(Class[] argumentTypes, Object[] arguments);
~~~

### 4.1.2 Callback

Callback是一个空的接口，在Cglib中它的实现类有以下几种：

1. `MethodInterceptor`
2. `NoOp`
3. `LazyLoader`
4. `Dispatcher`
5. `InvocationHandler`
6. `FixedValue`

**`MethodInterceptor`**

~~~
它可以实现类似于AOP编程中的环绕增强（around-advice）。

    它只有一个方法：
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args, MethodProxy proxy)

    代理类的所有方法调用都会转而执行这个接口中的intercept方法而不是原方法。
    如果需要在intercept方法中执行原方法可以使用参数method进行反射调用，
    或者使用参数proxy 一 proxy.invokeSuper(obj, args);
    后者会快一些（反射调用比正常的方法调用的速度慢很多）。
    MethodInterceptor允许我们完全控制被拦截的方法，并且提供了手段对原方法进行调用，

    因为 MethodInterceptor的效率不高，它需要产生不同类型的字节码，
    并且需要生成一些运行时对象（InvocationHandler就不需要），所以Cglib提供了其它的接口供我们选择。
~~~

**`InvocationHandler`**

它的使用方式和MethodInterceptor差不多。 需要注意的一点是，所有对invoke()方法的参数proxy对象的方法调用都会被委托给同一个InvocationHandler，所以可能会导致无限循环。

**`NoOp`**

 这个接口只是简单地把方法调用委托给了被代理类的原方法，不做任何其它的操作

**`LazyLoader`**

它也提供了一个方法：`Object loadObject()    loadObject()`方法会在第一次被代理类的方法调用时触发，它返回一个代理类的对象，这个对象会被存储起来然后负责所有被代理类方法的调用，一种lazy模式。如果被代理类或者代理类的对象的创建比较麻烦，而且不确定它是否会被使用，那么可以选择使用这种lazy模式来延迟生成代理。

**`Dispatcher`**

Dispatcher和LazyLoader接口相同，也是提供了loadObject()方法。不过它们之间不同的地方在于，Dispatcher的loadObject()方法在每次发生对原方法的调用时都会被调用并返回一个代理对象来调用原方法。也就是说Dispatcher的loadObject()方法返回的对象并不会被存储起来，可以类比成Spring中的Prototype类型，而LazyLoader则是lazy模式的Singleton。

### 4.1.3 FastClass

FastClass不使用反射类（Constructor或Method）来调用委托类方法，而是动态生成一个新的类（继承FastClass），向类中写入委托类实例直接调用方法的语句，用模板方式解决Java语法不支持问题，同时改善Java反射性能。

动态类为委托类方法调用语句建立索引，使用者根据方法签名（方法名+参数类型）得到索引值，再通过索引值进入相应的方法调用语句，得到调用结果。

## 4.2 实例：使用`CGLib`的EnHancer实现动态代理

被代理的类：

~~~java
public class Lisi{
    public void findHouse() {
        System.out.println("找房子");
    }

    public void discussPrice() {
        System.out.println("商议价格");
    }

    public void payAndHaveHousePropertyCard() {
        System.out.println("交钱办房产证");
    }
}
~~~

自定义拦截器实现`MethodInterceptor`:

~~~java
public class TestInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        // 前置增强
        System.out.println("中介了解详情");

        // 被代理类执行的方法（被增强的方法）
        // 注意这里是调用invokeSuper而不是invoke，否则死循环;
        // methodProxy.invokeSuper执行的是原始类的方法;
        // method.invoke执行的是子类的方法;
        Object invoke = methodProxy.invokeSuper(o, objects);

        // 后置增强
        System.out.println("中介进行处理");

        return invoke;
    }
}
~~~

使用Enhancer创建代理对象:

```java
public class Test3 {
    public static void main(String[] args) {
        //创建增强类
        Enhancer enhancer = new Enhancer();
        //传入被代理的类
        enhancer.setSuperclass(Lisi.class);
        //设置回调函数，传入自定义的拦截器
        enhancer.setCallback(new TestInterceptor());
        //获取代理对象
        Lisi proxy = (Lisi) enhancer.create();

        proxy.findHouse();
        proxy.discussPrice();
        proxy.payAndHaveHousePropertyCard();
        System.out.println("结束代理");
    }
}
```

结果：

> 中介了解详情
> 找房子
> 中介进行处理
> 中介了解详情
> 商议价格
> 中介进行处理
> 中介了解详情
> 交钱办房产证
> 中介进行处理
> 结束代理











