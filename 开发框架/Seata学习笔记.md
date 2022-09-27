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



