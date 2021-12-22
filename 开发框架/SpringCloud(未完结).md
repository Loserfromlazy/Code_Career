# SpringCloud学习文档

# 一、微服务架构

## 1.1 应用架构发展

**集中式架构**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%8D%95%E4%BD%93%E6%9E%B6%E6%9E%84png.png)

网站流量很小，一个应用将所有功能部署在一起，以减少部署节点和成本。

优点：系统开发速度快；维护成本低；适用于并发要求较低的系统

缺点：代码耦合高，维护困难；无法进行不同模块的针对性优化；无法水平拓展；单点容错率低，并发能力差。

**垂直拆分**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%9E%82%E7%9B%B4%E6%8B%86%E5%88%86png.png)

当访问量增大，为了应对更高的并发和业务需求，对业务进行拆分。

优点：系统拆分实现类流量分担，解决了并发，可以针对不同模块进行优化，方便水平拓展负载均衡，容错率提高。

缺点：系统相互独立，会有很多重复的开发工作，影响效率

**分布式服务**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%88%86%E5%B8%83%E5%BC%8F%E6%9E%B6%E6%9E%84png.png)

当垂直应用越来越多，应用交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端能更快速地相应多变的市场需求

优点：将基础服务抽取，系统间互相调用提高了代码的复用和开发效率

缺点：系统间耦合变高，调用关系错综复杂，难以维护。

**面向服务（SOA）**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/SOA%E6%9E%B6%E6%9E%84.png)

SOA面向服务架构，包含多个服务服务之间通过相互依赖最终提供一系列的功能。一个服务通常以独立的形式存在与操作系统进程中。各个服务之间通过网络调用。

缺点：每个供应商提供的ESB产品有偏差，自身实现较为复杂；应用服务粒度较大，ESB集成整合所有服务和协议、数据转换使得运维、测试部署困难。所有服务都通过一个通路通信，直接降低了通信速度。

**微服务**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84.png)

> API Gateway网关是一个服务器，是系统的唯一入口。为每个客户端提供一个定制API。API网关核心是，所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有的非业务功能。如它还可以具有其它职责，如身份验证、监控、负载均衡、缓存、请求分片与管理、静态响应处理。通常，网关提供RESTful/HTTP的方式访问服务。而服务端通过服务注册中心进行服务注册和管理。

**微服务架构**是使用**一套小服务来开发单个应用的方式或途径**，每个服务基于单一业务能力构建，运行在自己的进程中，并使用轻量级机制通信，通常是HTTP API，并能够通过自动化部署机制来独立部署。这些服务可以使用不同的编程语言实现，以及不同数据存储技术，并保持最低限度的集中式管理。

## 1.2 服务调用方式

**RPC和HTTP**

无论是微服务还是SOA，都面临着服务间的远程调用。

常见的远程调用方式有以下2种：

**RPC**：Remote Produce Call远程过程调用，**RPC基于Socket，工作在会话层。自定义数据格式**，速度快，效率高。早期的webservice，现在热门的dubbo，都是RPC的典型代表。

**Http**：http其实是**一种网络传输协议，基于TCP，工作在应用层，规定了数据传输的格式**。现在客户端浏览器与服务端通信基本都是采用Http协议，也可以用来进行远程服务调用。缺点是消息封装臃肿，优势是对服务的提供和调用方没有任何技术限定，自由灵活，更符合微服务理念。

> **区别**：RPC的机制是根据语言的API（language API）来定义的，而不是根据基于网络的应用来定义的。如果公司全部采用Java技术栈，那么使用Dubbo作为微服务架构是一个不错的选择。相反，如果公司的技术栈多样化，而且你更青睐Spring家族，那么Spring Cloud搭建微服务是不二之选。

**Http客户端工具**

既然微服务选择了Http，那么我们就需要考虑自己来实现对请求和响应的处理。不过开源世界已经有很多的http客户端工具，能够帮助我们做这些事情，例如：

HttpClient

OKHttp

URLConnection

不过这些不同的客户端，API各不相同。而Spring也有对http的客户端进行封装，提供了工具类叫RestTemplate。

**Spring的RestTemplate**

Spring提供了一个RestTemplate模板工具类，对基于Http的客户端进行了封装，并且实现了对象与json的序列化和

反序列化，非常方便。RestTemplate并没有限定Http的客户端类型，而是进行了抽象，目前常用的3种都有支持：HttpClient、OkHttp、JDK原生的URLConnection（默认的）

## 1.3 微服务中的一些概念

1. 服务注册和服务发现

   **服务注册**：服务提供者将所提供服务的信息（服务器IP和端⼝、服务访问协议等）注册/登记到注册中⼼。

   **服务发现**：服务消费者能够从注册中⼼获取到较为实时的服务列表，然后根究⼀定的策略选择⼀个服务访问。

2. 负载均衡

   负载均衡即将请求压⼒分配到多个服务器（应⽤服务器、数据库服务器等），以此来提⾼服务的性能、可靠性。

3. 熔断

   熔断即断路保护。微服务架构中，如果下游服务因访问压⼒过⼤⽽响应变慢或失败，上游服务为了保护系统整体可⽤性，可以暂时切断对下游服务的调⽤。这种牺牲局部，保全整体的措施就叫做熔断。

4. 链路追踪

   微服务架构越发流⾏，⼀个项⽬往往拆分成很多个服务，那么⼀次请求就需要涉及到很多个服务。不同的微服务可能是由不同的团队开发、可能使⽤不同的编程语⾔实现、整个项⽬也有可能部署在了很多服务器上（甚⾄百台、千台）横跨多个不同的数据中⼼。所谓链路追踪，就是对⼀次请求涉及的很多个服务链路进⾏⽇志记录、性能监控。

5. API网关

   微服务架构下，不同的微服务往往会有不同的访问地址，客户端可能需要调⽤多个服务的接⼝才能完成⼀个业务需求，如果让客户端直接与各个微服务通信可能出现以下问题：

   - 客户端需要调⽤不同的url地址，增加了维护调⽤难度
   - 在⼀定的场景下，也存在跨域请求的问题（前后端分离就会碰到跨域问题，原本我们在后端采⽤Cors就能解决，现在利⽤⽹关，那么就放在⽹关这层做好了）
   - 每个微服务都需要进⾏单独的身份认证

   那么，API⽹关就可以较好的统⼀处理上述问题，API请求调⽤统⼀接⼊API⽹关层，由⽹关转发请求。API⽹关更专注在安全、路由、流量等问题的处理上（微服务团队专注于处理业务逻辑即可），它的功能有：

   1. 统⼀接⼊——路由
   2. 安全防护——统⼀鉴权，负责⽹关访问身份认证验证，与“访问认证中⼼”通信，实际认证业务逻辑交移“访问认证中⼼”处理
   3. 黑白名单——实现通过IP地址控制禁⽌访问⽹关功能，控制访问
   4. 协议适配——实现通信协议校验、适配转换的功能
   5. 流量管控——限流
   6. 长短连接支持
   7. 容错能力（负载均衡）

# 二、Spring Cloud概述

## 2.1 什么是SpringCloud 

[百度百科]Spring Cloud是⼀系列框架的有序集合。它利⽤Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中⼼、消息总线、负载均衡、断路器、数据监控等，都可以⽤ Spring Boot的开发⻛格做到⼀键启动和部署。Spring Cloud并没有重复制造轮⼦，它只是将⽬前各家公司开发的⽐较成熟、经得起实际考验的服务框架组合起来，通Spring Boot⻛格进⾏再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了⼀套简单易懂、易部署和易维护的分布式系统开发⼯具包。

Spring Cloud 规范及实现意图要解决的问题其实就是微服务架构实施过程中存在的⼀些问题，⽐如微服务架构中的服务注册发现问题、⽹络问题（⽐如熔断场景）、统⼀认证安全授权问题、负载均衡问题、链路追踪等问题。

## 2.2 Spring Cloud架构

SpringCloud的核心组件按照发展可以分为第一代SpringCloud组件和第二代核心组件。

|                | 第一代(Netflix,SCN)       | 第二代(Alibaba,SCA)                     |
| -------------- | ------------------------- | --------------------------------------- |
| 注册中心       | Netflix Eureka            | 阿里 Nacos                              |
| 客户端负载均衡 | Netflix Ribbon            | 阿里 Dubbo LB、SpringCloud Loadbalancer |
| 熔断器         | Netflix Hystrix           | 阿里 Sentinel                           |
| 网关           | Netflix Zuul              | SpringCloud Gateway                     |
| 配置中心       | SpringCloud Config        | 阿里 Nacos、携程Apollo                  |
| 服务调用       | Netflix Feign             | 阿里 Dubbo RPC                          |
| 消息驱动       | SpringCloud Stream        |                                         |
| 链路追踪       | SpringCloud Sleuth/Zipkin |                                         |
|                |                           | 阿里巴巴Seata分布式事务                 |

![SpringCloud20211204](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/SpringCloud20211204.png)

Spring Cloud中的各组件协同⼯作，才能够⽀持⼀个完整的微服务架构。⽐如：

- 注册中⼼负责服务的注册与发现，很好将各服务连接起来
- API⽹关负责转发所有外来的请求
- 断路器负责监控服务之间的调⽤情况，连续多次失败进⾏熔断保护。
- 配置中⼼提供了统⼀的配置信息管理服务,可以实时的通知各个服务获取最新的配置信息

## 2.3 对dubbo和SpringBoot的关系

Dubbo是阿⾥巴巴公司开源的⼀个⾼性能优秀的服务框架，基于RPC调⽤，对于⽬前使⽤率较⾼的Spring Cloud Netflflix来说，它是基于HTTP的，所以效率上没有Dubbo⾼，但问题在于Dubbo体系的组件不全，不能够提供⼀站式解决⽅案，⽐如服务注册与发现需要借助于Zookeeper等实现，⽽Spring Cloud Netflflix则是真正的提供了⼀站式服务化解决⽅案，且有Spring⼤家族背景。

Spring Cloud 只是利⽤了Spring Boot 的特点，让我们能够快速的实现微服务组件开发，否则不使⽤Spring Boot的话，我们在使⽤Spring Cloud时，每⼀个组件的相关Jar包都需要我们⾃⼰导⼊配置以及需要开发⼈员考虑兼容性等各种情况。所以Spring Boot是我们快速把Spring Cloud微服务技术应⽤起来的⼀种⽅式。

# 三、微服务入门案例

我们在这部分模拟一个微服务之间的调用，在之后会一步一步使用SpringCloud组件对案例进行改造。

