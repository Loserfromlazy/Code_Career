# SpringCloud学习文档

## 一、系统架构演变

### 1.1 集中式架构

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%8D%95%E4%BD%93%E6%9E%B6%E6%9E%84png.png)

网站流量很小，一个应用将所有功能部署在一起，以减少部署节点和成本。

优点：系统开发速度快；维护成本低；适用于并发要求较低的系统

缺点：代码耦合高，维护困难；无法进行不同模块的针对性优化；无法水平拓展；单点容错率低，并发能力差。

### 1.2 垂直拆分

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%9E%82%E7%9B%B4%E6%8B%86%E5%88%86png.png)

当访问量增大，为了应对更高的并发和业务需求，对业务进行拆分。

优点：系统拆分实现类流量分担，解决了并发，可以针对不同模块进行优化，方便水平拓展负载均衡，容错率提高。

缺点：系统相互独立，会有很多重复的开发工作，影响效率

### 1.3 分布式服务

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%88%86%E5%B8%83%E5%BC%8F%E6%9E%B6%E6%9E%84png.png)

当垂直应用越来越多，应用交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端能更快速地相应多变的市场需求

优点：将基础服务抽取，系统间互相调用提高了代码的复用和开发效率

缺点：系统间耦合变高，调用关系错综复杂，难以维护。

### 1.4 面向服务（SOA）

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/SOA%E6%9E%B6%E6%9E%84.png)

SOA面向服务架构，包含多个服务服务之间通过相互依赖最终提供一系列的功能。一个服务通常以独立的形式存在与操作系统进程中。各个服务之间通过网络调用。

缺点：每个供应商提供的ESB产品有偏差，自身实现较为复杂；应用服务粒度较大，ESB集成整合所有服务和协议、数据转换使得运维、测试部署困难。所有服务都通过一个通路通信，直接降低了通信速度。

### 1.5 微服务

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84.png)

> API Gateway网关是一个服务器，是系统的唯一入口。为每个客户端提供一个定制API。API网关核心是，所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有的非业务功能。如它还可以具有其它职责，如身份验证、监控、负载均衡、缓存、请求分片与管理、静态响应处理。通常，网关提供RESTful/HTTP的方式访问服务。而服务端通过服务注册中心进行服务注册和管理。

**微服务架构**是使用**一套小服务来开发单个应用的方式或途径**，每个服务基于单一业务能力构建，运行在自己的进程中，并使用轻量级机制通信，通常是HTTP API，并能够通过自动化部署机制来独立部署。这些服务可以使用不同的编程语言实现，以及不同数据存储技术，并保持最低限度的集中式管理。

## 二、服务调用方式

### 2.1 RPC和HTTP

无论是微服务还是SOA，都面临着服务间的远程调用。

常见的远程调用方式有以下2种：

**RPC**：Remote Produce Call远程过程调用，**RPC基于Socket，工作在会话层。自定义数据格式**，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型代表。

**Http**：http其实是**一种网络传输协议，基于TCP，工作在应用层，规定了数据传输的格式**。现在客户端浏览器与服务端通信基本都是采用Http协议，也可以用来进行远程服务调用。缺点是消息封装臃肿，优势是对服务的提供和调用方没有任何技术限定，自由灵活，更符合微服务理念。

> **区别**：RPC的机制是根据语言的API（language API）来定义的，而不是根据基于网络的应用来定义的。如果公司全部采用Java技术栈，那么使用Dubbo作为微服务架构是一个不错的选择。相反，如果公司的技术栈多样化，而且你更青睐Spring家族，那么Spring Cloud搭建微服务是不二之选。

### 2.2 Http客户端工具

既然微服务选择了Http，那么我们就需要考虑自己来实现对请求和响应的处理。不过开源世界已经有很多的http客户端工具，能够帮助我们做这些事情，例如：

HttpClient

OKHttp

URLConnection

不过这些不同的客户端，API各不相同。而Spring也有对http的客户端进行封装，提供了工具类叫RestTemplate。

### 2.3 Spring的RestTemplate

Spring提供了一个RestTemplate模板工具类，对基于Http的客户端进行了封装，并且实现了对象与json的序列化和

反序列化，非常方便。RestTemplate并没有限定Http的客户端类型，而是进行了抽象，目前常用的3种都有支持：HttpClient、OkHttp、JDK原生的URLConnection（默认的）

