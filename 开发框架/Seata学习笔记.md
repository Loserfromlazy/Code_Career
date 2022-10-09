# Seata学习和源码剖析

> 本笔记的项目地址[LearnSpringCloud](https://github.com/Loserfromlazy/LearnSpringCloud)。此项目是我个人的SpringCloud学习的统一项目，我们这篇笔记里Seata相关的项目均是seata-xxx形式的。
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

## 4.1 TCC模式介绍

### 1 TCC模式

TCC的工作模式与AT类似，在AT模式中是基于支持本地 ACID事务 的关系型数据库；但是TCC模式不依赖于底层数据资源的事务支持，而是把 自定义 的分支事务纳入到全局事务的管理中。TCC模式各阶段工作如下：

- 一阶段 prepare 行为：调用 自定义 的 prepare 逻辑。
- 二阶段 commit 行为：调用 自定义 的 commit 逻辑。
- 二阶段 rollback 行为：调用 自定义 的 rollback 逻辑。

也就是说，我们的提交和回滚逻辑全部都是自定义完成的。因此TCC模式对业务的侵入性很高，Seata官方的建议是使用AT模式。

### 2 入门案例的改造思路

首先我们在数据库表中增加字段（如下图，每张表的最后一个字段为新加的字段）：

![image-20221004220414199](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221004220414199.png)

我们的思路是在prepare阶段（即try阶段），将数据存入我们为seata准备的中间字段中，比如我们调用购买商品接口时，设置新增订单状态，并将积分存入中间积分中，扣减的库存存入中间库存字段中。然后在第二阶段时进行处理，比如二阶段如果是提交那么就将中间库存数据在剩余库存中扣减，然后清除中间库存，其它的类似这么处理。然后如果是回滚，那么就清楚该状态的数据，或清除中间数据完成回滚。

> 注意我们的入门案例中，seata_client项目为TM端，其余为RM端

## 4.2 Seata的TCC模式的使用

### 1 RM端改造

在TM端我们需要手动指定Order的id，并去掉实体类中的id自增。因为我们的入门案例除了自增id以外没有一个有唯一索引的字段，这样seata在提交方法中无法获取这个订单，所以我们这里简单的通过手动设置id的方式进行处理，这是TM端即seata_client模块的改动。**注意：TCC模式依旧要使用`@GlobalTransactional`注解。**

> 这里是我一开始的设计失误，为了入门案例简单而没有考虑到的情况。

我们下面主要对三个作为RM端使用的项目进行改造。

首先我们对接口进行改造：以seata_order模块为例，先用`@LocalTCC`标注seata接口，然后使用`@TwoPhaseBusinessAction`标注一阶段try方法。此注解可以指明二阶段提交和回滚方法，然后我们在接口声明出对应的提交和回滚方法，注意这两个方法返回值必须为boolean，代码如下：

```java
@LocalTCC//表示接口被seata管理
public interface OrderTCCService extends IService<Order> {

    //声明try方法，其中name为全局唯一属性
    @TwoPhaseBusinessAction(name = "addOrder",commitMethod = "commitOrder",rollbackMethod = "rollbackOrder")
    Result<Boolean> addOrder(@BusinessActionContextParameter(paramName = "order") Order order);

    //这里返回值必须为boolean
    boolean commitOrder(BusinessActionContext context);
    //这里返回值必须为boolean
    boolean rollbackOrder(BusinessActionContext context);
}
```

然后我们按照逻辑实现对应的方法即可：

```java
@Override
    public Result<Boolean> addOrder(Order order) {
        order.setCreateTime(new Date());
        int notUse = 0;
        //0为不可用，事务未提交 1为可用，事务已经提交
        order.setStatus(notUse);//预检查
        try {
            orderMapper.insert(order);
        } catch (Exception e) {
            log.error("插入订单失败，错误信息{}", e.getMessage());
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
            return ResultUtils.resultInit(0, e.getMessage(), false);
        }
        log.info("新增订单成功");
        return ResultUtils.successBuild(true);
    }

    @Override
    public boolean commitOrder(BusinessActionContext context) {
        Object orderJson = context.getActionContext("order");
        Order order = JSON.parseObject(orderJson.toString(), Order.class);
        Order orderSelect = orderMapper.selectById(order.getId());
        if (orderSelect!=null){
            //二阶段提交
            orderSelect.setStatus(1);
            this.saveOrUpdate(orderSelect);
        }
        log.info("提交成功，xid={}",context.getXid());
        //这里需要返回true，如果为false会不断的去重试，可以在配置中心配置重试策略
        return true;
    }

    @Override
    public boolean rollbackOrder(BusinessActionContext context) {
        Object orderJson = context.getActionContext("order");
        Order order = JSON.parseObject(orderJson.toString(), Order.class);
        if (order!=null){
            //二阶段回滚，删除订单
            orderMapper.deleteById(order.getId());
        }
        log.info("回滚成功，xid={}",context.getXid());
        //这里需要返回true，如果为false会不断的去重试，可以在配置中心配置重试策略
        return true;
    }
```

修改完代码后，我们可以在配置文件中关闭数据库代理，因为seata的TCC模式不需要对数据库进行代理：

```yaml
seata:
  application-id: ${spring.application.name}
  enable-auto-data-source-proxy: false
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
  #data-source-proxy-mode: AT
  enabled: true
```

其他两个模块跟此模块一样，这里就不贴代码了。

总结TCC模式的使用流程就是：

1. 首先通过@LocalTCC指定seata管理的接口
2. 然后通过@TwoPhaseBusinessAction指明seata一、二阶段的方法，最后结合业务进行实现。

### 2 测试TCC模式

改造完后，我们来进行测试,首先测试能否正常购买商品，我们访问`http://localhost:8075/test/testBuy1`,然后我们可以在控制台查看三个模块都打印了我们的日志提交成功，且数据库数据均正确，日志图片如下（这里只展示一个服务的日志）：

![image-20221005215945522](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221005215945522.png)

然后看数据库数据，原始数据如下：

![image-20221005215915027](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221005215915027.png)

方法执行完后数据如下：

![image-20221005220106240](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221005220106240.png)

我们可以看到数据均是正常的。

然后我们将数据恢复为原始数据，并访问`http://localhost:8075/test/testBuy2`

我们查看日志，这里以warehouse服务为例：

![image-20221005220338889](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221005220338889.png)

然后在看数据库，发现数据没有变化，回滚成功。以上就是Seata的TCC模式的使用。

> PS：我在使用测试seata时如果重启一部分服务，那么没重启的服务就会事务失效，比如我重启了我的client和order项目，但是没重启user和warehouse模块，那么这俩没重启的模块就不会执行事务，或者说不参与此事务，日志如下：
>
> order模块日志：
>
> ~~~
> 2022-10-05 21:51:46.181  INFO 21048 --- [nio-8076-exec-5] c.l.service.impl.OrderTCCServiceImpl     : 新增订单成功
> 2022-10-05 21:51:46.185  WARN 21048 --- [nio-8076-exec-5] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.123.169:8091:2044947731123859477 to null
> 2022-10-05 21:51:46.490  INFO 21048 --- [h_RMROLE_1_3_24] i.s.c.r.p.c.RmBranchCommitProcessor      : rm client handle branch commit process:xid=192.168.123.169:8091:2044947731123859477,branchId=2044947731123859478,branchType=TCC,resourceId=addOrder,applicationData={"actionContext":{"action-start-time":1664977906076,"useTCCFence":false,"sys::prepare":"addOrder","sys::rollback":"rollbackOrder","sys::commit":"commitOrder","host-name":"192.168.123.169","order":{"createTime":1664977906072,"goodsId":1,"id":1,"name":"购买商品1","nums":10,"userId":1},"actionName":"addOrder"}}
> 2022-10-05 21:51:46.490  INFO 21048 --- [h_RMROLE_1_3_24] io.seata.rm.AbstractRMHandler            : Branch committing: 192.168.123.169:8091:2044947731123859477 2044947731123859478 addOrder {"actionContext":{"action-start-time":1664977906076,"useTCCFence":false,"sys::prepare":"addOrder","sys::rollback":"rollbackOrder","sys::commit":"commitOrder","host-name":"192.168.123.169","order":{"createTime":1664977906072,"goodsId":1,"id":1,"name":"购买商品1","nums":10,"userId":1},"actionName":"addOrder"}}
> 2022-10-05 21:51:46.604  INFO 21048 --- [h_RMROLE_1_3_24] c.l.service.impl.OrderTCCServiceImpl     : 提交成功，xid=192.168.123.169:8091:2044947731123859477
> 2022-10-05 21:51:46.604  INFO 21048 --- [h_RMROLE_1_3_24] io.seata.rm.AbstractResourceManager      : TCC resource commit result : true, xid: 192.168.123.169:8091:2044947731123859477, branchId: 2044947731123859478, resourceId: addOrder
> 2022-10-05 21:51:46.604  INFO 21048 --- [h_RMROLE_1_3_24] io.seata.rm.AbstractRMHandler            : Branch commit result: PhaseTwo_Committed
> ~~~
>
> user模块日志：
>
> ~~~
> 2022-10-05 21:51:46.334  INFO 6928 --- [nio-8077-exec-4] c.learn.service.impl.UserTCCServiceImpl  : 更新积分成功
> 2022-10-05 21:51:46.338  WARN 6928 --- [nio-8077-exec-4] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.123.169:8091:2044947731123859477 to null
> ~~~
>
> warehouse模块日志：
>
> ~~~
> 2022-10-05 21:51:46.479  INFO 11600 --- [nio-8078-exec-3] c.l.service.impl.GoodsTCCServiceImpl     : 扣减库存成功
> 2022-10-05 21:51:46.482  WARN 11600 --- [nio-8078-exec-3] c.a.c.seata.web.SeataHandlerInterceptor  : xid in change during RPC from 192.168.123.169:8091:2044947731123859477 to null
> ~~~
>
> 此问题暂时未找到原因，如有大佬懂此处，请指教。我的解决办法是，全部重启服务。

# 五、Seata源码学习

## 5.1 seata源码工程搭建

首先我们需要到[官方下载地址](https://seata.io/zh-cn/blog/download.html)下载seata源码并解压,然后导入到IDEA中，然后我们进行编译，如下图：

![image-20221006095625214](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006095625214.png)

**注意：seata需要在jdk1.8的环境下才能运行**

编译完成后可以在seata-server模块下编辑配置文件，可以直接将上面使用的配置文件拿过来，然后启动ServerApplication的main方法(如果是以前的版本那就是Server的main方法)。启动完成后nacos能注册上即可：

![image-20221006101855115](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006101855115.png)

> 注意：如果debug无法启动，错误信息如下：
>
> ![image-20221008110959787](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221008110959787.png)
>
> 这是新的Kotlin 1.4协程调试器中的一个错误，我们只需要在IDEA中关闭即可，如下图：
>
> ![image-20221008111048966](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221008111048966.png)

我们简单的了解一下seata各模块的作用：

- seata-common：提供seata工具类、异常类等。
- seata-core：提供RPC封装、数据模型、通信格式等。
- seata-config：从配置中心读取配置
- seata-discovery：用于TC注册到注册中心、TM从注册中心服务发现TC
- seata-rm：RM的核心实现
- seata-rm-datasource：对JDBC拓展，实现对MySQL等透明接入RM
- seata-server：对TC的核心实现，提供了事务协调、锁、事务状态等功能。
- seata-tm：对TM的实现，提供了全局事务管理，比如事务的提交回滚等。
- seata-tcc：对TCC事务模式的实现
- seata-spring：Spring对Seata集成的实现，比如@GlobalTransaction注解等

## 5.2 AT模式TM/RM注册TC

### 5.2.1 TM/RM注册流程

我们依旧从自动装配的spring.factories配置文件入手，进入到SeataAutoConfiguration自动配置类（在以前的seata版本可能入口在GlobalTransactionAutoConfiguration，在spring-cloud-alibaba-seata包下），如下图：

![image-20221006134659516](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006134659516.png)

这个配置类中会注入一个GlobalTransactionScanner全局事务扫描器的Bean，在这里会配置一些seata的配置，源码如下图：

![image-20221006140113806](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006140113806.png)

同时，GlobalTransactionScanner继承了InitialzingBean接口，因此我们重点关注一下afterPropertiesSet方法，源码如下：

> Spring启动后，初始化Bean时，若该Bean实现InitialzingBean接口，会自动调用afterPropertiesSet()方法，完成一些用户自定义的初始化操作

```java
@Override
public void afterPropertiesSet() {
    if (disableGlobalTransaction) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Global transaction is disabled.");
        }
        ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
                (ConfigurationChangeListener)this);
        return;
    }
    if (initialized.compareAndSet(false, true)) {
        //初始化客户端
        initClient();
    }
}
```

然后我们主要跟进initClient方法中：

```java
//GlobalTransactionScanner#initClient
private void initClient() {
    //。。省略部分代码
    //init TM
    TMClient.init(applicationId, txServiceGroup, accessKey, secretKey);
    if (LOGGER.isInfoEnabled()) {
        //打印日志
    }
    //init RM
    RMClient.init(applicationId, txServiceGroup);
    //。。省略部分代码
    registerSpringShutdownHook();

}
```

此方法中有两个比较重要的部分那就是TM和RM的客户端初始化，这俩个客户端的初始化过程类似，我们先看TM的初始化过程。

我们先跟进`TMClient.init`方法中： 

```java
public static void init(String applicationId, String transactionServiceGroup, String accessKey, String secretKey) {
    //创建Netty远程客户端
    TmNettyRemotingClient tmNettyRemotingClient = TmNettyRemotingClient.getInstance(applicationId, transactionServiceGroup, accessKey, secretKey);
    //初始化
    tmNettyRemotingClient.init();
}
```

我们先跟进getInstance方法中：

```java
//TmNettyRemotingClient#getInstance(...)
public static TmNettyRemotingClient getInstance(String applicationId, String transactionServiceGroup, String accessKey, String secretKey) {
    TmNettyRemotingClient tmRpcClient = getInstance();
    tmRpcClient.setApplicationId(applicationId);
    tmRpcClient.setTransactionServiceGroup(transactionServiceGroup);
    tmRpcClient.setAccessKey(accessKey);
    tmRpcClient.setSecretKey(secretKey);
    return tmRpcClient;
}
//---->
public static TmNettyRemotingClient getInstance() {
    if (instance == null) {
        synchronized (TmNettyRemotingClient.class) {
            if (instance == null) {
                NettyClientConfig nettyClientConfig = new NettyClientConfig();
                final ThreadPoolExecutor messageExecutor = new ThreadPoolExecutor(
                        nettyClientConfig.getClientWorkerThreads(), nettyClientConfig.getClientWorkerThreads(),
                        KEEP_ALIVE_TIME, TimeUnit.SECONDS,
                        new LinkedBlockingQueue<>(MAX_QUEUE_SIZE),
                        new NamedThreadFactory(nettyClientConfig.getTmDispatchThreadPrefix(),
                                nettyClientConfig.getClientWorkerThreads()),
                        RejectedPolicies.runsOldestTaskPolicy());
                //新建TmNettyRemotingClient对象
                instance = new TmNettyRemotingClient(nettyClientConfig, null, messageExecutor);
            }
        }
    }
    return instance;
}
```

然后我们进入到TmNettyRemotingClient构造函数，这里会调用父类构造函数，在父类构造函数中会初始化Netty的客户端启动助手，并新增Handler处理器，源码如下图：

![image-20221006142714073](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006142714073.png)

然后创建完TmNettyRemotingClient之后，回到`TMClient.init`方法中，接下来会调用`tmNettyRemotingClient.init()`方法：

```java
//TmNettyRemotingClient#init
@Override
public void init() {
    // registry processor注册处理器
    registerProcessor();
    if (initialized.compareAndSet(false, true)) {
        super.init();//调用init方法
        if (io.seata.common.util.StringUtils.isNotBlank(transactionServiceGroup)) {
            getClientChannelManager().reconnect(transactionServiceGroup);
        }
    }
}
```

在这里会先注册处理器，然后执行init方法，我们分别来看这个两个方法：

1. registerProcessor()

   ```java
   private void registerProcessor() {
       // 1.registry TC response processor注册TC响应处理器
       ClientOnResponseProcessor onResponseProcessor =
               new ClientOnResponseProcessor(mergeMsgMap, super.getFutures(), getTransactionMessageHandler());
       super.registerProcessor(MessageType.TYPE_SEATA_MERGE_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_GLOBAL_BEGIN_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_GLOBAL_COMMIT_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_GLOBAL_REPORT_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_GLOBAL_ROLLBACK_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_GLOBAL_STATUS_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_REG_CLT_RESULT, onResponseProcessor, null);
       super.registerProcessor(MessageType.TYPE_BATCH_RESULT_MSG, onResponseProcessor, null);
       // 2.registry heartbeat message processor 注册心跳信息处理器
       ClientHeartbeatProcessor clientHeartbeatProcessor = new ClientHeartbeatProcessor();
       super.registerProcessor(MessageType.TYPE_HEARTBEAT_MSG, clientHeartbeatProcessor, null);
   }
   ```

   这里调用父类的registerProcessor方法进行注册，此方法本质上是保存到map中：

   ```java
   
   protected final HashMap<Integer/*MessageType*/, Pair<RemotingProcessor, ExecutorService>> processorTable = new HashMap<>(32);
   
   @Override
   public void registerProcessor(int requestCode, RemotingProcessor processor, ExecutorService executor) {
       Pair<RemotingProcessor, ExecutorService> pair = new Pair<>(processor, executor);
       this.processorTable.put(requestCode, pair);
   }
   ```

2. super.init()

   此方法中会创建一个定时任务，主要作用是检测与server的连接是否断开，如果断开就重连并重新注册到server端。然后会启动ClientBootstrap。源码如下：

   ```java
   @Override
   public void init() {
       //启动了一个定时任务，这个定时任务会判断跟server端的连接是否还有效，如果已经断开，就会重新连上，并且重新把tm注册到server
       timerExecutor.scheduleAtFixedRate(new Runnable() {
           @Override
           public void run() {
               //检测、连接netty服务器
               clientChannelManager.reconnect(getTransactionServiceGroup());
           }
       }, SCHEDULE_DELAY_MILLS, SCHEDULE_INTERVAL_MILLS, TimeUnit.MILLISECONDS);
       if (this.isEnableClientBatchSendRequest()) {
           mergeSendExecutorService = new ThreadPoolExecutor(MAX_MERGE_SEND_THREAD,
               MAX_MERGE_SEND_THREAD,
               KEEP_ALIVE_TIME, TimeUnit.MILLISECONDS,
               new LinkedBlockingQueue<>(),
               new NamedThreadFactory(getThreadPrefix(), MAX_MERGE_SEND_THREAD));
           mergeSendExecutorService.submit(new MergedSendRunnable());
       }
       super.init();
       //启动ClientBootstrap
       clientBootstrap.start(); 
   }
   ```

   然后我们先跟进reconnect方法中：此方法先获取server端地址，然后根据地址获取Channel，源码如下图：

   ![image-20221006230715783](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006230715783.png)

   主要逻辑在acquireChannel方法中，我们跟进此方法：

   ```java
   Channel acquireChannel(String serverAddress) {
       //根据地址获取channel
       Channel channelToServer = channels.get(serverAddress);
       if (channelToServer != null) {
           channelToServer = getExistAliveChannel(channelToServer, serverAddress);
           if (channelToServer != null) {
               //如果能取到Channel且Channel依旧alive那么就直接返回Channel
               return channelToServer;
           }
       }
       if (LOGGER.isInfoEnabled()) {
           LOGGER.info("will connect to {}", serverAddress);
       }
       Object lockObj = CollectionUtils.computeIfAbsent(channelLocks, serverAddress, key -> new Object());
       //如果取不到Channel或Channel不是alive状态，就进行连接。
       synchronized (lockObj) {
           return doConnect(serverAddress);
       }
   }
   ```

   我们跟进doConnect方法，此方法中主要逻辑就是从对象池中获取Channel，源码如下图：

   ![image-20221006231446150](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006231446150.png)

   我们看构造函数：

   ![image-20221006231503103](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006231503103.png)

   我们注意在创建对象池时会传入一个工厂，当我们调用borrowObject获取对象时如果获取不到就会调用工厂去创建。最终的连接server端和注册是在此工厂的makeObject方法中完成的，源码流程如下：

   ![image-20221006232144435](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006232144435.png)
   
   我们可以debug查看注册请求的具体信息，如下图：
   
   ![image-20221008112033891](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221008112033891.png)
   
   进入到sendSyncRequest方法中，会构建消息：
   
   ![image-20221008112757229](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221008112757229.png)
   
   这里发送请求本质上是将信息写入通道，发送到Netty的Server端。

我们接下来回到`GlobalTransactionScanner#initClient`

接下来我们跟进RMClient.init中看一下：

```java
public static void init(String applicationId, String transactionServiceGroup) {
    RmNettyRemotingClient rmNettyRemotingClient = RmNettyRemotingClient.getInstance(applicationId, transactionServiceGroup);
    rmNettyRemotingClient.setResourceManager(DefaultResourceManager.get());
    rmNettyRemotingClient.setTransactionMessageHandler(DefaultRMHandler.get());
    rmNettyRemotingClient.init();
}
```

我们可以看到基本没有太大的区别，都是获取对应的XXNettyRemotingClient并调用init方法。只不过RM需要额外设置ResourceManager和TransactionMessageHandler。

接下来我们跟进`rmNettyRemotingClient.init()`方法，源码如下：

```java
@Override
public void init() {
    // registry processor
    //调用RmNettyRemotingClient#registerProcessor
    registerProcessor();
    if (initialized.compareAndSet(false, true)) {
        super.init();

        // Found one or more resources that were registered before initialization
        if (resourceManager != null
                && !resourceManager.getManagedResources().isEmpty()
                && StringUtils.isNotBlank(transactionServiceGroup)) {
            getClientChannelManager().reconnect(transactionServiceGroup);
        }
    }
}
```

我们可以看到RM端也是先进行注册然后调用父类的init方法，而TM和RM端的父类都是AbstractNettyRemotingClient，他的init方法我们在上面已经介绍了，这里就不再赘述，也就是说RM和TM的注册流程基本是一样的。

综上，是AT模式TM/RM注册的整体流程源码分析。

### 5.2.2 注册TC流程分析

在seata服务端，我们是通过ServerApplication启动的（在以前的版本是通过Server.main方法启动的），此方法就是启动了springboot项目，因此实际的入口在ServerRunner类中，此方法继承了CommandLineRunner接口，因此我们主要关注其run方法：

```java
@Override
public void run(String... args) {
    try {
        long start = System.currentTimeMillis();
        Server.start(args);
        started = true;

        long cost = System.currentTimeMillis() - start;
        LOGGER.info("seata server started in {} millSeconds", cost);
    } catch (Throwable e) {
        //部分代码略
    }
}
```

在这里主要调用了Server.start方法。源码如下：

![image-20221006234614760](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221006234614760.png)

我们可以看到在Server.start方法中主要是新建了一个NettyRemotingServer并调用了其init方法。我们先来看一下NettyRemotingServer的构造函数：

```java
//NettyRemotingServer extends AbstractNettyRemotingServer
public NettyRemotingServer(ThreadPoolExecutor messageExecutor) {
    super(messageExecutor, new NettyServerConfig());
}
//------> AbstractNettyRemotingServer构造函数：
public AbstractNettyRemotingServer(ThreadPoolExecutor messageExecutor, NettyServerConfig nettyServerConfig) {
    super(messageExecutor);
    //新建服务端Netty启动助手包装类
    serverBootstrap = new NettyServerBootstrap(nettyServerConfig);
    //设置服务端处理器
    serverBootstrap.setChannelHandlers(new ServerHandler());
}
```

我们可以看到新建NettyRemotingServer时，会将Netty的启动助手完成创建并配置服务端的通道处理器。

然后我们关注其init方法，代码如下：

```java
@Override
public void init() {
    // registry processor
    registerProcessor();
    if (initialized.compareAndSet(false, true)) {
        super.init();
    }
}
```

这个方法很眼熟，可以说和客户端的init方法几乎没有区别，在`registerProcessor()`方法中，还是注册了一些请求和心跳处理器。然后我们进入super.init()方法，代码如下：

```java
//AbstractNettyRemotingServer#init
@Override
public void init() {
    super.init();
    serverBootstrap.start();
}
//----->NettyServerBootstrap#start
@Override
    public void start() {
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupWorker)
            .channel(NettyServerConfig.SERVER_CHANNEL_CLAZZ)
            .option(ChannelOption.SO_BACKLOG, nettyServerConfig.getSoBackLogSize())
            .option(ChannelOption.SO_REUSEADDR, true)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSendBufSize())
            .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketResvBufSize())
            .childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
                new WriteBufferWaterMark(nettyServerConfig.getWriteBufferLowWaterMark(),
                    nettyServerConfig.getWriteBufferHighWaterMark()))
            .localAddress(new InetSocketAddress(getListenPort()))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) {
                    ch.pipeline().addLast(new IdleStateHandler(nettyServerConfig.getChannelMaxReadIdleSeconds(), 0, 0))
                        .addLast(new ProtocolV1Decoder())
                        .addLast(new ProtocolV1Encoder());
                    if (channelHandlers != null) {
                        addChannelPipelineLast(ch, channelHandlers);
                    }

                }
            });

        try {
            this.serverBootstrap.bind(getListenPort()).sync();
            XID.setPort(getListenPort());
            LOGGER.info("Server started, service listen port: {}", getListenPort());
            RegistryFactory.getInstance().register(new InetSocketAddress(XID.getIpAddress(), XID.getPort()));
            initialized.set(true);
        } catch (SocketException se) {
            throw new RuntimeException("Server start failed, the listen port: " + getListenPort(), se);
        } catch (Exception exx) {
            throw new RuntimeException("Server start failed", exx);
        }
    }
```

我们可以看到在这里会将Netty服务端进行启动。

在5.2.1中也讲过，TM/RM会连接Netty服务端发送注册请求，也就是将数据通过通道发送到Netty的Server端。因此我们回到TC的AbstractNettyRemotingServer类的构造函数中：在这里设置了netty的handler处理器ServerHandler。我们进入此类查看器channelRead方法：

```java
@Override
public void channelRead(final ChannelHandlerContext ctx, Object msg) throws Exception {
    if (!(msg instanceof RpcMessage)) {
        return;
    }
    processMessage(ctx, (RpcMessage) msg);
}
```

这里会将RpcMessage的消息交给processMessage方法进行处理，我们跟进此方法：

```java
protected void processMessage(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug(String.format("%s msgId:%s, body:%s", this, rpcMessage.getId(), rpcMessage.getBody()));
    }
    Object body = rpcMessage.getBody();
    if (body instanceof MessageTypeAware) {
        //将消息强制转换成MessageTypeAware，此接口中有一个getTypeCode方法，获取当前请求的类型
        MessageTypeAware messageTypeAware = (MessageTypeAware) body;
        //根据type类型，调用不同的处理器进行处理，这里的处理器就是上面注册处理器时注册的
        final Pair<RemotingProcessor, ExecutorService> pair = this.processorTable.get((int) messageTypeAware.getTypeCode());
        if (pair != null) {
            if (pair.getSecond() != null) {
                try {
                    pair.getSecond().execute(() -> {
                        try {
                            pair.getFirst().process(ctx, rpcMessage);
                        } catch (Throwable th) {
                            LOGGER.error(FrameworkErrorCode.NetDispatch.getErrCode(), th.getMessage(), th);
                        } finally {
                            MDC.clear();
                        }
                    });
                } catch (RejectedExecutionException e) {
                    //省略部分代码
                }
            } else {
                try {
                    pair.getFirst().process(ctx, rpcMessage);
                } catch (Throwable th) {
                    //省略日志代码
                }
            }
        } else {
            //省略日志代码
        }
    } else {
        //省略日志代码
    }
}
```

在此方法中根据type调用不同的处理器进行处理，TC端处理注册的主要是下面两个处理器：

![image-20221008131831870](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221008131831870.png)

我们这里以RM为例，我们跟进RegRmProcessor的process方法，此方法中核心代码就是`ChannelManager.registerRMChannel()`方法，在此方法中完成了RM的注册处理，注册完后会创建响应消息并回写给客户端，源码如下：

```java
@Override
public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
    onRegRmMessage(ctx, rpcMessage);
}

private void onRegRmMessage(ChannelHandlerContext ctx, RpcMessage rpcMessage) {
    //获取消息内容
    RegisterRMRequest message = (RegisterRMRequest) rpcMessage.getBody();
    String ipAndPort = NetUtil.toStringAddress(ctx.channel().remoteAddress());
    boolean isSuccess = false;
    String errorInfo = StringUtils.EMPTY;
    try {
        if (null == checkAuthHandler || checkAuthHandler.regResourceManagerCheckAuth(message)) {
            //注册处理消息
            ChannelManager.registerRMChannel(message, ctx.channel());
            Version.putChannelVersion(ctx.channel(), message.getVersion());
            isSuccess = true;
            if (LOGGER.isDebugEnabled()) {
                //日志代码省略
            }
        } else {
            if (LOGGER.isWarnEnabled()) {
                //日志代码省略
            }
        }
    } catch (Exception exx) {
        isSuccess = false;
        errorInfo = exx.getMessage();
        //日志代码省略
    }
    //新建响应消息
    RegisterRMResponse response = new RegisterRMResponse(isSuccess);
    if (StringUtils.isNotEmpty(errorInfo)) {
        response.setMsg(errorInfo);
    }
    //写回响应消息
    remotingServer.sendAsyncResponse(rpcMessage, ctx.channel(), response);
    if (isSuccess && LOGGER.isInfoEnabled()) {
        //日志代码省略
    }
}
```

然后我们主要跟进registerRMChannel方法：

```java
public static void registerRMChannel(RegisterRMRequest resourceManagerRequest, Channel channel)
    throws IncompatibleVersionException {
    //校验版本
    Version.checkVersion(resourceManagerRequest.getVersion());
    Set<String> dbkeySet = dbKeytoSet(resourceManagerRequest.getResourceIds());
    RpcContext rpcContext;
    if (!IDENTIFIED_CHANNELS.containsKey(channel)) {
        //新建RpcContext对象
        rpcContext = buildChannelHolder(NettyPoolKey.TransactionRole.RMROLE, resourceManagerRequest.getVersion(),
            resourceManagerRequest.getApplicationId(), resourceManagerRequest.getTransactionServiceGroup(),
            resourceManagerRequest.getResourceIds(), channel);
        //此方法将rpcContext和IDENTIFIED_CHANNELS绑定，并将当前channel和rpcContext绑定
        //IDENTIFIED_CHANNELS是一个Map，维护着channel与RpcContext之间的关系，所以通过RpcContext对象可以查看到所有的连接以及对应的RpcContext。
        rpcContext.holdInIdentifiedChannels(IDENTIFIED_CHANNELS);
    } else {
        rpcContext = IDENTIFIED_CHANNELS.get(channel);
        rpcContext.addResources(dbkeySet);
    }
    if (dbkeySet == null || dbkeySet.isEmpty()) { return; }
    //遍历注册数据库资源
    for (String resourceId : dbkeySet) {
        String clientIp;
        ConcurrentMap<Integer, RpcContext> portMap = CollectionUtils.computeIfAbsent(RM_CHANNELS, resourceId, key -> new ConcurrentHashMap<>())
                .computeIfAbsent(resourceManagerRequest.getApplicationId(), key -> new ConcurrentHashMap<>())
                .computeIfAbsent(clientIp = ChannelUtil.getClientIpFromChannel(channel), key -> new ConcurrentHashMap<>());
		//此方法将端口和RpcContext的关系保存进portMap中。
        rpcContext.holdInResourceManagerChannels(resourceId, portMap);
        //更新RM的资源信息
        updateChannelsResource(resourceId, clientIp, resourceManagerRequest.getApplicationId());
    }
}
//更新资源的原因是如果与RM的连接断开了，可以使用其他通道与RM进行通讯，因为RM与分支事务有关，比如通知分支事务回滚，而此时与RM的连接断开了，那么seata会选择同一个IP上同一个应用的不同端口的连接进行通知，以此来保证事务的一致性
private static void updateChannelsResource(String resourceId, String clientIp, String applicationId) {
    ConcurrentMap<Integer, RpcContext> sourcePortMap = RM_CHANNELS.get(resourceId).get(applicationId).get(clientIp);
    for (ConcurrentMap.Entry<String, ConcurrentMap<String, ConcurrentMap<String, ConcurrentMap<Integer,RpcContext>>>> rmChannelEntry : RM_CHANNELS.entrySet()) {
        //判断如果资源已经注册过则跳过
        if (rmChannelEntry.getKey().equals(resourceId)) { continue; }
        ConcurrentMap<String, ConcurrentMap<String, ConcurrentMap<Integer,
        RpcContext>>> applicationIdMap = rmChannelEntry.getValue();
        //如果应用不同则跳过
        if (!applicationIdMap.containsKey(applicationId)) { continue; }
        ConcurrentMap<String, ConcurrentMap<Integer,
        RpcContext>> clientIpMap = applicationIdMap.get(applicationId);
        //如果ip不同则跳过
        if (!clientIpMap.containsKey(clientIp)) { continue; }
        ConcurrentMap<Integer, RpcContext> portMap = clientIpMap.get(clientIp);
        //遍历资源信息
        for (ConcurrentMap.Entry<Integer, RpcContext> portMapEntry : portMap.entrySet()) {
            Integer port = portMapEntry.getKey();
            //如果新资源不包含旧资源的端口
            if (!sourcePortMap.containsKey(port)) {
                //将旧资源的端口和RpcContext保存进sourcePortMap（sourcePortMap就是新资源）
                RpcContext rpcContext = portMapEntry.getValue();
                sourcePortMap.put(port, rpcContext);
                //将新资源保存到旧资源的RpcContext上
                rpcContext.holdInResourceManagerChannels(resourceId, port);
            }
        }
    }
}
```

以上就是TC处理RM注册的流程，而TM的大体流程与RM类似，这里就不再赘述，感兴趣请自行跟踪源码。

### 5.2.3 AT模式TM/RM注册TC的整体流程

![Seata TM_RM注册TC](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Seata%20TM_RM%E6%B3%A8%E5%86%8CTC.png)

## 5.3 TM开启全局事务

我们依旧从自动装配入手，然后我们来到GlobalTransactionScanner类：

```java
public class GlobalTransactionScanner extends AbstractAutoProxyCreator
        implements ConfigurationChangeListener, InitializingBean, ApplicationContextAware, DisposableBean
```

我们看这里它继承了AbstractAutoProxyCreator，这是Spring的自动创建动态代理的类，它继承了BeanPostProcessor，当Bean初始化后会调用postProcessAfterInitialization方法进行自动代理相关的操作，而其中就有AbstractAutoProxyCreator的postProcessAfterInitialization方法，此方法最终会进入到wrapIfNecessary判断是否需要代理，如果需要则创建并返回代理类（更具体的原理可以看我的Spring的博客：[Spring学习和源码解析](https://www.cnblogs.com/yhr520/p/12554829.html)）。

而GlobalTransactionScanner主要通过复写wrapIfNecessary方法，来创建并返回代理对象（如果需要进行代理的话）。此方法源码如下：

```java
@Override
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    // do checkers做检查，比如此类是否已经生成了代理
    if (!doCheckers(bean, beanName)) {
        return bean;
    }

    try {
        //对已经被代理过的集合加锁
        synchronized (PROXYED_SET) {
            //如果已经被自动代理过则直接返回
            if (PROXYED_SET.contains(beanName)) {
                return bean;
            }
            interceptor = null;
            //check TCC proxy 检查是否是TCC模式
            if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
                // init tcc fence clean task if enable useTccFence
                TCCBeanParserUtils.initTccFenceCleanTask(TCCBeanParserUtils.getRemotingDesc(beanName), applicationContext);
                //TCC interceptor, proxy bean of sofa:reference/dubbo:reference, and LocalTCC
                interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
                ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
                        (ConfigurationChangeListener)interceptor);
            } else {
                //获取当前类的接口
                Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
                Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);
				//通过检查类或方法上是否有GlobalTransactional注解判断是否需要动态代理
                if (!existsAnnotation(new Class[]{serviceInterface})
                    && !existsAnnotation(interfacesIfJdk)) {
                    return bean;
                }
				
                if (globalTransactionalInterceptor == null) {
                    //创建全局事务方法拦截器
                    globalTransactionalInterceptor = new GlobalTransactionalInterceptor(failureHandlerHook);
                    ConfigurationCache.addConfigListener(
                            ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
                            (ConfigurationChangeListener)globalTransactionalInterceptor);
                }
                interceptor = globalTransactionalInterceptor;
            }

            LOGGER.info("Bean[{}] with name [{}] would use interceptor [{}]", bean.getClass().getName(), beanName, interceptor.getClass().getName());
            //如果当前Bean没有被AOP代理
            if (!AopUtils.isAopProxy(bean)) {
                //则通过Spring AOP生成代理对象
                bean = super.wrapIfNecessary(bean, beanName, cacheKey);
            } else {
                AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
                Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
                int pos;
                for (Advisor avr : advisor) {
                    // Find the position based on the advisor's order, and add to advisors by pos
                    //找到seata切面的位置
                    pos = findAddSeataAdvisorPosition(advised, avr);
                    advised.addAdvisor(pos, avr);
                }
            }
            //存入已经被代理的类的集合中
            PROXYED_SET.add(beanName);
            return bean;
        }
    } catch (Exception exx) {
        throw new RuntimeException(exx);
    }
}
```



## 5.4 RM分支事务注册

## 5.5 TM/RM处理事务提交和回滚