案例需求：在交友App中会有这样类似的一个功能，就是推荐”心动女生“，根据用户的心动条件，每天匹配一定数量的”心动女生“推荐给该用户，在推荐之前查询以下女生的资料是否是保密状态，如果是保密状态则不再推荐该用户。自动推荐功能在自动推荐微服务（微服务A）中，资料查询在个人资料微服务（微服务B）中，那么就涉及到服务之间的调用，这时A就是一个服务消费者，B就是一个服务提供者。

建表语句：

~~~mysql
DROP TABLE IF EXISTS `user_info`;
CREATE TABLE `user_info`  (
  `id` int(10) NOT NULL AUTO_INCREMENT,
  `name` varchar(40) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '姓名',
  `sex` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '性别',
  `birthday` date NULL DEFAULT NULL COMMENT '出生日期',
  `phone` varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '电话号码',
  `email` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '邮件',
  `status` varchar(80) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '状态',
  `head_pic` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '头像',
  `height` int(4) NULL DEFAULT NULL COMMENT '身高',
  `weight` int(4) NULL DEFAULT NULL COMMENT ' 体重',
  `isOpen` int(1) NULL DEFAULT NULL COMMENT ' 是否开放推荐自己简介0关闭1打开2资料不完整3从未设置',
  `isDel` tinyint(1) NULL DEFAULT NULL COMMENT '是否删除',
  `refuse_count` int(11) NULL DEFAULT NULL COMMENT '拒绝次数',
  `one_word` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '签名',
  `live_city` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '居住城市',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
~~~

创建完成后请手动建几个测试数据

## 3.1 父工程

根据以上需求，首先创建父工程，删除src，修改pom文件。导入springboot相关依赖，持久层框架使用mybatisplus。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.learn</groupId>
    <artifactId>MyGirl</artifactId>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>common</module>
        <module>user</module>
        <module>autodeliver</module>
    </modules>
    <packaging>pom</packaging>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.3.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
            <version>1.18.20</version>
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

</project>
```

然后创建三个子工程，继承父工程：如图所示

![image-20211208130932477](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208130932477.png)

## 3.2 Common工程

修改pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>MyGirl</artifactId>
        <groupId>com.learn</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>common</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.3.2</version>
        </dependency>
    </dependencies>
</project>
```

编写实体类

```java
@Data
@TableName
public class UserInfo {
    @TableId(value = "id",type = IdType.AUTO)
    private Integer id;
    private String name;
    private String sex;
    private Date birthday;
    private String phone;
    private String email;
    private String status;
    private String headPic;
    private Integer height;
    private Integer weight;
    @TableField("isOpen")
    private Integer open;
    @TableField("isDel")
    private Boolean del;
    private Integer refuseCount;
    private String oneWord;
    private String liveCity;
}
```

## 3.3 user工程

创建一个根据id查询user的一套mapper，service和controller方法。工程结构如下：

![image-20211208131326088](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208131326088.png)

Controller方法如下：

```java
@RestController
@RequestMapping("/user")
public class UserInfoController {

    @Autowired
    private UserInfoService userInfoService;

    @GetMapping("/findUserById")
    public UserInfo findUserById(@RequestParam("id") Integer id) {
        return userInfoService.findUserById(id);
    }
}
```

## 3.4 autodeliver工程

注入RestTemplate，创建controller调用user工程的方法

![image-20211208131651193](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208131651193.png)

```java
@RestController
@RequestMapping("/autoDeliver")
public class AutoDeliverController {

    @Autowired
    private RestTemplate restTemplate;


    @GetMapping("/findOpenStatusByUid")
    public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
        UserInfo forObject = restTemplate.getForObject("http://127.0.0.1:8090/user/findUserById?id=" + uid, UserInfo.class);
        System.out.println(forObject.getOpen());
        return forObject.getOpen();
    }
}
```

## 3.5 测试

测试user工程的方法

![image-20211208131831138](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208131831138.png)

测试autodeliver工程的方法，看是否能拿到数据

![image-20211208131941612](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208131941612.png)

测试结果发现结果为2跟上面的测试user方法时拿到的open数据相同测试完成。

## 3.6 此案例问题以及微服务改造思路

此案例目前使用RestTemplate来调用服务，但如果在分布式集群的环境下就会有问题发生。

问题：

1. url地址硬编码，不方便维护

2. 服务提供者现在只有一个实例，如果需要集群需要服务消费者自己实现负载均衡

3. 服务消费者不清楚服务提供者的状态。

4. 服务消费者调用服务提供者，如果出现故障会抛出异常很不友好

   例如服务提供者挂掉了

   ![image-20211208132648379](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208132648379.png)

   或者有异常发生

   ![image-20211208132756956](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208132756956.png)

5. RestTemplate是否可以像Dubbo那样优化，即像通过调用方法那样调用远程方法

6. 无法实现统一认证

7. 配置文件有很多修改麻烦

8. ......

以上的问题就是微服务可能出现的问题，在微服务中也需要进行解决。

在微服务中的解决思路：

1. 服务管理：自动注册与发现、状态管理
2. 服务负载均衡
3. 熔断
4. 远程调用
5. 网关
6. 统一认证
7. 集中式配置

在SpringCloud或者SpringCloudAlibaba都有对应的解决方案。

# 四、SpringCloud核心组件

> 由于种种原因，放弃Zuul的学习，网关直接学习GateWay。

## 4.1 Eureka服务注册中心

### 4.1.1 服务注册中心简介

服务注册中心，本质上是为了解耦服务提供者和服务消费者。

对于任何⼀个微服务，原则上都应存在或者⽀持多个提供者（⽐如某个微服务部署多个实例），这是由微服务的**分布式属性**决定的。更进⼀步，为了⽀持弹性扩缩容特性，⼀个微服务的提供者的数量和分布往往是动态变化的，也是⽆法预先确定的。因此，原本在单体应⽤阶段常⽤的静态LB机制就不再适⽤了，需要引⼊额外的组件来管理微服务提供者的注册与发现，⽽这个组件就是服务注册中⼼。

![注册中心20211208](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%8320211208.png)

分布式微服务架构中，服务注册中⼼⽤于存储服务提供者地址信息、服务发布相关的属性信息，消费者通过主动查询和被动通知的⽅式获取服务提供者的地址信息，⽽不再需要通过硬编码⽅式得到提供者的地址信息。消费者只需要知道当前系统发布了那些服务，⽽不需要知道服务具体存在于什么位置，这就是透明化路由。

**主流服务注册中心对比**

| 组件      | 语言 | CAP                        | 对外接口 |
| --------- | ---- | -------------------------- | -------- |
| Eureka    | Java | AP(自我保护机制，保证可用) | HTTP     |
| Consul    | Go   | CP                         | HTTP/DNS |
| Zookeeper | Java | CP                         | 客户端   |
| Nacos     | Java | CP/AP切换                  | HTTP     |

P：分区容错性；C：数据一致性；A高可用

### 4.1.2 Eureka架构

![Eureka架构20211208](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Eureka%E6%9E%B6%E6%9E%8420211208.png)

下图是官网架构：

![image-20211208144456102](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211208144456102.png)

Eureka 包含两个组件：Eureka Server 和 Eureka Client，Eureka Client是⼀个Java客户端，⽤于简化与Eureka Server的交互；Eureka Server提供服务发现的能⼒，各个微服务启动时，会通过Eureka Client向Eureka Server 进⾏注册⾃⼰的信息（例如⽹络信息），Eureka Server会存储该服务的信息。

图中us-east-1c、us-east-1d，us-east-1e代表不同的区也就是不同的机房。图中每⼀个Eureka Server都是⼀个集群。图中Application Service作为服务提供者向Eureka Server中注册服务，Eureka Server接受到注册事件会在集群和分区中进⾏数据同步，Application Client作为消费端（服务消费者）可以从Eureka Server中获取到服务注册信息，进⾏服务调⽤。微服务启动后，会周期性地向Eureka Server发送⼼跳（默认周期为30秒）以续约⾃⼰的信息。Eureka Server在⼀定时间内没有接收到某个微服务节点的⼼跳，EurekaServer将会注销该微服务节点（默认90秒）。每个Eureka Server同时也是Eureka Client，多个Eureka Server之间通过复制的⽅式完成服务注册列表的同步。Eureka Client会缓存Eureka Server中的信息。即使所有的Eureka Server节点都宕掉，服务消费者依然可以使⽤缓存中的信息找到服务提供者。

### 4.1.3 Eureka应用和集群

下面我们通过以下顺序完成Eureka的学习：

首先创建单实例EurekaServer，然后在搭建EurekaServer集群，之后我们改造入门案例注册到集群完成调用。

#### 创建单实例EurekaServer注册中心

在例子中创建eurekaserver8761工程，因为我们之后需要扩展集群，所以在项目名上加上端口号用来区分。

![image-20211210132912095](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211210132912095.png)

然后增加springcloud的eureka的依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

在父工程配置依赖的版本

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.RELEASE</version>
            <!-- <version>Hoxton.SR4</version> -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

> 如果是jdk9以后默认时不加载jaxb的所以需要手动加上。因为eureka需要用到。
>
> ~~~xml
> <!--引⼊Jaxb，开始-->
> <dependency>
>      <groupId>com.sun.xml.bind</groupId>
>      <artifactId>jaxb-core</artifactId>
>      <version>2.2.11</version>
> </dependency> 
> <dependency>
>      <groupId>javax.xml.bind</groupId>
>      <artifactId>jaxb-api</artifactId>
> </dependency> 
> <dependency>
>      <groupId>com.sun.xml.bind</groupId>
>      <artifactId>jaxb-impl</artifactId>
>      <version>2.2.11</version>
>  </dependency> 
> <dependency>
>      <groupId>org.glassfish.jaxb</groupId>
>      <artifactId>jaxb-runtime</artifactId>
>      <version>2.2.10-b140310.1920</version>
> </dependency> 
> <dependency>
>      <groupId>javax.activation</groupId>
>      <artifactId>activation</artifactId>
>      <version>1.1.1</version>
> </dependency>
> <!--引⼊Jaxb，结束-->
> ~~~

编写启动类和配置文件：

```yaml
server:
  port: 8761
spring:
  application:
    name: cloud-eureka-server #应用名称，会在eureka中作为server的唯一ID
eureka:
  instance:
    hostname: localhost #当前eureka实例的主机名
  client:
    service-url: 
    # 客户端与EurekaServer交互的地址，如果是集群，也需要写其它Server的地址
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
    register-with-eureka: false # 自己就是服务不需要注册自己
    fetch-registry: false #从Eureka Server获取服务信息,默认为true，置为false。自己就是服务不需要
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class,args);
    }
}
```

启动启动类访问127.0.0.1:8761看到以下页面说明EurekaServer发布成功。

![image-20211210133341167](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211210133341167.png)

#### 搭建Eureka Server HA高可用集群

互联网应用中服务实例很少单个，因为如果只有一个实例，如果他挂掉而且微服务消费者的本地缓存列表也不可用，那么整个系统都会受到影响。

在生产环境中，我们可以配置Eureka Server集群，它的集群通过P2P通信的方式共享服务注册表，我们可以在这里开启两台Server来搭建集群。集群的架构与单体类似：

![Eureka集群架构20211210](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Eureka%E9%9B%86%E7%BE%A4%E6%9E%B6%E6%9E%8420211210.png)

下面进行集群配置:

由于是在本地进⾏测试，很难模拟多主机的情况，Eureka配置server集群时需要执⾏host地址。 所以需要修改个⼈电脑中host地址。

~~~
127.0.0.1       CloudEurekaServerA
127.0.0.1       CloudEurekaServerB
~~~

因为eureka server本身也可以看作一个客户端，所以配置文件这样配即可，（可以看作是在其他server中注册自己）

```yaml
server:
  port: 8761
spring:
  application:
    name: cloud-eureka-server #应用名称，会在eureka中作为server的唯一ID
eureka:
  instance:
    hostname: CloudEurekaServerA #当前eureka实例的主机名
  client:
    service-url:
      # 客户端与EurekaServer交互的地址，如果是集群，也需要写其它Server的地址
      # 集群模式下，指向其他的Eureka Server ，如果有更多的实例逗号拼接即可
      defaultZone: http://CloudEurekaServerB:8762/eureka/
    register-with-eureka: true # 集群模式下可以改成true
    fetch-registry: true #集群模式下可以改成true
```

```yaml
server:
  port: 8762
spring:
  application:
    name: cloud-eureka-server #应用名称，会在eureka中作为server的唯一ID
eureka:
  instance:
    hostname: CloudEurekaServerB #当前eureka实例的主机名
  client:
    service-url:
      # 客户端与EurekaServer交互的地址，如果是集群，也需要写其它Server的地址
      defaultZone: http://CloudEurekaServerA:8761/eureka/
    register-with-eureka: true
    fetch-registry: true
```

启动两个服务，http://CloudEurekaServerA:8761和http://CloudEurekaServerB:8762

可以看到两个服务的信息，且实例数为2，代表配置成功。

![image-20211210144312988](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211210144312988.png)

#### 微服务注册到Eureka Server集群

在入门案例中有user工程作为服务提供者，autodeliver作为服务消费者，现在我们进行配置将服务提供者和服务消费者注册到Server集群。

**服务提供者User工程配置：**

首先在common工程中增加依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-commons</artifactId>
</dependency>
```

然后在user工程中增加依赖

```xml
<!--eureka client客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

然后在user工程中增加eureka配置,注释掉的配置可以不配，主要作用是修改实例名。

```yaml
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
#  instance:
#    #使⽤ip注册，否则会使⽤主机名注册了（此处考虑到对⽼版本的兼容，新版本经过实验都是ip）
#    prefer-ip-address: true
#    #⾃定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
#    instance-id: ${spring.cloud.client.ip.address}:${spring.application.name}:${server.port}:@project.version@
```

最后增加注解

```java
@SpringBootApplication
@MapperScan("com.learn.mapper")
//@EnableEurekaClient  //开启Eureka Client
@EnableDiscoveryClient  //开启注册中心客户端（通用型注解，比如注册到Eureka、Nacos等）
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class,args);
    }
}
```

打开Eureka Server看User工程（服务提供者）是否能注册成功，如图现在已经注册成功

![image-20211213161010856](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211213161010856.png)

> 如果想集群可以参照以上配置完成User工程的集群。
>

**服务消费者AutoDeliver工程配置：**

增加依赖

~~~xml
<!--eureka client客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
~~~

编写配置文件：

```yaml
server:
  port: 8070
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
#  instance:
#    #使⽤ip注册，否则会使⽤主机名注册了（此处考虑到对⽼版本的兼容，新版本经过实验都是ip）
#    prefer-ip-address: true
#    #⾃定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
#    instance-id: ${spring.cloud.client.ip.address}:${spring.application.name}:${server.port}:@project.version@
spring:
  application:
    name: autodeliver
```

启动类增加注解：

```java
@SpringBootApplication(exclude= {DataSourceAutoConfiguration.class})
@EnableDiscoveryClient
public class AutoDeliverApplication {
    public static void main(String[] args) {
        SpringApplication.run(AutoDeliverApplication.class,args);
    }

    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
}
```

然后我们启动工程看是否注册成功

![image-20211213163332117](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211213163332117.png)

> 如果想集群可以参照以上配置完成User工程的集群。
>

#### 服务消费者调用服务提供者

改造Controller：

```java
@Autowired
private DiscoveryClient discoveryClient;

@GetMapping("/findOpenStatusByUid")
public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
    /*1.从Eureka Server中获取关注的那个服务的实力信息*/
    List<ServiceInstance> user = discoveryClient.getInstances("user");
    /*2.如果有多个实例选择一个来使用(负载均衡)*/
    ServiceInstance serviceInstance = user.get(0);
    /*3.从元数据信息获取host port*/
    String host = serviceInstance.getHost();
    int port = serviceInstance.getPort();
    String url = "http://"+host+":"+port+"/user/findUserById?id="+uid;
    System.out.println("############URL##########:"+url);
    UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
    return forObject.getOpen();
}
```

调用查看结果：

![image-20211213164822717](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211213164822717.png)

### 4.1.4 Eureka的元数据

Eureka的元数据有两种：标准元数据和自定义元数据

**标准元数据：**主机名、IP地址、端⼝号等信息，这些信息都会被发布在服务注册表中，⽤于服务之间的调⽤。

**⾃定义元数据：**可以使⽤eureka.instance.metadata-map配置，符合KEY/VALUE的存储格式。这 些元数据可以在远程客户端中访问。在程序中可以使⽤DiscoveryClient 获取指定微服务的所有元数据信息。自定义元数据如下：

~~~yml
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
  instance:
    metadata-map:
      testkey: testvalue
      myname: haha
~~~

### 4.1.5 Eureka的客户端

服务提供者（也是Eureka客户端）要向EurekaServer注册服务，并完成服务续约等⼯作。

**服务注册（服务提供者）**

1. 当我们导⼊了eureka-client依赖坐标，配置Eureka服务注册中心地址
2. 服务在启动时会向注册中心发起注册请求，携带服务元数据信息
3. Eureka注册中心会把服务的信息保存在Map中。

**服务续约（服务提供者）**

每隔30s会向注册中心续约（心跳）一次，如果没有心跳，租约会在90s后到期，然后服务会被失效。

~~~yml
eureka: 
 instance:
   #发心跳间隔时间，默认30s
   lease-renewal-interval-in-seconds: 30
   #租约到期服务失效时间，默认90s
   lease-expiration-duration-in-seconds: 90
~~~

**获取服务列表（服务消费者）**

每隔30秒服务会从注册中⼼中拉取⼀份服务列表，这个时间可以通过配置修改。往往不需要我们调整。

~~~yml
eureka: 
 instance: 
  #每隔多久拉取一次服务
  registry-fetch-interval-seconds: 30
~~~

### 4.1.6 Eureka的服务端

**服务下线**

当服务正常关闭操作时，会发送服务下线的REST请求给EurekaServer。

服务中⼼接受到请求后，将该服务置为下线状态。

**失效剔除**

Eureka Server会定时（间隔值是eureka.server.eviction-interval-timer-in-ms，默认60s）进⾏检查，如果发现实例在在⼀定时间（此值由客户端设置的eureka.instance.lease-expiration-duration-in-seconds定义，默认值为90s）内没有收到⼼跳，则会注销此实例。

**自我保护：**

服务提供者 —> 注册中⼼定期的续约（服务提供者和注册中⼼通信），假如服务提供者和注册中⼼之间的⽹络有点问题，不代表服务提供者不可⽤，不代表服务消费者⽆法访问服务提供者。

如果在15分钟内超过85%的客户端节点都没有正常的⼼跳，那么Eureka就认为客户端与注册中⼼出现了⽹络故障，Eureka Server⾃动进⼊⾃我保护机制。

> 为什么会有⾃我保护机制？
>
> 默认情况下，如果Eureka Server在⼀定时间内（默认90秒）没有接收到某个微服务实例的⼼跳，Eureka Server将会移除该实例。但是当⽹络分区故障发⽣时，微服务与Eureka Server之间⽆法正常通信，⽽微服务本身是正常运⾏的，此时不应该移除这个微服务，所以引⼊了⾃我保护机制。

当处于⾃我保护模式时

1. 不会剔除任何服务实例（可能是服务提供者和EurekaServer之间⽹络问题），保证了⼤多数服务依然可⽤
2. Eureka Server仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上，保证当前节点依然可⽤，当⽹络稳定时，当前Eureka Server新的注册信息会被同步到其它节点中。
3. 在Eureka Server⼯程中通过eureka.server.enable-self-preservation配置可⽤关停⾃我保护，默认值是打开。

## 4.2 Ribbon负载均衡

负载均衡⼀般分为**服务器端负载均衡**和**客户端负载均衡**

所谓**服务器端负载均衡**，⽐如Nginx、F5这些，请求到达服务器之后由这些负载均衡器根据⼀定的算法将请求路由到⽬标服务器处理。

所谓**客户端负载均衡**，⽐如我们要说的Ribbon，服务消费者客户端会有⼀个服务器地址列表，调⽤⽅在请求前通过⼀定的负载均衡算法选择⼀个服务器进⾏访问，负载均衡算法的执⾏是在请求客户端进⾏。

Ribbon是Netflflix发布的负载均衡器。Eureka⼀般配合Ribbon进⾏使⽤，Ribbon利⽤从Eureka中读取到服务信息，在调⽤服务提供者提供的服务时，会根据⼀定的算法进⾏负载。

### 4.2.1 Ribbon的应用

在服务消费者中，也就是入门案例中的autodeliver工程进行改造。eureka-client中会引入Ribbon的相关jar包，所以我们不需要引入jar包，直接使用。

测试前需要复制一份user工程做集群以便查看负载均衡效果。

> 实际开发中，我们只需要部署在不同服务器即可，不需要复制项目来做集群。

![image-20211214111153065](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211214111153065.png)

下面进行Ribbon的使用，使用时，只需要加一个注解即可：

```java
@SpringBootApplication(exclude= {DataSourceAutoConfiguration.class})
@EnableDiscoveryClient
public class AutoDeliverApplication {
    public static void main(String[] args) {
        SpringApplication.run(AutoDeliverApplication.class,args);
    }

    @Bean
    //Ribbon负载均衡
    @LoadBalanced
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
}
```

修改controller：

```java
@GetMapping("/findOpenStatusByUid")
public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
    /*使用Ribbon负载均衡*/
    String url = "http://user/user/findUserById?id="+uid;
    System.out.println("############URL##########:"+url);
    UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
    return forObject.getOpen();
}
```

![image-20211214101914021](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211214101914021.png)



> 可以在服务提供者即User工程中返回当前的端口号，这样可以更好的查看负载均衡的效果。也可以加上日志如以下代码：
>
> ```java
> @GetMapping("/findUserById")
> public UserInfo findUserById(@RequestParam("id") Integer id) {
>     System.out.println("当前是8090，目前请求到这里了"+new Date());
>     return userInfoService.findUserById(id);
> }
> ```

### 4.2.2 Ribbon负载均衡的策略

Ribbon内置了多种负载均衡策略，内部负责复杂均衡的顶级接⼝com.netflflix.loadbalancer.IRule

![image-20211214103817604](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211214103817604.png)

| 策略                                            | 描述                                                         |
| ----------------------------------------------- | ------------------------------------------------------------ |
| RoundRobinRule：轮询策略                        | 默认超过10次获取到的server都不可⽤，会返回⼀个空的server     |
| RandomRule：随机策略                            | 如果随机到的server为null或者不可⽤的话，会while不停的循环选取 |
| RetryRule：重试策略                             | ⼀定时限内循环重试。默认继承RoundRobinRule，也⽀持⾃定义注⼊，RetryRule会在每次选取之后，对选举的server进⾏判断，是否为null，是否alive，并且在500ms内会不停的选取判断。⽽RoundRobinRule失效的策略是超过10次，RandomRule是没有失效时间的概念，只要serverList没都挂。 |
| BestAvailableRule：最小连接数策略               | 遍历serverList，选取出可⽤的且连接数最⼩的⼀个server。该算法⾥⾯有⼀个LoadBalancerStats的成员变量，会存储所有server的运⾏状况和连接数。如果选取到的server为null，那么会调⽤RoundRobinRule重新选取。 |
| AvailabilityFilteringRule：可⽤过滤策略         | 扩展了轮询策略，会先通过默认的轮询选取⼀个server，再去判断该server是否超时可⽤，当前连接数是否超限，都成功再返回 |
| ZoneAvoidanceRule：区域权衡策略**（默认策略）** | 扩展了轮询策略，继承了2个过滤器：ZoneAvoidancePredicate和AvailabilityPredicate，除了过滤超时和链接数过多的server，还会过滤掉不符合要求的zone区域⾥⾯的所有节点，AWS --ZONE 在⼀个区域/机房内的服务实例中轮询 |

## 4.3 Hystrix熔断器

### 4.3.1 微服务中的雪崩效应及解决方案

微服务中一个请求可能要多个微服务接口才能实现，会形成复杂的调用链路

![雪崩效应20211214](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E9%9B%AA%E5%B4%A9%E6%95%88%E5%BA%9420211214.png)

如上图，站在微服务B的角度上来说，上游微服务对它的调用是扇入，它对下游微服务的调用叫扇出。

扇⼊：代表着该微服务被调⽤的次数，扇⼊⼤，说明该模块复⽤性好

扇出：该微服务调⽤其他微服务的个数，扇出⼤，说明业务逻辑复杂

在微服务架构中，⼀个应⽤可能会有多个微服务组成，微服务之间的数据交互通过远程过程调⽤完成。这就带来⼀个问题，假设微服务A调⽤微服务B和微服务C，微服务B和微服务C⼜调⽤其它的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调⽤响应时间过⻓或者不可⽤，对微服务A的调⽤就会占⽤越来越多的系统资源，进⽽引起系统崩溃，所谓的“雪崩效应”。

如图中所示，最下游**微服务C**响应时间过⻓，⼤量请求阻塞，⼤量线程不会释放，会导致服务器资源耗尽，最终导致上游服务甚⾄整个系统瘫痪

解决雪崩效应主要有三种方法：

1. 服务熔断

   熔断机制是应对雪崩效应的⼀种微服务链路保护机制。当扇出链路的某个微服务不可⽤或者响应时间太⻓时，熔断该节点微服务的调⽤，进⾏服务的降级，快速返回错误的响应信息。当检测到该节点微服务调⽤响应正常后，恢复调⽤链路。

   > 服务熔断重点在**断**，切断对下游服务的调⽤
   >
   > 服务熔断和服务降级往往是⼀起使⽤的，Hystrix就是这样。

2. 服务降级

   通俗讲就是整体资源不够⽤了，先将⼀些不关紧的服务停掉（调⽤我的时候，给你返回⼀个预留的值，也叫做**兜底数据**），待渡过难关⾼峰过去，再把那些服务打开。服务降级⼀般是从整体考虑，就是当某个服务熔断之后，服务器将不再被调⽤，此刻客户端可以⾃⼰准备⼀个本地的fallback回调，返回⼀个缺省值，这样做，虽然服务⽔平下降，但好⽍可⽤，⽐直接挂掉要强。

3. 服务限流

   服务降级是当服务出问题或者影响到核⼼流程的性能时，暂时将服务屏蔽掉，待⾼峰或者问题解决后再打开；但是有些场景并不能⽤服务降级来解决，⽐如秒杀业务这样的核⼼功能，这个时候可以结合服务限流来限制这些场景的并发/请求量。

   > 限流措施也很多，⽐如
   >
   > - 限制总并发数（⽐如数据库连接池、线程池）
   > - 限制瞬时并发数（如nginx限制瞬时并发连接数）
   > - 限制时间窗⼝内的平均速率（如Guava的RateLimiter、nginx的limit_req模块，
   > - 限制每秒的平均速率）
   > - 限制远程接⼝调⽤速率、限制MQ的消费速率等

### 4.3.2 Hystrix简介

是由Netflflix开源的⼀个延迟和容错库，⽤于隔离访问远程系统、服务或者第三⽅库，防⽌级联失败，从⽽提升系统的可⽤性与容错性。Hystrix主要通过以下⼏点实现延迟和容错：

- 包裹请求：使⽤HystrixCommand包裹对依赖的调⽤逻辑。
- 跳闸机制：当某服务的错误率超过⼀定的阈值时，Hystrix可以跳闸，停⽌请求
  该服务⼀段时间。
- 资源隔离：Hystrix为每个依赖都维护了⼀个⼩型的线程池(舱壁模式)（或者信号量）。如果该线程池已满， 发往该依赖的请求就被⽴即拒绝，⽽不是排队等待，从⽽加速失败判定。
- 监控：Hystrix可以近乎实时地监控运⾏指标和配置的变化，例如成功、失败、超时、以及被拒绝 的请求等。
- 回退机制：当请求失败、超时、被拒绝，或当断路器打开时，执⾏回退逻辑。回退逻辑由开发⼈员 ⾃⾏提供，例如返回⼀个缺省值。
- ⾃我修复：断路器打开⼀段时间后，会⾃动进⼊“半开”状态。

### 4.3.3 Hystrix应用

如果我们的User工程长时间没有响应，服务消费者——自动投递微服务将快速失败给用户提示。下面我们进行修改：

首先在user工程（user8090和user8091修改一个即可，这样可以模拟两个服务实例其中一个坏掉）中进行线程休眠来模拟访问超时：

```java
@GetMapping("/findUserById")
public UserInfo findUserById(@RequestParam("id") Integer id) {
    //模拟访问超时
    try {
        Thread.sleep(10000);
    }catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("当前是8091，目前请求到这里了"+new Date());
    return userInfoService.findUserById(id);
}
```

然后在autodeliver工程中导入hystrix依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

启动类增加注解

```java
@SpringBootApplication(exclude= {DataSourceAutoConfiguration.class})
@EnableDiscoveryClient
//@EnableHystrix  //开启Hystrix功能
@EnableCircuitBreaker //开启熔断器功能
//@SpringCloudApplication //等价于@SpringBootApplication + @EnableDiscoveryClient + @EnableCircuitBreaker
public class AutoDeliverApplication {
    public static void main(String[] args) {
        SpringApplication.run(AutoDeliverApplication.class,args);
    }

    @Bean
    @LoadBalanced//Ribbon负载均衡
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
}
```

然后修改Controller代码

```java
//使用HystrixCommand注解进行熔断控制
@HystrixCommand(
        //commandProperties：熔断的一些细节属性配置其中的每一个属性都是HystrixProperty
        commandProperties = {
                //设置服务超时时间
                @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
        }
)
@GetMapping("/findOpenStatusByUid")
public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
    /*使用Ribbon负载均衡*/
    // String url = "http://"+host+":"+port+"/user/findUserById?id="+uid;
    String url = "http://user/user/findUserById?id="+uid;
    System.out.println("############URL##########:"+url);
    UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
    return forObject.getOpen();
}
```

> HystrixProperty的所有属性都在HystrixCommandProperties类中进行了配置。
>
> ![image-20211215113826902](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215113826902.png)

我们调用方法，因为我们只在一个服务提供者增加了休眠，所以我们会发现请求时一个会成功而另一个会超时熔断返回错误信息。

![image-20211215131432443](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215131432443.png)

服务提供者处理超时熔断会返回错误信息，或者抛出异常信息，这些信息都会返回到服务消费者这里，但是有时候我们不想将收集到的异常或者错误信息抛到它的上游，比如以下业务场景：

![example20211215](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/example20211215.png)

用户注册领取优惠卷，但优惠卷微服务返回了错误或异常信息，这时注册微服务直接返回给上游用户微服务很不好，所以我们可以返回一个默认数据（服务降级）。我们可以如下配置：

```java
@RestController
@RequestMapping("/autoDeliver")
public class AutoDeliverController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private DiscoveryClient discoveryClient;

    //使用HystrixCommand注解进行熔断控制
    @HystrixCommand(
            //commandProperties：熔断的一些细节属性配置其中的每一个属性都是HystrixProperty
            commandProperties = {
                    //设置服务超时时间
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
            },
            fallbackMethod = "myFallBack"
    )
    @GetMapping("/findOpenStatusByUid")
    public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
        /*使用Ribbon负载均衡*/
        // String url = "http://"+host+":"+port+"/user/findUserById?id="+uid;
        String url = "http://user/user/findUserById?id="+uid;
        System.out.println("############URL##########:"+url);
        UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
        return forObject.getOpen();
    }
    //回退方法，返回默认值
    public Integer myFallBack(Integer uid){
        return -1;//默认数据\兜底数据
    }
}
```

> 降级方法的形参和返回值必须与原方法相同
>
> 也可以在类上使⽤@DefaultProperties注解统⼀指定整个类中共⽤的降级⽅法

这时我们可以看到超时后会返回我们设置的-1

![image-20211215133314571](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215133314571.png)

### 4.3.4 Hystrix舱壁模式（线程池隔离策略）

默认Hystrix有一个线程池有10个线程，如果某一个请求比如下图的请求1发了十二个请求请求A服务，那么其他请求只能等待或者拒绝连接，比如请求2，这并不是B服务不可用，而是线程不够的原因。

![舱壁模式20211215](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E8%88%B1%E5%A3%81%E6%A8%A1%E5%BC%8F20211215.png)

为了避免问题服务请求过多导致正常服务⽆法访问，Hystrix 不是采⽤增加线程数，⽽是单独的为每⼀个控制⽅法创建⼀个线程池的⽅式，这种模式叫做“舱壁模式"，也是线程隔离的⼿段。如下图：

![舱壁模式二20211215](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E8%88%B1%E5%A3%81%E6%A8%A1%E5%BC%8F%E4%BA%8C20211215.png)

我们可以通过jps命令查看当前运行的Java进程，如图可以看到我们正在运行的五个进程：

![image-20211215144723971](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215144723971.png)

我们通过postman批量发请求发送20个请求，等待请求完成。

> postman的文件夹可以使用RUN collectin创建批量请求：
>
> ![image-20211215150737607](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215150737607.png)

然后我们可以使用jstack+pid 查看进程的线程信息可以通过grep过滤（windows下可以使用findstr代替）如下图：

![image-20211215150613760](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215150613760.png)

可以看到，虽然有20个请求，但是只有10个线程，所以如果不进⾏任何设置，所有熔断⽅法使⽤⼀个Hystrix线程池（10个线程），那么这样的话会因为我们的Hystrix的线程机制造成问题。

下面我们进行舱壁模式的修改，只需要设置线程池标识，并且唯一就可以每个方法使用自己的线程池，然后我们复制一个方法做对比：

```java
@RestController
@RequestMapping("/autoDeliver")
public class AutoDeliverController {

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private DiscoveryClient discoveryClient;


    @HystrixCommand(
            // 线程池标识，要保持唯一，不唯一的话就共用了
            threadPoolKey = "findOpenStatusByUid",
            // 线程池细节属性配置
            threadPoolProperties = {
                    @HystrixProperty(name="coreSize",value = "2"),// 线程数
                    @HystrixProperty(name="maxQueueSize",value="20") // 等待队列长度
            },
            commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
            },
            fallbackMethod = "myFallBack"
    )
    @GetMapping("/findOpenStatusByUid")
    public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
        String url = "http://user/user/findUserById?id="+uid;
        System.out.println("############URL##########:"+url);
        UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
        return forObject.getOpen();
    }

    @HystrixCommand(
            // 线程池标识，要保持唯一，不唯一的话就共用了
            threadPoolKey = "findOpenStatusByUid1",
            // 线程池细节属性配置
            threadPoolProperties = {
                    @HystrixProperty(name="coreSize",value = "2"),// 线程数
                    @HystrixProperty(name="maxQueueSize",value="20") // 等待队列长度
            },
            commandProperties = {
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
            },
            fallbackMethod = "myFallBack"
    )
    @GetMapping("/findOpenStatusByUid1")
    public Integer findOpenStatusByUid1(@RequestParam("uid") Integer uid){
        String url = "http://user/user/findUserById?id="+uid;
        System.out.println("############URL##########:"+url);
        UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
        return forObject.getOpen();
    }
    //回退方法，返回默认值
    public Integer myFallBack(Integer uid){
        return -1;//默认数据\兜底数据
    }
}
```

启动postman发10次请求，再查看线程情况，可以发现跟我们设置的是一致的，每个方法两个线程且不共用。

![image-20211215151911699](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215151911699.png)

### 4.3.5 Hystrix工作流程及配置

如下图，当出现调用问题开一个默认10s的时间窗进行判断，如果达到阈值则跳闸，跳闸后开一个默认5s的活动窗口判断远程服务是否调用成功，调用成功则再开一个时间窗口重复流程。

![Hystrix流程20211215](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Hystrix%E6%B5%81%E7%A8%8B20211215.png)

下面我们对上述条件进行配置

```java
@HystrixCommand(
        threadPoolKey = "findOpenStatusByUid",
        threadPoolProperties = {
                @HystrixProperty(name="coreSize",value = "2"),// 线程数
                @HystrixProperty(name="maxQueueSize",value="20") // 等待队列长度
        },
        commandProperties = {
                @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000"),
                //配置一个8s内请求次数达到2个失败率达到50%就跳闸的熔断器，跳闸后的活动窗口设置为3s
                @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds",value = "8000"),
                @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "2"),
                @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "50"),
                @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "3000")

        },
        fallbackMethod = "myFallBack"
)
@GetMapping("/findOpenStatusByUid")
public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
    String url = "http://user/user/findUserById?id="+uid;
    System.out.println("############URL##########:"+url);
    UserInfo forObject = restTemplate.getForObject(url, UserInfo.class);
    return forObject.getOpen();
}
```

下面我们对验证Hystrix的章台，首先我们在autodeliver工程的配置文件中增加健康检查状态的配置，可以使用`localhost:8070/actuator/health`查看状态

~~~yaml
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
   web:
    exposure:
     include: "*"
  # 暴露健康接⼝的细节
  endpoint:
   health:
    show-details: always
~~~

我们首先启动项目访问健康检查接口，查看hystrix的状态：

![image-20211215154932143](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215154932143.png)

然后我们跟之前一样使用postman执行5次请求，在请求过程中一直访问健康检查接口，可以看到它的跳闸状态，然后在活动窗口时间内可以自我修复。下图为跳闸状态：

![image-20211215155148461](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211215155148461.png)

这些配置也可以配置在配置文件中：我们将代码中的配置注释掉然后在配置文件中编写，重启项目再次验证即可。

```yaml
#Hystrix熔断策略
hystrix:
  command:
    default:
      circuitBreaker:
        # 强制打开熔断器，如果该属性设置为true，强制断路器进⼊打开状态，将会拒绝所有的请求。 默认false关闭的
        forceOpen: false
        # 触发熔断错误⽐例阈值，默认值50%
        errorThresholdPercentage: 50
        # 熔断后休眠时⻓，默认值5秒
        sleepWindowInMilliseconds: 3000
        # 熔断触发最⼩请求次数，默认值是20
        requestVolumeThreshold: 2
      execution:
        isolation:
         thread:
          # 熔断超时设置，默认为1秒
          timeoutInMilliseconds: 2000
```

## 4.4 Feign远程调用

### 4.4.1 Feign简介

之前服务消费者调用服务提供者时使用RestTemplate技术，但它存在不便，首先需要我们拼接URL，其次需要我们模板化的使用restTemplate。所以有了Feign来解决这个问题。

Feign是Netflflix开发的⼀个轻量RESTful的HTTP服务客户端（⽤它来发起请求，远程调⽤的），是以Java接口注解的⽅式调⽤Http请求，而不用像Java中通过封装HTTP请求报文的方式直接调用，Feign被⼴泛应⽤在Spring Cloud 的解决⽅案中。它类似于Dubbo，服务消费者拿到服务提供者的接⼝，然后像调⽤本地接口⽅法⼀样去调⽤，实际发出的是远程的请求。

SpringCloud对Feign进⾏了增强，使Feign⽀持了SpringMVC注解(OpenFeign)。

> Feign = RestTemplate + Ribbon + Hystrix

### 4.4.2 Feign的应用

**feign的应用只需要建立一个feignClient然后将服务提供者的接口放入feignClient即可**，下面结合入门案例进行演示。

之前是用RestTemplate+Ribbon的方式进行远程调用的，为了与之区分我们新建一个工程：

![image-20211217105330650](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211217105330650.png)

我们导入依赖，不需要导入Ribbon和Hystrix，因为Feign = RestTemplate + Ribbon + Hystrix。

```xml
<dependencies>
    <dependency>
        <artifactId>common</artifactId>
        <groupId>com.learn</groupId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

然后将配置文件导入：

```yaml
server:
  port: 8071
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
  instance:
    metadata-map:
      testkey: testvalue
      myname: haha
spring:
  application:
    name: autodeliver
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
   web:
    exposure:
     include: "*"
  # 暴露健康接⼝的细节
  endpoint:
   health:
    show-details: always
```

然后编写入口类，加入feign的注解：

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients //开启Feign客户端功能
public class AutoDeliverFeignApplication {
    public static void main(String[] args) {
        SpringApplication.run(AutoDeliverFeignApplication.class, args);
    }
}
```

然后我们先把原来的controller拿过来，去掉之前的东西，如下：

```java
@RestController
@RequestMapping("/autoDeliverFeign")
public class AutoDeliverFeignController {

    @GetMapping("/findOpenStatusByUid")
    public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
        return null;
    }
}
```

现在我们的项目结构如下：

![image-20211217110755709](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211217110755709.png)

现在我们编写Feign的代码：

首先我们创建FeignClient：在controller下建立一个feign的包，然后建立feign客户端接口，如下：

![image-20211217111939330](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211217111939330.png)

```java
//FeignClient表明当前类是一个FeignClient客户端；value指定要请求的服务名称，本项目中指的是user微服务
@FeignClient(name = "user")
@RequestMapping("/user")
public interface UserServiceFeignClient {

    @GetMapping("/findUserById")
    UserInfo findUserById(@RequestParam("id") Integer id);
}
```

然后我们在controller中调用feign方法即可，feign会自动帮我们访问远程接口

```java
@RestController
@RequestMapping("/autoDeliverFeign")
public class AutoDeliverFeignController {

    @Autowired
    private UserServiceFeignClient userServiceFeignClient;

    @GetMapping("/findOpenStatusByUid")
    public Integer findOpenStatusByUid(@RequestParam("uid") Integer uid){
        //表面上是调用本地方法，实际上feign帮我们拼接url访问远程方法
        UserInfo userById = userServiceFeignClient.findUserById(uid);
        return userById.getOpen();
    }
}
```

> 当然，上述过程需要我们访问一个微服务就需要在自己的微服务中建立一个feignClient。
>
> 其实我们可以为每一个服务提供者（本例中就是user工程）创建一个api工程，在这个api工程中编写FeignClient，这样就可以把服务提供者的接口暴露出来，供服务消费者调用。
>
> 比如创建一个userApi工程，然后将需要别人调用的接口编写成FeignClient，这样别人只需要导入userApi工程的feignClient即可，不需要在自己的项目中编写feignClient。

### 4.4.3 Feign对Ribbon的支持

Feign本身已经集成了Ribbon依赖和自动配置，以此不需要引入额外的依赖，可以通过ribbon.xx进行全局配置或者通过服务名对执行微服务进行配置。

```yaml
#针对的被调用的微服务名称,不加就是对所有调用的微服务生效
user:
  ribbon:
    #请求连接超时时间
    ConnectTimeout: 2000
    #请求处理超时时间
    ReadTimeout: 5000
    #对所有操作都进⾏重试
    OkToRetryOnAllOperations: true
    ####根据如上配置，当访问到故障请求的时候，它会再尝试访问1次当前实例（次数由MaxAutoRetries配置），
    ####如果不⾏，就换⼀个实例进⾏访问，如果还不⾏，再换1次实例访问（更换次数由MaxAutoRetriesNextServer配置），
    ####如果依然不⾏，返回失败信息。
    MaxAutoRetries: 0 #对当前选中实例重试次数，不包括第⼀次调⽤
    MaxAutoRetriesNextServer: 0 #切换实例的重试次数
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #负载策略调整
```

### 4.4.4 Feign日志输出

feign是http请求的客户端，它可以打印出一些详细信息，如果我们想看到Feign请求时的⽇志，我们可以进⾏配置，默认情况下Feign的⽇志没有开启。

首先开启Feign日志的功能和级别：

```java
@Configuration
public class FeignLogConfig {
// Feign的⽇志级别（Feign请求过程信息）
// NONE：默认的，不显示任何⽇志----性能最好
// BASIC：仅记录请求⽅法、URL、响应状态码以及执⾏时间----⽣产问题追踪
// HEADERS：在BASIC级别的基础上，记录请求和响应的header
// FULL：记录请求和响应的header、body和元数据----适⽤于开发及测试环境定位问题
    @Bean
    Logger.Level feignLevel(){
        return Logger.Level.FULL;
    }
}
```

然后配置log日志级别是debug：

```yml
logging:
  level:
    #Feign日志只会对日志级别为debug做出响应
    com.learn.controller.feign.UserServiceFeignClient: debug
```

### 4.4.5 Feign对熔断的支持

首先我们在配置文件中开启熔断

```yml
# 开启Feign的熔断功能
feign:
  hystrix:
   enabled: true
```

然后编写一个类继承feignClient接口，返回降级数据

```java
@Component
public class UserServiceFeignFallback implements UserServiceFeignClient{
    @Override
    public UserInfo findUserById(Integer id) {
        UserInfo userInfo = new UserInfo();
        userInfo.setOpen(-7);
        return userInfo;
    }
}
```

最后在feignClien指定降级实现类即可：

```java
//FeignClient表明当前类是一个FeignClient客户端；value指定要请求的服务名称，本项目中指的是user微服务
@FeignClient(name = "user", fallback = UserServiceFeignFallback.class, path = "/user")
//@RequestMapping("/user") //使用fallback时接口上的RequestMapping应该配置path属性中
public interface UserServiceFeignClient {

    @GetMapping("/findUserById")
    UserInfo findUserById(@RequestParam("id") Integer id);
}
```

这时我们开启服务用postman测试会发现很快就会熔断，这是因为hystrix的熔断超时时长远远小于我们之前4.4.3配置的feign的超时时长，所以会熔断。两者都能配置超时时长，生效取决于最短的时间，来进行熔断。

![image-20211217133306955](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211217133306955.png)

我们可以使用以下配置来修改hystrix的超时时长

```yaml
#配置hystrix的超时时长
hystrix:
 command:
  default:
   execution:
    isolation:
     thread:
      timeoutInMilliseconds: 15000
```

当前也可以使用FallBackFactory，我们创建一个类继承FallbackFactory接口，泛型是我们定义的FeignCclient。

```java
@Component
public class UserServiceFeignFallbackFactory implements FallbackFactory<UserServiceFeignClient> {

    @Override
    public UserServiceFeignClient create(Throwable throwable) {
        return new UserServiceFeignClient() {
            @Override
            public UserInfo findUserById(Integer id) {
                System.out.println(throwable.getMessage());
                UserInfo userInfo = new UserInfo();
                userInfo.setOpen(-7);
                return userInfo;
            }
        };
    }
}
```

然后FeignClient指定工厂：

```java
@FeignClient(name = "user", fallbackFactory = UserServiceFeignFallbackFactory.class, path = "/user")
//@RequestMapping("/user") //使用fallback时接口上的RequestMapping应该配置path属性中
public interface UserServiceFeignClient {

    @GetMapping("/findUserById")
    UserInfo findUserById(@RequestParam("id") Integer id);
}
```

> FallBackFactory与FallBack的区别就是FallBackFactory可以获取到失败原因
>
> ![image-20211217170355359](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211217170355359.png)

### 4.4.6 Feign的请求压缩

Feign ⽀持对请求和响应进⾏GZIP压缩，以减少通信过程中的性能损耗。通过下⾯的参数即可开启请求与响应的压缩功能：

```yaml
feign:
  hystrix:
   enabled: true
  compression:
    request:
      enabled: true
      min-request-size: 2048
      mime-types: text/html,application/xml,application/json
    response:
      enabled: true
```

## 4.5 GateWay

Spring Cloud GateWay是Spring Cloud的⼀个全新项⽬，⽬标是取代Netflflix Zuul，它基于Spring5.0+SpringBoot2.0+WebFlux（基于⾼性能的Reactor模式响应式通信框架Netty，异步⾮阻塞模型）等技术开发，性能⾼于Zuul，官⽅测试，GateWay是Zuul的1.6倍，旨在为微服务架构提供⼀种简单有效的统⼀的API路由管理⽅式。

### 4.5.1 GateWay工作流程

Spring Cloud GateWay不仅提供统⼀的路由⽅式（反向代理）并且基于 Filter(定义过滤器对请求过滤，完成⼀些功能) 链的⽅式提供了⽹关基本的功能，例如：鉴权、流量控制、熔断、路径重写、⽇志监控等。

![gateway20211221](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/gateway20211221.png)

Spring Cloud GateWay是天生异步非阻塞的基于Reactor模型，一个请求打到网关，网关需要根据一定的条件匹配，然后将请求转发到指定的服务地址，在这个转发的过程中我们可以进行一些具体的控制，比如限流、日志、黑白名单等。

1. 路由（route）： ⽹关最基础的部分，也是⽹关⽐较基础的⼯作单元。路由由⼀个ID、⼀个⽬标URL（最终路由到的地址）、⼀系列的断⾔（匹配条件判断）和Filter过滤器（精细化控制）组成。如果断⾔为true，则匹配该路由。
2. 断⾔（predicates）：参考了Java8中的断⾔java.util.function.Predicate，开发⼈员可以匹配Http请求中的所有内容（包括请求头、请求参数等）（类似于nginx中的location匹配⼀样），如果断⾔与请求相匹配则路由。
3. 过滤器（fifilter）：⼀个标准的Spring webFilter，使⽤过滤器，可以在请求之前或者之后执⾏业务逻辑。

![image-20211221150512460](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221150512460.png)

上图来自官网客户端向Spring Cloud GateWay发出请求，然后在GateWay Handler Mapping中找到与请求相匹配的路由，将其发送到GateWay Web Handler；Handler再通过指定的过滤器链来将请求发送到我们实际的服务执⾏业务逻辑，然后返回。过滤器之间⽤虚线分开是因为过滤器可能会在发送代理请求之前（pre）或者之后（post）执⾏业务逻辑。

Filter在“pre”类型过滤器中可以做参数校验、权限校验、流量监控、⽇志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改、⽇志的输出、流量监控等。GateWay核⼼逻辑：路由转发+执⾏过滤器链。

### 4.5.2 GateWay应用

我们在案例中的应用主要是通过网关对自动投递微服务进行代理（相当于隐藏了具体微服务的信息，对外暴露网关）

我们创建gateway8060项目，注意不继承父工程因为gateway使用的是webflux，但父工程引入了web模块，然后编辑pom文件，增加依赖。

> PS：也可以自行改变项目结构，将web从父模块移除，因为很麻烦所以这里直接不继承父工程，直接导入依赖。

![image-20211221155111019](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221155111019.png)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!--spring boot ⽗启动器依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.3.RELEASE</version>
        <relativePath/><!--项目报错可以加上这个-->
    </parent>
    <groupId>com.learn</groupId>
    <artifactId>gateway8060</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-commons</artifactId>
        </dependency>
        <!--eureka client客户端-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <!--gateway-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <!--webflux-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
            <version>1.18.20</version>
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

> 如果报错'parent.relativePath' of POM com.learn:gateway8060:1.0-SNAPSHOT可以加上`<relativePath/>`有一解释说报错原因是本身在父项目中，但是`<parent>`不是父项目

然后我们编辑启动类：

```java
@SpringBootApplication
@EnableDiscoveryClient  //开启注册中心客户端（通用型注解，比如注册到Eureka、
public class GateWay8060 {
    public static void main(String[] args) {
        SpringApplication.run(GateWay8060.class,args);
    }
}
```

最后编写配置文件：

```yaml
server:
  port: 8060
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: cloud-autodeliver # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
          uri: http://127.0.0.1:8071
          predicates:
            - Path=/autoDeliverFeign/** # 断言，路径相匹配的进行路由
        - id: cloud-user # 路由的id,没有规定规则但要求唯一,建议配合服务名
            #匹配后提供服务的路由地址
          uri: http://127.0.0.1:8091
          predicates:
            - Path=/user/** # 断言，路径相匹配的进行路由
```

然后我们可以启动工程，访问网关，可以看到访问结果成功：

![image-20211221162808629](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221162808629.png)

### 4.5.3 GatwWay路由规则

**时间匹配**

Spring Cloud Gateway支持设置一个时间，在请求进行转发的时候，可以通过判断在这个时间之前或者之后或者之间进行转发

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Path=/user/**
        - After=2020-01-09T00:00:00+08:00[Asia/Shanghai]
        #- Before=2020-01-09T00:00:00+08:00[Asia/Shanghai]
        #- Between=2020-01-01T00:00:00+08:00[Asia/Shanghai],2020-01-09T00:00:00+08:00[Asia/Shanghai]
      - id: oss-route
        uri: lb://test-OSS
        predicates:
        - Path=/oss/**
~~~

**Cookie匹配**

例子如下：带上cookies访问，且userCode为123456，则正常返回。

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Path=/user/**
        - Cookie=userCode,123456
~~~

**Header匹配**

Header 匹配 和 Cookie 匹配 一样，也是接收两个参数，一个 header 中属性名称和一个正则表达式，这个属性值和正则表达式匹配则执行。例子：上面的匹配规则，就是请求头要有token属性，并且值必须为数字和字母组合的正则表达式。

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Path=/user/**
        - Header=token,^(?!\d+$)[\da-zA-Z]+$
~~~

**Host匹配**

Host匹配 接收一组参数，一组匹配的域名列表，它通过参数中的主机地址作为匹配规则。

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Host=**.baidu.com
~~~

**请求方式匹配**

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Method=GET //或者POST，PUT
~~~

**请求路径匹配**

~~~yml
server:
  port: 8080
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
      - id: route
        uri: lb://test
        predicates:
        - Path=/user/{segment}
~~~

**请求参数匹配**

Query匹配 支持传入两个参数，一个是属性名，一个为属性值，属性值可以是正则表达式。

~~~yml
predicates:
 - Query=token, \d+
~~~

**请求地址匹配**

通过设置某个 ip 区间号段的请求才会路由。

~~~yml
predicates:
 - RemoteAddr=192.168.1.1/24
~~~

### 4.5.4 动态路由

在4.5.2中我们直接写死了实例地址，但如果我们有集群的话就不是很好，所以我们可以使用动态路由，从注册中心拿。动态路由可以通过`lb://服务名称`写。

修改配置文件：

```yaml
server:
  port: 8060
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
spring:
  application:
    name: gateway
  cloud:
    gateway:
      routes:
        - id: cloud-autodeliver # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
          uri: lb://autodeliver
          predicates:
            - Path=/autoDeliverFeign/** # 断言，路径相匹配的进行路由
        - id: cloud-user # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
          uri: lb://user
          predicates:
            - Path=/user/** # 断言，路径相匹配的进行路由
```

我们测试user服务来进行测试是否是动态路由（因为autodeliver看不出来）

我们postman发送请求

![image-20211221164842669](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221164842669.png)

然后看user的两个服务

![image-20211221164913775](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221164913775.png)

![image-20211221164925613](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211221164925613.png)

可以看到可以进行动态路由。

### 4.5.5 过滤器Filter

过滤器的生命周期主要有两个

| 生命周期 | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| pre      | 这种过滤器在请求被路由之前调⽤。我们可利⽤这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等 |
| post     | 这种过滤器在路由到微服务以后执⾏。这种过滤器可⽤来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。 |

从过滤器类型的角度。SpringCloudGateway分为GateWayFilter和GlobalFilter两种

| 类型          | 影响范围           |
| ------------- | ------------------ |
| GateWayFilter | 应用到单个路由器上 |
| GlobalFilter  | 应用到所有路由器上 |

首先是GateWayFilter,我们可以配置到单个路由之下，对当前路由使用。

~~~yml
predicates:
- Path=/user/name/getname # 断言，路径相匹配的进行路由
filters: 
- StripPrefix=1 
~~~

StripPrefix过滤器可以去掉当前的第一个路由，比如我们之前的路径是`localhost:8080/user/name/getname`在加上过滤器之后就变成了`localhost:8080/name/getname`。测试结果：

![image-20211222091828406](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222091828406.png)

![image-20211222091842158](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222091842158.png)

当然实际上GlobalFilter的使用频率更高，我们下面对GlobalFilter进行自定义配置。（PS：GateWayFilter也可以自定义，但是一般只用默认配置）

我们通过实现自定义全局过滤器实现IP访问限制的例子来进行GlobalFilter进行配置。

在gateway8060中新建类：

```java
package com.learn.filter;

import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

@Component
public class BlackListFilter implements GlobalFilter, Ordered {
    //模拟黑名单，实际可以存入redis或mysql
    private static List<String> blackList = new ArrayList<>();

    static {
        blackList.add("0:0:0:0:0:0:0:1");//模拟本地地址
    }

    /**
     * 过滤器的核心方法
     *
     * @param exchange 封装了request和response对象的上下文
     * @param chain    网关过滤器链（包含全局和单路由过滤器）
     * @return
     */
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        //获取客户端ip，判断是否在黑名单中，在的话拒绝，不在就放行
        ServerHttpRequest request = exchange.getRequest();
        ServerHttpResponse response = exchange.getResponse();
        String clientIp = request.getRemoteAddress().getHostString();
        if (blackList.contains(clientIp)) {
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            System.out.println("在黑名单中，拒绝访问，clientIP="+clientIp);
            DataBuffer wrap = response.bufferFactory().wrap("Request be refused".getBytes(StandardCharsets.UTF_8));
            return response.writeWith(Mono.just(wrap));
        }
        return chain.filter(exchange);
    }

    /**
     * 返回值表示当前过滤器的优先级（顺序），数值越小优先级越高
     *
     * @return
     */
    @Override
    public int getOrder() {
        return 0;
    }
}
```

我们通过postman测试，测试结果：

![image-20211222091552276](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222091552276.png)

> 因为网关在微服务体系十分重要，所以需要实现高可用，我们可以通过集群的方式通过上游用Nginx完成负载均衡即可。

## 4.6 Spring Cloud Config

### 4.6.1 简介

在项目中我们需要配置文件进行一些配置，在单体应用中，配置信息的管理不会很麻烦，因为就一个工程，但是在微服务体系下有很多工程，我们不可能一个个的去配置重启然后生效，有时候我们还需要在运行期间动态调整配置信息。所以我们需要对配置文件进行集中式管理，这就是分布式配置中心的作用。

Spring Cloud Conﬁg是⼀个分布式配置管理⽅案，包含了Server端和Client端两个部分。

Server 端：提供配置⽂件的存储、以接⼝的形式将配置⽂件的内容提供出去，通过使⽤@EnableConﬁgServer注解在Spring boot 应⽤中非常简单的嵌⼊
Client 端：通过接⼝获取配置数据并初始化自己的应用。

![CloudConfig20121222](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/CloudConfig20121222.png)

### 4.6.2 Config配置中心应用

首先我们在Gitee上创建一个仓库然后增加一个测试用的配置文件：

![image-20211222093957001](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222093957001.png)

创建configserver工程引入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>MyGirl</artifactId>
        <groupId>com.learn</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>serverConfig</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
    </dependencies>
</project>
```

