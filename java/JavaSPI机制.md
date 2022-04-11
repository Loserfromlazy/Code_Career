# JavaSPI机制

转载请声明，禁止抄袭，请珍惜他人成果，本文中如有错误欢迎各位大佬指正，感激不尽。

SPI全称Service Provider Interface，是Java提供的一套用来被第三方实现或者扩展的API，它可以用来启用框架扩展和替换组件。

## 一、简介

### 1.1 定义

在jdk6里面引进的一个新的特性ServiceLoader，从官方的文档来说，它主要是用来装载一系列的service provider。而且ServiceLoader可以通过service provider的配置文件来装载指定的service provider。当服务的提供者，提供了服务接口的一种实现之后，我们只需要在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。

> SPI java 官方文档定义：A simple service-provider loading facility.一个简单的服务提供者加载工具。
>
> A *service* is a well-known set of interfaces and (usually abstract) classes. A *service provider* is a specific implementation of a service. The classes in a provider typically implement the interfaces and subclass the classes defined in the service itself. Service providers can be installed in an implementation of the Java platform in the form of extensions, that is, jar files placed into any of the usual extension directories. Providers can also be made available by adding them to the application's class path or by some other platform-specific means.
>
> For the purpose of loading, a service is represented by a single type, that is, a single interface or abstract class. (A concrete class can be used, but this is not recommended.) A provider of a given service contains one or more concrete classes that extend this *service type* with data and code specific to the provider. The *provider class* is typically not the entire provider itself but rather a proxy which contains enough information to decide whether the provider is able to satisfy a particular request together with code that can create the actual provider on demand. The details of provider classes tend to be highly service-specific; no single class or interface could possibly unify them, so no such type is defined here. The only requirement enforced by this facility is that provider classes must have a zero-argument constructor so that they can be instantiated during loading.
>
> A service provider is identified by placing a *provider-configuration file* in the resource directory `META-INF/services`. The file's name is the fully-qualified [binary name](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassLoader.html#name) of the service's type. The file contains a list of fully-qualified binary names of concrete provider classes, one per line. Space and tab characters surrounding each name, as well as blank lines, are ignored. The comment character is `'#'` (`'\u0023'`, NUMBER SIGN); on each line all characters following the first comment character are ignored. The file must be encoded in UTF-8.
>
> If a particular concrete provider class is named in more than one configuration file, or is named in the same configuration file more than once, then the duplicates are ignored. The configuration file naming a particular provider need not be in the same jar file or other distribution unit as the provider itself. The provider must be accessible from the same class loader that was initially queried to locate the configuration file; note that this is not necessarily the class loader from which the file was actually loaded.
>
> Providers are located and instantiated lazily, that is, on demand. A service loader maintains a cache of the providers that have been loaded so far. Each invocation of the [`iterator`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html#iterator--) method returns an iterator that first yields all of the elements of the cache, in instantiation order, and then lazily locates and instantiates any remaining providers, adding each one to the cache in turn. The cache can be cleared via the [`reload`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html#reload--) method.
>
> Service loaders always execute in the security context of the caller. Trusted system code should typically invoke the methods in this class, and the methods of the iterators which they return, from within a privileged security context.
>
> Instances of this class are not safe for use by multiple concurrent threads.
>
> Unless otherwise specified, passing a `null` argument to any method in this class will cause a [`NullPointerException`](https://docs.oracle.com/javase/8/docs/api/java/lang/NullPointerException.html) to be thrown.

当然这么看定义肯定不直观，因此我们以一个入门案例来更深的了解一下Java的SPI机制和ServiceLoader

### 1.2 入门案例

我们先定义出一个接口

```java
package com.learn.testSPI;
public interface SPIService {
    void execute();
}
```

然后定义一下这个接口的两个实现类：

```java
package com.learn.testSPI;
public class SPIServiceImpl1 implements SPIService{
    @Override
    public void execute() {
        System.out.println("111111111111111111");
    }
}
```

```java
package com.learn.testSPI;
public class SPIServiceImpl2 implements SPIService{
    @Override
    public void execute() {
        System.out.println("222222222222222222");
    }
}
```

然后我们在resources下创建META-INF/services目录，然后创建一个名称为接口的全限定类名的文件，如下图：

![image-20220411143716569](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220411143716569.png)

然后在这个文件中写下我们要使用的该接口的实现类：

```
com.learn.testSPI.SPIServiceImpl1
//PS：实现类也可以写多个，因为SPI加载会获取一个实现类集合
//com.learn.testSPI.SPIServiceImpl2
```

然后我们编写测试类：

```java
public class Test {
    public static void main(String[] args) {
        //通过SPI机制，加载SPIService的实现类
        ServiceLoader<SPIService> load = ServiceLoader.load(SPIService.class);
        //第一种方法使用增强for循环
        for (SPIService spiService : load) {
            spiService.execute();
        }
        System.out.println("--------------------------------");
        //第二种方法，用迭代器
        Iterator<SPIService> iterator = load.iterator();
        while(iterator.hasNext()) {
            SPIService ser = iterator.next();
            ser.execute();
        }
    }
}
```

执行结果

~~~
111111111111111111
--------------------------------
111111111111111111
~~~

综上，Java提供了ServiceLoader类用于SPI机制获取某一个接口的实现类，这么做的好处就是可以为很多框架扩展提供了可能。

### 1.3 优点

使用 Java SPI 机制的优势是实现解耦，使得接口的定义与具体业务实现分离，而不是耦合在一起。应用进程可以根据实际业务情况启用或替换具体组件。以 java 中的 JDBC 数据库驱动为例，java 官方在核心库制定了 java.sql.Driver 数据库驱动接口，使用该接口实现了数据库链接等逻辑，但是并没有具体实现数据库驱动接口，而是交给 MySql 等厂商去实现具体的数据库接口。

## 二、源码解析

我们下面通过阅读源码的方式来了解ServiceLoader类。

我们首先看一下ServiceLoader类的私有属性：

```java
//配置文件路径，Java直接写死在类中，所以我们要在这个文件夹下创建SPI的配置文件
private static final String PREFIX = "META-INF/services/";

//正在加载的服务类
// The class or interface representing the service being loaded
private final Class<S> service;

//类加载器
// The class loader used to locate, load, and instantiate providers
private final ClassLoader loader;

// The access control context taken when the ServiceLoader is created
private final AccessControlContext acc;
//已加载的服务类集合
// Cached providers, in instantiation order
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();
//内部类，用于加载实现类
// The current lazy-lookup iterator
private LazyIterator lookupIterator;
```

我们在加载实现类时通过load方法`ServiceLoader.load(SPIService.class);`，此方法用于返回一个新的ServiceLoader实例：

```java
public static <S> ServiceLoader<S> load(Class<S> service,
                                        ClassLoader loader)
{
    return new ServiceLoader<>(service, loader);
}
```

既然要返回一个新的ServiceLoader实例，那么必然执行构造函数，因此我们看一下构造函数：

```java
public void reload() {
    //清空已加载的服务类集合
    providers.clear();
    //新建LazyIterator内部类
    lookupIterator = new LazyIterator(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
    //获取要加载的类，同时判空
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    //获取类加载器
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    //获取访问控制器
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}
```

可以看到，构造函数主要功能就是构造一个新的LazyIterator内部类。这个类主要就是实现加载实现类的。

在我们获取到了ServiceLoader实例之后就会调用`iterator()`方法获取迭代器，然后调用`next()`方法获取这个接口的实现类（for循环本质跟迭代器是一样的）。

我们先看一下`iterator()`方法：

```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();
		//判断是否还有下一个
        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }
		//返回实现类
        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

此方法直接返回了一个迭代器，这个迭代器迭代ServiceLoader类的Map（就是ServiceLoader的私有属性providers）。当我们调用迭代器的hasNext或next方法时，实际上是调用LazyIterator内部类的方法。

我们这里以next为例，首先看一下LazyIterator内部类的方法next方法，发现这里调用了nextService方法

```java
public S next() {
    if (acc == null) {
        return nextService();
    } else {
        PrivilegedAction<S> action = new PrivilegedAction<S>() {
            public S run() { return nextService(); }
        };
        return AccessController.doPrivileged(action, acc);
    }
}
```

我们跟进nextService方法中：

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
        S p = service.cast(c.newInstance());
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

在这个方法中，会加载类，然后存入map中（这个Map就是ServiceLoader的私有属性providers）。

到这里，我们已经对ServiceLoader类很清楚了。总结起来就是ServiceLoader去一个定义好的文件夹下（META-INF/services/）加载传入接口的实现类。

## 三、框架案例分析

### 3.1 Sentinel1.8 中使用SPI机制

在Sentinel中也有使用SPI机制的地方，在Sentinel中客户端需要与DashBoard的进行心跳交互，在心跳发送类的init方法中，就使用了SPI机制。（我们这里仅以心跳举例）

```java
@Override
public void init() {
    //SPI机制，如果我们添加了http的依赖，那么 SimpleHttpHeartbeatSender 就会被加载
    HeartbeatSender sender = HeartbeatSenderProvider.getHeartbeatSender();
    if (sender == null) {
        RecordLog.warn("[HeartbeatSenderInitFunc] WARN: No HeartbeatSender loaded");
        return;
    }

    initSchedulerIfNeeded();
    //设置心跳任务发送的时间间隔，默认10s
    long interval = retrieveInterval(sender);
    setIntervalIfNotExists(interval);
    //启动心跳任务
    scheduleHeartbeatTask(sender, interval);
}
```

我们可以进入到getHeartbeatSender方法中看一下：

```java
private static void resolveInstance() {
    //通过SPI加载HeartbeatSender.class接口的实现类
    HeartbeatSender resolved = SpiLoader.loadHighestPriorityInstance(HeartbeatSender.class);
    if (resolved == null) {
        RecordLog.warn("[HeartbeatSenderProvider] WARN: No existing HeartbeatSender found");
    } else {
        heartbeatSender = resolved;
        RecordLog.info("[HeartbeatSenderProvider] HeartbeatSender activated: " + resolved.getClass()
            .getCanonicalName());
    }
}

public static HeartbeatSender getHeartbeatSender() {
    return heartbeatSender;
}
```

在上面的方法中可以看到HeartbeatSender是通过封装的SPI工具类获取的（如果感兴趣可以自行看一下这个工具类的源码）

那么既然用了SPI，就肯定有配置文件，如下图：

![image-20220411160327251](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220411160327251.png)

到这里我们就可以知道了，Sentinel提供了netty和http两种方式发送心跳，当我们引入simple-http模块时，SPI机制就会去加载SimpleHttpHeartbeatSender，这样就可以实现不同的心跳发送方式的区分。