## 三、SpringCloud&SpringCloudAlibaba简介

**Spring Cloud**是Spring旗下的项目之一，Spring最擅长的就是集成，把世界上最好的框架拿过来，集成到自己的项目中。Spring Cloud也是一样，它将现在非常流行的一些技术整合到一起，实现了诸如：配置管理，服务发现，智能路由，负载均衡，熔断器，控制总线，集群状态等功能；协调分布式环境中各个系统，为各类服务提供模板性配置。其主要涉及的组件包括：

Eureka：注册中心

Zuul、Gateway：服务网关

Ribbon：负载均衡

Feign：服务调用

Hystrix或Resilience4j：熔断器

**Spring Cloud Alibaba** 旨在为微服务开发提供一站式解决方案。该项目包含了开发分布式应用和服务所需的组件，让开发者可以使用Spring Cloud编程模型轻松开发分布式应用。

使用Spring Cloud Alibaba，您只需要添加一些注解和配置，您就可以在您的应用中使用阿里巴巴的分布式解决方案，并通过阿里巴巴中间件构建您自己的分布式系统。

Spring Cloud 阿里巴巴的特点：

1. **流量控制和服务降级**：支持WebServlet、WebFlux、OpenFeign、RestTemplate、Dubbo接入限流降级功能。可以在运行时通过控制台实时修改限流和降级规则，同时支持限流Metrics的监控。
2. **服务注册和发现**：可以注册服务，客户端可以使用 Spring 管理的 bean，自动集成 Ribbon 来发现实例。
3. **分布式配置**：支持分布式系统中的外化配置，配置改变时自动刷新。
4. **Rpc Service**：扩展 Spring Cloud 客户端 RestTemplate 和 OpenFeign 以支持调用 Dubbo RPC 服务。
5. **事件驱动**：支持构建与共享消息系统连接的高度可扩展的事件驱动微服务。
6. **分布式事务**：支持高性能、易用的分布式事务解决方案。
7. **阿里云对象存储**：海量、安全、低成本、高可靠的云存储服务。支持随时随地在任何应用程序中存储和访问任何类型的数据。
8. **阿里云SchedulerX**：精准、高可靠、高可用的定时作业调度服务，响应时间秒级。
9. **阿里云短信**：覆盖全球的短信服务，阿里短信提供便捷、高效、智能的通讯能力，帮助企业快速联系客户。

Spring Cloud Alibaba是阿里巴巴公司基于Spring Cloud标准实现一套微服务开发框架集合，它和Netflflix一样都是Spring Cloud微服务开发实现方案。

### 3.1 微服务场景模拟





## 四、服务发现Nacos

### 4.1 区别与选型

EureKa已停止更新，本文档只学习Nacos

> Nacos与Eureka均提供注册中心和服务治理功能，以下为两者差异和选型方案。
>
> **功能差异**
>
> | 模块     | Nacos | Eureka | 说明                                                         |
> | -------- | ----- | ------ | ------------------------------------------------------------ |
> | 注册中心 | 是    | 是     | 服务治理基本功能，负责服务中心化注册                         |
> | 配置中心 | 是    | `否`   | Eureka需要配合Config实现配置中心，且不提供管理界面           |
> | 动态刷新 | 是    | `否`   | Eureka需要配合MQ实现配置动态刷新，Nacos采用Netty保持TCP长连接实时推送 |
> | 可用区AZ | 是    | 是     | 对服务集群划分不同区域，实现区域隔离，并提供容灾自动切换     |
> | 分组     | 是    | `否`   | Nacos可用根据业务和环境进行分组管理                          |
> | 元数据   | 是    | 是     | 提供服务标签数据，例如环境或服务标识                         |
> | 权重     | 是    | `否`   | Nacos默认提供权重设置功能，调整承载流量压力                  |
> | 健康检查 | 是    | 是     | Nacos支持由客户端或服务端发起的健康检查，Eureka是由客户端发起心跳 |
> | 负载均衡 | 是    | 是     | 均提供负责均衡策略，Eureka采用Ribion                         |
> | 管理界面 | 是    | `否`   | Nacos支持对服务在线管理，Eureka只是预览服务状态              |
>
> **部署安装**
>
> | 模块     | Nacos    | Eureka                  | 说明                                   |
> | -------- | -------- | ----------------------- | -------------------------------------- |
> | MySql    | 是       | `否`                    | Nacos需要采用MySql进行数据进行持久化   |
> | MQ       | `否`     | 是                      | Eureka需要采用MQ进行配置中心刷新       |
> | 配置中心 | 是       | `否`                    | Eureka结合Config或者Consul实现配置中心 |
> | 配置文件 | 在线编辑 | 本地文件或者Git远程文件 | Eureka结合Config或者Consul             |
> | 集群     | 是       | 是                      | Nacos需要配置集群ip再启动              |
>
> **稳定及扩展性**
>
> | 模块     | Nacos    | Eureka  | 说明                                                         |
> | -------- | -------- | ------- | ------------------------------------------------------------ |
> | 版本     | 1.0.0    | 1.9.9   | Eureka2.0已停止开发,Nacos处于1.x-2.0开发                     |
> | 厂商     | 阿里巴巴 | Netflix | Netflix已长期用于生产,阿里刚起步                             |
> | 生产建议 | `否`     | 是      | Nacos0.8以前不可用于生产,建议生产采用Nacos1.0,便于节省配置中心集群和服务管理 |
> | 未来发展 | 是       | `否`    | Nacos 2.0主要关注在统一服务管理、服务共享及服务治理体系的开放的服务平台的建设 |

