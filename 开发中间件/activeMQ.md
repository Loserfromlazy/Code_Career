# activeMQ学习笔记

# 一、JMS

JMS即Java消息服务（Java Message Service）应用程序接口，是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持（百度百科给出的概述）。

JMS具有两种通信模式：

1. Point-to-Point Messaging Domain 点对点

   　在点对点通信模式中，应用程序由消息队列，发送方，接收方组成。每个消息都被发送到一个特定的队列，接收者从队列中获取消息。队列保留着消息，直到他们被消费或超时。

   ![P2P20220308](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/P2P20220308.png)

2. Publish/Subscribe Messaging Domain 发布订阅模式

   在发布/订阅消息模型中，发布者发布一个消息，该消息通过topic传递给所有的客户端。该模式下，发布者与订阅者都是匿名的，即发布者与订阅者都不知道对方是谁。并且可以动态的发布与订阅Topic。Topic主要用于保存和传递消息，且会一直保存消息直到消息被传递给客户端。

   ![Topic20220308](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Topic20220308.png)

在JMS API出现之前，大部分产品使用“点对点”和“发布/订阅”中的任一方式来进行消息通讯。JMS定义了这两种消息发送模型的规范，它们相互独立。任何JMS的提供者可以实现其中的一种或两种模型，这是它们自己的选择。JMS规范提供了通用接口保证我们基于JMS API编写的程序适用于任何一种模型。

JMS的编程模型，整体结构如下图，主要有以下几个部分：

1. 管理对象、连接工厂(ConnectionFactory)和目的地(Destination)
2. 连接对象(Connection)
3. 会话(Session)
4. 消息生产者(Message  Producer)
5. 消息消费者(Message Comsumer)
6. 消息监听者(Message Listener)

![JSM20220308](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/JSM20220308.png)



# 二、ActiveMQ

ActiveMQ 是Apache出品，最流行的，能力强劲的开源消息总线。ActiveMQ 是一个完全支持JMS1.1和J2EE 1.4规范的 JMS Provider实现,尽管JMS规范出台已经是很久的事情了,但是JMS在当今的J2EE应用中间仍然扮演着特殊的地位。

**特点：**

1. 支持来自Java，C，C ++，C＃，Ruby，Perl，Python，PHP的各种跨语言客户端和协议
2. 完全支持JMS客户端和Message Broker中的企业集成模式
3. 支持许多高级功能，如消息组，虚拟目标，通配符和复合目标
4. 完全支持JMS 1.1和J2EE 1.4，支持瞬态，持久，事务和XA消息
5. Spring支持，以便ActiveMQ可以轻松嵌入到Spring应用程序中，并使用Spring的XML配置机制进行配置
6. 专为高性能集群，客户端 - 服务器，基于对等的通信而设计
7. CXF和Axis支持，以便ActiveMQ可以轻松地放入这些Web服务堆栈中以提供可靠的消息传递
8. 可以用作内存JMS提供程序，非常适合单元测试JMS
9. 支持可插拔传输协议，例如in-VM，TCP，SSL，NIO，UDP，多播，JGroups和JXTA传输
10. 使用JDBC和高性能日志支持非常快速的持久性

## 2.1 名词简介

**Destination**

目的地，JMS Provider负责维护，用于对Message进行管理的对象，Message  Producer需要指定Destination才能发送消息，Message Comsumer需要指定Destination才能收到消息。

**Producer**

消息生产者，负责发送Message到目的地，应用接口为MessageProducer。

**Consumer**

消息消费者，负责从目的地中消费、处理、监听、订阅消息。

**Message**

消息，消息封装一次通信的内容。常见的类型有：StreamMessage、ByteMessage、TextMessage、ObjectMessage、MapMessage

**ConnectionFactory**

连接工厂，用于创建连接的工厂

**Connection**

连接，用于建立访问ActiveMQ连接的类型，由连接工厂创建。

**Session**

会话，一次持久有效有状态的访问，由连接创建

**Queue&Topic**

Queue是队列目的地，Topic是主体目的地，都是Destination的子接口。

**PTP**

点对点消息模型

**PUB&SUB**

发布订阅消息模型

## 2.2 安装ActiveMQ

安装极其简单，下载对应的版本解压即可。linux上进入bin目录使用`./activemq start`启动即可。

> 注意：activeMQ有两个端口，61616和8161。其中61616是activemq的使用端口，8161是web页面管理端口。



>  未完结，后续有时间进行整理