然后创建启动类：注意需要注解@EnableConfigServer

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
public class ServerConfigApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServerConfigApplication.class,args);
    }
}
```

编写配置文件：

```yml
server:
  port: 8050
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
      defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
spring:
  application:
    name: serverconfig
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/yhr520/config-server.git #git地址
          username: #git用户名
          password: #git密码
          search-paths:
            - config-server
      label: master #分支
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
    web:
      exposure:
        include: "*"
  # 暴露健康接⼝的细节
  endpoint:
    health:
      show-details: always
```

![image-20211222094630180](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222094630180.png)

搭建完工程后我们启动工程然后访问`http://localhost:8050/master/test-dev.yml`

第一个master是分支名称，第二个是文件名称，测试结果如下：可以获取配置文件信息

![image-20211222093718227](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222093718227.png)

### 4.6.3 Config客户端应用

我们在入门案例的autodeliver工程进行修改，其他工程几乎一样，可以自己完成修改。

首先在项目中增加依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
~~~

然后将application.yml改为bootstrap.yml,然后增加config客户端的配置

```yml
spring:
  application:
    name: autodeliver
  cloud:
    config:
      name: test #配置文件名文件名可以配多个逗号分隔
      profile: dev #后缀名
      label: master #分支名
      uri: http://localhost:8050
```

