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

如果我们使用AT模式，那么需要在自己的业务库中创建UNDO_LOG表，[下载地址](https://github.com/seata/seata/tree/develop/script/client/at/db)。如下图：

![image-20220930112813585](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220930112813585.png)





