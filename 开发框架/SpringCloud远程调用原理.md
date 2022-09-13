# Spring Cloud 远程调用原理

> 此笔记是基于我的[Spring Cloud笔记](https://github.com/Loserfromlazy/Code_Career/blob/master/%E5%BC%80%E5%8F%91%E6%A1%86%E6%9E%B6/SpringCloud.md)上对Feign和Ribbon进行深入学习的。学习前可以先看一下我的[SpringCloud](https://github.com/Loserfromlazy/Code_Career/blob/master/%E5%BC%80%E5%8F%91%E6%A1%86%E6%9E%B6/SpringCloud.md)第4.2Ribbon负载均衡和4.4章Feign远程调用。
>
> 参考资料：springcloud Nginx高并发核心编程；互联网等资源。

## 一、RPC入门与代理模式

这里主要通过一个RPC远程调用的客户端实现类的入门案例，并通过代理模式进行改造，来初步了解一下Feign的基本原理。如果不了解代理模式及JDK代理可以看我的笔记[Java代理](https://www.cnblogs.com/yhr520/p/15601620.html)

> 此笔记`第一章RPC入门与代理模式`的代码都在此项目[LearnSpringCloud](https://github.com/Loserfromlazy/LearnSpringCloud/)下的**rpcDemo和rpcDemoProvider**这两个项目中

### 1.1 入门案例：简单的RPC实现类

客户端RPC实现类主要有以下功能或职责：

- 拼装URI：根据Java接口的参数，拼装目标RESTFUL
- 发送请求并获取结果：通过Java的Http组件，调用provider微服务的REST接口，并且获取REST响应
- 解码：解析REST的响应结果，封装成POJO对象并且返回

我们在进行远程调用时，会自动生成一个客户端实现类，这样的话调用过程对用户来说就是透明的。下面我们来实现一个简单的RPC实现类。首先我们先编写供我们调用的接口，这里很简单用springboot随便实现两个接口即可，主要代码如下：

```java
package com.learn.rpcdemoprovider.controller;
//。。。略
@RestController
@RequestMapping("/demo")
public class ProviderController {
    @GetMapping("/test")
    public Result<Boolean> test(){
        Result<Boolean> result = new Result<>();
        result.setResult(true);
        result.setMsg("请求成功");
        result.setCode(100);
        return result;
    }
    @GetMapping("/print")
    public Result<String> print(@RequestParam("input") String input){
        Result<String> result = new Result<>();
        result.setResult(input);
        result.setMsg("请求成功");
        result.setCode(100);
        return result;
    }
}
```

然后定义RPC客户端的接口：

```java
public interface RpcDemoClient {

    Result<Boolean> test();

    Result<String> testPrint(String input);
}
```

然后我们实现实现类：

```java
@Slf4j
public class RpcDemoClientImpl implements RpcDemoClient{
    final String contextPath = RpcConstant.CONTEXT;

    @Override
    public Result<Boolean> test() {
        //封装URI
        String uri = "/test";
        String restUri = contextPath + uri;
        log.info("restUri={}",restUri);
        //发起HTTP请求
        String request = null;
        try {
            request = HttpUtils.getRequest(restUri);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //解析响应结果
        if (StringUtils.hasText(request)){
            return JSONObject.parseObject(request, Result.class);
        }else {
            return ResultUtils.resultInit(0,"失败",false);
        }
    }

    @Override
    public Result<String> testPrint(String input) {
        //封装URI
        String uri = "/print?input={0}";
        String uriParams = MessageFormat.format(uri, input);
        String restUri = contextPath + uriParams;
        log.info("restUri={}",restUri);
         //发起HTTP请求
        String request = null;
        try {
            request = HttpUtils.getRequest(restUri);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //解析响应结果
        if (StringUtils.hasText(request)){
            return JSONObject.parseObject(request, Result.class);
        }else {
            return ResultUtils.resultInit(0,"失败",null);
        }
    }
}
```

测试类：

```java
@Test
void testRpcDemoClient(){
    RpcDemoClient client = new RpcDemoClientImpl();
    Result<Boolean> test = client.test();
    System.out.println(test);
    Result<String> stringResult = client.testPrint("123456");
    System.out.println(stringResult);
}
```

以上我们就完成了一个简单的RPC实现类

### 1.2 静态代理改造

通常来说我们在远程调用时会通过代理对目标对象进行委托，在客户端和目标对象起到中介作用。那么为什么要用代理呢？因为我们实际会通过代理类对委托类进行拓展，比如远程调用的负载均衡、熔断、重试功能都要通过代理实现。

静态代理进行RPC调用需要的类为：

1. 远程接口，比如上面的RpcDemoClient类
2. 真实委托类，比如上面的RpcDemoClientImpl类
3. 代理类，比如下面的ClientStaticProxy类

下面我们先用静态代理进行简单改造，为动态代理做个铺垫：

```java
@Slf4j
@AllArgsConstructor
public class ClientStaticProxy implements RpcDemoClient{
    //真正执行的实现类
    private RpcDemoClient rpcDemoClient;

    @Override
    public Result<Boolean> test() {
        log.info("静态代理，test方法执行");
        return rpcDemoClient.test();
    }

    @Override
    public Result<String> testPrint(String input) {
        log.info("静态代理，print方法执行");
        return rpcDemoClient.testPrint(input);
    }
}
```

然后我们编写测试类：

```java
@Test
void testRpcStaticClientProxy(){
    RpcDemoClient client = new RpcDemoClientImpl();
    ClientStaticProxy proxy = new ClientStaticProxy(client);
    Result<Boolean> test = proxy.test();
    System.out.println(test);
    Result<String> stringResult = proxy.testPrint("123456");
    System.out.println(stringResult);
}
```

以上就是简单的静态代理的改造。

### 1.3 动态代理改造

实际开发中我们肯定不能为每一个实现类实现代理类，所以我们可以通过动态代理的方式实现代理。动态代理改造需要的类跟静态代理类似，只不过代理类是用jdk动态代理生成的。

主要代码如下：

```java
@Test
public void testDynamicClientProxy(){
    //真实委托类
    RpcDemoClient client = new RpcDemoClientImpl();
    //动态代理生成
    RpcDemoClient proxyClient = (RpcDemoClient) Proxy.newProxyInstance(RpcDemoApplicationTests.class.getClassLoader(), new Class[]{RpcDemoClient.class}, (proxy, method, args) -> {
        System.out.println("动态代理，代理方法执行");
        return method.invoke(client, args);
    });
    //调用并返回结果
    Result<Boolean> test = proxyClient.test();
    System.out.println(test);
    Result<String> stringResult = proxyClient.testPrint("1234567");
    System.out.println(stringResult);
}
```

我们简单介绍一下Proxy#newProxyInstance方法，这里的第一个参数是类加载器，第二个参数是被代理类的接口，第三个参数是一个InvocationHandler，我们需要在这个类的invoke方法中实现代理的逻辑。其实Feign的RPC调用的各种增强就是通过InvocationHandler实现的。

> InvocationHandler是一个Java反射包中的函数式接口，因此我在上面代码中使用了lambda的形式简化开发。
>
> ```java
> public interface InvocationHandler {
>     public Object invoke(Object proxy, Method method, Object[] args)
>         throws Throwable;
> }
> ```

## 二、模拟Feign的动态代理实现类

> 此笔记`第二章模拟Feign的动态代理实现类`的代码都在此项目[LearnSpringCloud](https://github.com/Loserfromlazy/LearnSpringCloud/)下的**rpcDemo和rpcDemoProvider**这两个项目中

下面我们通过简单的模拟Feign的动态代理实现类，来了解一下Feign，为后面学习Feign的相关组件源码做铺垫。

### 2.1 模拟Feign的方法处理器MethodHandler

由于每一个Feign客户端一般都会包含多个远程调用方法，因此Feign为远程调用封装了一个专门的接口MethodHandler，此接口仅包含一个invoke抽象方法，方法的作用是组装URL，完成REST远程调用，返回JSON结果。主要代码如下：

```java
public interface MyMethodHandler {

    /**
     * 组装URL，完成REST远程调用，返回JSON结果
     *
     * @param args RPC方法的参数
     * @return REST接口响应
     * @throws Throwable 异常
     */
    Object invoke(Object[] args) throws Throwable;
}
```

实现类如下：

```java
@Slf4j
public class MyMethodHandlerImpl implements MyMethodHandler{

    /**
     * REST URL的前半部分，来自Feign接口的类级别注释
     * 比如：“http://www.demo.com:8111/demo”
     */
    final String contextPath;
    /**
     * REST URL的后半部分，来自Feign接口的方法级别注释
     * 比如："/print"或"/test"
     */
    final String url;

    public MyMethodHandlerImpl(String contextPath, String url) {
        this.contextPath = contextPath;
        this.url = url;
    }

    @Override
    public Object invoke(Object[] args) throws Throwable {
        String restUrl = contextPath + MessageFormat.format(url,args);
        log.info("restUrl={}",restUrl);
        String response = HttpUtils.getRequest(restUrl);
        Result result = JSONObject.parseObject(response, Result.class);
        return result;
    }
}
```

### 2.2 模拟Feign的调用处理器InvocationHandler

调用处理器是一个相对简单的类，它里面有一个Map成员dispatch，保存着RPC方法反射实例和MethodHandler之间的映射。它是通过 Java 反射扫描远程调用接口中的每一个方法的反射注解，组装出一个对应的 Map 映射实例，它的 key 值为 RPC 方法的反射实例，value 值为 MockRpcMethodHandler 方法的处理器实例。主要代码如下：

```java
public class MyInvocationHandler implements InvocationHandler {

    /**
     * 远程调用的映射，根据方法名称，分发方法处理器
     * key：Feign接口的方法名称；value：方法处理器
     */
    private Map<Method,MyMethodHandler> dispatch;

    public static <T> T newInstance(Class<T> clazz){
        //获取contextPath
        RestController controllerAnno = clazz.getAnnotation(RestController.class);
        if (controllerAnno == null){
            return null;
        }
        String contextPath = controllerAnno.value();
        MyInvocationHandler invocationHandler = new MyInvocationHandler();
        invocationHandler.dispatch = new LinkedHashMap<>();
        //遍历获取url
        for (Method method : clazz.getMethods()) {
            RequestMapping requestMapping = method.getAnnotation(RequestMapping.class);
            if (requestMapping == null){
                continue;
            }
            String uri = requestMapping.name();
            //缓存MethodHandler
            MyMethodHandler methodHandler = new MyMethodHandlerImpl(contextPath,uri);
            invocationHandler.dispatch.put(method,methodHandler);
        }
        T proxy = (T) Proxy.newProxyInstance(clazz.getClassLoader(), new Class[]{clazz}, invocationHandler);
        return proxy;
    }



    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if ("equals".equals(method.getName())){
            Object o = args.length > 0 && args[0] != null ? args[0] : null;
            return equals(o);
        }else if ("hashCode".equals(method.getName())){
            return hashCode();
        }else if ("toString".equals(method.getName())){
            return toString();
        }
        //从dispatch映射中取出MyMethodHandler，并进行远程调用
        MyMethodHandler methodHandler = dispatch.get(method);
        return methodHandler.invoke(args);
    }
}
```

此类的newInstance方法是一个静态方法，用于组装和完成dispatch的映射，最后通过jdk的Proxy.newProxyInstance方法生成代理对象并返回。

### 2.3 测试远程调用

下面我们来进行测试，我们就使用1.1中的RpcDemoClient，但是我们需要加上注解，因为我们现在是通过注解获取的URI，代码如下：

```java
@RestController(RpcConstant.CONTEXT)
public interface RpcDemoClient {

    //这里我们是简单实现，所以用name属性标识url，并且用{?}表示参数
    @RequestMapping(name = "/test")
    Result<Boolean> test();

    @RequestMapping(name = "/print?input={0}")
    Result<String> testPrint(String input);
}
```

测试类代码如下：

```java
@Test
public void testMyFeignImpl(){
    RpcDemoClient client = MyInvocationHandler.newInstance(RpcDemoClient.class);
    Result<Boolean> test = client.test();
    System.out.println(test);
    Result<String> stringResult = client.testPrint("123456");
    System.out.println(stringResult);
}
```

Feign的RPC客户端实现类是一种JDK动态代理类，能完成对简单RPC类的动态代理，其次Feign通过对InvocationHandler、MethodHandler完成了对RPC被委托类的增强，其InvocationHandler可以通过使用Ribbon、Hystrix使Feign客户端具备负载均衡、熔断等能力。下面我们来正式的看一下Feign的组件。

## 三、Feign的重要组件

### 3.1 Feign的调用处理器组件

通过JDK动态代理生成代理类的核心步骤就是定制一个调用处理器，实现JDK的反射包下的InvocationHandler接口，并且实现invoke方法。

在Feign中提供了一个默认的调用处理器，名为FeignInvocationHandler，在feign-core核心包中，当然此类是可以替换的，如果与Hystrix一起使用就会被替换为HystrixInvocationHandler，此类在feign-hystrix的jar包中。

FeignInvocationHandler的源码如下：

```java
//此类是ReflectiveFeign的内部类，是默认的调用处理器
static class FeignInvocationHandler implements InvocationHandler {

  private final Target target;
    //method和MethodHandler的映射
  private final Map<Method, MethodHandler> dispatch;
	//构造函数
  FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
    this.target = checkNotNull(target, "target");
    this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }
	//从映射中取得MethodHandler的实例，然后调用MethodHandler的invoke方法
    return dispatch.get(method).invoke(args);
  }

  //重写toString等方法略。。。
}
```

### 3.2 Feign的方法处理器组件

Feign的MethodHandler是feign的自定义接口，在InvocationHandlerFactory内部，默认有两个实现类DefaultMethodHandler和SynchronousMethodHandler，源码如下：

```java
public interface InvocationHandlerFactory {

  InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch);
    
  interface MethodHandler {
	//完成URL请求
    Object invoke(Object[] argv) throws Throwable;
  }

  static final class Default implements InvocationHandlerFactory {

    @Override
    public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
      return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
    }
  }
}
```

![image-20220905124804460](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220905124804460.png)

其中内置的 SynchronousMethodHandler 同步方法处理实现类是 Feign 的一个重要类，提供了基本的远程URL同步请求响应处理，源码如下：

```java
final class SynchronousMethodHandler implements MethodHandler {

  private static final long MAX_RESPONSE_BUFFER_SIZE = 8192L;

  //RPC远程调用的元数据，保存了RPC方法的配置键，格式为“接口名#方法名(形参表)”；其次保存了RPC方法的请求模板（包括URL、请求方法等），然后还保存了returnType返回类型和一些其他属性
  private final MethodMetadata metadata;
  //RPC远程调用Java接口的元数据，保存了RPC接口的类名称服务名称等信息，也就是说@FeignClient注解中的配置主要属性值都在target实例中
  private final Target<?> target;
  private final Client client;//Feign的客户端实例，执行REST的请求和处理响应
  private final Retryer retryer;
  private final List<RequestInterceptor> requestInterceptors;//请求拦截器
  private final Logger logger;
  private final Logger.Level logLevel;
  private final RequestTemplate.Factory buildTemplateFromArgs;
  private final Options options;
  private final Decoder decoder;//结果解码器
  private final ErrorDecoder errorDecoder;
  private final boolean decode404;//是否反编码404
  //其余属性略。。。
    
  //执行rpc请求，解析响应信息
  @Override
  public Object invoke(Object[] argv) throws Throwable {
    //生成requestTemplate实例
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
          //执行REST请求和处理响应
        return executeAndDecode(template, options);
      } catch (RetryableException e) {
        try {
          retryer.continueOrPropagate(e);
        } catch (RetryableException th) {
          Throwable cause = th.getCause();
          if (propagationPolicy == UNWRAP && cause != null) {
            throw cause;
          } else {
            throw th;
          }
        }
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
  
  Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    //通过请求模板实例生成目标request的请求实例，主要完成请求的URL、参数和请求头等内容的封装。
    Request request = targetRequest(template);

    if (logLevel != Logger.Level.NONE) {
      logger.logRequest(metadata.configKey(), logLevel, request);
    }

    Response response;
    long start = System.nanoTime();
    try {
      //发起真正的RPC远程调用
      response = client.execute(request, options);
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
      }
      throw errorExecuting(request, e);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    boolean shouldClose = true;
    try {
      if (logLevel != Logger.Level.NONE) {
        response =
            logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      }
      if (Response.class == metadata.returnType()) {
        if (response.body() == null) {
          return response;
        }
        if (response.body().length() == null ||
            response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
          shouldClose = false;
          return response;
        }
        // Ensure the response body is disconnected
        byte[] bodyData = Util.toByteArray(response.body().asInputStream());
        return response.toBuilder().body(bodyData).build();
      }
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
          //将请求结果进行解码
          Object result = decode(response);
          shouldClose = closeAfterDecode;
          return result;
        }
      } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      } else {
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      if (logLevel != Logger.Level.NONE) {
        logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
      }
      throw errorReading(request, response, e);
    } finally {
      if (shouldClose) {
        ensureClosed(response.body());
      }
    }
  }
    //其余方法略。。。
}
```

Feign的SynchronousMethodHandler 同步方法处理除了上面的属性和方法还有一个内部工厂类，源码如下：

```java
static class Factory {

  private final Client client;
  private final Retryer retryer;
  private final List<RequestInterceptor> requestInterceptors;
  private final Logger logger;
  private final Logger.Level logLevel;
  private final boolean decode404;
  private final boolean closeAfterDecode;
  private final ExceptionPropagationPolicy propagationPolicy;

  //省略构造器

  //工厂的默认创建方法，创建一个方法调用器
  public MethodHandler create(Target<?> target,
                              MethodMetadata md,
                              RequestTemplate.Factory buildTemplateFromArgs,
                              Options options,
                              Decoder decoder,
                              ErrorDecoder errorDecoder) {
    return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger,
        logLevel, md, buildTemplateFromArgs, options, decoder,
        errorDecoder, decode404, closeAfterDecode, propagationPolicy);
  }
}
```

### 3.3 Feign的客户端组件

客户端组件是feign中的重要组件，负责最终HTTP请求的执行，核心逻辑是发送Request请求到服务器，接收到Response后进行解码，并返回结果。

feign.Client是顶层接口，源码如下：

```java
public interface Client {
  /**
   * Executes a request against its {@link Request#url() url} and returns a response.
   * 提交HTTP请求，并且接收response响应后进行解码
   * @param request safe to replay.
   * @param options options to apply to this request.
   * @return connected response, {@link Response.Body} is absent or unread.
   * @throws IOException on a network error connecting to {@link Request#url()}.
   */
  Response execute(Request request, Options options) throws IOException;
}
```

此接口不同的实现类其内部的HTTP的技术是不同的：

- Client.Default:默认的实现类，使用JDK的HttpURLConnection类提交HTTP请求

  ![image-20220905131638800](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220905131638800.png)

- ApacheHttpClient:内部使用Apache HttpClient开源组件提交HTTP请求

  客户端实现类 ApacheHttpClient 处于 feign-httpclient 独立 JAR 包中，如果使用，还需引入配套版本的 JAR 包依赖：

  ~~~xml
  <dependency>
   <groupId>io.github.openfeign</groupId>
   <artifactId>feign-httpclient</artifactId>
   <version>${feign-httpclient.version}</version>
  </dependency>
  <dependency>
   <groupId>org.apache.httpcomponents</groupId>
   <artifactId>httpclient</artifactId>
   <version>${httpclient.version}</version>
  </dependency>
  ~~~

  在配置文件中将配置项 feign.httpclient.enabled 的值设置为 true，表示需要启用ApacheHttpClient。

- OkHttpClient:内部使用OkHttp3开源组件提交HTTP请求

  OkHttp3 组件是 Square公司开发的，用于替代 HttpUrlConnection 和 ApacheHttpClient 的高性能 HTTP 组件。OkHttp3 较好地支持 SPDY 协议（SPDY 是 Google 开发的基于 TCP 的传输层协议。

- LoadBalancerFeignClient:内部使用Ribbon负载均衡技术完成HTTP请求处理

  该客户端类处于 Feign 核心 JAR 包中，在内部使用 Ribbon 开源组件实现多个 Provider 实例之间的负载均衡。它的内部有一个封装的 delegate 被委托客户端成员，该成员才是最终的 HTTP请求提交者。Ribbon 负载均衡组件计算出合适的服务端 Provider 实例之后，由 delegate 被委托客户端完成到 Provider 服务端之间的 HTTP 请求。

## 四、Feign的动态代理的创建流程

### 4.1 Feign创建动态代理的整体流程

根据上面的学习，我们能大概了解feign的工作原理，在应用程序启动的初始化过程中，Feign完成了以下工作：

1. 对于每一个RPC远程调用的Java接口，Feign根据@FeignClient注解生成本地JDK动态代理实例。
2. 对于Java接口中的每一个方法，Feign根据Spring MVC类型的注解生成方法处理器实例，该实例内部有一个RequestTemplate请求模板实例

在远程调用的过程中，Feign完成了以下工作：

1. Feign使用远程方法调用的实际参数替换掉RequestTemplate请求模板实例中的参数，生成最终的HTTP请求
2. 将HTTP请求通过feign.Client客户端实例发送到Provider服务端。

综上，Feign的整体运行流程如下：

1. 通过应用启动类上的@EnableFeignClients 注解开启 Feign 的装配和远程代理实例创建。在@EnableFeignClients 注解源码中可以看到导入了 FeignClientsRegistrar 类，该类用于扫描@FeignClient 注解过的 RPC 接口。

   ```java
   //略。。。
   @Import(FeignClientsRegistrar.class)
   public @interface EnableFeignClients {
   //略。。。
   }
   ```

2. 通过对@FeignClient 注解 RPC 接口扫描创建远程调用的动态代理实例。FeignClientsRegistrar类会进行包扫描，扫描所有包下@FeignClient注解过的接口，创建 RPC接口的FactoryBean工厂类实例，并将这些FactoryBean 注入 Spring IOC 容器中，当其他类需要注入Feign客户端时，Spring 就会通过注册的 FactoryBean 工厂类实例的 getObject()方法获取 RPC 接口的动态代理实例。部分源码如下：

   ```java
   //FeignClientsRegistrar#registerFeignClients方法部分源码，省略其余代码
   for (BeanDefinition candidateComponent : candidateComponents) {
      if (candidateComponent instanceof AnnotatedBeanDefinition) {
         // verify annotated class is an interface
         AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
         AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
         Assert.isTrue(annotationMetadata.isInterface(),
               "@FeignClient can only be specified on an interface");
   
         Map<String, Object> attributes = annotationMetadata
               .getAnnotationAttributes(
                     FeignClient.class.getCanonicalName());
   
         String name = getClientName(attributes);
         registerClientConfiguration(registry, name,
               attributes.get("configuration"));
   	//此方法是类的内部方法，此方法主要作用是将FeignClient注解属性封装、注册到Spring IOC容器，通过Spring提供的工具类BeanDefinitionReaderUtils#registerBeanDefinition进行注册
         registerFeignClient(registry, annotationMetadata, attributes);
      }
   }
   
   //FeignClientsRegistrar#registerFeignClient方法源码，省略其余代码
   private void registerFeignClient(BeanDefinitionRegistry registry,
   			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
       String className = annotationMetadata.getClassName();
       //将FeignClient注解的属性等相关信息封装进BeanDefinitionBuilder，注意此时封装的类型是FeignClientFactoryBean.class
       BeanDefinitionBuilder definition = BeanDefinitionBuilder
           .genericBeanDefinition(FeignClientFactoryBean.class);
       validate(attributes);
       definition.addPropertyValue("url", getUrl(attributes));
       definition.addPropertyValue("path", getPath(attributes));
       String name = getName(attributes);
       definition.addPropertyValue("name", name);
       String contextId = getContextId(attributes);
       definition.addPropertyValue("contextId", contextId);
       definition.addPropertyValue("type", className);
       definition.addPropertyValue("decode404", attributes.get("decode404"));
       definition.addPropertyValue("fallback", attributes.get("fallback"));
       definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
       definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
   
       String alias = contextId + "FeignClient";
       AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
   
       boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
       // null
   
       beanDefinition.setPrimary(primary);
   
       String qualifier = getQualifier(attributes);
       if (StringUtils.hasText(qualifier)) {
           alias = qualifier;
       }
   
       BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                                                              new String[] { alias });
       BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
   }
   ```

   在创建 RPC 接口的动态代理实例时，Feign还会为每一个 RPC 接口创建一个调用处理器，也会为接口的每一个 RPC 方法创建一个方法处理器，并且将方法处理器缓存在调用处理器的 dispatch映射成员中。同时还会生成一个RequesTemplate 请求模板实例，RequestTemplate 中包含请求的所有信息，如请求 URL、请求类型、请求参数等。

3. 发生RPC调用时，通过动态代理实例类完成远程Provider的 HTTP 调用，Feign会根据 RPC 方法的反射实例从调用处理器的dispatch 成员中取得方法处理器，然后由 MethodHandler 方法处理器开始 HTTP 请求处理。

   MethodHandler 会结合实际的调用参数，通过 RequesTemplate 模板实例生成 Request 请求实例。最后，将 Request 请求实例交给 feign.Client 客户端实例进一步完成 HTTP 请求处理。

4. 在完成远程 HTTP 调用前需要进行客户端负载均衡等处理，生产环境下，Feign 必须和 Ribbon 结合在一起使用，所以方法处理器MethodHandler的客户端client成员必须是具备负载均衡能力的LoadBalancerFeignClient 类型，而不是完成 HTTP 请求提交的 ApacheHttpClient 等类型。只有在负载均衡计算出最佳的 Provider 实例之后，才能开始HTTP 请求的提交。

FeignClient的创建详细流程：

![FeignCreate20220912](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/FeignCreate20220912.png)

### 4.2 RPC动态代理容器实例的FactiryBean

为了方便 Feign 的 RPC 客户端动态代理实例的使用，还需要将其注册到 Spring IOC 容器，以方便使用者通过@Resource 或@Autoware 注解将其注入其他的依赖属性。Feign 的 RPC 客户端动态代理 IOC 容器实例只能通过 FactoryBean 方式创建，原因有两点：

- 代理对象为通过 JDK 反射机制动态创建的 Bean，不是直接定义的普通实现类；
- 它配置的属性值比较多，而且是通过 @FeignClient 注解配置完成的

> FactoryBean 通常是用来创建比较复杂的bean，一般的bean 直接用xml配置和基于Java配置即可，但如果一个bean的创建过程中涉及到很多其他的bean 和复杂的逻辑，用xml配置比较困难，这时可以考虑用FactoryBean。FactoryBean 注册到容器之后，从Spring上下文通过ID或者类型获取IOC容器Bean时，获取的实际上是FactoryBean的getObject()返回的对象，而不是FactoryBean本身.

Feign 提供了一个用于获取 RPC 容器实例的工厂类，名为 FeignClientFactoryBean 类，部分源码如下：

```java
class FeignClientFactoryBean
      implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
   private Class<?> type;//RPC接口的class对象
   private String name;//RPC接口配置的远程provider微服务名称
   private String url;//RPC接口配置的url值，由@FeignClient注解配置
   private String contextId;
   private String path;//RPC接口配置的path值，由@FeignClient注解配置
   private boolean decode404;
   private ApplicationContext applicationContext;
   private Class<?> fallback = void.class;
   private Class<?> fallbackFactory = void.class;
   
    //获取IOC容器的Feign.Builder建造者Bean
   protected Feign.Builder feign(FeignContext context) {
       FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
       Logger logger = loggerFactory.create(this.type);

       // @formatter:off
       //从IOC容器中获取Feign.Builder实例，并且设置编解码器、日志器、方法解析器
       Feign.Builder builder = get(context, Feign.Builder.class)
           // required values
           .logger(logger)
           .encoder(get(context, Encoder.class))
           .decoder(get(context, Decoder.class))
           .contract(get(context, Contract.class));
       // @formatter:on
       configureFeign(context, builder);
       return builder;
   }
    
   //通过ID或类型获取IOC容器的Bean时调用
   @Override
	public Object getObject() throws Exception {
        //委托给getTarget()方法
		return getTarget();
	} 
    //委托方法：获取RPC动态代理的Bean
    <T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
        //根据当前是Hystrix还是Sentinel注入不同的Builder
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			builder.client(client);
		}
        //Targeter是一个目标类，最终本质上是调用build().newInstance(target);方法
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}
    
    protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
   //其余方法略。。。
}
	
```

在扫描注解时就会将FeignClient包装成FeignClientFactoryBean（在上面4.1中的总流程的第二步源码中有体现），注入到容器中，在使用时获取FactoryBean的getObject()返回的对象。

### 4.3 Feign.Builder建造者容器实例

当从 Spring IOC 容器获取 RPC 接口的动态代理实例时，也就是当 FeignClientFactoryBean 的getObject()方法被调用时，其调用的 getTarget()方法首先从 IOC 容器获取配置好的 Feign.Builder建造者容器实例，然后通过 Feign.Builder 建造者容器实例的 target()方法完成 RPC 动态代理实例的创建。源码见上一小节4.2。

当然如果项目中使用了Hystrix或Sentinel将会引用他们的Builder，这里只以

Feign.Builder 类是 feign.Feign 抽象类的一个内部类，部分源码如下：

```java
public abstract class Feign {

  //建造者方法
  public static Builder builder() {
    return new Builder();
  }
    
  //内部类：建造者类
  public static class Builder {
      public <T> T target(Class<T> apiType, String url) {
          return target(new HardCodedTarget<T>(apiType, url));
      }
	  //创建RPC客户端的动态代理实例	
      public <T> T target(Target<T> target) {
          return build().newInstance(target);
      }
      //建造方法
      public Feign build() {
          //方法处理器工厂的实例
          SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
              new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                  logLevel, decode404, closeAfterDecode, propagationPolicy);
          //RPC方法解析器
          ParseHandlersByName handlersByName =
              new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
                  errorDecoder, synchronousMethodHandlerFactory);
          //反射式Feign实例
          return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
    }
    //其余方法略。。。
  }
  
  //其余方法略。。。
}
```

当FeignClientFactoryBean工厂类的getObject()方法被调用后，通过Feign.Builder容器实例的target()方法完成 RPC 动态代理实例的创建。然后target方法首先调用内部的 build()方法创建一个 Feign 实例，然后通过该实例的 newInstance(...)方法（注意这里的newInstance是抽象类Feign的抽象方法，最终会在其实现类中通过反射创建代理对象）创建最终的 RPC 动态代理实例。

### 4.4 默认的RPC动态代理实例的创建流程

默认情况下，Feign.Builder 建造者实例的 target()方法会调用自身的 build()方法创建一个ReflectiveFeign（反射式 Feign）实例，然后调用该实例的 newInstance()方法创建远程接口最终的JDK 动态代理实例。

我们可以看一下ReflectiveFeign的源码：

```java
public class ReflectiveFeign extends Feign {

  private final ParseHandlersByName targetToHandlersByName;
  private final InvocationHandlerFactory factory;
  private final QueryMapEncoder queryMapEncoder;
  
  @Override
  public <T> T newInstance(Target<T> target) {
    //方法名和MethodHandler的映射
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
    //方法反射对象和MethodHandler的映射
    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    //创建一个InvocationHandler调用处理器和方法处理器类似，它的创建也是通过工厂模式完成的。默认的InvocationHandler实例是通过InvocationHandlerFactory工厂类完成的
    InvocationHandler handler = factory.create(target, methodToHandler);
    //JDK动态代理生成代理对象
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
  //其余代码略。。。
}
```

从上面我们可以看到此方法步骤跟我们之前的简单模拟的流程类似：

1. 首先进行方法解析，进行方法名称和方法处理器的映射
2. 创建方法反射对象和MethodHandler的映射
3. 创建一个InvocationHandler
4. 创建一个动态代理对象

下面我们来看看newInstance一开始的TargetToHandlersByName#apply方法是如何做名称映射的。TargetToHandlersByName是ReflectiveFeign的内部类作用是方法解析器，其apply方法源码如下：

```java
public Map<String, MethodHandler> apply(Target key) {
  //解析RPC方法元数据，返回一个方法元数据列表
  List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
  Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
  //遍历RPC方法元数据
  for (MethodMetadata md : metadata) {
    BuildTemplateByResolvingArgs buildTemplate;
    if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
      buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
    } else if (md.bodyIndex() != null) {
      buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder, queryMapEncoder);
    } else {
      buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder);
    }
    //通过方法处理器工厂factory创建SynchronousMethodHandler同步方法处理器实例
    result.put(md.configKey(),
        factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
  }
  return result;
}
```

从上面源码可知，TargetToHandlersByName是通过方法处理器工厂factory的create方法完成的，而默认的方法处理器工厂定义在SynchronousMethodHandler类中，我们可以直接跟进去，源码如下：

```java
static class Factory {

  private final Client client;
  private final Retryer retryer;
  private final List<RequestInterceptor> requestInterceptors;
  private final Logger logger;
  private final Logger.Level logLevel;
  private final boolean decode404;
  private final boolean closeAfterDecode;
  private final ExceptionPropagationPolicy propagationPolicy;

  //构造函数略。。。

  public MethodHandler create(Target<?> target,
                              MethodMetadata md,
                              RequestTemplate.Factory buildTemplateFromArgs,
                              Options options,
                              Decoder decoder,
                              ErrorDecoder errorDecoder) {
    return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger,
        logLevel, md, buildTemplateFromArgs, options, decoder,
        errorDecoder, decode404, closeAfterDecode, propagationPolicy);
  }
}
```

从上面源码可见默认方法处理器工厂类 Factory 的 create()方法创建的正是同步方法处理器SynchronousMethodHandler 的实例。

### 4.5 Contract远程调用协议规则类

在通过 ReflectiveFeign.newInstance()方法创建本地 JDK Proxy 实例时，首先需要调用方法解析器 ParseHandlersByName 的 apply()方法，获取方法名和方法处理器的映射。在ParseHandlersByName.apply()方法中，需要通过Contract协议规则类将远程调用Feign接口中的所有方法配置和注解解析成一个` List<MethodMetadata>`方法元数据列表。

Spring Cloud Feign 中有两个协议规则解析类：一个为 Feign 默认协议规则解析类（DefaultContract）；另一个为 SpringMvcContract 协议规则解析类，后者用于解析使用了 Spring MVC 规则配置 RPC方法。虽然Feign 有一套自己的默认协议规则，定义了一系列 RPC 方法的配置注解，用于 RPC 方法所对应的 HTTP 请求相关的参数，但是为了降低学习成本，Spring Cloud并没有推荐采用 Feign 自己的协议规则注解来进行 RPC 接口配置，而是推荐部分 Spring MVC 协议规则注解来进行 RPC 接口的配置，并且通过SpringMvcContract 协议规则解析类进行解析。

## 五、Feign远程调用的执行流程

在Feign中生成RPC接口JDK动态代理实例涉及到的InvocationHandler调用处理器有很多种，导致Feign远程调用的执行流程稍有不同，但主要步骤是一致的，这里主要介绍两类：

1. 默认的调用处理器FeignInvocationHandler
2. 与Hystrix有关的HystrixInvocationHandler

### 5.1 FeignInvocationHandler执行流程

Feign的整体流程主要如下：

1. 通过 Spring IOC 容器实例完成动态代理实例的装配
2. 执行 InvocationHandler 调用处理器的 invoke(...)方法
3. 执行 MethodHandler 方法处理器的 invoke(...)方法
4. 通过 feign.Client 客户端成员完成远程 URL 请求执行和获取远程结果

其中MethodHandler默认的方法处理器为SynchronousMethodHandler同步调用处理器，它的 invoke(...)方法主要通过内部 feign.Client 类型的 client 成员实例完成远程 URL 请求执行和获取远程结果。MethodHandler通过什么来完成请求主要由eign.Client 客户端类型决定（比如JDK自带HTTP工具，又或者是HttpClient等等），不同的类型完成 URL 请求处理的具体方式不同。

默认的基于 FeignInvocationHandler 调用处理器的执行流程在运行机制和调用性能上都满足不了生产环境的要求，大致原因有以下两点：

在远程调用过程中没有异常的熔断监测和恢复机制且没有用到高性能的 HTTP 连接池技术。

### 5.2 HystrixInvocationHandler执行流程

HystrixInvocationHandler 是具备 RPC 保护能力的调用处理器，它实现了 InvocationHandler 接口，对接口的 invoke(...)抽象方法的实现源码如下：

```java
final class HystrixInvocationHandler implements InvocationHandler {

    private final Map<Method, MethodHandler> dispatch;
    @Override
    public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
        //equals等方法处理略
        //创建一个HystrixCommand命令，对同步方法调用器进行封装。
        HystrixCommand<Object> hystrixCommand =
            new HystrixCommand<Object>(setterMethodMap.get(method)) {
            @Override
            protected Object run() throws Exception {
                try {
                    //invoke方法会创建HystrixCommand实例，对从dispatch中获取的SynchronousMethodHandler实例进行封装，然后对 RPC 方法实例 method 进行判断，判断是直接返回 hystrixCommand 命令实例，还是立即执行其 execute()方法
                    return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
                } catch (Exception e) {
                    throw e;
                } catch (Throwable t) {
                    throw (Error) t;
                }
            }

            @Override
            protected Object getFallback() {
                //省略异常回调源代码
            };

            //根据method的返回值类型，返回hystrixCommand或直接执行它的execute()方法
            if (Util.isDefault(method)) {
                return hystrixCommand.execute();
            } else if (isReturnsHystrixCommand(method)) {
                return hystrixCommand;
            } else if (isReturnsObservable(method)) {
                // Create a cold Observable
                return hystrixCommand.toObservable();
            } else if (isReturnsSingle(method)) {
                // Create a cold Observable as a Single
                return hystrixCommand.toObservable().toSingle();
            } else if (isReturnsCompletable(method)) {
                return hystrixCommand.toObservable().toCompletable();
            } else if (isReturnsCompletableFuture(method)) {
                return new ObservableCompletableFuture<>(hystrixCommand);
            }
            return hystrixCommand.execute();
        }
    }
```

HystrixCommand 具备熔断、隔离、回退等能力，如果它的 run()方法执行发生异常，就会执行 getFallback()失败回调方法。也就是说如果 MethodHandler 内的 RPC 调用出现异常，比如远程 server 宕机、网络延迟太大而导致请求超时、远程 server 来不及响应等，hystrixCommand 命令器就会调用失败回调方法 getFallback()返回回退结果。而 hystrixCommand 的 getFallback()方法最终会调用配置在 RPC 接口@FeignClient 注解的fallback 属性上的失败回退类中对应的回退方法，执行业务级别的失败回退处理。

使用HystrixInvocationHandler方法处理器进行远程调用，总体流程与使用默认的方法处理器FeignInvocationHandler进行远程调用大致是相同的

### 5.3 Feign的流程及特性

Spring Cloud Feign具有以下特性：

1. 可插拔的注解支持，包括feign注解和mvc注解
2. 支持可插拔的HTTP编解码器
3. 支持Hystrix和RPC保护机制
4. 支持ribbon负载均衡
5. 支持HTTP请求和响应的压缩

Feign的流程（一图流）：

![image-20220906105436989](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220906105436989.png)

## 六、Hystrix Feign动态代理创建流程

### 6.1 HystrixFeign.Builder 建造者容器实例

Feign 中默认的远程接口的 JDK 动态代理实例创建是通过 Feign.Builder 建造者容器实例的 target()方法来完成的。而target()方法的第一步是通过自身的 build()方法来构造一个ReflectiveFeign实例，第二步是通过反射式 Feign 实例的 newInstance()方法创建真正的 JDK Proxy 代理实例。

而HystrixFeign，有自己的建造者类 HystrixFeign.Builder 类，该类继承了 feign.Feign.Builder 默认的建造者，重写了它获得 Feign 实例的 build()方法，源码如下：

```java
public final class HystrixFeign {
  //创建新的构造者实例
  public static Builder builder() {
    return new Builder();
  }
  //内部类：HystrixFeign.Builder 类
  public static final class Builder extends Feign.Builder {

    private Contract contract = new Contract.Default();
    private SetterFactory setterFactory = new SetterFactory.Default();
      
    public <T> T target(Target<T> target, T fallback) {
      return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
          .newInstance(target);
    }

    public <T> T target(Target<T> target, FallbackFactory<? extends T> fallbackFactory) {
      return build(fallbackFactory).newInstance(target);
    }

	//其余方法略。。。
    @Override
    public Feign build() {
      return build(null);
    }
      //重载的 build 方法替换了基类的 invocationHandlerFactory然后调用基类的 build()方法建造一个 ReflectiveFeign（反射式 Feign）的实例
    Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override
        public InvocationHandler create(Target target,
                                        Map<Method, MethodHandler> dispatch) {
          //返回的是 HystrixInvocationHandler
          return new HystrixInvocationHandler(target, dispatch, setterFactory,
              nullableFallbackFactory);
        }
      });
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
    }
  }
}
```

也就是说，HystrixFeign.Builder类其实还是返回了一个ReflectiveFeign实例，只不过是将其中的invocationHandlerFactory和contract替换成了Hystrix自己的，这样的话，当ReflectiveFeign调用newInstance方法生成代理对象时，此时被替换过的处理器工厂将创建带RPC 保护功能的 HystrixInvocationHandler 类型的调用处理器。

### 6.2 配置HystrixFeign.Builder

使用 HystrixFeign.Builder 实例替换 feign.Feign.Builder 实例，在 FeignClientsConfiguration 中自动配置类的源码完成。部分源码如下：

```java
@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {
    
    //其余代码略。。。

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
    protected static class HystrixFeignConfiguration {
        @Bean
        @Scope("prototype")
        @ConditionalOnMissingBean
        @ConditionalOnProperty(name = "feign.hystrix.enabled")
        public Feign.Builder feignHystrixBuilder() {
            return HystrixFeign.builder();
        }
    }
}
```

从源码中我们可知，如果想创建一个 HystrixFeign.Builder 类型的实例必须同时满足两个条件：

1. 在类路径中同时存在 HystrixCommand.class 和 HystrixFeign.class 两个类。
2. 应用的配置文件中存在着 feign.hystrix.enabled 的配置项

## 七、远程调用的负载均衡Ribbon源码详解

### 7.1 Ribbon的使用

在实际使用中Ribbon通常与RestTemplate一起使用，使用方式如下，首先配置Ribbon：

```java
@Bean
@LoadBalanced//Ribbon负载均衡
public RestTemplate getRestTemplate() {
    return new RestTemplate();
}
```

然后在接口中注入RestTemplate使用即可：

```java
@Service
public class RibbonService {

    @Autowired
    private RestTemplate restTemplate;
    
    public String hi(String id) {
        return restTemplate.getForObject("http://xxx?id="+id,String.class);
    }
}
```

我们下面来看一下@LoadBalanced这个注解的源码：

```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
//被 @Inherited 注解修饰的注解，如果作用于某个类上，其子类是可以继承的该注解的。反之，如果一个注解没有被 @Inherited注解所修饰，那么他的作用范围只能是当前类，其子类是不能被继承的
@Inherited
@Qualifier
//Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient.
public @interface LoadBalanced {
}
```

根据源码注释可知，此类的作用是将RestTemplate bean标记为配置为使用LoadBalancerClient。

### 7.2 RestTemplate绑定拦截器

我们接着往下看，有@LoadBalanced注解，肯定就有支持这个注解工作的类，我们这时就可以从Spring自动装配入手，我们在ribbon的spring-cloud-netflix-ribbon的jar包下找spring.factorys文件：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

然后我们跟入RibbonAutoConfiguration配置类：

```java
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
//负载均衡要基于注册中心来做，所以RibbonAutoConfiguration加载要在Eureka初始化完毕后进行
@AutoConfigureAfter(
		name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
		AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
		ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {

}
```

这里我们关注@AutoConfigureBefore注解，表示RibbonAutoConfiguration此类需要在其之前进行加载。因此我们关注一下LoadBalancerAutoConfiguration这个配置类，源码如下，此类中有三个需要理解的地方，这三个地方就是Ribbon如何给RestTemplate绑定ribbon的拦截器，我在下面进行了标注：

```java
@Configuration(proxyBeanMethods = false)
//只有当RestTemplate存在时才进行装配
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
	//1.这里将标注了@LoadBalanced的RestTemplate注解注入到下面的List集合
   @LoadBalanced
   @Autowired(required = false)
   private List<RestTemplate> restTemplates = Collections.emptyList();

   @Autowired(required = false)
   private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

   @Bean
   public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
         final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
       //3.通过下面的定制器给集合中的每一个RestTemplate添加拦截器
      return () -> restTemplateCustomizers.ifAvailable(customizers -> {
          //遍历注入的RestTemplate集合
         for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
             //通过定制器定制添加拦截器
            for (RestTemplateCustomizer customizer : customizers) {
               customizer.customize(restTemplate);
            }
         }
      });
   }
    
    //省略部分代码

   @Configuration(proxyBeanMethods = false)
   @ConditionalOnClass(RetryTemplate.class)
   public static class RetryInterceptorAutoConfiguration {

      @Bean
      @ConditionalOnMissingBean
      public RetryLoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRetryProperties properties,
            LoadBalancerRequestFactory requestFactory,
            LoadBalancedRetryFactory loadBalancedRetryFactory) {
         return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
               requestFactory, loadBalancedRetryFactory);
      }
	//2.定制器，此类作用是将loadBalancerInterceptor加入到restTemplate
      @Bean
      @ConditionalOnMissingBean
      public RestTemplateCustomizer restTemplateCustomizer(
            final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
         return restTemplate -> {
             //获取restTemplate全部拦截器的集合
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                  restTemplate.getInterceptors());
             //将loadBalancerInterceptor加入集合，然后再塞回去
            list.add(loadBalancerInterceptor);
            restTemplate.setInterceptors(list);
         };
      }
   }
}
```

### 7.3 Ribbon的拦截器

在看拦截器之前，我们先来看看刚才略过的RibbonAutoConfiguration，这个配置类会注入一些Ribbon常用的类，其中有两个类需要注意：

```java
//注入SpringClientFactory
@Bean
public SpringClientFactory springClientFactory() {
   SpringClientFactory factory = new SpringClientFactory();
   factory.setConfigurations(this.configurations);
   return factory;
}
//注入LoadBalancerClient
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
   return new RibbonLoadBalancerClient(springClientFactory());
}
```

其中SpringClientFactory的构造函数中引入了RibbonClientConfiguration

```java
public SpringClientFactory() {
   super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
}
```

在RibbonClientConfiguration注入了一些常用组件：

```java
public class RibbonClientConfiguration {
    
	@Bean
	@ConditionalOnMissingBean
	public IRule ribbonRule(IClientConfig config) {
		if (this.propertiesFactory.isSet(IRule.class, name)) {
			return this.propertiesFactory.get(IRule.class, config, name);
		}
        //默认轮询策略，其父类是PredicateBasedRule
		ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
		rule.initWithNiwsConfig(config);
		return rule;
	}
    @Bean
	@ConditionalOnMissingBean
	@SuppressWarnings("unchecked")
	public ServerList<Server> ribbonServerList(IClientConfig config) {
		if (this.propertiesFactory.isSet(ServerList.class, name)) {
			return this.propertiesFactory.get(ServerList.class, config, name);
		}
		ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
		serverList.initWithNiwsConfig(config);
		return serverList;
	}
    @Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
        //默认使用ZoneAwareLoadBalancer负载均衡器
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
    //略。。。
}
```

这些类在下面都会用到，在看完Ribbon的自动配置类都注入了哪些类之后，我们进入正题来看一下Ribbon的拦截器。在IDEA中可以使用Double Shift来进行搜索，我们搜索到此拦截器，然后重点关注一下拦截方法：

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

   private LoadBalancerClient loadBalancer;

   private LoadBalancerRequestFactory requestFactory;

   //构造函数略

   @Override
   public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
         final ClientHttpRequestExecution execution) throws IOException {
       //获取源URI比如：http://cloud-service-user/user/getUserList;其中cloud-service-user是我们的微服务名称
      final URI originalUri = request.getURI();
       //获取服务名，就是上面的cloud-service-user
      String serviceName = originalUri.getHost();
      Assert.state(serviceName != null,
            "Request URI does not contain a valid hostname: " + originalUri);
       //然后拦截器将工作交给了loadBalancer去执行
      return this.loadBalancer.execute(serviceName,
            this.requestFactory.createRequest(request, body, execution));
   }

}
```

从源码中我们可以看到Ribbon的拦截器将工作最后委托给了loadBalancer，而loadBalancer实际上是一个LoadBalancerClient，这个类在上面我们看到了是Ribbon自动配置类注入的。接下来我们就跟进execute方法中：

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {

   <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
  
   <T> T execute(String serviceId, ServiceInstance serviceInstance,
         LoadBalancerRequest<T> request) throws IOException;
  
   URI reconstructURI(ServiceInstance instance, URI original);

}
```

跟进去我们发现LoadBalancerClient是一个接口，然后我们继续进入他的实现类，如果有多个（选择Ribbon的实现类）：

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {
    //其余代码略。。。
	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
        //进入下面贴的execute重载方法
		return execute(serviceId, request, null);
	}
    
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
        //从容器中获取loadBalancer，该类是自动装配时注入的
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
        //获取服务实例
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
        //将服务实例封装成RibbonServer
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
		//继续执行请求
		return execute(serviceId, ribbonServer, request);
	}
}
```

上面源码中我们主要有以下几个关注点：

![image-20220909105420223](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220909105420223.png)

我们下面一一跟进：

1. 如何获取的loadBalancer，这点比较简单，我在注释中也标明了，springboot自动装配时装配了此类，在RibbonClientConfiguration配置类中。

2. 如何获取服务实例，我们可以跟进getServer方法

   ```java
   protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
      if (loadBalancer == null) {
         return null;
      }
      // Use 'default' on a null hint, or just pass it on?
      return loadBalancer.chooseServer(hint != null ? hint : "default");
   }
   ```

   这里我们发现，它调用了loadBalancer的chooseServer方法，然后我们跟进此方法，然后进入实现类，实现类我们选择ZoneAwareLoadBalancer实现类，因为在之前我们也了解了这是Ribbon自动装配时默认的loadBalancer，如下图：

   ![image-20220909110340182](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220909110340182.png)

   进入后，发现此类由调用了父类的方法：

   ```java
   @Override
   public Server chooseServer(Object key) {
       //这里一般只关心这里即可，略去的代码一般不会走到
       //原因是ZoneAwareLoadBalancer是为了适配亚马逊云做的，我们一般只会有一个Zone,也就是说下面if判断的size一般都等于1
       if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
           logger.debug("Zone aware logic disabled or there is only one zone");
           return super.chooseServer(key);
       }
   	//其余代码略。。。
   }
   ```

   我们继续跟进：

   ```java
   public Server chooseServer(Object key) {
       if (counter == null) {
           counter = createCounter();
       }
       counter.increment();
       if (rule == null) {
           return null;
       } else {
           try {
               //通过负载均衡规则进行选择
               return rule.choose(key);
           } catch (Exception e) {
               logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
               return null;
           }
       }
   }
   ```

   我们进入到接口，在实现类中我们选择PredicateBaseRule，因为我们自动装配的默认规则ZoneAvoidanceRule就是继承了这个类：

   ![image-20220909110504176](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220909110504176.png)

   我们查看它的choose方法，这里主要调用了chooseRoundRobinAfterFiltering方法，并通过lb.getAllServers()获取全部服务实例作为参数传入：

   ```java
   @Override
   public Server choose(Object key) {
       ILoadBalancer lb = getLoadBalancer();
       Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
       if (server.isPresent()) {
           return server.get();
       } else {
           return null;
       }       
   }
   ```

   我们只关心主线，因此我们跟进chooseRoundRobinAfterFiltering方法：

   ```java
   public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
       //获取符合条件的服务集合。Eligible：符合条件的
       List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
       if (eligible.size() == 0) {
           return Optional.absent();
       }
       //关键代码，用Optional包装选择出的实例
       return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
   }
   ```

   这里最关键的就是return返回的这句代码，用Optional包装的选择出来的实例。那么是怎么选择的呢，我们进入incrementAndGetModulo方法：

   ```java
   //此方法传入所有符合条件实例的总数
   private int incrementAndGetModulo(int modulo) {
       for (;;) {
           //获取当前服务实例的索引值
           int current = nextIndex.get();
           //通过求余的方式记录下一个索引值
           int next = (current + 1) % modulo;
           //通过CAS设置下一个索引值
           if (nextIndex.compareAndSet(current, next) && current < modulo)
               return current;
       }
   }
   ```

   到此就是获取服务实例的全过程

3. 如何发起、执行请求

   我们继续跟进execute方法：

   ```java
   @Override
   public <T> T execute(String serviceId, ServiceInstance serviceInstance,
   		LoadBalancerRequest<T> request) throws IOException {
   	Server server = null;
   	if (serviceInstance instanceof RibbonServer) {
   		server = ((RibbonServer) serviceInstance).getServer();
   	}
   	if (server == null) {
   		throw new IllegalStateException("No instances available for " + serviceId);
   	}
   	
   	RibbonLoadBalancerContext context = this.clientFactory
   			.getLoadBalancerContext(serviceId);
       //Ribbon的状态记录类
   	RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);
   
   	try {
           //发起、执行请求,这个request就是一直传进来的ruquest，调用apply方法就会根据Server实例封装调用RestTemplate进行HTTP请求
   		T returnVal = request.apply(serviceInstance);
           //记录返回值
   		statsRecorder.recordStats(returnVal);
   		return returnVal;
   	}
   	// catch IOException and rethrow so RestTemplate behaves correctly
   	catch (IOException ex) {
   		statsRecorder.recordStats(ex);
   		throw ex;
   	}
   	catch (Exception ex) {
   		statsRecorder.recordStats(ex);
   		ReflectionUtils.rethrowRuntimeException(ex);
   	}
   	return null;
   }
   ```

   我们可以查看execute方法传入的这个request，如下图：

   ![image-20220909113700892](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220909113700892.png)

   createRequest方法如下：

   ```java
   public LoadBalancerRequest<ClientHttpResponse> createRequest(
         final HttpRequest request, final byte[] body,
         final ClientHttpRequestExecution execution) {
      return instance -> {
         HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
               this.loadBalancer);
         if (this.transformers != null) {
            for (LoadBalancerRequestTransformer transformer : this.transformers) {
               serviceRequest = transformer.transformRequest(serviceRequest,
                     instance);
            }
         }
         return execution.execute(serviceRequest, body);
      };
   }
   ```

### 7.4 Ribbon的ServerList

上面我们介绍了Ribbon的拦截器，拦截器的主要作用就是将请求交给Ribbon，然后由ribbon去负载均衡发送请求，那么在chooseServer时的服务列表是从何而来，下面我们就来分析一下。

首先ServerList是在Ribbon自动装配的过程中被实现的：

```java
@Bean
@ConditionalOnMissingBean
@SuppressWarnings("unchecked")
public ServerList<Server> ribbonServerList(IClientConfig config) {
    //从配置中获取Server集合。这里是因为Ribbon不需要nacos或eureka也能使用
   if (this.propertiesFactory.isSet(ServerList.class, name)) {
      return this.propertiesFactory.get(ServerList.class, config, name);
   }
   ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
   serverList.initWithNiwsConfig(config);
    //这里一般情况下只是注入了一个空的ServerList，数据由后续进行注入
   return serverList;
}
```

接下来我们看一下LoadBalancer，因为在这里执行chooseServer方法时已经有了，所以我们看一下使用ServerList的地方，我们看一下LoadBalancer的自动装配：

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
      ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
      IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
   if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
      return this.propertiesFactory.get(ILoadBalancer.class, config, name);
   }
   return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
         serverListFilter, serverListUpdater);
}
```