然后我们创建一个Controller，测试能否拿到配置中心的配置

```java
@RestController
@RequestMapping("config")
public class ConfigTestController {

    @Value("${testname.namestr}")
    private String testStr;

    @RequestMapping("testGetConfig")
    public String testGetConfig(){
        return testStr;
    }
}
```

打开postman进行测试，测试结果：

![image-20211222095948355](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222095948355.png)

> 其实我们可以将全部配置信息都放入Condig远端，然后本地只留一个bootstrap拉取其他配置，这样所有配置信息都在远端管理，但是本例只是学习用所以并没有那么做。
>
> 即bootstrap只留以下信息：
>
> ```yml
> spring:
>   application:
>     name: autodeliver
>   cloud:
>     config:
>       name: test #配置文件名文件名可以配多个逗号分隔
>       profile: dev #后缀名
>       label: master #分支名
>       uri: http://localhost:8050
> #注册到Eureka服务中心
> eureka:
>   client:
>     service-url:
>       # 注册到集群，把多个server地址用逗号连接即可。如果Eureka Server是单实例就写一个就行。
>       defaultZone: http://CloudEurekaServerA:8761/eureka/,http://CloudEurekaServerB:8762/eureka/
> ```

### 4.6.4 Spring Cloud Config的配置文件刷新

