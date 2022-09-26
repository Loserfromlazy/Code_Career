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

假设现在有业务需求，我们客户在买东西后，后台需要执行很多步骤比如需要新增订单、积分返现并扣减库存，如下图：

![image-20220926142236907](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220926142236907.png)

我们现在创建项目实现此功能，首先创建我们需要的四个项目，并引入nacos、feign和mybatisplus等依赖，如下图：

![image-20220926142859407](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220926142859407.png)