我们跟进ZoneAwareLoadBalancer类的构造函数，会发现他会调用父类的构造函数：

```java
public ZoneAwareLoadBalancer(IClientConfig clientConfig, IRule rule,
                             IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,
                             ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping, serverList, filter, serverListUpdater);
}
```

我们继续跟进进入到了DynamicServerListLoadBalancer的构造方法：

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                     ServerList<T> serverList, ServerListFilter<T> filter,
                                     ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping);
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    restOfInit(clientConfig);
}
```

在这个构造方法中主要关注`restOfInit(...)`这个方法:

```java
void restOfInit(IClientConfig clientConfig) {
    boolean primeConnection = this.isEnablePrimingConnections();
    // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
    this.setEnablePrimingConnections(false);
    //启动发现实例
    enableAndInitLearnNewServersFeature();
	//更新服务列表，因为上面是延时启动，所以我们还需要立即启动一次
    updateListOfServers();
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections()
                .primeConnections(getReachableServers());
    }
    this.setEnablePrimingConnections(primeConnection);
    LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
}
```

我们进入到`enableAndInitLearnNewServersFeature();`方法：

```java
public void enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
    serverListUpdater.start(updateAction);
}
```

我们先来看一下这个updateAction是什么：

```java
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
    @Override
    public void doUpdate() {
        updateListOfServers();
    }
};
```

可以看到updateAction中有一个doUpdate()方法，且此方法会执行`updateListOfServers();`方法，在这里就会进行服务的拉取。代码如下：

~~~java
@VisibleForTesting
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        //拉取服务实例，nacos和eureka都实现了ServerList
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
~~~

然后我们跟进start方法：

![image-20220909162749927](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220909162749927.png)

我们这里选择第二个实现类，可以通过debug断点的方式进行确定。

> - 通过定时任务进行更新。由这个实现类 PollingServerListUpdater 做到的。
> - 利用 Eureka 的事件监听器来更新。由这个实现类 EurekaNotificationServerListUpdater 做到的。

```java
@Override
public synchronized void start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                if (!isActive.get()) {
                    if (scheduledFuture != null) {
                        scheduledFuture.cancel(true);
                    }
                    return;
                }
                try {
                    //调用doUpdate方法或者说是调用doUpdate中的updateListOfServers()
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };
		//启动延迟任务，每隔一定时间执行wrapperRunnable也就是执行updateListOfServers()
        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                wrapperRunnable,
                initialDelayMs,
                refreshIntervalMs,
                TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