我们修改远端仓库的值：

![image-20211222100922362](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222100922362.png)

在访问配置中心的端口：

![image-20211222101001238](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222101001238.png)

发现配置中心能够实时获取配置的修改。然后我们在访问配置的客户端（即Autodeliver）的接口

![image-20211222101107079](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222101107079.png)

发现客户端会有缓存。这时我们可以手动刷新。

**手动刷新**

首先在boostrap中加上如下配置

```yml
management:
  endpoints:
   web:
    exposure:
     include: "*" #暴露的端口也可以指定refresh只暴露刷新端口
```

然后再客户端使用到配置信息的类上添加@RefreshScope

然后手动发起请求`http://localhost:8071/actuator/refresh`

测试前将远端仓库重新修改为haha222，因为添加@RefreshScope需要重启工程。

然后我们发起请求，测试结果如下：

![image-20211222101738006](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222101738006.png)

![image-20211222101759550](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222101759550.png)

## 4.7 Spring Cloud Stream消息驱动组件

Spring Cloud Stream 消息驱动组件帮助我们更快速，更⽅便，更友好的去构建消息驱动微服务。

### 4.7.1 Stream简介

MQ消息中间件广泛用在应用解耦合，异步消息处理，流量削峰等场景中。不同的MQ消息中间件内部机制会有不同，比如RabbitMQ中有Exchange概念，kafka有Topic、Partition分区等概念，MQ的差异性不利于我们上层开发，因为可能应用程序和具体的MQ耦合了，当我们想切换MQ产品时，我们发现比较困难，很多需要重写。