### 4.2 理解服务发现

在微服务架构中，整个系统会按职责能力划分为多个服务，通过服务间写作实现业务目标。这样在我们的代码中免不了要进行服务间的远程调用，服务的消费方要调用服务的生产方，为了完成一次请求，消费方需要知道服务生产方的网络位置(IP地址和端口号)。 

解决方案：可以通过读取配置文件的方式读取服务生产方网络位置。

新建maven工程，添加pom配置

~~~xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.3.3.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
~~~

新建provider子模块，添加依赖

~~~xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
~~~

创建配置文件

~~~yml
server:
  port: 56010
~~~

创建controller和启动类（略）

~~~java
@RestController
public class RestProviderController {

    @GetMapping("/service")
    public String service(){
        System.out.println("provider invoke");
        return "provider invoke";
    }

}
~~~

新建consume模块，依赖与provider相同

创建配置文件

~~~yml
server:
  port: 56020

provider:
  address: 127.0.0.1:56010
~~~

创建controller

~~~java
@RestController
public class RestConsumerController {

    @Value("${provider.address}")
    private String providerAddress;

    @GetMapping("/service")
    public String service(){
        RestTemplate restTemplate = new RestTemplate();
        String forObject = restTemplate.getForObject("http://" + providerAddress + "/service", String.class);
        return "consumer invoke |"+ forObject;
    }
}
~~~

访问localhost:56020/service输出`consumer invoke |provider invoke`

以上的解决方案虽然很完美，但是对于微服务却行不通，微服务部署在云环境，服务网络位置或许是动态分配的，每一个服务也许会有多个实例做负载均衡，基于以上问题，服务之间如何发现，服务如何管理就是服务发现的问题了。服务发现就是服务消费方通过服务发现中心发现服务提供方，从而进行远程调用。如图：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/nacos/nacos1.png)

服务实例本身不记录服务生产方的网络地址，所有服务实例都会包含服务发现客户端。

1. 每个服务启动后会向服务发现中心上报自己的网络位置，在服务发现中心形成一个服务注册表，服务注册表是核心部分，是包含所有服务实例的网络地址的数据库。
2. 服务发现客户端会定期从服务发现中心同步服务注册表，并缓存在客户端内。
3. 当需要对某服务进行请求时，服务实力通过改注册表，定位到目标服务网络地址，若目标服务存在多个网络地址，则使用负载均衡算法从多个服务实例中选择出一个，案后发出请求。

### 4.3 Nacos

Nacos是阿里的一个开源产品，它是针对微服务架构中的服务发现、配置管理、服务治理的综合型解决方案。

**安装Nacos**

安装前需要配置java和maven环境

下载安装包并解压

**启动服务器**

nacos默认端口是8848，进入安装程序的bin目录。

Linux启动

`sh startup.sh -m standalone`

Window启动

`cmd startup.cmd`或者双击运行文件

window若报错闪退可以编辑cmd文件第27行将集群启动修改为单机启动

默认用户名密码：nacos

localhost:8848/nacos

### 4.4 RESTFul服务发现

**测试环境**

这里使用SpringCloud Alibaba解决

1.服务发现客户端从服务发现中心获取服务列表