```

总结，经过上面源码分析我们可以看到，Ribbon自己内部的服务实例缓存是通过定时任务，然后用nacos或eureka实现的ServerList中定时获取最新的服务列表。

这也是Ribbon与nacos或eureka结合使用的原理，也就是说核心就是这个ServerList接口：

![image-20220913092005015](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220913092005015.png)

## 八、Feign+Ribbon+Hystrix/Sentinel进行远程调用

### 8.1 调用流程

了解Feign和Ribbon之后我们看看一个Feign+Ribbon+Hystrix/Sentinel远程调用的具体流程：

![FeignInvoke20220912](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/FeignInvoke20220912.png)

### 8.2 Feign和Ribbon的整合分析

根据上面整体的远程调用流程图（可以根据上面的流程自己跟踪源码），我们可以看到Feign的LoadBalanceFeignClient客户端底层还是对Ribbon的封装和使用。这其中有一个关键的Feign和Ribbon整合的类就是CachingSpringLoadBalancerFactory，在LoadBalancerFeignClient#execute方法中有一句关键代码：

```java
return lbClient(clientName).executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
```

其中的lbClient方法就会调用lbClientFactory创建FeignLoadBalancer，源码如下：

~~~java
private FeignLoadBalancer lbClient(String clientName) {
    return this.lbClientFactory.create(clientName);
}
~~~

这个方法其实就是CachingSpringLoadBalancerFactory#create方法，我们来看一下源码：

```java
public FeignLoadBalancer create(String clientName) {
   FeignLoadBalancer client = this.cache.get(clientName);
   if (client != null) {
      return client;
   }
    //获取Ribbon相关的类
   IClientConfig config = this.factory.getClientConfig(clientName);
   ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
   ServerIntrospector serverIntrospector = this.factory.getInstance(clientName,
         ServerIntrospector.class);
   client = this.loadBalancedRetryFactory != null
         ? new RetryableFeignLoadBalancer(lb, config, serverIntrospector,
               this.loadBalancedRetryFactory)
         : new FeignLoadBalancer(lb, config, serverIntrospector);
   this.cache.put(clientName, client);
   return client;
}
```

这个方法先根据服务名从缓存中获取一个FeignLoadBalancer，获取不到就创建一个。创建的过程就是从每个服务对应的容器中获取到IClientConfig和ILoadBalancer，其中ILoadBalancer类就是Ribbon的负载均衡的核心类。

那么这个create方法最后返回的FeignLoadBalancer是什么呢？

```java
public class FeignLoadBalancer extends
    AbstractLoadBalancerAwareClient<FeignLoadBalancer.RibbonRequest, FeignLoadBalancer.RibbonResponse> {

    private final RibbonProperties ribbon;

    protected int connectTimeout;

    protected int readTimeout;

    protected IClientConfig clientConfig;

    protected ServerIntrospector serverIntrospector;

    public FeignLoadBalancer(ILoadBalancer lb, IClientConfig clientConfig,
                             ServerIntrospector serverIntrospector) {
        super(lb, clientConfig);
        this.setRetryHandler(RetryHandler.DEFAULT);
        this.clientConfig = clientConfig;
        this.ribbon = RibbonProperties.from(clientConfig);
        RibbonProperties ribbon = this.ribbon;
        this.connectTimeout = ribbon.getConnectTimeout();
        this.readTimeout = ribbon.getReadTimeout();
        this.serverIntrospector = serverIntrospector;
    }
}
```

FeignLoadBalancer继承自AbstractLoadBalancerAwareClient，这个类主要作用是通过ILoadBalancer组件获取一个Server，然后基于这个Server重构了URI，也就是将你的请求路径`http://服务名/demo/hello`转换成类似`http://127.0.0.1:8080/demo/hello`这种路径，也就是将原服务名替换成服务所在的某一台机器ip和端口，替换之后就交由子类实现的exceut方法来发送http请求。

AbstractLoadBalancerAwareClient#executeWithLoadBalancer(...)源码如下：

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
```

以上就是Feign和Ribbon的整合分析。
