# Spring Cloud 远程调用原理

# 一、RPC入门与代理模式

这里主要通过一个RPC远程调用的客户端实现类的入门案例，并通过代理模式进行改造，来初步了解一下Feign的基本原理。如果不了解代理模式及JDK代理可以看我的笔记[Java代理](https://www.cnblogs.com/yhr520/p/15601620.html)

> 此笔记所有代码都在此项目(LearnSpringCloud)路径[代码地址](https://github.com/Loserfromlazy/LearnSpringCloud/)下的**rpcDemo和rpcDemoProvider**这两个项目中

## 1.1 入门案例：简单的RPC实现类

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
@Service
public class RpcDemoClientImpl implements RpcDemoClient{
    final String contextPath = RpcConstant.CONTEXT;

    @Override
    public Result<Boolean> test() {
        //封装URI
        String uri = "/demo/test";
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
        String uri = "/demo/print?input={0}";
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

## 1.2 静态代理改造

通常来说我们在远程调用时会通过代理对目标对象进行委托，在客户端和目标对象起到中介作用。那么为什么要用代理呢？因为我们实际会通过代理类对委托类进行拓展，比如远程调用的负载均衡、熔断、重试功能都要通过代理实现。

静态代理进行RPC调用需要的类为：

1. 远程接口，比如上面的RpcDemoClient类
2. 真实委托类，比如上面的RpcDemoClientImpl类
3. 代理类，比如下面的ClientStaticProxy类

下面我们先用静态代理进行简单改造，为动态代理做个铺垫：

```java
@Slf4j
@Service
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

## 1.3 动态代理改造

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

# 二、模拟Feign的动态代理实现类

下面我们通过简单的模拟Feign的动态代理实现类，来了解一下Feign，为后面学习Feign的相关组件源码做铺垫。

2.1 模拟Feign的方法处理器MethodHandler

2.2 模拟Feign的调用处理器InvocationHandler

2.3 模拟Feign的RCP动态代理

2.4 测试远程调用

2.5 feign的弹性RPC客户端实现类