Spring Cloud Stream就进行了很好的上层抽象，可以让我们与具体的消息中间件解耦合，屏蔽掉了底层具体MQ消息中间件的细节差异，就像Hibernate屏蔽掉了具体数据库，目前Spring Cloud Stream支持RabbitMQ和KafKa。

# 五、SpringCloudAlibaba核心组件

## 5.1 Nacos服务发现和配置中心

Nacos （Dynamic Naming and Confifiguration Service）是阿⾥巴巴开源的⼀个针对微服务架构中服务发现、配置管理和服务管理平台。Nacos就是注册中心+配置中心的组合Nacos=Eureka+Config+Bus

### 5.1.1 Nacos单例服务部署

官网下载安装包，解压。然后将 startup.cmd记事本打开，修改mode为单机模式，如下图：

![image-20211222104600489](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222104600489.png)

然后双击启动即可。linux在解压后执行命令即可

~~~sh
sh startup.sh -m standalone
~~~

输入`localhost://127.0.0.1:8848/nacos`账号密码都是nacos

### 5.1.2 服务提供者注册到Nacos

首先我们创建user8092nacos工程，这是为了与之前eureka做区分，所以新建一个工程然后在这里进行修改。

![image-20211222124941202](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222124941202.png)

我们在父pom中增加阿里的微服务依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.2.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