2.服务消费方通过负载均衡获取服务地址

在4.2理解服务发现创建的父工程中添加依赖管理

~~~xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.3.3.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.2.2.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
~~~

在服务提供和消费工程中添加依赖，此依赖的作用是服务发现

~~~xml
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
~~~

**服务注册**

在服务提供工程中配置nacos服务发现的相关配置

~~~yml
server:
  port: 56010
spring:
  application:
    name: nacos-restful-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #配置nacos地址
~~~

启动nacos，启动springboot服务，观察nacos服务列表注册成功

服务名称：每个服务在服务注册中心的标识，相当于Java中的类名

服务实例：网络中提供服务的实例，具有IP和端口，相当于Java中的对象，一个实例即为运行在服务器上的一个进程。

**服务发现**

配置文件

~~~yml
server:
  port: 56020
spring:
  application:
    name: nacos-restful-consumer
  cloud:
    nacos:
     discovery:
       server-addr: 127.0.0.1:8848
~~~

编写controller

~~~java
@RestController
public class RestConsumerController {
    
    private String serviceId = "nacos-restful-provider";
    
    @Autowired
    LoadBalancerClient loadBalancerClient;

    @GetMapping("/service")
    public String service(){
        RestTemplate restTemplate =new RestTemplate();
        ServiceInstance serviceInstance = loadBalancerClient.choose(serviceId);
        URI uri = serviceInstance.getUri();
        String forObject = restTemplate.getForObject(uri + "/servide", String.class);
        return "consumer invoke |"+ forObject;
    }
}
~~~

**负载均衡**

负载均衡就是将用户的请求（流量）通过一定的策略分摊在多个服务实例上执行，他是系统处理高并发、缓解网络压力和进行服务端扩容的重要手段之一。它分为服务端负载均衡和客户端负载均衡。

*服务器端负载均衡*

在负载均衡器中维护一个可用的服务实例清单，当客户端请求来临时，负载均衡服务器按照某种配置好的规则(**负载均衡算法**)从可用服务实例清单中选取其一去处理客户端的请求。这就是服务端负载均衡。 例如Nginx，通过Nginx进行负载均衡，客户端发送请求至Nginx，Nginx通过负载均衡算法，在多个服务器之间选择一个进行访问。即在服务器端再进行负载均衡算法分配。

*客户端服务负载均衡*

LoadBalancerClient就是一个客户端负载均衡器，具体使用的是Ribbon客户端负载均衡器。Ribbon在发送请求前通过负载均衡算法选择一个服务实例，然后进行访问，这是客户端负载均衡。即在客户端就进行负载均衡的分配。 

Ribbon是一个客户端负载均衡服务器，他的责任是从一组实例列表中挑选合适的实例

Ribbon核心组件IRule是负载均衡策略接口

## 五、配置中心Nacos  //todo

### 5.1 理解配置中心

应用程序在启动和运行时往往需要读取一些配置信息，配置基本上伴随着整个应用程序的生命周期，比如数据库连接参数、启动参数等。

配置特点：

- 配置是独立于程序的只读变量
  - 配置对于程序是只读的，程序通过读取配置来改变自己的行为，但是程序不应该去改变配置。
- 配置伴随应用的整个生命周期
  - 配置贯穿于应用的整个生命周期，应用在启动时通过读取配置来初始化，在运行时根据配置调整行为。
- 配置可以有多种加载方式
  - 常见的有程序内部hard code，配置文件，环境变量，启动参数，基于数据库等
- 配置需要治理
  - 同一个程序在不同的环境（开发测试生产）不同的集群（如不同的数据中心）经常有不同的配置，所以需要有完善的环境、集群配置管理。

在微服务体系下，当系统从一个单体应用被拆分成系统上一个个的服务节点后，配置文件也必须跟着迁移，这样配置就跟着分散了而且还有冗余。

配置中心的服务流程如下：

1. 用户在配置中心更新配置信息
2. 服务A和B及时得到配置更新通知，从配置中心获取配置。

一个合格的配置中心需要满足如下特性：

- 配置项容易读取和修改
- 分布式环境下应用配置的可管理性，即提供远程管理配置的能力
- 支持对配置的修改的检视以把控风险
- 可以查看配置修改的历史纪录
- 不同部署环境下应用配置的隔离性

### 5.2 Nacos配置管理

**发布配置**















