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

首先在user工程（user8090和user8091都要修改）中进行线程休眠来模拟访问超时：

```java
@GetMapping("/findUserById")
public UserInfo findUserById(@RequestParam("id") Integer id) {
    //模拟访问超时
    try {
        Thread.sleep(10000000);
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



# 五、SpringCloud高级组件



# 六、SpringCloudAlibaba核心组件