然后再user8092nacos工程的pom中增加nacos注册中心依赖

```xml
<!--nacos client客户端-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

然后编辑application.yml

```yml
server:
  port: 8092
spring:
  application:
    name: user
  #配置注册到nacos服务端
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/my_girl?allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: 123456
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
    web:
      exposure:
        include: "*"
  # 暴露健康接⼝的细节
  endpoint:
    health:
      show-details: always
```

然后登录nacos页面查看

![image-20211222125538669](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222125538669.png)

点击详情查看

![image-20211222132225160](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222132225160.png)

服务已经注册上来了，然后我们以同样的方式在创建一个工程，做一下集群（**再强调一遍，正常情况下是同一个工程在不同服务器上实现集群，目前因为是学习项目，所以才这样做集群**）：

![image-20211222125911769](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222125911769.png)

启动注册到nacos上，这时我们发现

![image-20211222125949222](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222125949222.png)

> 保护阈值：这实际上是一个0-1的浮点数，其实是一个比例值：当前服务健康实例数/当前服务总实例数。
>
> 一般情况下，nacos在服务为消费者获取一个服务的实例信息时，会把健康的实例信息返回，但在高并发的情况下会出现问题，比如A服务有100个实例，但是95个事不健康的，如果nacos只返回那五个健康的实例的话，流量洪峰会让这5个实例也扛不住，如果这5个也挂了，就会造成雪崩。
>
> 所以当一个服务的健康实例数/总实例数<保护阈值时，说明健康实例不多了，这时候保护阈值就会触发（状态改为true），nacos会把健康的不健康的都返回给消费者，这样虽然会造成请求失败，但是比雪崩会好，因为这样时牺牲了一些请求来保证整个系统的可用。

### 5.1.3 服务消费者注册到Nacos

同样的，为了与eureka做对比，我们也给autodeliver也新建一个工程切换到nacos上，配置方式与上面相同。

![image-20211222131357946](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222131357946.png)

配置文件：

```yml
server:
  port: 8072
