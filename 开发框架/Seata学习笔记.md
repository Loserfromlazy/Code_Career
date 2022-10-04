# Seata学习和源码剖析

> 本笔记的项目地址[LearnSpringCloud](https://github.com/Loserfromlazy/LearnSpringCloud)。此项目是SpringCloud学习的统一项目，我们这篇笔记里Seata相关的项目均是seata-xxx形式的。
>
> 参考资料：[Seata官方文档](https://seata.io/zh-cn/docs/overview/what-is-seata.html)

# 一、Seata概述

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。下图来自官网：

![](https://user-images.githubusercontent.com/68344696/145942191-7a2d469f-94c8-4cd2-8c7e-46ad75683636.png)

此图中有一些Seata的术语：

- TC (Transaction Coordinator) - 事务协调者
  维护全局和分支事务的状态，驱动全局事务提交或回滚。
- TM (Transaction Manager) - 事务管理器
  定义全局事务的范围：开始全局事务、提交或回滚全局事务。
- RM (Resource Manager) - 资源管理器
  管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

我们下面来看一下Seata的AT、TCC和Saga模式：

- AT模式：AT 模式是⼀种⽆侵⼊的分布式事务解决⽅案。在 AT 模式下，⽤户只需关注⾃⼰的“业务 SQL”，⽤户的 “业务 SQL” 作为⼀阶段，Seata 框架会⾃动⽣成事务的⼆阶段提交和回滚操作。
- TCC模式：TCC模式主要是需要用户根据业务场景实现try、confirm、cancel三个方法。事务在以阶段执行try方法；在二阶段提交执行confirm；在二阶段回滚执行cancel方法。
- SAGA模式：Saga模式是Seata的长事务解决方案，在saga模式中，业务流程中的每一个参与者都提交事务，当出现某一个参与者失败，则补偿前面成功的参与者，去执行各参与者的逆向回滚操作，使分布式事务回到初始状态。

# 二、Seata入门案例

> 为了方便和统一，我们在之前SpringCloud笔记中使用过的项目下创建Seata的测试项目，Seata测试项目与LearnSpringCloud没有关系仅继承了LearnSpringCloud的根pom文件，使用了一致的微服务版本，Seata测试项目全部以seata-xxx命名。
>
> 这里不展示全部代码，只写出思路和部分代码，请移步代码仓库自行下载查看[LearnSpringCloud](https://github.com/Loserfromlazy/LearnSpringCloud)，以seata-xxx命名全是本笔记的项目代码，数据库脚本都在该项目的路径下。
>
> 入门案例主要使用springboot+springcloud(openfeign+nacos)+mybatisplus+mysql8.0.18等技术实现。

假设现在有业务需求，我们客户在买东西后，后台需要执行很多步骤比如需要新增订单、积分返现并扣减库存，如下图：

![image-20220926142236907](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220926142236907.png)

我们现在创建项目实现此功能，首先创建我们需要的四个项目，并引入nacos、feign和mybatisplus等相关依赖，如下图：

![image-20220926142859407](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220926142859407.png)

然后我们建出数据库表（建表语句我放在了项目sql目录下）以及相关实体类和mapper和Service，（当然也可以用mybatisplus的代码生成器）这里以order为例，最终项目结构如下：

![image-20220927131451979](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220927131451979.png)

然后我们编写Client测试项目，我们的整体流程如下（具体各模块实现代码，比如新增订单等这种业务增删改查代码这里略，可以自行翻阅本项目的代码仓库或自行编写CURD代码）：

购买-->创建订单-->用户积累积分-->扣减库存

我们来看一下Client模块的测试代码，首先是Controller代码：

```java
@GetMapping("/testBuy1")
public Result<Boolean> testBuy1() {
    return clientService.buy(10,1,1);
}

@GetMapping("/testBuy2")
public Result<Boolean> testBuy2() {
    return clientService.buy(100,1,1);
}
```

然后是Service代码：

```java
@Override
public Result<Boolean> buy(Integer num, Integer userId, Integer goodsId) {
    Order order = new Order();
    order.setGoodsId(goodsId);
    order.setName("购买商品"+goodsId);
    order.setUserId(userId);
    order.setNums(num);
    order.setCreateTime(new Date());
    Result<Boolean> result = orderFeign.addOrder(order);
    if (!result.getResult()){
        return ResultUtils.resultInit(0,"购买失败，原因"+result.getMsg(),false);
    }
    Result<Boolean> result1 = userFeign.addPoints(userId, num);
    if (!result1.getResult()){
        return ResultUtils.resultInit(0,"购买失败，原因"+result.getMsg(),false);
    }
    Result<Boolean> result2 = warehouseFeign.reduceGoods(goodsId, num);
    if (!result2.getResult()){
        return ResultUtils.resultInit(0,"购买失败，原因"+result.getMsg(),false);
    }
    return ResultUtils.successBuild(true);
}
```

其余Feign接口及实现代码不在此粘贴，然后在数据库中造一个用户数据，一个库存为50的商品数据：

![image-20220927143859496](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220927143859496.png)

下面我们进行测试，首先我们调用`http://localhost:8075/test/testBuy1`,结果显示成功，数据全部正常：

![image-20220927145840647](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220927145840647.png)

然后我们测试`http://localhost:8075/test/testBuy2`,我们可以发现虽然购买失败了库存表没变化，但是用户表的积分已经更改了，同时订单表中该订单也进行新增了。

![image-20220927151622101](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220927151622101.png)

此问题就是分布式微服务下的事务问题，即使我们在ClientServiceImpl#buy方法上加上了事务注解也不能解决，因为那个事务是本地事务，但我们此场景下的事务属于分布式事务。针对此类问题，我们就需要使用seata之类的分布式事务框架来解决此问题。

# 三、Seata的AT模式

首先我们先了解Seata如何使用。首先我们需要搭建Seata的服务端（TC），然后需要在业务模块引入客户端。具体步骤如下：

## 3.1 Seata的服务端（TC）搭建

**我们这里使用nacos作为注册和配置中心，并使用db进行存储，在Windows环境下搭建**

### 1 下载配置seata

首先我们要下载Seata服务端，[下载地址](https://github.com/seata/seata/releases)。我们这里以152版本进行演示（与其他版本的配置方法基本类似）。解压后我们需要修改其配置文件：

![image-20220930104623860](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930104623860.png)

> seata1.5.0以后的版本对配置文件进行了简化，如果是150以前的版本配置文件如下：
>
> ![image-20220930105505177](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930105505177.png)
>
> 当然只是文件类型不同，具体的配置方法是大体一致的。

我们打开application.yml进行配置，如下（已标注注解）：

~~~yaml
#端口号
server:
  port: 7091 
#服务名称
spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata
  extend:
    logstash-appender:
      destination: 127.0.0.1:4560
    kafka-appender:
      bootstrap-servers: 127.0.0.1:9092
      topic: logback_to_logstash
#控制台
console:
  user:
    username: seata
    password: seata
#seata配置中心，这里以nacos为例
seata:
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    #配置nacos
    nacos: 
      server-addr: 127.0.0.1:8848
      namespace:
      group: SEATA_GROUP
      username: nacos
      password: nacos
      #data-id: seataServer.properties 这个配置对应下面的第一种导入方法。
  #seata注册中心，这里以nacos为例
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace:
      cluster: default
      username: nacos
      password: nacos
  #这里不需要配置也可以，因为我们config选择了nacos，会从nacos中拉取stroe的相关配置
  #如果使用file模式，那么这里是需要配置的
  store:
    # support: file 、 db 、 redis
    mode: db
#  server:
#    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
~~~

### 2 配置中心添加配置信息

然后我们需要去nacos上加入seata的配置。配置可以从Seata的Github的[Config-Center](https://github.com/seata/seata/tree/develop/script/config-center)进行下载。需要下载其中的`config.txt`文件。然后我们主要修改此文件其中的store的相关配置，如下图：

![image-20220930110716086](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930110716086.png)

> config.txt中的各个配置的意思，在nacos官方文档中可以查询到。

然后我们去数据库中创建一个seata的数据库，创建出相关表，建表sql可以从Seata的Github的[SQL脚本](https://github.com/seata/seata/tree/develop/script/server/db)进行下载，我们这里下载mysql.sql，然后去执行这个脚本。

![image-20220930111039199](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930111039199.png)

然后我们去配置中心也就是nacos中添加seata配置，这里有两种方式：

第一种是application.yml中的配置nacos的config时加上data-id配置，我在上面的配置中用注释标注了。然后我们在nacos中新建配置其data-id与我们配置的一样，然后将config.txt中的数据拷贝到这个配置中，如下图：

![image-20220930112124660](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930112124660.png)

第二种是去下载seata的到配置脚本，下载地址：[Naocs配置脚本](https://github.com/seata/seata/tree/1.5.2/script/config-center/nacos)我们这里下载sh脚本（如下图），将其放在config目录下，然后将刚才我们修改的config.txt放在seata根目录下。

![image-20220930111304245](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930111304245.png)

然后用git bash执行此脚本,命令为：`sh nacos-config.sh -h 127.0.0.1`，然后相关配置就能自动导入了：

![image-20220930111515461](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930111515461.png)

### 3 启动seata服务端

配置完以上信息后就可以启动seata的服务端了，我们进入bin目录下启动bat脚本即可。

![image-20220930112450792](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930112450792.png)

## 3.2 业务模块引入Seata客户端（整合TM/RM）

**我们这里用入门案例中的demo项目进行客户端整合，即通过Spring Cloud整合Seata**

如果我们使用AT模式，那么需要在自己的业务库中创建UNDO_LOG表，[下载地址](https://github.com/seata/seata/tree/develop/script/client/at/db)。如下图：

![image-20220930112813585](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930112813585.png)

Seata官方的部署文档地址[Seata部署](https://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html)，下面是我的部署流程

### 1 导入依赖

先给出我导入的依赖，

```xml
<!--        seata Client-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
    <version>2.2.0.RELEASE</version>
    <exclusions>
        <exclusion>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>1.5.2</version>
</dependency>
```

我的父工程的版本是：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.3.RELEASE</version>
</parent>

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
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.2.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

注意点： 

1. 首先spring cloud整合依赖是`spring-cloud-starter-alibaba-seata`，然后最好排除掉springboot的整合依赖`seata-spring-boot-starter`，然后引入与使用的Seata服务端相同版本的springboot版本。

2. 然后默认`spring-cloud-starter-alibaba-seata`跟着spring cloud alibaba版本走即可，但是我在启动项目的时候报错，错误信息是：

   ~~~
   tried to access class org.springframework.cloud.openfeign.loadbalancer.FeignBlockingLoadBalancerClient from class com.alibaba.cloud.seata.feign.SeataFeignObjectWrapper
   ~~~

   然后我找了很久也没有解决方案，在官方的issue中[Seata 1.4.0与spring-cloud-starter-openfeign兼容问题](https://github.com/seata/seata/issues/3449)和[SeataFeignObjectWrapper 创建 feignClient 失败](https://github.com/seata/seata/issues/3115)里面也有人询问了关于这个问题，但是里面的解决方法都不好使，最后在官方博客[Spring Cloud集成Seata分布式事务-TCC模式](https://seata.io/zh-cn/blog/integrate-seata-tcc-mode-with-spring-cloud.html)里面的示例项目中我发现了它使用的是2.2.0.RELEASE的seata版本，因此我也尝试了直接引入此版本，解决了此问题。目前原因暂未进行深入分析，如果有大佬能解答此问题或者有更好的解决方法也欢迎来留言指教。

### 2 编辑配置

```yml
seata:
  application-id: ${spring.application.name}
  #启动自动数据源 代理
  enable-auto-data-source-proxy: true
  #事务分组，使用时必须与注册中心配置一致：service.vgroupMapping.default_tx_group
  #tx-service-group: default_tx_group
  #seata的配置中心和注册中心需要和seata server保持一致
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      username: nacos
      password: nacos
      dataId: "seataServer.properties"
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      username: nacos
      password: nacos
  data-source-proxy-mode: AT
  enabled: true
```

注意点：

1. 首先在配置seata时我们需要配置其注册和配置中心，这一部分需要与seata Server保持一致

2. seata可以使用事务分组，这部分必须与配置中心的`service.vgroupMapping.xxx`保持一致，后面xxx的部分就是可以自定义的部分。

3. seata需要配置数据源代理，默认使用自动代理即可`enable-auto-data-source-proxy: true`

   当然也可以手动配置数据源代理，我们添加配置类：

   ~~~java
   @Configuration
   public class DataSourceConfiguration {
   	//配置数据源
       @Bean
       @ConfigurationProperties("spring.datasource")
       public DataSource dataSource(){
           return new DruidDataSource();
       }
   
       //配置seata代理数据源
       @Bean
       @Primary//设置首选数据源
       public DataSourceProxy dataSourceProxy(DataSource dataSource){
           //使用DataSourceProxy包装原有的数据源
           return new DataSourceProxy(dataSource);
       }
   }
   ~~~

   当然如果要使用手动数据源配置那么就需要关闭自动代理数据源即：`enable-auto-data-source-proxy: false`

   最后在我们的启动类上扫描到我们的这个配置类即可。

### 3 使用AT模式

我们在我们入门案例中的四个项目都进行上面的配置。在上面的配置完成后，我们只要在需要事务的地方加上注解`@GlobalTransactional(rollbackFor = Exception.class)`即可。

> 注意：我们在使用seata时，如果使用了fallback降级就会导致异常被全局处理，从而导致事务失效，我们需要抛出异常而不是降级：
>
> ```java
> public class WarehouseFeignFallback implements FallbackFactory<WarehouseFeign> {
> 
>     @Override
>     public WarehouseFeign create(Throwable cause) {
>         return new WarehouseFeign() {
>             @Override
>             public Result<Boolean> reduceGoods(Integer id, Integer nums) {
>                 throw new RuntimeException("进入到reduceGoods的降级方法，抛出异常，便于seata回滚");
> //                return ResultUtils.resultInit(0, cause.getMessage(), false);
>             }
>         };
>     }
> }
> ```
>
> 当然也可以使用seata提供的方法进行手动回滚`GlobalTransactionContext.reload(RootContext.getXID()).rollback()`

我们在reduceGoods扣减库存这个方法上打上断点，再次访问`http://localhost:8075/test/testBuy2`，这次我们会发现数据并没有变化，然后我们观察seata控制台，会发现所有分支均已经回滚：

![image-20221004170928918](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221004170928918.png)

到此AT模式的Seata配置使用已经了解完成。

## 3.3 Seata AT模式工作机制介绍

Seata的AT模式整体上是两阶段提交协议的演变：

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：
  - 提交异步化，非常快速地完成。
  - 回滚通过一阶段的回滚日志进行反向补偿。

在官方文档中有AT模式的工作机制介绍（以下内容来自官方文档）：

> **工作机制**
>
> 以一个示例来说明整个 AT 分支的工作过程。
>
> 业务表：`product`
>
> | Field | Type         | Key  |
> | ----- | ------------ | ---- |
> | id    | bigint(20)   | PRI  |
> | name  | varchar(100) |      |
> | since | varchar(100) |      |
>
> AT 分支事务的业务逻辑：
>
> ```sql
> update product set name = 'GTS' where name = 'TXC';
> ```
>
> **一阶段**
>
> 过程：
>
> 1. 解析 SQL：得到 SQL 的类型（UPDATE），表（product），条件（where name = 'TXC'）等相关的信息。
> 2. 查询前镜像：根据解析得到的条件信息，生成查询语句，定位数据。
>
> ```sql
> select id, name, since from product where name = 'TXC';
> ```
>
> 得到前镜像：
>
> | id   | name | since |
> | ---- | ---- | ----- |
> | 1    | TXC  | 2014  |
>
> 1. 执行业务 SQL：更新这条记录的 name 为 'GTS'。
> 2. 查询后镜像：根据前镜像的结果，通过 **主键** 定位数据。
>
> ```sql
> select id, name, since from product where id = 1;
> ```
>
> 得到后镜像：
>
> | id   | name | since |
> | ---- | ---- | ----- |
> | 1    | GTS  | 2014  |
>
> 1. 插入回滚日志：把前后镜像数据以及业务 SQL 相关的信息组成一条回滚日志记录，插入到 `UNDO_LOG` 表中。
>
> ```json
> {
> 	"branchId": 641789253,
> 	"undoItems": [{
> 		"afterImage": {
> 			"rows": [{
> 				"fields": [{
> 					"name": "id",
> 					"type": 4,
> 					"value": 1
> 				}, {
> 					"name": "name",
> 					"type": 12,
> 					"value": "GTS"
> 				}, {
> 					"name": "since",
> 					"type": 12,
> 					"value": "2014"
> 				}]
> 			}],
> 			"tableName": "product"
> 		},
> 		"beforeImage": {
> 			"rows": [{
> 				"fields": [{
> 					"name": "id",
> 					"type": 4,
> 					"value": 1
> 				}, {
> 					"name": "name",
> 					"type": 12,
> 					"value": "TXC"
> 				}, {
> 					"name": "since",
> 					"type": 12,
> 					"value": "2014"
> 				}]
> 			}],
> 			"tableName": "product"
> 		},
> 		"sqlType": "UPDATE"
> 	}],
> 	"xid": "xid:xxx"
> }
> ```
>
> 1. 提交前，向 TC 注册分支：申请 `product` 表中，主键值等于 1 的记录的 **全局锁** 。
> 2. 本地事务提交：业务数据的更新和前面步骤中生成的 UNDO LOG 一并提交。
> 3. 将本地事务提交的结果上报给 TC。
>
> **二阶段-回滚**
>
> 1. 收到 TC 的分支回滚请求，开启一个本地事务，执行如下操作。
>
> 2. 通过 XID 和 Branch ID 查找到相应的 UNDO LOG 记录。
>
> 3. 数据校验：拿 UNDO LOG 中的后镜与当前数据进行比较，如果有不同，说明数据被当前全局事务之外的动作做了修改。这种情况，需要根据配置策略来做处理，详细的说明在另外的文档中介绍。
>
> 4. 根据 UNDO LOG 中的前镜像和业务 SQL 的相关信息生成并执行回滚的语句：
>
>    `update product set name = 'TXC' where id = 1;`
>
> 5. 提交本地事务。并把本地事务的执行结果（即分支事务回滚的结果）上报给 TC。
>
> **二阶段-提交**
>
> 1. 收到 TC 的分支提交请求，把请求放入一个异步任务的队列中，马上返回提交成功的结果给 TC。
> 2. 异步任务阶段的分支提交请求将异步和批量地删除相应 UNDO LOG 记录。

简单的总结来说就是，Seata的一阶段会在数据库本地事务中提交，这其中会将回滚信息，分支信息存入seata的数据库表中，然后在第二阶段seata会根据提交或回滚进行不同的操作：如果是提交那么就返回提交并异步删除undo_log等表记录数据；如果是回滚，则根据之前记录的回滚日志进行回滚，然后将结果返回给TC。

# 四、Seata的TCC模式

## 4.1 Seata的TCC模式的使用

### 1 RM端改造

### 2 TM端改造

## 4.2 Seata的TCC模式工作机制介绍

