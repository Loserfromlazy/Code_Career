# RocketMQ学习笔记

# 一、概述

## 1.1 概述

RocketMQ是阿里巴巴自研的高性能、高吞吐量、低延迟、高可用、高可靠得分布式消息中间件，开源后捐给apache社区。RocketMQ在阿里内部叫做Metaq，RocketMQ是Metaq 3.0之后开源的版本。Metaq在阿里巴巴集团内部、蚂蚁金服、菜鸟等各业务中被广泛使用，接入了上万个应用系统中。并平稳支撑了历年的双十一大促（万亿级的消息），在性能、稳定性、可靠性等方面表现出色，在整个阿里技术体系和大中台战略中发挥着举足轻重的作用。

## 1.2 RocketMQ业务架构

RocketMQ由Producer、Broker、Consumer三部分组成，Producer负责生产消息，Consumer负责消费消息，Broker负责存储消息。Broker在实际部署过程中对应一台服务器，每个Broker可以存储多个Topic的消息，每个Topic得消息也可以分片存储于不同的Broker。Message Queue用于存储消息得物理地址，每个Topic中的消息地址存储于多个Message Queue中。RocketMQ得业务架构如下图：

![image-20230116094803794](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230116094803794.png)

- producer：

  消息发布的角色，支持分布式集群方式部署。Producer通过MQ得负载均衡模块选择相应的Broker集群队列进行消息投递，投递过程支持快速失败并且低延迟。

- consumer：

  消息消费的角色，支持分布式集群方式部署，支持push、pull两种模式对消息的消费，同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。

- name server：

  name server是一个非常简单的Topic路由注册中心，角色类似Dubbo中的zookeeper，支持broker的动态注册与发现。主要包括两个功能：

  1. broker管理：nameserver接收broker集群的注册信息，并且保存下来作为路由信息的基本数据，然后提供心跳检测机制，检查broker是否存活。

  2. 路由信息管理：每个NameServer将保存关于Broker集群的整个路由信息和用于客户端查询的队列信息。然后Producer和Conumser通过NameServer就可以知道整个Broker集群的路由信息，从而进行消息的投递和消费。NameServer通常也是集群的方式部署，各实例间相互不进行信息通讯。

     Broker是向每一台NameServer注册自己的路由信息，所以每一个NameServer实例上面都保存一份完

     整的路由信息。当某个NameServer因某种原因下线了，Broker仍然可以向其它NameServer同步其路由信息，Producer,Consumer仍然可以动态感知Broker的路由的信息。

- broker：

  Broker主要负责消息的存储、投递和查询以及服务高可用保证，为了实现这些功能，Broker包含了以下几个重要子模块。

  Remoting Module：整个Broker的实体，负责处理来自clients端的请求。

  Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息

  Store Service：提供方便简单的API接口处理消息存储到物理硬盘和查询功能。

  HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能。

  Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询。

## 1.3 应用场景



# 参考资料

- 《横扫全网：rocketmq工业级高可用架构原理与实操v2》尼恩
- [RocketMQ官方文档](https://rocketmq.apache.org/zh/docs/4.x/producer/04concept1)