spring:
  application:
    name: autodeliver
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
# springboot中暴露健康检查等断点接⼝
management:
  endpoints:
   web:
    exposure:
     include: "*"
  # 暴露健康接⼝的细节
  endpoint:
   health:
    show-details: always
#针对的被调用的微服务名称,不加就是对所有调用的微服务生效
user:
  ribbon:
    #请求连接超时时间
    ConnectTimeout: 2000
    #请求处理超时时间
    ReadTimeout: 8000
    #对所有操作都进行重试
    OkToRetryOnAllOperations: true
    ####根据如上配置，当访问到故障请求的时候，它会再尝试访问1次当前实例（次数由MaxAutoRetries配置），
    ####如果不行，就换一个实例进行访问，如果还不行，再换1次实例访问（更换次数由MaxAutoRetriesNextServer配置），
    ####如果依然不行，返回失败信息。
    MaxAutoRetries: 0 #对当前选中实例重试次数，不包括第一次调用
    MaxAutoRetriesNextServer: 0 #切换实例的重试次数
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #负载策略调整
logging:
  level:
    #Feign日志只会对日志级别为debug做出响应
    com.learn.controller.feign.UserServiceFeignClient: debug
# 开启Feign的熔断功能
feign:
  hystrix:
   enabled: true
  compression:
    request:
      enabled: true #开启请求压缩
      min-request-size: 2048 #出发压缩的大小下限
      mime-types: text/html,application/xml,application/json #设置压缩的数据类型
    response:
      enabled: true #开启响应压缩
#配置hystrix的超时时长
hystrix:
 command:
  default:
   execution:
    isolation:
     thread:
      timeoutInMilliseconds: 8000
```

> Nacos客户端引⼊的时候，会关联引⼊Ribbon的依赖包，我们使⽤OpenFiegn的时候也会引⼊Ribbon的依赖，Ribbon包括Hystrix都按原来⽅式进⾏配置即可。

创建完启动工程，查看nacos：

![image-20211222131523774](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222131523774.png)

我们可以通过postman进行测试：可以看到访问成功且进行了负载均衡

![image-20211222132424499](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222132424499.png)

![image-20211222132611233](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222132611233.png)

![image-20211222132622606](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222132622606.png)

### 5.1.4数据模型

![nacosmodel20211222](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/nacosmodel20211222.png)

> Namespace：命名空间，对不同的环境进⾏隔离，比如隔离开发环境、测试环境和⽣产环境
>
> Group：分组，将若⼲个服务或者若⼲个配置集归为⼀组，通常习惯⼀个系统归为⼀个组
>
> Service：某⼀个服务，⽐如简历微服务
>
> DataId：配置集或者可以认为是⼀个配置⽂件
>
> Namespace + Group + Service如同Maven中的GAV坐标，GAV坐标是为了锁定Jar，而这里是为了锁定服务
>
> Namespace + Group + DataId是为了锁定配置⽂件

### 5.1.5 nacos集群

安装3个或以上nacos，修改nacos的application.properties配置文件中的server.port属性8848、8849、8850，然后修改ip地址为127.0.0.1，三个nacos都要改，如下图

![image-20211222141735736](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222141735736.png)

然后我们创建cluster.conf在文件中输入集群中的全部ip和端口，然后拷贝到每一个nacos的conf文件夹中。

~~~
#it is ip
#example
127.0.0.1:8848
127.0.0.1:8849
127.0.0.1:8850
~~~

将三个nacos都启动即可。

> 我的nacos版本是1.3.2，集群启动会报错，需要配置数据库才能启动。
>
> 配置数据库在applicatin.properties中配置数据库连接和密码
>
> ![image-20211222155242373](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222155242373.png)
>
> 配置完后需要创建对应的数据库，同时在数据库的表中需要执行conf文件夹下的nacos-mysql.sql文件。

启动后我们可以看到

![image-20211222161923827](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222161923827.png)

刚启动节点的状态会变成CANDIDATE，也就是大家都在竞争leader

![image-20211222160625303](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222160625303.png)

等一会后再刷新我么可以看到nacos已经竞争出了leader和follower

![image-20211222161804590](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222161804590.png)

这种leader和follower的模式我们是需要3台以上的机器来搭建集群。

客户端访问到集群跟Eureka类似，逗号分隔即可：

```yml
spring:
  application:
    name: autodeliver
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.22.162:8848,192.168.22.162:8849,192.168.22.162:8850
      config:
        server-addr: 192.168.22.162:8848,192.168.22.162:8849,192.168.22.162:8850
        file-extension: yml
        namespace: autodeliver
        extension-configs[0]:
          data-id: test.yml
          group: DEFAULT_GROUP
          refresh: true
        extension-configs[1]:
          data-id: test.properties
          group: DEFAULT_GROUP
          refresh: true
```

### 5.1.6 nacos作为配置中心

nacos作为配置中心非常简单，不再需要git仓库和bus。

跟SpringConfig一样，是可以只留boostrap，其他配置全部从nacos中拉取，因为是学习项目所以只在nacos中建一个简单的配置文件，然后让客户端去拉去配置文件,下面进行配置。

我们在nacos中建立配置文件：

![image-20211222133833803](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222133833803.png)

![image-20211222140824367](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222140824367.png)

![image-20211222140838399](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222140838399.png)

然后我们在微服务中开启nacos作为配置中心。（在autodeliver中演示配置，user同理）

首先添加依赖：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

微服务通过 Namespace + Group + dataId 来锁定配置⽂件，Namespace不指定就默认public，Group不指定就默认 DEFAULT_GROUP

DataId的格式：`${prefix}-${spring.profile.active}.${file-extension}`

- prefix默认为spring.applicatin.name的值，也可以通过spring.cloud.nacos.config.prefix来配置
- spring.profile.active即当前环境的profile，当此项为空时，dataId直接变成`${prefix}.${file-extension}`
- file-extension为配置内容的数据格式，可以通过file-extension来配置，目前只支持properties和yaml

添加配置：

```yml
spring:
  application:
    name: autodeliver
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yml
        namespace: autodeliver
        extension-configs[0]:
          data-id: test.yml
          group: DEFAULT_GROUP
          refresh: true
        extension-configs[1]:
          data-id: test.properties
          group: DEFAULT_GROUP
          refresh: true
```

优先级：根据规则⽣成的dataId > 扩展的dataId（对于扩展的dataId，[n] n越⼤优先级越⾼）

然后我们在需要使用的类上使用@RefreshScope实现配置的自动更新。

```java
@RestController
@RequestMapping("config")
@RefreshScope
public class ConfigTestController {

    @Value("${test.name}")
    private String name;
    @Value("${test.sex}")
    private String sex;

    @RequestMapping("testGetConfig")
    public String testGetConfig(){
        return "name="+name+";sex="+sex;
    }
}
```

使用postman进行测试

![image-20211222140804424](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222140804424.png)

然后我们测试动态刷新：先修改配置

![image-20211222140923969](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222140923969.png)

然后访问postman，发现可以进行动态刷新配置：

![image-20211222141000714](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211222141000714.png)

## 5.2 Sentinal服务熔断限流

> 未完结

# 六、SpringCloud高级组件

> 未完结
>

