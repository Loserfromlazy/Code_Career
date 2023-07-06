# RocketMQ学习笔记

> 本文是RocketMQ的学习笔记，是对RocketMQ的学习过程的记录，便于日后复习翻阅。
>
> **未经授权请勿转载本笔记。创作不易，违者必究！！！**。
>
> PS：本文的目的是作为学习笔记，禁止转载，博客只会放入笔记链接。
>
> 关于学习过程中的参考资料请见本文的参考资料章节，该章节放在了最后。
>
> 本文涉及到的图片除特殊标注外均为自己绘制，请勿盗图。

# 一、概述

## 1.1 概述

RocketMQ是阿里巴巴自研的高性能、高吞吐量、低延迟、高可用、高可靠得分布式消息中间件，开源后捐给apache社区。RocketMQ在阿里内部叫做Metaq，RocketMQ是Metaq 3.0之后开源的版本。Metaq在阿里巴巴集团内部、蚂蚁金服、菜鸟等各业务中被广泛使用，接入了上万个应用系统中。并平稳支撑了历年的双十一大促（万亿级的消息），在性能、稳定性、可靠性等方面表现出色，在整个阿里技术体系和大中台战略中发挥着举足轻重的作用。

## 1.2 RocketMQ业务架构

### 1.2.1 业务架构

RocketMQ由Producer、Broker、Consumer三部分组成，Producer负责生产消息，Consumer负责消费消息，Broker负责存储消息。Broker在实际部署过程中对应一台服务器，每个Broker可以存储多个Topic的消息，每个Topic得消息也可以分片存储于不同的Broker。Message Queue用于存储消息得物理地址，每个Topic中的消息地址存储于多个Message Queue中。RocketMQ的业务架构如下图：

![image-20230116094803794](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230116094803794.png)

- producer：

  消息发布的角色，支持分布式集群方式部署。Producer通过MQ的负载均衡模块选择相应的Broker集群队列进行消息投递，投递过程支持快速失败并且低延迟。

- consumer：

  消息消费的角色，支持分布式集群方式部署，支持push、pull两种模式对消息的消费，同时也支持集群方式和广播方式的消费，它提供实时消息订阅机制，可以满足大多数用户的需求。

  在消费者中消费组的有非常重要的作用，如果多个消费者设置了相同的Consumer Group，我们认为这些消费者在同一个消费组内。

  在 Apache RocketMQ 有两种消费模式，分别是：

  - 集群消费模式：当使用集群消费模式时，RocketMQ 认为任意一条消息只需要被消费组内的任意一个消费者处理即可。
  - 广播消费模式：当使用广播消费模式时，RocketMQ 会将每条消息推送给消费组所有的消费者，保证消息至少被每个消费者消费一次。

  集群消费模式适用于每条消息只需要被处理一次的场景，也就是说整个消费组会Topic收到全量的消息，而消费组内的消费分担消费这些消息，因此可以通过扩缩消费者数量，来提升或降低消费能力，具体示例如下图所示，图片来自官网。

  ![image-20230327155014355](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327155014355.png)

  广播消费模式适用于每条消息需要被消费组的每个消费者处理的场景，也就是说消费组内的每个消费者都会收到订阅Topic的全量消息，因此即使扩缩消费者数量也无法提升或降低消费能力，具体示例如下图所示，图片来自官网：

  ![image-20230327155028931](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327155028931.png)

  MQ的消费模式可以大致分为两种，一种是推Push，一种是拉Pull。

  - Push是服务端主动推送消息给客户端，优点是及时性较好，但如果客户端没有做好流控，一旦服务端推送大量消息到客户端时，就会导致客户端消息堆积甚至崩溃。
  - Pull是客户端需要主动到服务端取数据，优点是客户端可以依据自己的消费能力进行消费，但拉取的频率也需要用户自己控制，拉取频繁容易造成服务端和客户端的压力，拉取间隔长又容易造成消费不及时。

  Apache RocketMQ既提供了Push模式也提供了Pull模式。

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

### 1.2.2 RocketMQ的分区概念

在RocketMQ中，为了支持高并发和水平扩展，需要对 Topic 进行分区，在 RocketMQ 中这被称为队列，一个 Topic 可能有多个队列，并且可能分布在不同的 Broker 上。如下图（图片来自官网）：

![image-20230327154515089](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327154515089.png)

一般来说一条消息，如果没有重复发送（比如因为服务端没有响应而进行重试），则只会存在在 Topic 的其中一个队列中，消息在队列中按照先进先出的原则存储，每条消息会有自己的位点，每个队列会统计当前消息的总条数，这个称为最大位点 MaxOffset；队列的起始位置对应的位置叫做起始位点 MinOffset。队列可以提升消息发送和消费的并发度。

同时在RocketMQ中每个队列都会记录自己的最小位点、最大位点（如下图，图片来自官网）。针对于消费组，还有消费位点的概念，在集群模式下，消费位点是由客户端提给交服务端保存的，在广播模式下，消费位点是由客户端自己保存的。一般情况下消费位点正常更新，不会出现消息重复，但如果消费者发生崩溃或有新的消费者加入群组，就会触发重平衡，重平衡完成后，每个消费者可能会分配到新的队列，而不是之前处理的队列。为了能继续之前的工作，消费者需要读取每个队列最后一次的提交的消费位点，然后从消费位点处继续拉取消息。但在实际执行过程中，由于客户端提交给服务端的消费位点并不是实时的，所以重平衡就可能会导致消息少量重复。

![image-20230327155359905](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327155359905.png)

## 1.3 应用场景todo

### 1.3.1 流量削峰

在秒杀等场景下

### 1.3.2 异步解耦



### 1.3.3 顺序消息



### 1.3.4 事务消息



## 1.4 RocketMQ的安装部署

> 本文使用4.6版本RocketMQ进行学习安装

### 1.4.1 单机部署

这里以windows为例

首先下载并解压rocketmq的压缩包，下载地址：[RocketMQ下载](https://rocketmq.apache.org/download)

然后配置环境变量，如下图：

![image-20230130162051545](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230130162051545.png)

配置好后就可以启动了，分别启动nameserver和broker。也就是说均是单节点，且broker没有slave的模式，命令如下：

~~~
//启动nameserver
D:\rocketmq\rocketmq-all-4.6.0-bin-release\bin>start mqnamesrv.cmd
//启动broker
D:\rocketmq\rocketmq-all-4.6.0-bin-release\bin>start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true  
~~~

至此，rocketmq就完成了启动。下面我们需要启动控制台：

首先要下载dashboard项目代码：地址：[rocketmq-dashboard的Github地址](https://github.com/apache/rocketmq-dashboard)

然后将项目导入IDEA启动，注意在配置文件中修改为rocketmq的地址，

最后浏览器访问即可：

![image-20230130162407109](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230130162407109.png)

### 1.4.2 集群部署与高可用

rocketmq有两种集群部署模式，主要是针对broker的。两种方式如下：

1. broker多master部署
2. broker多master多slave部署

> 这里broker不考虑单master部署，因为单点会成为瓶颈，当然这样适合个人学习使用。

对于多master模式的broker，broker由多个master 节点组成集群，单个 master 节点宕机或者重启对应用没有影响。这样部署有优点也有缺点：

- 优点：高可用，性能高
- 缺点：可能会有少量消息丢失，单台机器（其中一台broker）重启或宕机期间，该机器（broker节点）下未被消费的消息在机器恢复前不可订阅，影响消息实时性。

同时，每个broker可以为master节点配置slave节点。根据Master和Slave之节的数据同步方式可以分为：

- 多 master 多 slave 异步复制模式
- 多 master 多 slave 同步复制模式

当采用多Master方式时，Master与Master之间是不需要知道彼此的，这样的设计直接降低了Broker实现的复杂性。如果Master与Master之间需要知道彼此的存在，那么在Master之中一定会维护一个网络的Master列表，而且必然设计到Master发现和活跃Master数量变更等诸多状态更新问题，所以最简单也最可靠的做法就是Master只做好自己的事情（比如和Slave进行数据同步）即可。

这时如果想知道网络中有多少master，那么就需要用到一个统一的管理，Kafka中用的是ZooKeeper，在RocketMQ中使用的就是NameServer。

这样的话多master多slave部署后的集群架构如下：

整体工作流程如下：

1. 首先启动，NameServer。NameServer起来后监听端口，等待Broker、Producer、Consumer连上来，

   相当于一个路由控制中心。

2. 然后启动Broker，跟所有的NameServer保持长连接，定时发送心跳包。心跳包中包含当前Broker信息(IP+端口等)以及存储所有Topic信息。注册成功后，NameServer集群中就有Topic跟Broker的映射关系。

3. 收发消息前，先创建Topic，创建Topic时需要指定该Topic要存储在哪些Broker上，也可以在发送消息时自动创建Topic。

4. Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取当前发送的Topic存在哪些Broker上，轮询从队列列表中选择一个队列，然后与队列所在的Broker建立长连接从而向Broker发消息。

5. Consumer跟Producer类似，跟其中一台NameServer建立长连接，获取当前订阅Topic存在哪些Broker上，然后直接跟Broker建立连接通道，开始消费消息。

与此同时，在多master模式的基础上，每个 master 节点都有至少一个对应的 slave。master 节点可读可写，但是 slave 只能读不能写，类似于 mysql 的主备模式。这样做的优点是在 master 宕机时，消费者可以从 slave 读取消息，消息的实时性不会受影响，性能几乎和多master 一样。当然缺点也有，即使用异步复制的同步方式有可能会有消息丢失的问题。

> 和kafka不一样，RocketMQ中并没有 replica 的 leader 选举功能，在RocketMQ集群中，1台机器只能要么是Master，要么是Slave，这个在初始的机器配置里面，就决定了。不会像kafka那样存在master动态选举、leader选举 的功能，所以通过配置多个master节点来保证rocketMQ的高可用。

其中Master的broker id = 0，Slave 的broker id > 0。有点类似于mysql的主从概念，master挂了以后，slave仍然可以提供读服务，但是由于有多主的存在，当一个master挂了以后，可以写到其他的master上。和所有的集群角色定位一样，master节点负责接受事务请求、slave节点只负责接收读请求，并且接收master同步过来的数据和slave保持一致。

由于有主从节点，所以主从节点的数据的复制分为同步复制和异步复制。同步复制与异步复制相比，同步复制可以保证舒服不丢失更可靠，但是发送单个消息 RT 会略长，性能相比异步复制低10%左右。同时不同节点的刷盘策略也分为同步刷盘和异步刷盘（即节点自身数据是同步还是异步存储）。

### 1.4.3 docker服务编排部署高可用学习环境

> 关于docker的相关操作可以见我的docker的学习笔记：[Github笔记地址](https://github.com/Loserfromlazy/Code_Career/blob/master/%E5%BC%80%E5%8F%91%E5%B8%B8%E7%94%A8%E5%B7%A5%E5%85%B7/Docker%E7%AC%94%E8%AE%B0.md)；[博客笔记地址](https://www.cnblogs.com/yhr520/p/17093452.html)

这里我们使用docker-compose来安装部署集群的环境。首先我们需要下载RocketMQ和控制台的docker镜像，代码如下：

~~~sh
#这里apacherocketmq是官方的docker镜像，rocketmq下的镜像很老不推荐使用
docker pull apacherocketmq/rocketmq:4.6.0
docker pull styletang/rocketmq-console-ng:latest
~~~

拉取成功后就可以看到这两个镜像了：

![image-20230205152616833](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230205152616833.png)

然后我们可以在自己的目录下创建并赋予权限：

~~~sh
mkdir -p /usr/local/soft/rocketmq/data/{broker-a,broker-b,namesrv-a,namesrv-b}/{log,store}
chmod 777  /usr/local/soft/rocketmq/data/{broker-a,broker-b,namesrv-a,namesrv-b}/{log,store}
~~~

然后我们编写docker-compose服务编排文件：

~~~yaml
version: '3.5'
services:
  rmqnamesrv-a:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqnamesrv-a
    ports:
      - 9876:9876
    restart: always
    environment:
      TZ: Asia/Shanghai
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    volumes:
      - /usr/local/soft/rocketmq/data/namesrv-a/logs:/home/rocketmq/logs
      - /usr/local/soft/rocketmq/data/namesrv-a/store:/home/rocketmq/store
    command: sh mqnamesrv
    networks:
      rmq:
        aliases:
          - rmqnamesrv-a
  rmqnamesrv-b:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqnamesrv-b
    ports:
      - 9877:9876
    restart: always
    environment:
      TZ: Asia/Shanghai
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    volumes:
      - /usr/local/soft/rocketmq/data/namesrv-b/logs:/home/rocketmq/logs
      - /usr/local/soft/rocketmq/data/namesrv-b/store:/home/rocketmq/store
    command: sh mqnamesrv
    networks:
      rmq:
        aliases:
          - rmqnamesrv-b
  rmqbroker-a:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqbroker-a
    ports:
     - 10911:10911
     - 10912:10912
    volumes:
      - /usr/local/soft/rocketmq/data/broker-a/logs:/home/rocketmq/logs
      - /usr/local/soft/rocketmq/data/broker-a/store:/home/rocketmq/store
      - /usr/local/soft/rocketmq/conf/brokera.conf:/opt/rocketmq-4.6.0/conf/broker.conf
    environment:
      TZ: Asia/Shanghai
      NAMESRV_ADDR: "rmqnamesrv-a:9876"
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    command: sh mqbroker -c /opt/rocketmq-4.6.0/conf/broker.conf autoCreateTopicEnable=true &
    links:
      - rmqnamesrv-a:rmqnamesrv-a
      - rmqnamesrv-b:rmqnamesrv-b
    networks:
      rmq:
        aliases:
          - rmqbroker-a
  rmqbroker-b:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqbroker-b
    ports:
     - 10921:10911
     - 10922:10912
    volumes:
      - /usr/local/soft/rocketmq/data/broker-b/logs:/home/rocketmq/logs
      - /usr/local/soft/rocketmq/data/broker-b/store:/home/rocketmq/store
      - /usr/local/soft/rocketmq/conf/brokerb.conf:/opt/rocketmq-4.6.0/conf/broker.conf
    environment:
      TZ: Asia/Shanghai
      NAMESRV_ADDR: "rmqnamesrv-a:9876"
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    command: sh mqbroker -c /opt/rocketmq-4.6.0/conf/broker.conf autoCreateTopicEnable=true &
    links:
      - rmqnamesrv-a:rmqnamesrv-a
      - rmqnamesrv-b:rmqnamesrv-b
    networks:
      rmq:
        aliases:
          - rmqbroker-b
  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    restart: always
    ports:
      - 9001:9001
    environment:
      TZ: Asia/Shanghai
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    environment:
      JAVA_OPTS: "-Drocketmq.config.namesrvAddr=rmqnamesrv-a:9876;rmqnamesrv-b:9877 -Dcom.rocketmq.sendMessageWithVIPChannel=false -Dserver.port=9001"
    networks:
      rmq:
        aliases:
          - rmqconsole
    networks:
      rmq:
        aliases:
          - rmqbroker-b
networks:
  rmq:
    name: rmq
    driver: bridge
~~~

broker的配置文件(这里以brokerb.conf为例)：

~~~
brokerClusterName = rocketmq-cluster
#brokera.conf中仅修改下面的broke名称即可
brokerName = broker-b
brokerId = 0
#宿主机的ip地址
brokerIP1 = 192.168.123.223
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
namesrvAddr = 192.168.123.223:9876;192.168.123.223:9877;
autoCreateTopicEnable = true
listenPort = 10911

#haListenPort 此端口在haService中使用 默认值 listenPort+1
~~~

启动命令：

~~~sh
#启动容器
docker-compose --compatibility up -d
#关闭容器
docker-compose down
#查看日志
docker-compose logs -f
~~~

启动成功后，访问控制台：

![image-20230205204609844](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230205204609844.png)

# 二、Java客户端使用

> 本章的例子全部来自官方文档，本章除2.1节以外，其余小节都可以作为工具章节略过，后续章节需要时可以回来看代码示例。

## 2.1 RocketMQ的消息结构

Message 可以设置的属性值包括：

| 字段名         | 默认值 | 必要性 | 说明                                                         |
| -------------- | ------ | ------ | ------------------------------------------------------------ |
| Topic          | null   | 必填   | 消息所属 topic 的名称                                        |
| Body           | null   | 必填   | 消息体                                                       |
| Tags           | null   | 选填   | 消息标签，方便服务器过滤使用。目前只支持每个消息设置一个     |
| Keys           | null   | 选填   | 代表这条消息的业务关键词                                     |
| Flag           | 0      | 选填   | 完全由应用来设置，RocketMQ 不做干预                          |
| DelayTimeLevel | 0      | 选填   | 消息延时级别，0 表示不延时，大于 0 会延时特定的时间才会被消费 |
| WaitStoreMsgOK | true   | 选填   | 表示消息是否在服务器落盘后才返回应答。                       |

当发送给消息时，你需要获取一个包含发送状态的结果，Broker在不同的配置下可能会返回不同响应状态。消息的发送状态共四种，源码如下：

```java
public enum SendStatus {
    SEND_OK,//发送成功，但并不意味着可靠，要保证消息不丢失，你需要开启同步主(SYNC_MASTER)和同步刷盘(SYNC_FLUSH)
    FLUSH_DISK_TIMEOUT,//如果broker消息存储设置的参数是FlushDiskType=SYNC_FLUSH(默认是 ASYNC_FLUSH)，如果broker没有在配置的5秒时间内完成消息的存储，会返回这个状态值
    FLUSH_SLAVE_TIMEOUT,//如果broker的角色是SYNC_MASTER(默认 ASYNC_MASTER)，如果5秒内 slave broker没有完成同步，会返回这个状态
    SLAVE_NOT_AVAILABLE,//如果broker的角色是SYNC_MASTER(默认 ASYNC_MASTER)，但是没有 slave broker 被配置，会返回这个状态
}
```

## 2.2 普通消息

首先需要导入Java客户端的依赖：

~~~xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.6.0</version>
</dependency>
~~~

然后编写一个消费者：

```java
//消费信息
@Test
public void consumerTest() throws Exception {
    AtomicInteger integer = new AtomicInteger(0);
    //实例化消费者，指定消费者组，这里是please_rename_unique_group_name
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
    //设置nameSever
    consumer.setNamesrvAddr("192.168.123.223:9876");
    //订阅一个或多个topic，并指定tag过滤条件，这里指定*表示接收所有tag的消息
    consumer.subscribe("TopicTest", "*");
    //注册回调实现类处理消息
    consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
            System.out.println(Thread.currentThread().getName() + "Receive new message:" + list);
            for (int i = 0; i < list.size(); i++) {
                MessageExt messageExt = list.get(i);
                log.info(messageExt.getBody());
                String content = new String(messageExt.getBody());
                log.info("收到消息:{}", messageExt.getMsgId() + " " + messageExt.getTopic() + " " + messageExt.getTags() + " " + content);
                integer.incrementAndGet();
                System.out.println("###当前是第" + integer.get() + "条消息");
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    });
    consumer.start();
    System.out.println("consumer is started");
    Thread.sleep(Integer.MAX_VALUE);
}
```

### 2.2.1 同步消费

同步发送是最常用的方式，是指消息发送方发出一条消息后，会在收到服务端同步响应之后才发下一条消息的通讯方式，可靠的同步传输被广泛应用于各种场景，如重要的通知消息、短消息通知等。流程如下：

![image-20230207220342161](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230207220342161.png)

生产者代码如下：

```java
//同步消息
@Test
public void produceSync() throws Exception {
    //实例化生产者
    DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    producer.setNamesrvAddr("192.168.123.223:9876");
    producer.start();
    //发送消息
    for (int i = 0; i < 100; i++) {
        byte[] bytes = ("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET);
        //创建一条消息，并指定topic、tag、body等信息，tag可以理解成标签，对消息进行再归类，RocketMQ可以在消费端对tag进行过滤
        Message message = new Message("TopicTest", "TahA", bytes);
        SendResult sendResult = producer.send(message);
        System.out.println(sendResult);
    }
    producer.shutdown();
}
```

> 同步发送方式请务必捕获发送异常，并做业务侧失败兜底逻辑，如果忽略异常则可能会导致消息未成功发送的情况。

### 2.2.2 异步消息

异步发送是指发送方发出一条消息后，不等服务端返回响应，接着发送下一条消息的通讯方式。

消息发送方在发送了一条消息后，不需要等待服务端响应即可发送第二条消息，发送方通过回调接口接收服务端响应，并处理响应结果。异步发送一般用于链路耗时较长，对响应时间较为敏感的业务场景。例如，视频上传后通知启动转码服务，转码完成后通知推送转码结果等。流程如下：

![image-20230207221604311](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230207221604311.png)

生产者代码如下：

```java
//异步消息
    @Test
    public void produceAsync() throws Exception {
        //实例化生产者
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("192.168.123.223:9876");
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
        int count = 100;
        //发送消息
        for (int i = 0; i < count; i++) {
            byte[] bytes = ("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET);
            Message message = new Message("TopicTest", "TahA", bytes);
            int finalI = i;
            producer.send(message, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.println("msg" + finalI + " is OK,msgId=" + sendResult.getMsgId());
                }

                @Override
                public void onException(Throwable throwable) {
                    System.out.println("msg" + finalI + " is Fail,e=" + throwable.getMessage());
                }
            });
        }
        //等5秒，防止消息没发完，测试线程就结束了
        Thread.sleep(5000);
        producer.shutdown();
    }
```

> 异步发送与同步发送代码唯一区别在于调用send接口的参数不同，异步发送不会等待发送返回，取而代之的是send方法需要传入 SendCallback 的实现，SendCallback 接口主要有onSuccess 和 onException 两个方法，表示消息发送成功和消息发送失败。

### 2.2.3 单向消息

发送方只负责发送消息，不等待服务端返回响应且没有回调函数触发，即只发送请求不等待应答。此方式发送消息的过程耗时非常短，一般在微秒级别。适用于某些耗时非常短，但对可靠性要求并不高的场景，例如日志收集。流程如下：

![image-20230207221350788](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230207221350788.png)

生产者代码如下：

```java
//单向消息
@Test
public void produceOneWay() throws Exception {
    //实例化生产者
    DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    producer.setNamesrvAddr("192.168.123.223:9876");
    producer.start();
    //发送消息
    for (int i = 0; i < 100; i++) {
        byte[] bytes = ("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET);
        Message message = new Message("TopicTest", "TahA", bytes);
        producer.sendOneway(message);
    }
    producer.shutdown();
}
```

## 2.3 顺序消息

顺序消息是一种对消息发送和消费顺序有严格要求的消息。

对于一个指定的Topic，消息严格按照先进先出（FIFO）的原则进行消息发布和消费，即先发布的消息先消费，后发布的消息后消费。

并且在 Apache RocketMQ 中支持分区顺序消息，如下图所示（图片来自官方文档）。我们可以按照某一个标准对消息进行分区（比如图中的ShardingKey），同一个ShardingKey的消息会被分配到同一个队列中，并按照顺序被消费。

> 为了支持高并发和水平扩展，需要对 Topic 进行分区，在 RocketMQ 中这被称为队列，一个 Topic 可能有多个队列，并且可能分布在不同的 Broker 上。

![image-20230207222454342](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230207222454342.png)

顺序消息的应用场景也非常广泛，在有序事件处理、撮合交易、数据实时增量同步等场景下，异构系统间需要维持强一致的状态同步，上游的事件变更需要按照顺序传递到下游进行处理。

例如创建订单的场景，需要保证同一个订单的生成、付款和发货，这三个操作被顺序执行。如果是普通消息，订单A的消息可能会被轮询发送到不同的队列中，不同队列的消息将无法保持顺序，而顺序消息发送时将ShardingKey相同（同一订单号）的消息序路由到一个逻辑队列中。

下面是一个顺序消息的生产者和消费者示例代码：

```java
@Slf4j
public class TestOrderMQ {

    //内部类模拟订单
    @Data
    static class OrderPo {
        private Integer oid;
        private String ono;
        private Integer number;
    }

    //构建订单方法
    public List<OrderPo> buildOrders() {
        List<OrderPo> orderPos = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            OrderPo orderPo = new OrderPo();
            orderPo.setOid(i);
            orderPo.setOno("ono" + i);
            orderPo.setNumber(i * 10);
            orderPos.add(orderPo);
        }
        return orderPos;
    }

    //发送顺序消息
    @Test
    public void produceOrdered() throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("192.168.123.223:9876");
        producer.start();
        String[] tags = new String[]{"TagA", "TagB", "TagC"};
        List<OrderPo> orderPos = buildOrders();

        for (int i = 0; i < orderPos.size(); i++) {
            //给顺序消息一个时间前缀
            Date date = new Date();
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS");
            String format = sdf.format(date);
            byte[] bytes = (format + " Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET);
            //这里和下面获取队列(分区)时都对下标进行了取模，所以相同的tag进入的是相同的队列(分区)
            Message message = new Message("TopicTest", tags[i%tags.length],"KEY"+i, bytes);
            //发送顺序消息，顺序消息的关键就是MessageQueueSelector接口
            SendResult sendResult = producer.send(message, new MessageQueueSelector() {
                // select方法第一个参数: 指该Topic下有的队列(分区)集合
                // 第二个参数: 发送的消息
                // 第三个参数: 消息将要进入的队列(分区)下标，它与send方法的第三个参数相同
                @Override
                public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                    return list.get((Integer) o % list.size());
                }
            }, orderPos.get(i).getOid());
            System.out.println("发送状态：" + sendResult.getSendStatus() + "队列id：" + sendResult.getMessageQueue().getQueueId() + "body:" + format + " Hello RocketMQ" + i);
        }
    }

    //接收顺序消息
    @Test
    public void consumerOrdered() throws Exception {
        AtomicInteger integer = new AtomicInteger(0);
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        consumer.setNamesrvAddr("192.168.123.223:9876");
        //设置consumer第一次启动是从队列头部开始消费还是队列尾部开始消费，如果是非第一次启动则按照上次消费的位置继续消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("TopicTest", "TagA || TagB || TagC");
        consumer.registerMessageListener(new MessageListenerOrderly() {
            Random random = new Random();
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
                consumeOrderlyContext.setAutoCommit(true);
                for (MessageExt messageExt : list) {
                    //每个queue都有唯一的consumer线程消费，订单对每个queue（分区）有序
                    System.out.println(Thread.currentThread().getName() + "queueId" + messageExt.getQueueId()+"content"+new String(messageExt.getBody()));
                    String content = new String(messageExt.getBody());
                    log.info("收到消息{}",messageExt.getMsgId()+" QueueID="+messageExt.getQueueId()+" "+ messageExt.getTags()+" "+content);
                    integer.incrementAndGet();
                    System.out.println("###当前是第"+integer.get()+"个消息");
                    try {
                        //模拟业务处理
                        TimeUnit.SECONDS.sleep(random.nextInt(10));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        consumer.start();
        System.out.println("consumer ordered is started");
        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

运行结果略（太长了），发送时不同的queue（分区）我们用不同的Tag进行了区分，如果运行上面的代码，那么从运行结果中可以看出，分区消息是有序的，且发送时和消费时必须在同一个线程中。

> MessageQueueSelector的接口如下：
>
> ```java
> public interface MessageQueueSelector {
>     MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
> }
> ```
>
> 其中 mqs 是可以发送的队列，msg是消息，arg是上述send接口中传入的Object对象，返回的是该消息需要发送到的队列。上述例子里，是以orderId作为分区分类标准，对所有队列个数取余，来对将相同orderId的消息发送到同一个队列中。
>
> 生产环境中建议选择最细粒度的分区键进行拆分，例如，将订单ID、用户ID作为分区键关键字，可实现同一终端用户的消息按照顺序处理，不同用户的消息无需保证顺序。

保证消息的顺序消费有三个关键点：

- 消息顺序发送

  多线程发送无法保证有序，所以同一个标准的消息（上面例子是同一个订单id）需要在同一个，对应到mq中，消息发送方法就得使用同步发送，关键点在于单线程同步顺序发送消息。

- 消息顺序存储

  mq的topic下会存在多个queue，要保证消息的顺序存储，同一个业务编号的消息需要被发送到一个queue中。对应到mq中，需要使用MessageQueueSelector来选择要发送的queue，即对业务编号 sharding key 进行hash，然后根据队列数量对hash值取余，将消息发送到一个queue中。

- 消息顺序消费

  要保证消息顺序消费，同一个queue就只能被一个消费者所消费，因此对broker中消费队列加锁是无法避免的。同一时刻，一个消费队列只能被一个消费者消费，消费者内部，也只能有一个消费线程来消费该队列。

## 2.4 延迟消息

延迟消息发送是指消息发送到Apache RocketMQ后，并不期望立马投递这条消息，而是延迟一定时间后才投递到Consumer进行消费。

在分布式定时调度触发、任务超时处理等场景，需要实现精准、可靠的延时事件触发。使用 RocketMQ 的延时消息可以简化定时调度任务的开发逻辑，实现高性能、可扩展、高可靠的定时触发能力。

Apache RocketMQ 一共支持18个等级的延迟投递，具体时间如下（表格来自官网）：

> 在RocketMQ5.0中，已经支持任意时间的延迟消息了

| 投递等级（delay level） | 延迟时间 | 投递等级（delay level） | 延迟时间 |
| ----------------------- | -------- | ----------------------- | -------- |
| 1                       | 1s       | 10                      | 6min     |
| 2                       | 5s       | 11                      | 7min     |
| 3                       | 10s      | 12                      | 8min     |
| 4                       | 30s      | 13                      | 9min     |
| 5                       | 1min     | 14                      | 10min    |
| 6                       | 2min     | 15                      | 20min    |
| 7                       | 3min     | 16                      | 30min    |
| 8                       | 4min     | 17                      | 1h       |
| 9                       | 5min     | 18                      | 2h       |

生产者消费者代码如下：

```java
@Slf4j
public class TestDelayedMQ {
    @Test
    public void consumerDelayed() throws Exception {
        AtomicInteger integer = new AtomicInteger(0);
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ExampleConsumer");
        consumer.setNamesrvAddr("127.0.0.1:9876");
        consumer.subscribe("TopicTest", "*");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                System.out.println(Thread.currentThread().getName() + "Receive new message:" + list);
                for (int i = 0; i < list.size(); i++) {
                    MessageExt messageExt = list.get(i);
                    String content = new String(messageExt.getBody());
                    SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss SSS");
                    log.info("###当前是第" + integer.get() + "条消息;收到消息:{}", messageExt.getMsgId() + " " + messageExt.getTopic() + " " + sdf.format(new Date()) + " " + content);
                    integer.incrementAndGet();
                    System.out.println("");
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("consumer is started");
        Thread.sleep(Integer.MAX_VALUE);
    }

    @Test
    public void produceDelayed() throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("ExampleProduceGroup");
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();
        //发送消息
        log.info("###发送时间{}",new Date());
        for (int i = 0; i < 100; i++) {
            byte[] bytes = ("Hello RocketMQ" + i).getBytes(RemotingHelper.DEFAULT_CHARSET);
            Message message = new Message("TopicTest", "TahA", bytes);
            //设置消息延时等级
            message.setDelayTimeLevel(3);
            SendResult sendResult = producer.send(message);
            System.out.println(sendResult);
        }
        producer.shutdown();
    }
}
```

> 延时消息的实现逻辑需要先经过定时存储等待触发，延时时间到达后才会被投递给消费者。因此，如果将大量延时消息的定时时间设置为同一时刻，则到达该时刻后会有大量消息同时需要被处理，会造成系统压力过大，导致消息分发延迟，影响定时精度。

## 2.5 批量消息

在对吞吐率有一定要求的情况下，Apache RocketMQ可以将一些消息聚成一批以后进行发送，可以增加吞吐率，并减少API和网络调用次数。

### 2.5.1 发送批量消息

生产者代码如下（消费者用2.2中的消费者即可）：

```java
@Test
public void batchProducer() throws Exception{
    //实例化生产者
    DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
    producer.setNamesrvAddr("127.0.0.1:9876");
    producer.start();
    List<Message> messageList = new ArrayList<>();
    messageList.add(new Message("TopicTest","TagA","111","message1".getBytes(StandardCharsets.UTF_8)));
    messageList.add(new Message("TopicTest","TagA","222","message2".getBytes(StandardCharsets.UTF_8)));
    messageList.add(new Message("TopicTest","TagA","333","message3".getBytes(StandardCharsets.UTF_8)));
    try {
        //限制是这些批量消息应该有相同的topic，相同的waitStoreMsgOK，而且不能是延时消息
        producer.send(messageList);
    }catch (Exception e){
        e.printStackTrace();
    }
    producer.shutdown();
}
```

> 这里调用非常简单，将消息打包成 `Collection<Message> msgs` 传入方法中即可，需要注意的是批量消息的大小不能超过 1MiB（否则需要自行分割），其次同一批 batch 中 topic 必须相同。

### 2.5.2 消息分割

官方建议批量消息的大小不要超过1M，超过就要进行分割，下面给一个消息分割的例子：

```java
public class MesListSplit implements Iterator<List<Message>> {

    private final int SIZE_LIMIT = 1024*1024;

    private List<Message> messageList;

    private int currentIndex;

    public MesListSplit(List<Message> messageList) {
        this.messageList = messageList;
    }

    @Override
    public boolean hasNext() {
        return currentIndex < messageList.size();
    }

    @Override
    public List<Message> next() {
        int nextIndex = currentIndex;
        int totalSize = 0;
        for (; nextIndex < messageList.size(); nextIndex++) {
            Message message = messageList.get(nextIndex);
            int tempSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry : properties.entrySet()) {
                tempSize += entry.getKey().length() + entry.getValue().length();
            }
            tempSize += 20;//日志开销20字节
            if (tempSize > SIZE_LIMIT){
                if (nextIndex - currentIndex == 0){
                    //判断下一个子列表有没有元素，没有元素就把当前这个消息也加上
                    nextIndex++;
                }
                break;
            }
            if (tempSize + totalSize > SIZE_LIMIT){
                break;
            }else {
                totalSize += tempSize;
            }
        }
        List<Message> messageList = this.messageList.subList(currentIndex, nextIndex);
        currentIndex = nextIndex;
        return messageList;
    }
}
```

使用时将消息列表装入分割器类即可：

```java
MesListSplit mesListSplit = new MesListSplit(messageList);
while (mesListSplit.hasNext()){
    List<Message> next = mesListSplit.next();
    try {
        producer.send(next);
    }catch (Exception e){
        e.printStackTrace();
    }
}
```

测试时可以将上面例子中的`SIZE_LIMIT`调小，以观看现象。

## 2.6 消息过滤

### 2.6.1 简单过滤

在上面的例子中，我们发现可以通过Tag标签进行消息的过滤，只需要在消息发送时标记好该消息的标签：

```java
messageList.add(new Message("TopicTest","TagA","message11".getBytes(StandardCharsets.UTF_8)));
```

然后在消费者订阅时，订阅想订阅的Tag：

~~~java
 consumer.subscribe("TopicTest", "TagA");//*表示订阅该Topic下的全部Tag
~~~

### 2.6.2 复杂过滤

在RocketMQ中可以使用SQL表达式来对消息进行过滤。

这个模式的关键是在消费者端使用MessageSelector.bySql(String sql)返回的一个MessageSelector。这里面的sql语句是按照SQL92标准来执行的。sql中可以使用的参数有默认的TAGS和一个在生产者中加入的属性。

RocketMQ只定义了一些基本语法来支持这个特性。你也可以很容易地扩展它。

- 数值比较，比如：>，>=，<，<=，BETWEEN，=；
- 字符比较，比如：=，<>，IN；
- IS NULL 或者 IS NOT NULL；
- 逻辑符号 AND，OR，NOT；

常量支持类型为：

- 数值，比如：123，3.1415；
- 字符，比如：‘abc’，必须用单引号包裹起来；
- NULL，特殊的常量
- 布尔值，TRUE 或 FALSE

**注意只有使用推模式的消费者才能使用sql语句**，接口如下：

~~~
public void subscribe(finalString topic, final MessageSelector messageSelector)
~~~

使用时broker中需要加上`enablePropertyFilter=true`配置。

使用时生产者需要配置消息属性：

```java
message.putUserProperty("a","1");
```

消费者需要订阅时使用sql语句：

~~~java
consumer.subscribe("SqlFilterTest",
            MessageSelector.bySql("(TAGS is not null and TAGS in ('TagA', 'TagB'))" +
                "and (a is not null and a between 0 and 3)"));
~~~

## 2.7 事务消息

在一些对数据一致性有强需求的场景，可以用 Apache RocketMQ 事务消息来解决，从而保证上下游数据的一致性。

以电商交易场景为例，用户支付订单这一核心操作的同时会涉及到下游物流发货、积分变更、购物车状态清空等多个子系统的变更。当前业务的处理分支包括：

- 主分支订单系统状态更新：由未支付变更为支付成功。
- 物流系统状态新增：新增待发货物流记录，创建订单物流记录。
- 积分系统状态变更：变更用户积分，更新用户积分表。
- 购物车系统状态变更：清空购物车，更新用户购物车记录。

整体流程如下图（图片来自RocketMQ官方文档）：

![image-20230208134149213](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230208134149213.png)

由于上面这些不同的系统是独立的服务模块，所以有可能出现某一模块失败造成订单整体状态或数据不对。比如，更新用户积分时失败了，导致用户已经下单了但并没有增加积分。

这种问题的本质是本地事务与消息发送的原子性问题。普通消息无法像单机数据库事务一样，具备提交、回滚和统一协调的能力。 而基于 RocketMQ 的分布式事务消息功能，在普通消息基础上，支持二阶段的提交能力。将二阶段提交和本地事务绑定，实现全局提交结果的一致性。

RocketMQ事务消息发送分为两个阶段。第一阶段会发送一个**半事务消息**，半事务消息是指暂不能投递的消息，生产者已经成功地将消息发送到了 Broker，但是Broker 未收到生产者对该消息的二次确认，此时该消息被标记成“暂不能投递”状态，如果发送成功则执行本地事务，并根据本地事务执行成功与否，向 Broker 半事务消息状态（commit或者rollback），半事务消息只有 commit 状态才会真正向下游投递。如果由于网络闪断、生产者应用重启等原因，导致某条事务消息的二次确认丢失，Broker 端会通过扫描发现某条消息长期处于“半事务消息”时，需要主动向消息生产者询问该消息的最终状态（Commit或是Rollback）。这样最终保证了本地事务执行成功，下游就能收到消息，本地事务执行失败，下游就收不到消息。总而保证了上下游数据的一致性。整体流程如下（图片来自RocketMQ的官方文档）：

![image-20230208135414069](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230208135414069.png)

> PS：注意RocketMQ提供的是半事务消息，也就是说只能保证生产者的消息的事务。

下面是使用事务消息的例子：

```java
@Slf4j
public class TestTransMQ {


    @Test
    public void produceTrans() throws Exception {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("127.0.0.1:9876");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });
        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                        new Message("TopicTest", tags[i % tags.length], "KEY" + i,
                                ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }

    static class TransactionListenerImpl implements TransactionListener {

        private AtomicInteger transactionIndex = new AtomicInteger(0);

        private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

        @Override
        public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
            int value = transactionIndex.getAndIncrement();
            int status = value % 3;
            localTrans.put(msg.getTransactionId(), status);
            log.info("tag={},status={}",msg.getTags(),status);
            return LocalTransactionState.UNKNOW;
        }

        @Override
        public LocalTransactionState checkLocalTransaction(MessageExt msg) {
            Integer status = localTrans.get(msg.getTransactionId());
            if (null != status) {
                switch (status) {
                    case 0:
                        return LocalTransactionState.UNKNOW;
                    case 1:
                        return LocalTransactionState.COMMIT_MESSAGE;
                    case 2:
                        return LocalTransactionState.ROLLBACK_MESSAGE;
                    default:
                        return LocalTransactionState.COMMIT_MESSAGE;
                }
            }
            return LocalTransactionState.COMMIT_MESSAGE;
        }
    }
}
```

执行结果略，最终只有状态为COMMIT_MESSAGE的消息被消费者所接收。

事务消息的注意事项：

1. 事务消息不支持延时和批量。
2. 为了避免单个消息被检查太多次而导致半队列消息累积，默认将单个消息的检查次数限制为15次，但是用户可以通过 Broker 配置文件的 transactioncheckMax 参数来修改此限制。如果已经检查某条消息超过 N次的话(N= transactioncheckMax ) 则 Broker 将丢弃此消息，并在默认情况下同时打印错误日志。用户可以通过重写 AbstractTransactioncheckListener 类来修改这个行为。
3. 事务消息将在 Broker 配置文件中的参数transactionMsgTimeout 这样的特定时间长度之后被检查。当发送事务消息时，用户还可以通过设置用户属性 CHECK IMMUNITY TIME IN SECONDS来改变这个限制，该参数优先于 transactionMsgTimeout 参数。
4. 事务性消息可能不止一次被检查或消费

## 2.8 rocketmq在springboot中的开发方式

> rocketmq-spring-boot-starter暂不支持5.0的部分功能，比如设置任意时间的延迟消息

首先导入依赖：

```xml
<!--rocketmq 消息中间件-->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```

然后编写配置文件：

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: mygroup
```

> 下面是一个网上参考的详细配置文件：
>
> ~~~
> # rocketmq 配置项，对应 RocketMQProperties 配置类
> rocketmq:
>   name-server: 127.0.0.1:9876 # RocketMQ Namesrv
>   # Producer 配置项
>   producer:
>     group: erbadagang-producer-group # 生产者分组
>     send-message-timeout: 3000 # 发送消息超时时间，单位：毫秒。默认为 3000 。
>     compress-message-body-threshold: 4096 # 消息压缩阀值，当消息体的大小超过该阀值后，进行消息压缩。默认为 4 * 1024B
>     max-message-size: 4194304 # 消息体的最大允许大小。。默认为 4 * 1024 * 1024B
>     retry-times-when-send-failed: 2 # 同步发送消息时，失败重试次数。默认为 2 次。
>     retry-times-when-send-async-failed: 2 # 异步发送消息时，失败重试次数。默认为 2 次。
>     retry-next-server: false # 发送消息给 Broker 时，如果发送失败，是否重试另外一台 Broker 。默认为 false
>     access-key: # Access Key ，可阅读 https://github.com/apache/rocketmq/blob/master/docs/cn/acl/user_guide.md 文档
>     secret-key: # Secret Key
>     enable-msg-trace: true # 是否开启消息轨迹功能。默认为 true 开启。可阅读 https://github.com/apache/rocketmq/blob/master/docs/cn/msg_trace/user_guide.md 文档
>     customized-trace-topic: RMQ_SYS_TRACE_TOPIC # 自定义消息轨迹的 Topic 。默认为 RMQ_SYS_TRACE_TOPIC 。
>   # Consumer 配置项
>   consumer:
>     listeners: # 配置某个消费分组，是否监听指定 Topic 。结构为 Map<消费者分组, <Topic, Boolean>> 。默认情况下，不配置表示监听。
>       erbadagang-consumer-group:
>         topic1: false # 关闭 test-consumer-group 对 topic1 的监听消费
> 
> ~~~

然后编写消息生产者，在springboot项目中直接注入`RocketMQTemplate`即可使用：

```java
@Slf4j
@Component
public class MQProducerService {

    private static final String topic = "TestTopic";

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 普通发送
     */
    public void send(String msg) {
        rocketMQTemplate.convertAndSend(topic + ":tag1", msg);
//        rocketMQTemplate.send(topic + ":tag1", MessageBuilder.withPayload(user).build()); // 等价于上面一行
    }

    /**
     * 发送同步消息（阻塞当前线程，等待broker响应发送结果，这样不太容易丢失消息）
     * （msgBody也可以是对象，sendResult为返回的发送结果）
     */
    public SendResult sendMsg(String msgBody) {
        SendResult sendResult = rocketMQTemplate.syncSend(topic, MessageBuilder.withPayload(msgBody).build());
        log.info("【sendMsg】sendResult={}", JSON.toJSONString(sendResult));
        return sendResult;
    }

    /**
     * 发送异步消息（通过线程池执行发送到broker的消息任务，执行完后回调：在SendCallback中可处理相关成功失败时的逻辑）
     * （适合对响应时间敏感的业务场景）
     */
    public void sendAsyncMsg(String msgBody) {
        rocketMQTemplate.asyncSend(topic, MessageBuilder.withPayload(msgBody).build(), new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                // 处理消息发送成功逻辑
            }
            @Override
            public void onException(Throwable throwable) {
                // 处理消息发送异常逻辑
            }
        });
    }

    /**
     * 发送延时消息（上面的发送同步消息，delayLevel的值就为0，因为不延时）
     * 在start版本中 延时消息一共分为18个等级分别为：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
     */
    public void sendDelayMsg(String msgBody, int delayLevel) {
        rocketMQTemplate.syncSend(topic, MessageBuilder.withPayload(msgBody).build(), delayLevel);
    }

    /**
     * 发送单向消息（只负责发送消息，不等待应答，不关心发送结果，如日志）
     */
    public void sendOneWayMsg(String msgBody) {
        rocketMQTemplate.sendOneWay(topic, MessageBuilder.withPayload(msgBody).build());
    }

    /**
     * 发送带tag的消息，直接在topic后面加上":tag"
     */
    public SendResult sendTagMsg(String msgBody) {
        return rocketMQTemplate.syncSend(topic + ":tag2", MessageBuilder.withPayload(msgBody).build());
    }


}
```

消费者代码如下：

```java
@Slf4j
@Service
@RocketMQMessageListener(topic = "TestTopic", consumerGroup = "mygroup")
public class MQListener implements RocketMQListener<String> {

    @Override
    public void onMessage(String s) {
      log.info("收到消息:{}",s);
    }
}
```

# 三、RocketMQ的存储架构

## 3.1 存储概要

RocketMQ存储的文件主要包括CommitLog文件、ConsumeQueue文件、Index文件。RocketMQ将所有主题的消息存储在同一个文件中，确保消息发送时按顺序写文件，尽最大的能力确保消息发送的高性能与高吞吐量。因为消息中间件一般是基于消息主题的订阅机制，所以给按照消息主题检索消息带来了极大的不便。为了提高消息消费的效率，RocketMQ引入了ConsumeQueue消息消费队列文件，每个消息主题包含多个消息消费队列，每一个消息队列有一个消息文件。Index索引文件的设计理念是为了加速消息的检索性能，根据消息的属性从CommitLog文件中快速检索消息。

也就是说，RocketMQ共有三种文件：

- CommitLog是消息存储，所有的消息主题的消息都存储在CommitLog文件中
- ConsumeQueue是消息消费队列，当消息到达CommitLog文件后，将异步转发到ConsumeQueue文件中，共消息消费者进行消费。
- Index：消息索引，存储Key和offset的对应关系

RocketMQ在消息写入的过程中会追求极致的磁盘顺序写，所有主题的消息会全部写入一个文件即CommitLog文件中。所有消息按照抵达的顺序依次追加到CommitLog文件中，消息一旦写入不支持修改。但是基于文件编程有一个很大的问题就是没有现成的数据结构，所以为了方便查找，RocketMQ为每条消息引入了一个身份标志：消息的物理偏移量，也就是消息存储在文件的起始位置。也因此CommitLog文件的命名方式也是极具技巧性，使用存储在该文件的第一条消息在整个CommitLog文件组中的偏移量来命名，例如第一个CommitLog文件为`0000000000000000000`，第二个CommitLog文件为`00000000001073741824`，依次类推。这样做可以通过二分法进行查找，快速定位这个文件的位置，然后用消息物理偏移量减去所在文件的名称，得到的差值就是在该文件中的绝对地址。

但消息消费模型是基于主题订阅机制的，即一个消费组是消费特定主题的消息。根据主题从CommitlLog文件中检索消息，绝不是一个好主意，因为这样只能从文件的第一条消息逐条检索，其性能可想而知。为了解决基于topic的消息检索问题，RocketMQ引入了ConsumeQueue文件。

ConsumeQueue文件是消息消费队列文件，是CommitLog文件基于topic的索引文件，主要用于消费者根据topic消费消息，其组织方式为`/topic/queue`，同一个队列中存在多个消息文件。ConsumeQueue的设计极具技巧，每个条目长度固定，包括8字节CommitLog物理偏移量、4字节消息长度、8字节tag哈希码。这里不是存储tag的原始字符串，而是存储哈希码，目的是确保每个条目的长度固定，可以使用访问类似数组下标的方式快速定位条目，极大地提高了ConsumeQueue文件的读取性能。消息消费者根据topic、消息消费进度（ConsumeQueue逻辑偏移量），即第几个ConsumeQueue条目，这样的消费进度去访问消息，通过逻辑偏移量logicOffset×20，即可找到该条目的起始偏移量（ConsumeQueue文件中的偏移量），然后读取该偏移量后20个字节即可得到一个条目，无须遍历ConsumeQueue文件。

RocketMQ与Kafka相比具有一个强大的优势，就是支持按消息属性检索消息，引入ConsumeQueue文件解决了基于topic查找消息的问题，但如果想基于消息的某一个属性进行查找，ConsumeQueue文件就无能为力了。故RocketMQ又引入了Index索引文件，实现基于文件的哈希索引。Index文件基于物理磁盘文件实现哈希索引。

下面我们来分别详细了解一下每个文件。

### 3.1.1 CommitLog

CommitLog存储生产者写入的消息主体内容，单个文件默认大小为1G。文件名称长为20位，左边补零，为起始偏移量。当文件写满了，就写入下一个文件。

比如第一个文件是00000000000000000000，每个文件是1G，也就是1073741824，所以第二个文件就是00000000001073741824，如下图：

![image-20230210103014297](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210103014297.png)

commitLog的默认存储目录为`${RocketMQ_HOME}/store/commitLog`。

RocketMQ采用的是混合型的存储结构，文件存储的结构如下图（图片来自RocketMQ技术内幕：RocketMQ架构设计与实现原理（第2版））：

![image-20230210103333525](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210103333525.png)

### 3.1.2 ConsumeQueue

RocketMQ基于主题实现消息消费，所以出现了ConsumeQueue，此文件是为了适应消息消费的检索需求，可以看作CommitLog关于消息消费的索引文件。ConsumeQueue的第一级目录为消息主题，第二级目录为主题的消息队列，如下图：

![image-20230210113033465](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210113033465.png)

ConsumeQueue也可以看作kafka中的分区，是一个逻辑队列，存储的是Queue在CommitLog中的起始物理位置的offset、消息内容的大小和tag的hashcode。存储格式如下图：

![image-20230210113055177](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210113055177.png)

从实际物理存储来说，ConsumeQueue对应每个Topic和queue下面的文件。

单个ConsumeQueue文件默认包含30万个条目，单个文件长度为`30*10^6*20字节`。单个ConsumeQueue文件可以看作是一个ConsumeQueue条目的数组，其下标为ConsumeQueue的逻辑偏移量。ConsumeQueue就是CommitLog的索引文件，其构建机制是当消息到达CommitLog后，由专门的线程产生消息转发任务，从而构建ConsumeQueue文件和IndexFile文件。

> 偏移量：
>
> - CommitLog中的偏移量（物理偏移量）
>
>   对应这个CommitLog文件所有消息中的起始偏移量（方便通过ConsumeQueue.commitlogOffset找到当前要消费的消息存在于哪个commitlog文件）
>
> - ConsumeQueue中的commitlogOffset偏移量
>
>   定位了当前这条消息在commitlog中的偏移量

### 3.1.3 IndexFile

IndexFile（索引文件）为了消息查询提供了一种通过key或时间区间来查询消息的方法，也就是说除了满足正常消费的消息偏移索引功能外，还需要满足根据某些关键字查询消息的功能，为了保证查询速度，索引文件是少不了的。这些关键字对应的索引就存在index文件中，一个broker对应一组index文件。在实际的物理存储上，文件名则是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引；lndexFile 总共包含 lndexHeader、 Hash 槽、 Hash 条目。索引文件布局如下图所示（图片来自RocketMQ技术内幕：RocketMQ架构设计与实现原理（第2版））：

![image-20230210151112082](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210151112082.png)

其中 Index文件头包含40字节，记录该Index的统计信息其结构如下：

1. beginTimestamp：Index文件中消息的最小存储时间。
2. endTimestamp：Index文件中消息的最大存储时间。
3. beginPhyoffset：Index文件中消息的最小物理偏移量（CommitLog文件偏移量）。
4. endPhyoffset：Index文件中消息的最大物理偏移量（CommitLog文件偏移量）。
5. hashslotCount：hashslot个数，并不是哈希槽使用的个数，在这里意义不大。
6. indexCount：Index条目列表当前已使用的个数，Index条目在Index条目列表中按顺序存储。

一个Index默认包含500万个哈希槽。哈希槽存储的是落在该哈希槽的哈希码最新的Index索引。默认一个Index文件包含2000万个条目，每个Index条目结构如下：

1. hashcode：key的哈希码。
2. phyoffset：消息对应的物理偏移量。
3. timedif：该消息存储时间与第一条消息的时间戳的差值，若小于0，则该消息无效。
4. pre index no：该条目的前一条记录的Index索引，当出现哈希冲突时，构建链表结构。

整个Index File的结构如上图，40 Byte 的Header用于保存一些总的统计信息，`4*500W`的 Slot Table并不保存真正的索引数据，而是保存每个槽位对应的单向链表的头。`20*2000W` 是真正的索引数据，即一个Index File 可以保存 2000W个索引。

IndexFile文件的存储位置是：`$HOME\store\index${fileName}`，文件名fileName是以创建时的时间戳命名的，文件大小是固定的，等于`40+500W*4+2000W*20= 420000040个字节`大小。

## 3.2 RocketMQ消息写入和消费流程

整体流程如下图（图片来自RocketMQ技术内幕：RocketMQ架构设计与实现原理（第2版））

![image-20230213110355652](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213110355652.png)

主要可以总结为以下几步：

- 消息统一存储：producer发送消息最终写入CommitLog。CommitLog使用混合型存储，所有Topic共用一个CommitLog文件。

- dispatch建立逻辑队列：consumer针对ConsumerQueue进行订阅。当消费者拉取订阅的消息时，首先从自己的ConsumerQueue找到消息的物理偏移量offset，然后去CommitLog找到消息主体。

  同时broker通过ReputMessageService异步线程，不断地异步生成ConsumerQueue队列的元素。这样只要消息写入并刷盘到CommitLog之后，消息就不会丢失，即使ConsumerQueue中的数据丢失了，也可以通过CommitLog进行恢复。

- ConsumerQueue消费：消息消费时，消费者端顺序读取ConsumerQueue，然后根据起始物理位置的偏移量，读取CommitLog中的消息真实内容。

- 消息过滤：RocketMQ的消息过滤方式有别于其他中间件，它是订阅时进行过滤。ConsumeQueue的结构如下：

  ![image-20230210113055177](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210113055177.png)

  在Broker端进行消息tag的比对，先遍历ConsumeQueue，如果存储的消息tag与订阅的message tag不符合，那么就对比下一个，如果符合就传输给消费者。这时对比的是hashcode。

  消费者收到过滤的消息后，同样需要执行broker的对比操作，但此时对比的是message tag的字符串。

  这样做是因为：

  - MessageTag存储Hashcode，是为了在ConsumeQueue定长方式存储，节约空间。
  - 过滤过程中不会访问CommitLog数据，可以保证堆积情况下也能高效过滤。
  - 即使存在Hash冲突，也可以在Consumer端进行修正，保证万无一失。
  - Consumer可以对tag进行最终的判断，如果数据不对，可以丢弃

# 四、RocketMQ的高可用todo

## 4.1 namesrv高可用

由于namesrv节点是无状态的，且各个节点的数据是一致的，所以存在多节点namesrv的情况下，部分namesrv不可用也可以保证MQ服务正常运行。

## 4.2 Broker高可用



## 4.3 生产者消费者高可用

### 4.3.1 生产者高可用

下面来看看消费者高可用

### 4.3.2 消费模式

消息系统的重要作用之一是削峰填谷，但比如在电商大促的场景中，如果下游的消费者消费能力不足的话，大量的瞬时流量进入会后堆积在服务端。此时，消息的端到端延迟（从发送到被消费的时间）就会增加，对服务端而言，一直消费历史数据也会产生冷读。因此需要增加消费能力来解决这个问题，除了去优化消息消费的时间，最简单的方式就是扩容消费者。

但是否随意增加消费者就能提升消费能力？ 首先需要了解消费组的概念。在消费者中消费组的有非常重要的作用，如果多个消费者设置了相同的Consumer Group，我们认为这些消费者在同一个消费组内。

在 Apache RocketMQ 有两种消费模式，分别是：

- 集群消费模式：当使用集群消费模式时，RocketMQ 认为任意一条消息只需要被消费组内的任意一个消费者处理即可。
- 广播消费模式：当使用广播消费模式时，RocketMQ 会将每条消息推送给消费组所有的消费者，保证消息至少被每个消费者消费一次。

集群消费模式适用于每条消息只需要被处理一次的场景，也就是说整个消费组会Topic收到全量的消息，而消费组内的消费分担消费这些消息，因此可以通过扩缩消费者数量，来提升或降低消费能力，具体示例如下图所示，是最常见的消费方式。

广播消费模式适用于每条消息需要被消费组的每个消费者处理的场景，也就是说消费组内的每个消费者都会收到订阅Topic的全量消息，因此即使扩缩消费者数量也无法提升或降低消费能力，具体示例如下图所示。

### 4.3.3 消费负载均衡

# 五、RocketMQ的高可靠todo

生产者消息可靠发送

broker消息的可靠存储

消费者消息的可靠消费

# 六、RocketMQ源码分析

> 这里使用4.60分支进行源码阅读与学习，学习源码主要学习优秀的思想和核心底层的原理，所以这里不对各个版本的差别等进行详细分析。如果使用其他版本阅读学习，可以以此笔记进行参考进行查缺补漏。
>
> RocketMQ进行通信是用Netty进行的，所以研读源码前需要掌握Netty的使用。这里推荐先简单过一遍我的[Netty学习笔记](https://github.com/Loserfromlazy/Code_Career/blob/master/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E5%AD%A6%E4%B9%A0/Java%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%AF%87%E4%B8%83NIO%E4%B8%8ENetty.md)，了解Netty的基本使用。
>
> **注意：源码分析中的所有流程图均是我在研究完源码后画出来的流程图，是一个总结和归纳，可以进行参考学习。请勿盗图**
>
> PS：技术需要深耕，本笔记只是我的学习过程的整理与总结，可以作为学习和源码思路的参考，但是具体还是需要自己看源码，自己debug，自己画图整理，才能学习理解的深刻。

## 6.1 源码环境搭建

> 这里构建单namesrv双broker的源码环境

### 6.1.1 下载导入编译源码

首先下载源码到本地：[Github源码地址](https://github.com/apache/rocketmq)，下载完成后导入IDEA。

编辑pom文件，然后去除不用的插件。

去除gpg插件

```xml
<!--在这里可以指定要签名的POM及相关文件、Maven仓库的地址和ID，Maven GPG Plugin就会帮你签名文件并部署到仓库中。GPG签名这一步骤只有在项目发布时才显得必要，对日常的SNAPSHOT构建进行签名不仅没有多大的意义，GPG签名会比较耗时-->
<!--                    <plugin>-->
<!--                        <artifactId>maven-gpg-plugin</artifactId>-->
<!--                        <version>1.6</version>-->
<!--                        <executions>-->
<!--                            <execution>-->
<!--                                <id>sign-artifacts</id>-->
<!--                                <phase>verify</phase>-->
<!--                                <goals>-->
<!--                                    <goal>sign</goal>-->
<!--                                </goals>-->
<!--                            </execution>-->
<!--                        </executions>-->
<!--                    </plugin>-->
```

去除FailSafe插件

```xml
<!--maven FailSafe插件是用来执行集成测试的，Surefire插件则是用来执行单元测试的。-->
<!--                    <plugin>-->
<!--                        <artifactId>maven-failsafe-plugin</artifactId>-->
<!--                        <version>2.19.1</version>-->
<!--                        <configuration>-->
<!--                            <argLine>@{failsafeArgLine}</argLine>-->
<!--                            <excludes>-->
<!--                                <exclude>**/NormalMsgDelayIT.java</exclude>-->
<!--                            </excludes>-->
<!--                        </configuration>-->
<!--                        <executions>-->
<!--                            <execution>-->
<!--                                <goals>-->
<!--                                    <goal>integration-test</goal>-->
<!--                                    <goal>verify</goal>-->
<!--                                </goals>-->
<!--                            </execution>-->
<!--                        </executions>-->
<!--                    </plugin>-->
```

然后就是编译项目，注意跳过测试：

![image-20230213134155835](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213134155835.png)

### 6.1.2 配置启动namesrv

然后在创建一个目录，目录名位置任意。我这里为了方便直接建到项目里。然后在此目录下创建conf、logs、store目录，并将`distribution/logback_namesrc.xml`复制到我们新建的conf目录下，如下图：

![image-20230213161105295](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213161105295.png)

然后编辑这个文件，新增一个property指向新建的目录，并将user.home全局替换为新建的property。

![image-20230213161146442](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213161146442.png)

然后编辑namesrv启动类参数，如下图：

![image-20230213161519961](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213161519961.png)

最后启动即可。

### 6.1.3 配置启动broker

Broker跟namesrc的配置是一样的，新建文件夹，并拷贝文件，如下图：

![image-20230213161830411](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213161830411.png)

其中logback_broker.xml修改方式与上面一致。然后修改broker.conf文件，如下：

```properties
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

namesrvAddr=127.0.0.1:9876
#这里Windows下一定要配置双斜杠，否则会读取目录失败导致code=-3并退出
storePathRootDir=D:\\IDEA\\rocketmq\\Broker-a-master\\store
storePathCommitLog=D:\\IDEA\\rocketmq\\Broker-a-master\\store\\commitLog
storePathConsumeQueue=D:\\IDEA\\rocketmq\\Broker-a-master\\store\\consumeQueue
storePathIndex=D:\\IDEA\\rocketmq\\Broker-a-master\\store\\index
#这是个文件不要自己创建，mq启动时会自己创建，否则会报错导致code=-3并退出
storeCheckpoint=D:\\IDEA\\rocketmq\\Broker-a-master\\store\\checkpoint
abortFile=D:\\IDEA\\rocketmq\\Broker-a-master\\store\\abort
```

> PS：为了调试方便，可以将namesrv和broker的root根日志，换成打印到控制台。
>
> ```xml
> <root>
>     <level value="INFO"/>
>     <!--    <appender-ref ref="DefaultAppender"/>  -->
>     <appender-ref ref="STDOUT"/>
> </root>
> ```

然后编辑broker的启动类，配置启动参数，如下图：

![image-20230213162113325](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213162113325.png)

最后启动broker即可。

**调试源码时可能会需要broker的master集群，那么需要将此步骤在完成一遍，并更换名字。**比如再弄一个broker-b，如下图：

![image-20230329115326302](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329115326302.png)

注意：IDEA需要开启多实例才能用，即上图中的`Allow multiple instances`，同时brokerb的conf配置文件需要更改listenPort，如下图：

![image-20230329115453116](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329115453116.png)

### 6.1.4 配置启动dashboard

同1.4.1单机部署。下载启动dashboard。

### 6.1.5 测试消息的生产和消费

测试消息发送：在`org.apache.rocketmq.example.quickstart.Producer#main`类的主方法下，配置地址，然后执行此方法：

![image-20230213164438546](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213164438546.png)

控制台打印发送成功（这里略），然后存储目录下有对应的主题的目录（在consumequeue下）：

![image-20230213164652987](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213164652987.png)

消费者测试类在`org.apache.rocketmq.example.quickstart.Consumer#main`下，其余与上面同理，启动后，会收到刚才发送的消息，并在控制台打印。

## 6.2 Name Server服务注册、剔除、发现源码分析

> **在namesrv端和生产者消费者这两端流转的数据，可以自行debug查看，这里由于篇幅和文本记录的不便，这里仅展示部分数据。**

nameserver作为一个名称服务，需要提供服务注册、服务剔除、服务发现这些基本功能。nameserver的架构如下，图片来自官网：

![image-20230213165325729](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230213165325729.png)

NameServer本质上是一个AP的设计，虽然在早期也是依赖Zookeeper的，但是在3.0版本就去掉了Zookeeper的依赖，而是使用了自己的NameServer。因为NameServer需要保证最终一致，而并不需要强一致性，所以使用自己的降低维护成本。这里需要注意，NameServer不像Zookeeper（使用Zab协议）、etcd（使用raft协议），NameServer节点互不通信，无法进行数据复制。

> CAP：
>
> - C：Consistency 一致性
> - A：Availability 可用性
> - P:Partition Tolerance 分区容错性

下面就围绕着这些核心功能对nameserver源码进行学习研读。

### 6.2.1 Name Server 启动流程源码分析

Name Server最核心的就是NamesrvController，Name Server的启动本质上就是启动NamesrvController。Name Server的启动从`org.apache.rocketmq.namesrv.NamesrvStartup#main`开始，我们跟进此方法：

```java
public static void main(String[] args) {
    main0(args);
}

public static NamesrvController main0(String[] args) {

    try {
        //创建NamesrvController
        NamesrvController controller = createNamesrvController(args);
        //启动NamesrvController
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }
    return null;
}
```

可以看到namesrv的启动方法中，先创建了NamesrvController，然后启动了这个controller。我们先跟进createNamesrvController方法，此方法比较长，这里截取部分关键点：

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
    //省略部分代码。。。
    //解析启动参数
    Options options = ServerUtil.buildCommandlineOptions(new Options());
    commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
    if (null == commandLine) {
        System.exit(-1);
        return null;
    }
	//创建配置类
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);
    //解析 -c参数
    if (commandLine.hasOption('c')) {
        //省略过程
    }
    //解析 -p参数
    if (commandLine.hasOption('p')) {
        //省略过程
    }
    //将所有启动参数填充到namesrvConfig
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
    //省略部分代码。。。
    //创建controller
    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    // remember all configs to prevent discard
    controller.getConfiguration().registerConfig(properties);

    return controller;
}
```

创建完之后会启动controler，我们跟进NamesrvStartup#start：

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {

    if (null == controller) {
        throw new IllegalArgumentException("NamesrvController is null");
    }
    //初始化controller
    boolean initResult = controller.initialize();
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }
	//注册JVM钩子
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));
	//启动controller
    controller.start();

    return controller;
}
```

上面代码主要有两个核心点。一是初始化Controller；二是启动Controller。我们先看初始化controller，代码如下：

```java
public boolean initialize() {
    //加载kv配置
    this.kvConfigManager.load();
    //创建 rpcServer
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
	//注册业务处理器，注意后续的各种业务处理都是在这里注册的
    this.registerProcessor();
    //定时任务 每10s移除一次不活跃的broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
    //开启定时任务:每隔10min打印一次KV配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        // Register a listener to reload SslContext
        //处理tls模式，代码略
    }
    return true;
}
```

> 上面的registerProcessor方法这里就不做展开了，此方法中默认会进行processor的注册，在navesrv启动时默认会注册DefaultRequestProcessor，想注册broker等后续操作都是在这里注册的，如下图：
>
> ![image-20230320090616603](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320090616603.png)
>
> 像这种Processor是rocketmq的rpc业务处理器，本节不展开，后面会详细学习。

在上面初始化的过程中最重要的就是**启动了心跳检测**，也就是第一个定时任务：NameServer每隔10s扫描一次Broker，移除处于未激活状态的Broker。

下一步就是启动controller，代码如下：

```java
public void start() throws Exception {
    //启动rpc服务
    this.remotingServer.start();//底层是启动netty服务端

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

代码如上，start方法中会启动rpc服务，这里rocketmq将其封装为`remotingServer`类，本质上这个类封装的是netty。我们跟进`this.remotingServer.start()`后会发现是熟悉的netty服务端启动。这里不过多关注netty的部分，这里给出部分netty相关的代码：

```java
ServerBootstrap childHandler =
    this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
        .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
        .option(ChannelOption.SO_BACKLOG, 1024)
        .option(ChannelOption.SO_REUSEADDR, true)
        .option(ChannelOption.SO_KEEPALIVE, false)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
        .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
        .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ch.pipeline()
                    .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                    .addLast(defaultEventExecutorGroup,
                        encoder,
                        new NettyDecoder(),
                        new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                        connectionManageHandler,
                        serverHandler
                    );
            }
        });
```

综上，就是rocketmq的nameserver的启动流程。下面给出我整理的流程图：

![namesrvstart202303200900](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/namesrvstart202303200900.jpg)

### 6.2.2 服务注册——broker注册

在nameserver启动后就会监听9876（默认）端口，这时broker就会将元数据注册上来。broker元数据（路由）注册可以分为两种情况：

- 主动注册：Broker开启时进行注册。 broker调用registerBrokerAll 方法进行注册
- 定期（被动注册）：这也是broker的心跳，broker通过定时任务（线程）调用registerBrokerAll方法进行元数据的注册与更新。 默认30s心跳。

下面我们来看代码，在brokerq启动时会像namesrv一样创建BrokerController，然后启动这个controller，源码如下：

```java
public static void main(String[] args) {
    start(createBrokerController(args));
}

public static BrokerController start(BrokerController controller) {
    try {
        controller.start();
		//省略部分代码
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

然后我们跟进start方法中，在这里我们抓住主干，只关注关于注册部分的代码，源码如下：

```java
public void start() throws Exception {
    //省略部分代码

    if (!messageStoreConfig.isEnableDLegerCommitLog()) {
        startProcessorByHa(messageStoreConfig.getBrokerRole());
        handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
        //主动注册
        this.registerBrokerAll(true, false, true);
    }
	//定时任务，定时的去调用registerBrokerAll，实现定期（被动注册），这里我们可以看到，心跳会延时10s后才开始第一次发送
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
    //省略部分代码

}
```

这个方法中我们会看到，在刚启动时会调用一次注册方法`registerBrokerAll`，然后会启动一个定时任务定期进行注册。

> 关于定时任务的时间，rocketmq这里用了`Math.max`和`Math.min`方法将其周期定死在了10-60s中,默认是30s注册一次。

接下来我们看一下`registerBrokerAll`注册方法：

```java
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
    TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

    //省略部分代码

    if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.brokerConfig.getRegisterBrokerTimeoutMills())) {
        //注册broker
        doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
    }
}
```

这里我们关注其最核心的方法，即doRegisterBrokerAll方法：

```java
private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway,
    TopicConfigSerializeWrapper topicConfigWrapper) {
    //通过brokerOuterAPI完成注册消息的发送
    List<RegisterBrokerResult> registerBrokerResultList = this.brokerOuterAPI.registerBrokerAll(
        this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.getHAServerAddr(),
        topicConfigWrapper,
        this.filterServerManager.buildNewFilterServerList(),
        oneway,
        this.brokerConfig.getRegisterBrokerTimeoutMills(),
        this.brokerConfig.isCompressedRegister());
    //省略部分代码
}
```

> 在rocketmq中，broker即是服务端又是客户端。服务端是他会接收处理生产者和消费者的请求，客户端是他需要作为namesrv的客户端向namesrv发送注册等相关请求。在broker中BrokerOuterAPI就是broker作为客户端与namesrv交互的，一个封装好的类，内部维护了一个remotingClient成员，该成员是netty的客户端封装类。

我们继续跟进`brokerOuterAPI.registerBrokerAll`方法:

```java
public List<RegisterBrokerResult> registerBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final boolean oneway,
    final int timeoutMills,
    final boolean compressed) {

    //获取NameServer地址列表
    final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
    //这里是从缓存中获取；缓存中的地址一开始是在broker初始化时从配置文件中拿到的。这里不过多赘述，可以自己跟一下getNameServerAddressList这个方法就能找到
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {

        //封装请求头
        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        requestHeader.setBrokerAddr(brokerAddr);
        requestHeader.setBrokerId(brokerId);
        requestHeader.setBrokerName(brokerName);
        requestHeader.setClusterName(clusterName);
        requestHeader.setHaServerAddr(haServerAddr);
        requestHeader.setCompressed(compressed);

        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
        requestBody.setFilterServerList(filterServerList);
        final byte[] body = requestBody.encode(compressed);
        final int bodyCrc32 = UtilAll.crc32(body);
        requestHeader.setBodyCrc32(bodyCrc32);
        //下面通过阻塞异步任务完成注册，通过countDownLatch完成阻塞，通过线程池执行异步任务
        final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
        for (final String namesrvAddr : nameServerAddressList) {
            //遍历nameServerAddressList然后异步执行注册
            brokerOuterExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        //调用registerBroker注册方法
                        RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                        if (result != null) {
                            registerBrokerResultList.add(result);
                        }

                        log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                    } catch (Exception e) {
                        log.warn("registerBroker Exception, {}", namesrvAddr, e);
                    } finally {
                        //注册完成后，计数减一
                        countDownLatch.countDown();
                    }
                }
            });
        }

        try {
            //显示等待
            countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
        }
    }

    return registerBrokerResultList;
}
```

在此方法中会遍历地址列表，然后阻塞异步调用注册方法，具体可见上面的注释。我们下面继续跟进看看注册的`registerBroker`方法：

```java
private RegisterBrokerResult registerBroker(
    final String namesrvAddr,
    final boolean oneway,//注册Broker时传入的是false
    final int timeoutMills,
    final RegisterBrokerRequestHeader requestHeader,
    final byte[] body
) throws RemotingCommandException, MQBrokerException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException,
    InterruptedException {
        //设置请求业务码，这个很重要在namesrv服务端就是通过这个业务码区分业务，执行不同的处理方法，这里是REGISTER_BROKER = 103
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.REGISTER_BROKER, requestHeader);
    request.setBody(body);
	//省略部分代码。。。
	//调用rpc服务发送请求，底层是netty
    RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, timeoutMills);
    assert response != null;
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: {
            RegisterBrokerResponseHeader responseHeader =
                (RegisterBrokerResponseHeader) response.decodeCommandCustomHeader(RegisterBrokerResponseHeader.class);
            RegisterBrokerResult result = new RegisterBrokerResult();
            result.setMasterAddr(responseHeader.getMasterAddr());
            result.setHaServerAddr(responseHeader.getHaServerAddr());
            if (response.getBody() != null) {
                result.setKvTable(KVTable.decode(response.getBody(), KVTable.class));
            }
            return result;
        }
        default:
            break;
    }

    throw new MQBrokerException(response.getCode(), response.getRemark());
}
```

然后此方法就是设置了请求体和业务状态码，然后调用rpc服务类发送请求（底层是netty服务，这里暂不深究netty，后面的章节会详细学习了解rocketmq是如何通过netty实现rpc的）。

我们可以debug查看请求时发送的请求数据，如下图：

![image-20230220092951299](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230220092951299.png)

从这里可以看到上面代码中封装的请求头和请求体的所有数据。

综上，这就是broker主动和被动注册的源码分析。

### 6.2.2 番外 并发异步RPC请求模式

在broker注册时有这么一段代码：

~~~java
for (final String namesrvAddr : nameServerAddressList) {
    //遍历nameServerAddressList然后异步执行注册
    brokerOuterExecutor.execute(new Runnable() {
        @Override
        public void run() {
            try {
                //调用registerBroker注册方法
                RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                if (result != null) {
                    registerBrokerResultList.add(result);
                }

                log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
            } catch (Exception e) {
                log.warn("registerBroker Exception, {}", namesrvAddr, e);
            } finally {
                //注册完成后，计数减一
                countDownLatch.countDown();
            }
        }
    });
}

try {
    //显示等待
    countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
} catch (InterruptedException e) {
}
~~~

这段代码就是并发异步的RPC请求模式。在请求时通过countDownLatch阻塞当前线程，等待异步任务完成。同时通过线程池提交异步任务，这些任务都是并发执行的。这种方式可以大大加快业务处理速度，是一个countDownLatch的很好的使用案例，值得学习。

### 6.2.3 服务注册——namesrv注册broker

上一小节6.2.2中分析了broker注册时的源码。broker发送请求时会设置业务状态码为103。在namesrv中会根据发送请求时的这个业务状态码，找到对应的处理方法，找业务状态码的步骤我放到`6.2.3番外`中，现在我们直接看处理103broker注册的处理方法：

```java
case RequestCode.REGISTER_BROKER:
    Version brokerVersion = MQVersion.value2Version(request.getVersion());
	//当mq大于3.0.11版本
    if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
    	//注册broker
        return this.registerBrokerWithFilterServer(ctx, request);
    } else {
        return this.registerBroker(ctx, request);
    }
```

我们跟进`registerBrokerWithFilterServer`注册broker的方法：

```java
public RemotingCommand registerBrokerWithFilterServer(ChannelHandlerContext ctx, RemotingCommand request)
    throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
    final RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.readCustomHeader();
    final RegisterBrokerRequestHeader requestHeader =
        (RegisterBrokerRequestHeader) request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);

    if (!checksum(ctx, request, requestHeader)) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("crc32 not match");
        return response;
    }

    RegisterBrokerBody registerBrokerBody = new RegisterBrokerBody();

    if (request.getBody() != null) {
        try {
            registerBrokerBody = RegisterBrokerBody.decode(request.getBody(), requestHeader.isCompressed());
        } catch (Exception e) {
            throw new RemotingCommandException("Failed to decode RegisterBrokerBody", e);
        }
    } else {
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setCounter(new AtomicLong(0));
        registerBrokerBody.getTopicConfigSerializeWrapper().getDataVersion().setTimestamp(0);
    }
	//通过NameServer的Controller获取RouteInfoManager（元数据管理类），然后调用其registerBroker方法
    RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
        requestHeader.getClusterName(),
        requestHeader.getBrokerAddr(),
        requestHeader.getBrokerName(),
        requestHeader.getBrokerId(),
        requestHeader.getHaServerAddr(),
        registerBrokerBody.getTopicConfigSerializeWrapper(),
        registerBrokerBody.getFilterServerList(),
        ctx.channel());

    responseHeader.setHaServerAddr(result.getHaServerAddr());
    responseHeader.setMasterAddr(result.getMasterAddr());

    byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
    response.setBody(jsonValue);

    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}
```

此方法中我们重点关注`this.namesrvController.getRouteInfoManager().registerBroker`方法，在这里我们将broker请求发送过来的信息注册到namesrv中。在NameSrv中，MQ专门封装了一个RouteInfoManager类用于元数据管理。我们简单看一下这个类，如下图：

![image-20230216113600754](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230216113600754.png)

这个类主要有五个Map（在rocketmq中将这些map叫做table）。分别保存着各种元数据的对应关系：

- topicQueueTable：消息队列路由信息，消息发送时根据路由进行负载均衡。

  每个topic有多个QueueData，每个QueueData对应一个brokerName（也就是每个broker-master对应一个QueueData对象）。

- brokerAddrTable：broker基础信息，包含brokerName、所属集群名称、主备broker地址。

- clusterAddrTable：broker集群信息，存储集群中所有broker名称。多个Broker组成一个cluster，默认只有一个cluster即dafaultCluster。

- brokerLiveTable：broker状态信息，nameServer每次收到一台broker的心跳包后会更新该信息（BrokerLiveInfo中的lastUpdateTimestamp）。

- filterServerTable：broker上filterServer列表，用于消息过滤。

> RocketMQ基于订阅发布机制，一个topic拥有多个消息队列，一个Broker默认为每一主题创建4个读队列和4个写队列。多个Broker组成一个集群，BrokerName由相同的多台Broker组成主从架构，brokerId=0代表主节点，brokerId>0表示从节点。BrokerLiveInfo中的lastUpdateTimestamp存储上次收到Broker心跳包的时间。

以上就是元数据管理RouteInfoManager的主要属性。接下来我们继续跟进`RouteInfoManager#registerBroker`方法：

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            //申请写锁
            this.lock.writeLock().lockInterruptibly();
			//保存集群的broker信息
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            boolean registerFirst = false;
			//保存broker信息
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            
            Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
            //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
            //The same IP:PORT must only have one record in brokerAddrTable
            Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> item = it.next();
                if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                    it.remove();
                }
            }
			
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            //保存消息队列路由信息
            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

            //保存broker在线信息
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                new BrokerLiveInfo(
                    System.currentTimeMillis(),
                    topicConfigWrapper.getDataVersion(),
                    channel,
                    haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
            }

            //保存过滤数据
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            //保存HaServerAddr。haServerAddr是做m-s数据同步用的
            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            //解除写锁
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
```

从上面也可以看出，这里主要就是将请求过来的各种元数据在namesrv缓存中进行保存。同时，在此方法中有rocketmq对于读写锁的使用案例，在6.5.6番外中，会结合更多rocketmq中的案例，学习读写锁的使用、读写锁的使用场景。

### 6.2.3 番外 namesrv找处理方法的流程

这部分其实主要是netty的部分，所以不放在上面6.2.3一节中，不影响rocketmq主要流程的学习和研读。

在namesrv启动时，会启动netty服务监听端口，在启动netty的过程中会装配netty的handler处理器，其中会装配serverHandler，如下图：

![image-20230216151041344](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230216151041344.png)

然后我们看一下这个处理器：

![image-20230216151339206](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230216151339206.png)

在这里会调用`processMessageReceived`方法，`processMessageReceived`方法中会根据请求和响应分发到不同的方法中。我们这里关注一下请求的方法，即processRequestCommand：

![image-20230216151507397](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230216151507397.png)

这里主要关注一下processRequest方法，即处理请求的方法，跟进去之后，可以看到会根据请求的状态码找到不同的方法去处理对应的请求，如下图：

![image-20230216151615338](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230216151615338.png)

以上就是namesrv分发各种请求的流程，这里仅作为简单介绍，后面的章节会更详细的学习在rocketmq中netty的使用。

### 6.2.2-3 服务注册的流程整理

根据上面对源码的学习，这里给出我整理的rocketmq服务注册的整体的流程整理：

![rocketmqregister202303200930](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/rocketmqregister202303200930.jpg)

### 6.2.4 路由移除

路由移除分为主动移除和被动移除：

- 主动剔除路由：broker正常关闭，调用unregisterBroker
- 被动剔除路由：namesrv定时扫描brokerLiveTablle ，namesrv收到心跳包时会更新brokerLiveTable中的lastUpdateTimestamp，当lastUpdateTimestamp大于120s则移除该broker的信息。

下面我们分别来看看两种情况的源码：

#### 主动移除

在创建brokerController时会注册关闭的钩子方法，在这里会调用controller的shutdown方法，代码如下：

```java
Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
    private volatile boolean hasShutdown = false;
    private AtomicInteger shutdownTimes = new AtomicInteger(0);

    @Override
    public void run() {
        synchronized (this) {
            log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
            if (!this.hasShutdown) {
                this.hasShutdown = true;
                long beginTime = System.currentTimeMillis();
                controller.shutdown();
                long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
            }
        }
    }
}, "ShutdownHook"));
```

在shutdown方法中会调用`this.unregisterBrokerAll();`去取消注册，这一步就是broker的主动移除。我们跟进`unregisterBrokerAll`方法中，代码流程如下：

```java
private void unregisterBrokerAll() {
    this.brokerOuterAPI.unregisterBrokerAll(
        this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId());
}
// --->BrokerOuterAPI#unregisterBrokerAll
public void unregisterBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId
) {
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null) {
        for (String namesrvAddr : nameServerAddressList) {
            try {
                //取消注册
                this.unregisterBroker(namesrvAddr, clusterName, brokerAddr, brokerName, brokerId);
                log.info("unregisterBroker OK, NamesrvAddr: {}", namesrvAddr);
            } catch (Exception e) {
                log.warn("unregisterBroker Exception, {}", namesrvAddr, e);
            }
        }
    }
}
```

我们继续跟进`unregisterBroker`方法中:

在这里就会调用rpc服务，通过Netty向服务端发送取消注册的请求：

```java
public void unregisterBroker(
    final String namesrvAddr,
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId
) throws RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException, MQBrokerException {
    UnRegisterBrokerRequestHeader requestHeader = new UnRegisterBrokerRequestHeader();
    requestHeader.setBrokerAddr(brokerAddr);
    requestHeader.setBrokerId(brokerId);
    requestHeader.setBrokerName(brokerName);
    requestHeader.setClusterName(clusterName);
    //设置取消注册的业务状态码
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.UNREGISTER_BROKER, requestHeader);
	//调用rpc服务发送取消注册请求
    RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, 3000);
    assert response != null;
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: {
            return;
        }
        default:
            break;
    }

    throw new MQBrokerException(response.getCode(), response.getRemark());
}
```

netty的部分这里不做深究，我们下面来看看namesrv端处理取消注册的请求的流程。具体netty如何转发到处理取消注册方法的过程在6.2.3番外中详细的介绍了，这里不过多赘述。我们直接来到这个方法：

```java
public RemotingCommand unregisterBroker(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final UnRegisterBrokerRequestHeader requestHeader =
        (UnRegisterBrokerRequestHeader) request.decodeCommandCustomHeader(UnRegisterBrokerRequestHeader.class);
	//通过namesrvController拿到元数据管理类。调用元数据管理类的unregisterBroker方法
    this.namesrvController.getRouteInfoManager().unregisterBroker(
        requestHeader.getClusterName(),
        requestHeader.getBrokerAddr(),
        requestHeader.getBrokerName(),
        requestHeader.getBrokerId());

    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
}
```

我们跟进unregisterBroker方法：

```java
public void unregisterBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId) {
    try {
        try {
            //获取写锁
            this.lock.writeLock().lockInterruptibly();
            //获取在线broker信息
            BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);
            log.info("unregisterBroker, remove from brokerLiveTable {}, {}",
                brokerLiveInfo != null ? "OK" : "Failed",
                brokerAddr
            );

            this.filterServerTable.remove(brokerAddr);
			
            boolean removeBrokerName = false;
            //根据brokerName获取Broker基本信息
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            //移除基本信息
            if (null != brokerData) {
                String addr = brokerData.getBrokerAddrs().remove(brokerId);
                log.info("unregisterBroker, remove addr from brokerAddrTable {}, {}",
                    addr != null ? "OK" : "Failed",
                    brokerAddr
                );
                if (brokerData.getBrokerAddrs().isEmpty()) {
                    this.brokerAddrTable.remove(brokerName);
                    log.info("unregisterBroker, remove name from brokerAddrTable OK, {}",
                        brokerName
                    );
                    removeBrokerName = true;
                }
            }
			//移除clusterAddrTable和topicQueueTable中的信息
            if (removeBrokerName) {
                Set<String> nameSet = this.clusterAddrTable.get(clusterName);
                if (nameSet != null) {
                    boolean removed = nameSet.remove(brokerName);
                    log.info("unregisterBroker, remove name from clusterAddrTable {}, {}",
                        removed ? "OK" : "Failed",
                        brokerName);

                    if (nameSet.isEmpty()) {
                        this.clusterAddrTable.remove(clusterName);
                        log.info("unregisterBroker, remove cluster from clusterAddrTable {}",
                            clusterName
                        );
                    }
                }
                //从topicQueueTable中移除
                this.removeTopicByBrokerName(brokerName);
            }
        } finally {
            //释放写锁
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("unregisterBroker Exception", e);
    }
}
```

在这个方法中会将该broker在缓存中的所有信息进行移除。以上就是主动移除的流程。

#### 被动移除

在nameController初始化时（流程在6.2.1中）会启动一个定时任务，默认每十秒扫描一次不活跃的broker，如下：

```java
//定时任务 每10s移除一次不活跃的broker
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
}, 5, 10, TimeUnit.SECONDS);
```

我们跟进scanNotActiveBroker方法中查看如何进行处理，代码如下：

```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        //当最后更新时间大于120秒则移除该broker
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            //移除出brokerLiveTable
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            //销毁通道
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```

以上就是被动移除的流程，整体流程比较简单。

#### 服务剔除整体流程

根据上面对源码的学习和整理，下面给出服务剔除的流程图：

![rocketmqserviceremove202303200940](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/rocketmqserviceremove202303200940.jpg)

### 6.2.5 服务发现

下面我们来学习了解一下生产者消费者的服务发现。在微服务中每个服务既是元数据提供者，又是使用者。而在rocketmq中broker是元数据的提供者，生产者消费者才是元数据的使用者。

rocketmq的服务发现分为两种方式，跟服务移除一样也是分为主动进行服务发现和被动刷新服务在线列表。

#### 生产者服务发现

对于生产者来说，在发送第一条消息时和start后，会根据topic从nameserver获取路由信息。下面我们来跟踪一下源码：

我们先来看一下生产者启动时如何获取路由信息，首先我们从producer的start方法作为入口进行跟进，源码如下：

```java
@Override
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    //启动默认生产者实现
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

这里我们主要关注`defaultMQProducerImpl.start()`方法，然后我们跟进此方法：

```java
public void start() throws MQClientException {
    this.start(true);
}
//---->
public void start(final boolean startFactory) throws MQClientException {
    //这里刚启动时serviceState==CREATE_JUST
    switch (this.serviceState) {
        case CREATE_JUST:
            //代码会进入此分支
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();

            if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                this.defaultMQProducer.changeInstanceNameToPID();
            }
			//创建MQClientInstance
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
			//注册生产者，会将当前生产者存入producerTable中，具体见下面的图片
            boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }
			//维护topicPublishInfoTable（元数据）数据，这里的topic是TBW102，用于topic的自动创建
            this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

            if (startFactory) {
                //核心点：启动MQClientInstance
                mQClientFactory.start();
            }

            log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                this.defaultMQProducer.isSendMessageWithVIPChannel());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The producer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }

    //省略部分代码。。。
}
```

> 在上面注册生产时，会将当前要启动的生产者存放进MQClientInstance#producerTable中
>
> ![image-20230320103245557](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320103245557.png)

然后我们关注主要流程，跟进`mQClientFactory.start()`方法：

```java
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                this.startScheduledTask();
                // Start pull service
                this.pullMessageService.start();
                // Start rebalance service
                this.rebalanceService.start();
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

在此方法中会启动很多服务，我们这里只关注启动定时任务的方法`this.startScheduledTask()`，进入此方法，此方法中会启动很多定时任务分别执行不同的功能，我们这里只关注其中一个定时任务，如下：

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            MQClientInstance.this.updateTopicRouteInfoFromNameServer();
        } catch (Exception e) {
            log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
        }
    }
}, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
```

这里会延迟10s，然后每30s调用一次updateTopicRouteInfoFromNameServer方法，这个方法就是根据topic从服务端拉取元数据的方法。也就是说在启动后，生产者会延时10s去拉取元数据信息，并且每30s都会更新一次。

到此我们先暂停，暂时不深入updateTopicRouteInfoFromNameServer方法。我们回到开头，一开始我们说了生产者在启动后会去拉取元数据信息，在上面的源码分析中我们已经了解了其流程。但是一般生产者都是在发送消息时才会知道，自己到底要往哪个topic发消息，这时如果只依靠定时任务来获取元数据信息是没有实效性的。所以rocketmq在发送消息时也会根据我们指定的topic去拉取对应的元数据。我们下面看看在发送时是如何获取元数据的，我们从send方法作为入口进行跟进：

```java
@Override
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    //校验信息
    Validators.checkMessage(msg, this);
    //设置topic
    msg.setTopic(withNamespace(msg.getTopic()));
    //发送信息
    return this.defaultMQProducerImpl.send(msg);
}
```

我们主要跟进发送信息的方法中：

```java
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return send(msg, this.defaultMQProducer.getSendMsgTimeout());
}
//----->
public SendResult send(Message msg,
                       long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
}
```

然后我们跟进sendDefaultImpl方法中，此方法特别长我们这里只关注其发送消息时是如何获取元数据信息的，如下图：

![image-20230320134229233](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320134229233.png)

我们进入tryToFindTopicPublishInfo方法：

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    //先从本地缓存中获取元数据信息，从上面的启动流程也能看到，第一次发送消息时这里只有TWB102的元数据信息，所以这里为空
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        //维护当前topic的元数据信息
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        //调用updateTopicRouteInfoFromNameServer方法，更新此topic的元数据信息
        //注意如果这个topic当前没有被创建过，那么这里返回时，topicPublishInfoTable内部的messageQueueList是为空的，这样在下面就会走到else分支
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
	//如果不是第一次发送信息，那么topic的元数据信息不为空，这里会判断topicPublishInfoTable内部的messageQueueList是否为空，不为空直接返回即可
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        //如果为空，这里就会再去拉取一遍，把队列信息补全，注意这时isDefault为true，会基于TBW102创建路由元数据
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

> 我们可以debug查看上述流程：
>
> 首先我们用一个没有创建过的新topic发消息，然后打上断点，如下：
>
> ![image-20230320152308503](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320152308503.png)
>
> 我们可以看到因为是第一次发消息，所以topicPublishInfo为null，这时会进第一个if分支，去namesrv拉取元数据，但是此时由于没有此topic，所以没有消费队列信息如上图。这时就会进到下面的else分支再次拉取元数据，注意这时isDefault为true，会基于TBW102创建路由元数据，直接获取返回如下图：
>
> ![image-20230327141603885](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327141603885.png)
>
> 拉取完成后，消费队列信息就有了：
>
> ![image-20230320152753693](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320152753693.png)
>
> 当下次再次发送消息时，这时从本地缓存中就可以拿到topicPublishInfo，如下图，这时就会直接进入到下面的if分支，直接返回。
>
> ![image-20230320153016238](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320153016238.png)
>
> 这时我们进行正常的消息发送。如果broker的配置autoCreateTopicEnable配置为true，在初始化Broker的时候会自动创建TBW102，然后消息发过来broker会检查消息是否存在，不存在会基于defaultTopic的配置去创建该Topic（关于broker的流程会在下面的章节进行源码的学习和笔记的整理）。
>
> 然后我们在重启producer，因为是第一次发消息所以topicPublishInfo为null。这时还是会进入第一个if分支去拉取元数据。但是因为这个topic我们刚才创建了，所以这时一次就能直接查询到元数据信息和消费队列信息，所以再走到下面的if时就会直接返回。
>
> ![image-20230320152924276](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320152924276.png)
>
> 
>
> 总结一下，当Producer在发送一个不存在的Topic消息时，首先会从NameServer拉取Topic路由数据，第一次拉取必然失败，第二次会直接拉取TBW102的路由数据，基于它创建TopicPublishInfo并缓存到本地，进行正常的消息发送，发送封装Header时会将defaultTopic设置为TBW102。Broker接收到消息时，先对消息做Check，检查到Topic不存在，会基于defaultTopic的配置去创建该Topic，然后注册到NameServer上，这样一个全新的Topic就被自动创建了。我们重启producer之后，由于在borker已经有了该消息，所以会同步到namesrv中，这时produer就会拉取对应的元数据路由信息了。

到此，我们可以看到，不管是走哪个流程，元数据信息在生产者这端最终都会调用updateTopicRouteInfoFromNameServer方法去从namesrv中获取更新元数据信息。因此下面我们来看看这个方法，我们先跟进去看看源码：

```java
public boolean updateTopicRouteInfoFromNameServer(final String topic) {
    return updateTopicRouteInfoFromNameServer(topic, false, null);
}

// -------->
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,
                                                  DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
            try {
                TopicRouteData topicRouteData;
                if (isDefault && defaultMQProducer != null) {
                    //通过netty向namesrv发送GET_ROUTEINTO_BY_TOPIC（105）请求，获取topic的元数据信息
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                                                                                                 1000 * 3);
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else {
                    topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
                }
                if (topicRouteData != null) {
                    //判断元数据是否有变化
                    TopicRouteData old = this.topicRouteTable.get(topic);
                    boolean changed = topicRouteDataIsChange(old, topicRouteData);
                    if (!changed) {
                        changed = this.isNeedUpdateTopicRouteInfo(topic);
                    } else {
                        //省略日志代码
                    }
				
                    if (changed) {
                        TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();
					//维护broker的元数据信息
                        for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                            this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
                        }

                        // Update Pub info更新生产者信息
                        {
                            //将拉取的topicRouteData转换成可存储的TopicPublishInfo
                            TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
                            publishInfo.setHaveTopicRouterInfo(true);
                            Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQProducerInner> entry = it.next();
                                MQProducerInner impl = entry.getValue();
                                if (impl != null) {
                                    //保存进topicPublishInfoTable
                                    impl.updateTopicPublishInfo(topic, publishInfo);
                                }
                            }
                        }

                        // Update sub info更新消费者信息
                        {
                            Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
                            Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQConsumerInner> entry = it.next();
                                MQConsumerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicSubscribeInfo(topic, subscribeInfo);
                                }
                            }
                        }
                        //省略日志代码
                        this.topicRouteTable.put(topic, cloneTopicRouteData);
                        return true;
                    }
                } else {
                    //省略日志代码
                }
            } catch (MQClientException e) {
                //省略catch代码
            } catch (RemotingException e) {
                //省略catch代码
            } finally {
                this.lockNamesrv.unlock();
            }
        } else {
            //省略日志代码
        }
    } catch (InterruptedException e) {
        //省略日志代码
    }

    return false;
}
```

> 维护broker信息的debug数据，将元数据信息保存在brokerAddrTable
>
> ![image-20230320154606034](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320154606034.png)
>
> 保存生产者信息元数据时，需要先做一个转换，如下图：
>
> ![image-20230320155009780](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230320155009780.png)

以上就是生产者服务发现流程。

#### 消费者服务发现

与生产者不同，对于消费者而言，订阅的Topic是一定的，所以在启动时就会拉取元数据。

我们以start方法作为入口：

```java
@Override
public void start() throws MQClientException {
    setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
    //关键代码，启动defaultMQPushConsumerImpl
    this.defaultMQPushConsumerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

然后我们跟进`defaultMQPushConsumerImpl.start()`，这个方法比较长，这里展示出服务发现的关键代码，如下图：

![image-20230327114532865](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327114532865.png)

我们可以看到这里会调用`mQClientFactory.start()`方法，这个方法我们在上一小节刚见过，他会启动一个延时定时任务，然后定期更新服务地址，这里我们就不赘述了。当然与生产者同理，如果只依靠定时任务是没有时效性的，所以这个方法后续会主动更新一次元数据，代码如下图：

![image-20230327115915916](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327115915916.png)

我们跟进此方法：

```java
private void updateTopicSubscribeInfoWhenSubscriptionChanged() {
    //获取当前订阅的主题
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            //根据主题获取元数据
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        }
    }
}
```

> debug可以看到相关的数据，下图消费者启动时，除了我们自己订阅的的TopicTest，还会更新重试队列的主题元数据：
>
> ![image-20230327131542935](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327131542935.png)

此方法会先获取我们订阅的主题，然后更新相关的元数据，这里会调用`updateTopicRouteInfoFromNameServer`方法进行更新，后续的流程与生产者类似，这里就不再赘述。

#### namesrv端返回元数据信息

生产者和消费者在拉取元数据时，请求code是105（GET_ROUTEINTO_BY_TOPIC），我们这里直接找到对应的请求处理方法：

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    //创建请求对应的响应
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final GetRouteInfoRequestHeader requestHeader =
        (GetRouteInfoRequestHeader) request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);
	//从namesrv的缓存中获取对应的元数据
    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());

    if (topicRouteData != null) {
        if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
            String orderTopicConf =
                this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                    requestHeader.getTopic());
            topicRouteData.setOrderTopicConf(orderTopicConf);
        }
		//封装并返回
        byte[] content = topicRouteData.encode();
        response.setBody(content);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }
	//如果没有找到对应的主题元数据就在这里封装并返回
    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
        + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
}
```

此方法主要就是从namesrv的缓存中拿到topic对应的元数据并返回。获取数据的方法是`this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic())`，我们可以跟进这个方法：

```java
public TopicRouteData pickupTopicRouteData(final String topic) {
    TopicRouteData topicRouteData = new TopicRouteData();
    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    //封装返回的数据格式
    Set<String> brokerNameSet = new HashSet<String>();
    List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
    topicRouteData.setBrokerDatas(brokerDataList);

    HashMap<String, List<String>> filterServerMap = new HashMap<String, List<String>>();
    topicRouteData.setFilterServerTable(filterServerMap);

    try {
        try {
            //加读锁，准备获取数据
            this.lock.readLock().lockInterruptibly();
            //先获取topicQueueTable中的数据
            List<QueueData> queueDataList = this.topicQueueTable.get(topic);
            if (queueDataList != null) {
                topicRouteData.setQueueDatas(queueDataList);
                foundQueueData = true;

                Iterator<QueueData> it = queueDataList.iterator();
                while (it.hasNext()) {
                    QueueData qd = it.next();
                    brokerNameSet.add(qd.getBrokerName());
                }
				//获取封装brokerAddrTable中的数据
                for (String brokerName : brokerNameSet) {
                    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                    if (null != brokerData) {
                        BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData
                            .getBrokerAddrs().clone());
                        brokerDataList.add(brokerDataClone);
                        foundBrokerData = true;
                        for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                            List<String> filterServerList = this.filterServerTable.get(brokerAddr);
                            filterServerMap.put(brokerAddr, filterServerList);
                        }
                    }
                }
            }
        } finally {
            //获取完数据释放锁
            this.lock.readLock().unlock();
        }
    } catch (Exception e) {
        log.error("pickupTopicRouteData Exception", e);
    }

    log.debug("pickupTopicRouteData {} {}", topic, topicRouteData);
	//返回获取的数据
    if (foundBrokerData && foundQueueData) {
        return topicRouteData;
    }

    return null;
}
```

这里都是从缓存中获取的数据，是6.2.3中介绍的五个本地map缓存。数据是broker注册时缓存的。

以上就是namesrv端服务发现的流程，整体比较简单。

#### 服务发现整体流程

根据上面的源码的学习，下面给出我对服务发现的整体的流程整理：

![recketmqservicefound202303201602](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/recketmqservicefound202303201602.jpg)

在生产者和消费者这端，最终都会调用`MQClientInstance#updateTopicRouteInfoFromNameServer(...)`方法，这个方法的本质就是通过netty向namesrv发起请求。在RocketMQ的代码架构中消费者和生产者本质都是RocketMQ的客户端，所以我们可以看到最后调用的方法都是这个方法。

同时消费者和生产者都会主动和被动的拉取元数据，体现在上面的代码和流程中就是消费者启动时主动更新订阅的主题、生产者启动时更新TBW102主题、生产者发送消息时更新对应的主题；同时生产者和消费者启动时都会启动一个定时任务用于元数据的被动刷新。

在服务端方面，namesrv需要做的就只是查询对应的元数据返回给客户端（生产者和消费者）即可。

### 6.2.6 客户端选择NameServer

RocketMQ客户端启动时会将用户设置的nameserver列表设置进NettyRemotingClient类的namesrvAddrList字段中，这里的代码流程略，非常简单，感兴趣可以自己查看源码。

然后在客户端节点发消息时，就会随机找一个NameServer进行连接，如果出问题后面的是使用轮询策略。rocketmq这么做是为了减少服务端的压力，因为对于namesrv来说服务端就几个，但客户端可能会有很多比如broker、consumer、producer，如果客户端与所有的服务端都建立连接那么对于服务端来说压力会很大，所以客户端会随机找一个服务端进行连接，如果出问题后面的是使用轮询策略，下面我们来看一下源码流程：

我们直接来到从客户端发消息的类的代码中，在`NettyRemotingClient#invokeSync`方法，我们直接跟进此方法源码：

```java
@Override
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    long beginStartTime = System.currentTimeMillis();
    //根据传入的服务端地址获取namesrv的channel
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            //执行前置方法
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTimeoutException("invokeSync call timeout");
            }
            //执行发消息，内部是channel直接将消息刷入服务端
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
            //执行后置方法
            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            return response;
        } catch (RemotingSendRequestException e) {
            log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        } catch (RemotingTimeoutException e) {
            if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                this.closeChannel(addr, channel);
                log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
            }
            log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

在上面的方法中我们主要关注getAndCreateChannel方法，从此方法中会根据传入的地址拿到对应的channel，我们跟进此方法：

```java
private Channel getAndCreateChannel(final String addr) throws RemotingConnectException, InterruptedException {
    //如果地址为空，会调用getAndCreateNameserverChannel方法，此方法内部创建channel时也是调用下面的createChannel方法
    if (null == addr) {
        return getAndCreateNameserverChannel();
    }
	//否则从本地缓存中拿到channel
    ChannelWrapper cw = this.channelTables.get(addr);
    if (cw != null && cw.isOK()) {
        return cw.getChannel();
    }
	//如果本地缓存也没拿到，那就创建一个channel
    return this.createChannel(addr);
}
```

> netty客户端刚启动时是没有channel的，因为这时还不知道要连接到哪个服务端，源码如下
>
> ![image-20230327145922339](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327145922339.png)

我们下面跟进getAndCreateNameserverChannel方法：

```java
private Channel getAndCreateNameserverChannel() throws RemotingConnectException, InterruptedException {
    //先尝试获取上次连接的服务端地址
    String addr = this.namesrvAddrChoosed.get();
    if (addr != null) {
        //如果不为空，直接从channel缓存中拿到channel并返回
        ChannelWrapper cw = this.channelTables.get(addr);
        if (cw != null && cw.isOK()) {
            return cw.getChannel();
        }
    }

    //获取配置文件中的namesrv的地址
    final List<String> addrList = this.namesrvAddrList.get();
    if (this.lockNamesrvChannel.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
        try {
            //先尝试获取上次连接的服务端地址
            addr = this.namesrvAddrChoosed.get();
            if (addr != null) {
                ChannelWrapper cw = this.channelTables.get(addr);
                if (cw != null && cw.isOK()) {
                    return cw.getChannel();
                }
            }
		   //遍历配置文件中的namesrv的地址
            if (addrList != null && !addrList.isEmpty()) {
                for (int i = 0; i < addrList.size(); i++) {
                    //轮询获取namesrv的地址
                    //namesrvIndex，最开始是一个随机值
                    int index = this.namesrvIndex.incrementAndGet();
                    index = Math.abs(index);
                    index = index % addrList.size();
                    String newAddr = addrList.get(index);
					//设置获取的地址为上次连接的服务端地址
                    this.namesrvAddrChoosed.set(newAddr);
                    log.info("new name server is chosen. OLD: {} , NEW: {}. namesrvIndex = {}", addr, newAddr, namesrvIndex);
                    //创建channel
                    Channel channelNew = this.createChannel(newAddr);
                    if (channelNew != null) {
                        return channelNew;
                    }
                }
                throw new RemotingConnectException(addrList.toString());
            }
        } finally {
            this.lockNamesrvChannel.unlock();
        }
    } else {
        log.warn("getAndCreateNameserverChannel: try to lock name server, but timeout, {}ms", LOCK_TIMEOUT_MILLIS);
    }

    return null;
}
```

我们可以看到上面会轮询获取服务端地址，且为了防止每个客户端启动获取的namesrv地址是一样的造成单个点压力过大，轮询时还设置了一个随机数，代码如下：

```java
private final AtomicInteger namesrvIndex = new AtomicInteger(initValueIndex());

//initValueIndex()方法如下：
private static int initValueIndex() {
    Random r = new Random();

    return Math.abs(r.nextInt() % 999) % 999;
}
```

拿到地址后就会创建channel了，我们跟进createChannel方法，此方法比较简单，主要流程是先从缓存获取channel，如果拿不到就创建一个，创建的方法就是通过netty客户端连接到服务端，核心代码如下图：

![image-20230327151849287](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230327151849287.png)

以上就是客户端选择服务端的整体流程。

### 6.2.7 番外rocketmq使用读写锁

我们下面来学习一下rocketmq读写锁的用法。读写锁的使用场景是读多写少。在NameSrv中写是broker进行写，但是读的话是被N个生产者消费者进行读取。为了保证并发下的一致性和高性能，RocketMQ使用了读写锁。我们这里以namesrv端返回元数据信息的方法为例：

```java
try {
    try {
        //调用可被打断的加锁方法lockInterruptibly
        this.lock.readLock().lockInterruptibly();
        //操作加锁数据
        List<QueueData> queueDataList = this.topicQueueTable.get(topic);
        if (queueDataList != null) {
            topicRouteData.setQueueDatas(queueDataList);
            foundQueueData = true;

            Iterator<QueueData> it = queueDataList.iterator();
            while (it.hasNext()) {
                QueueData qd = it.next();
                brokerNameSet.add(qd.getBrokerName());
            }

            for (String brokerName : brokerNameSet) {
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null != brokerData) {
                    BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData
                        .getBrokerAddrs().clone());
                    brokerDataList.add(brokerDataClone);
                    foundBrokerData = true;
                    for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                        List<String> filterServerList = this.filterServerTable.get(brokerAddr);
                        filterServerMap.put(brokerAddr, filterServerList);
                    }
                }
            }
        }
    } finally {
        //解锁
        this.lock.readLock().unlock();
    }
} catch (Exception e) {
    log.error("pickupTopicRouteData Exception", e);
}
```

这里RocketMQ使用了读写锁，并且使用了两层try的方式进行加锁，这里调用了可被打断的lock方法。

## 6.3 Producer消息发送源码分析研读

producer是生产者，主要功能就是发送消息到broker，并供消费者使用。在RocketMQ中消息发送共有三种模式：

- 同步：收到服务端响应才能继续发送消息。
- 异步：不用等待服务端响应，可以继续发送消息，发送方通过回调接口接收服务端响应。
- 单向：不等待服务端响应，且不触发回调函数

消息发送主要可以归结为四大步骤，分别是消息校验、路由元数据的查找、队列的选择、消息发送到对端。下面我们通过源码进行了解。

### 6.3.1 消息发送的第一步——消息校验

我们从producer的send方法做入口，这里是以同步的send方法为例:

```java
@Override
public SendResult send(
    Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    //消息校验
    Validators.checkMessage(msg, this);
    //设置消息命名空间
    msg.setTopic(withNamespace(msg.getTopic()));
    //发送消息
    return this.defaultMQProducerImpl.send(msg);
}
```

在上面的方法中就会进行消息的校验，我们跟进`Validators.checkMessage(msg, this);`方法：

在此方法中会对空值，topic等进行校验：

```java
public static void checkMessage(Message msg, DefaultMQProducer defaultMQProducer)
    throws MQClientException {
    if (null == msg) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message is null");
    }
    // topic 检查topic非空、长度等校验
    Validators.checkTopic(msg.getTopic());

    // body
    if (null == msg.getBody()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body is null");
    }

    if (0 == msg.getBody().length) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body length is zero");
    }

    if (msg.getBody().length > defaultMQProducer.getMaxMessageSize()) {
        throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL,
            "the message body size over max value, MAX: " + defaultMQProducer.getMaxMessageSize());
    }
}
```

以上就是消息发送的第一步，在消息确定没有问题后，就会调用send方法发送消息。

### 6.3.2 消息发送的第二步——消息路由元数据的查找

我们继续上面的流程跟进send方法，最终会来到`DefaultMQProducerImpl#sendDefaultImpl`方法

> DefaultMQProducerImpl#sendDefaultImpl这个方法比较长，本小节只展示关键源码，此方法也是消息发送的核心方法。

整体的调用链可以debug查看调用堆栈，如下图：

![image-20230329101632458](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329101632458.png)

在sendDefaultImpl方法中第一步就是去获取当前topic的路由元数据信息，如下图：

![image-20230329101902727](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329101902727.png)

我们跟进`DefaultMQProducerImpl#tryToFindTopicPublishInfo`方法中：

```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    //先从本地缓存中获取元数据信息，从上面的启动流程也能看到，第一次发送消息时这里只有TWB102的元数据信息，所以这里为空
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        //维护当前topic的元数据信息
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        //调用updateTopicRouteInfoFromNameServer方法，更新此topic的元数据信息
        //注意如果这个topic当前没有被创建过，那么这里返回时，topicPublishInfoTable内部的messageQueueList是为空的，这样在下面就会走到else分支
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
	//如果不是第一次发送信息，那么topic的元数据信息不为空，这里会判断topicPublishInfoTable内部的messageQueueList是否为空，不为空直接返回即可
    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        //如果为空，这里就会再去拉取一遍，把队列信息补全，注意这时isDefault为true，会基于TBW102创建路由元数据
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

这个方法在上面`6.2.5 生产者的服务发现`一节中已经详细分析研读过了，包括自动创建topic的原理。这里就不再赘述了。此方法最终获取的路由信息示例如下图：

![image-20230329114433462](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329114433462.png)

### 6.3.3 消息发送的第三步——选择消费队列

在`DefaultMQProducerImpl#sendDefaultImpl`方法中，当获取到消息的路由元数据之后，就会去根据路由信息选择一个消费队列，源码如下图：

![image-20230329104121170](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230329104121170.png)

我们跟进selectOneMessageQueue，

> 这里需要注意，此方法（selectOneMessageQueue）的lastBrokerName参数是上一次发送失败的brokerName，因为此方法是在重试的循环中，所以如果上一次发送消息失败了，那么这句代码中`String lastBrokerName = null == mq ? null : mq.getBrokerName();`mq就不为空，就可以拿到上一次失败的brokerName

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    return this.mqFaultStrategy.selectOneMessageQueue(tpInfo, lastBrokerName);
}
```

继续跟进：

```java
//MQFaultStrategy#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    //是否启用了失败延时策略，默认不启用
    if (this.sendLatencyFaultEnable) {
        //代码暂略
    }
	//选择一个消费队列
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

我们这里暂时先略过失败延时策略（下一小节在详细学习这个策略），直接进入`tpInfo.selectOneMessageQueue(lastBrokerName);`方法：

```java
//TopicPublishInfo#selectOneMessageQueue(java.lang.String)
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    //判断上一次失败的的BrokerName是否为空，可以理解为上一次本条消息的的发送是否成功
    if (lastBrokerName == null) {
        //如果没有失败则直接选择一个队列
        return selectOneMessageQueue();
    } else {
        //如果上一次失败了，那么就会走这个分支
        //首先获取一个用于负载均衡的index随机数
        int index = this.sendWhichQueue.getAndIncrement();
        //遍历消息队列列表
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
            if (pos < 0)
                pos = 0;
            MessageQueue mq = this.messageQueueList.get(pos);
            //隔离上一个失败的broker，选一个其他broker的队列
            if (!mq.getBrokerName().equals(lastBrokerName)) {
                return mq;
            }
        }
        //兜底，如果只有一个Broker，那么就会一直是上一个失败的BrokerName，
        //就会走这个方法重新选一个消费队列
        return selectOneMessageQueue();
    }
}
```

这个方法中会根据上一次本条消息是否发送失败，来决定是否规避上一次失败的broker。我们再来看一下`selectOneMessageQueue`方法：

```java
//TopicPublishInfo#selectOneMessageQueue()
public MessageQueue selectOneMessageQueue() {
    //轮询算法，选择一个队列
    int index = this.sendWhichQueue.getAndIncrement();
    //取模，防止超出队列范围
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    return this.messageQueueList.get(pos);
}
```

从上面方法可以看到，本质上就是一个轮询算法，但需要注意的是，这里的index变量的获取是一个threadLocal+随机数的形式，这么做的目的是避免了锁的操作。我们可以看一下`getAndIncrement()`方法：

```java
public int getAndIncrement() {
    //从threadLocal拿到上一次的index
    Integer index = this.threadLocalIndex.get();
    //如果没有就用随机数生成一个并存入threadLocal
    if (null == index) {
        index = Math.abs(random.nextInt());
        if (index < 0)
            index = 0;
        this.threadLocalIndex.set(index);
    }
	//如果有就加一，并保存进threadLocal
    index = Math.abs(index + 1);
    if (index < 0)
        index = 0;

    this.threadLocalIndex.set(index);
    return index;
}
```

以上的流程都比较简单，下面我们来看看如果开启了故障延时策略的队列选择。

### 6.3.4 选择队列时的故障延时策略

我们回到MQFaultStrategy#selectOneMessageQueue方法：

> PS：可以通过producer对象开启：`producer.setSendLatencyFaultEnable(true);`

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        //如果开启了故障延时策略
        try {
            //获取index,通过轮询策略获取index
            int index = tpInfo.getSendWhichQ  ueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                //取模获得pos
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                //判断当前broker是否被隔离，没被隔离就返回该队列，否则就继续遍历选一个合适的
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }
			//所有队列都被隔离中的话，就从faultItemTable选择一个相对较好的队列
            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }
		//兜底，如果故障列表也没有可写的队列，则随机选一个队列
        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```

如果开启了故障延时策略，那么就有三种情况，：

1. 轮询获取一个没被隔离的消费队列
2. 如果所有的队列都被隔离，那么就从faultItemTable选择一个相对较好的队列
3. 最后，如果故障列表也没有可写的队列，则随机选一个队列

我们先来看一下如何判断当前broker是否被隔离，我们跟进`LatencyFaultToleranceImpl#isAvailable`方法：

```java
//LatencyFaultToleranceImpl#isAvailable
@Override
public boolean isAvailable(final String name) {
    //faultItemTable是内部维护的隔离缓存，本质是个Map
    final FaultItem faultItem = this.faultItemTable.get(name);
    if (faultItem != null) {
        //此方法或根据brokerName判断是否此broker被隔离，从faultItemTable拿到对应的FaultItem
        return faultItem.isAvailable();
    }
    return true;
}
```

我们跟进`faultItem.isAvailable()`方法：

```java
//LatencyFaultToleranceImpl.FaultItem#isAvailable
public boolean isAvailable() {
    //startTimestamp = System.currentTimeMillis() + notAvailableDuration
    //判断隔离是否结束，startTimestamp是隔离时间；当前时间减去隔离时间如果大于0说明隔离结束
    return (System.currentTimeMillis() - startTimestamp) >= 0;
}
```

> 我们可以看一下这个startTimestamp：
>
> 这个属性是隔离缓存项中的属性，如下图：
>
> ![image-20230402092243253](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402092243253.png)
>
> 实际上这写属性是updateFaultItem方法维护的。此（updateFaultItem）方法会在DefaultMQProducerImpl#sendDefaultImpl方法中被调用的（如下图），用于维护隔离列表也就是faultItemTable。
>
> ![image-20230402103034721](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402103034721.png)
>
> ![image-20230402103115118](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402103115118.png)
>
> updateFaultItem方法源码如下：
>
> ~~~java
> //isolation参数是否需要隔离，正常是false，否则就是30s
> public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
>     //计算隔离截止时间
>     if (this.sendLatencyFaultEnable) {
>         //计算出故障时间，默认30s，由isolation参数控制
>         long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
>         //更新隔离缓存项：FaultItem，此方法很简单这里略，可自行查看
>         this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
>     }
> }
> ~~~
>
> 这里的关键就是计算隔离截止时间computeNotAvailableDuration方法：
>
> ```java
> private long computeNotAvailableDuration(final long currentLatency) {
>     //根据性能选择隔离时长
>     //currentLatency越大，那么对应的notAvailableDuration（不可用时间）越大
>     //这里就是latencyMax每一个级别对应着不可用时间notAvailableDuration的每一个级别
>     for (int i = latencyMax.length - 1; i >= 0; i--) {
>         if (currentLatency >= latencyMax[i])
>             return this.notAvailableDuration[i];
>     }
> 
>     return 0;
> }
> ```
>
> 实际上上面的方法本质就是做了一个对应，如下面代码：
>
> ```java
> private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L};
> private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L};
> ```
>
> 根据传入的总响应时间，得到对应的broker的隔离时间，而如果发送消息报错或抛异常updateFaultItem的isolation会传入true，这时就会传入默认30s的响应时间，而这时这个broker的隔离时间就是600000L毫秒，基本上就会短时间不用此broker。

如果broker全部被隔离了，就会从broker中选择一个次优的broker，我们看一下pickOneAtLeast方法：

```java
@Override
public String pickOneAtLeast() {
    final Enumeration<FaultItem> elements = this.faultItemTable.elements();
    List<FaultItem> tmpList = new LinkedList<FaultItem>();
    while (elements.hasMoreElements()) {
        final FaultItem faultItem = elements.nextElement();
        tmpList.add(faultItem);
    }

    if (!tmpList.isEmpty()) {
        //洗牌，打乱顺序
        Collections.shuffle(tmpList);
        //排序，FaultItem有排序接口
        Collections.sort(tmpList);

        final int half = tmpList.size() / 2;
        if (half <= 0) {
            return tmpList.get(0).getName();
        } else {
            final int i = this.whichItemWorst.getAndIncrement() % half;
            return tmpList.get(i).getName();
        }
    }

    return null;
}
```

此方法会将隔离缓存中的所有缓存项（faultItem）先打乱顺序，然后在重新进行排序，排序的原则是先根据响应时间排序，然后根据根据隔离时间排序，最后选择延迟时间最短、规避时间最短的broker返回。

> 这里贴出排序接口源码：
>
> ```java
> @Override
> public int compareTo(final FaultItem other) {
>     if (this.isAvailable() != other.isAvailable()) {
>         if (this.isAvailable())
>             return -1;
> 
>         if (other.isAvailable())
>             return 1;
>     }
>     //优先根据响应时间排序
>     if (this.currentLatency < other.currentLatency)
>         return -1;
>     else if (this.currentLatency > other.currentLatency) {
>         return 1;
>     }
> 
>     //其次根据隔离时间排序
>     if (this.startTimestamp < other.startTimestamp)
>         return -1;
>     else if (this.startTimestamp > other.startTimestamp) {
>         return 1;
>     }
> 
>     return 0;
> }
> ```

以上就是整个的故障延时策略。

### 6.3.5 消息发送的第四步——将消息发送到对端

选择完消费队列之后，就会发送消息，如下图：

![image-20230402150327196](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402150327196.png)

我们跟进此方法，在此方法中会做一些发送前的准备，做一些检查、封装请求头等等，我们这里略过这些部分（感兴趣可以自行查看源码和细节），直接看最关键的发送消息的部分：

```java
SendResult sendResult = null;
//根据发送模式，进行不同的方式发送消息
switch (communicationMode) {
    case ASYNC:
        Message tmpMessage = msg;
        boolean messageCloned = false;
        if (msgBodyCompressed) {
            //If msg body was compressed, msgbody should be reset using prevBody.
            //Clone new message using commpressed message body and recover origin massage.
            //Fix bug:https://github.com/apache/rocketmq-externals/issues/66
            tmpMessage = MessageAccessor.cloneMessage(msg);
            messageCloned = true;
            msg.setBody(prevBody);
        }

        if (topicWithNamespace) {
            if (!messageCloned) {
                tmpMessage = MessageAccessor.cloneMessage(msg);
                messageCloned = true;
            }
            msg.setTopic(NamespaceUtil.withoutNamespace(msg.getTopic(), this.defaultMQProducer.getNamespace()));
        }

        long costTimeAsync = System.currentTimeMillis() - beginStartTime;
        if (timeout < costTimeAsync) {
            throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
        }
        //同步消息
        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
            brokerAddr,
            mq.getBrokerName(),
            tmpMessage,
            requestHeader,
            timeout - costTimeAsync,
            communicationMode,
            sendCallback,
            topicPublishInfo,
            this.mQClientFactory,
            this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(),
            context,
            this);
        break;
    case ONEWAY:
    case SYNC:
        long costTimeSync = System.currentTimeMillis() - beginStartTime;
        if (timeout < costTimeSync) {
            throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
        }
        
        //异步和抽象消息
        sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
            brokerAddr,
            mq.getBrokerName(),
            msg,
            requestHeader,
            timeout - costTimeSync,
            communicationMode,
            context,
            this);
        break;
    default:
        assert false;
        break;
}
```

这里同步、异步和单向消息最终都会进行到下面的方法：

```java
public SendResult sendMessage(
    final String addr,
    final String brokerName,
    final Message msg,
    final SendMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    RemotingCommand request = null;
    //创建消息请求
    String msgType = msg.getProperty(MessageConst.PROPERTY_MESSAGE_TYPE);
    boolean isReply = msgType != null && msgType.equals(MixAll.REPLY_MESSAGE_FLAG);
    //isReply默认为false
    if (isReply) {
        if (sendSmartMsg) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_REPLY_MESSAGE, requestHeader);
        }
    } else {
        if (sendSmartMsg || msg instanceof MessageBatch) {
            SendMessageRequestHeaderV2 requestHeaderV2 = SendMessageRequestHeaderV2.createSendMessageRequestHeaderV2(requestHeader);
            request = RemotingCommand.createRequestCommand(msg instanceof MessageBatch ? RequestCode.SEND_BATCH_MESSAGE : RequestCode.SEND_MESSAGE_V2, requestHeaderV2);
        } else {
            request = RemotingCommand.createRequestCommand(RequestCode.SEND_MESSAGE, requestHeader);
        }
    }
    //设置请求体
    request.setBody(msg.getBody());
	//选择消息发送模式
    switch (communicationMode) {
        case ONEWAY:
            this.remotingClient.invokeOneway(addr, request, timeoutMillis);
            return null;
        case ASYNC:
            final AtomicInteger times = new AtomicInteger();
            long costTimeAsync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeAsync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            this.sendMessageAsync(addr, brokerName, msg, timeoutMillis - costTimeAsync, request, sendCallback, topicPublishInfo, instance,
                retryTimesWhenSendFailed, times, context, producer);
            return null;
        case SYNC:
            long costTimeSync = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTimeSync) {
                throw new RemotingTooMuchRequestException("sendMessage call timeout");
            }
            return this.sendMessageSync(addr, brokerName, msg, timeoutMillis - costTimeSync, request);
        default:
            assert false;
            break;
    }

    return null;
}
```

在上面的方法中，首先会组装消息请求，设置消息体等，然后就是直接根据不同的模式发送消息，下面分别来看，首先是同步消息，跟进`sendMessageSync`方法：

```java
private SendResult sendMessageSync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request
) throws RemotingException, MQBrokerException, InterruptedException {
    //发送消息
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
    assert response != null;
    //处理响应并返回
    return this.processSendResponse(brokerName, msg, response);
}
```

这里会调用invokeSync方法，然后处理响应：

```java
@Override
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    long beginStartTime = System.currentTimeMillis();
    //获取对应的channel通道
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            //前置钩子方法
            doBeforeRpcHooks(addr, request);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTimeoutException("invokeSync call timeout");
            }
            //执行同步发送方法
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
            //后置钩子方法
            doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            return response;
        } catch (RemotingSendRequestException e) {
            //省略异常和日志代码
        } catch (RemotingTimeoutException e) {
            //省略异常和日志代码
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```

我们看一下invokeSyncImpl方法：

```java
public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
    final long timeoutMillis)
    throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
    final int opaque = request.getOpaque();

    try {
        //新建ResponseFuture
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
        //在响应的缓存中保存ResponseFuture
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
        //通过channel将消息写入到对端
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (f.isSuccess()) {
                    responseFuture.setSendRequestOK(true);
                    return;
                } else {
                    responseFuture.setSendRequestOK(false);
                }
				//发送失败，从响应缓存中移除
                responseTable.remove(opaque);
                responseFuture.setCause(f.cause());
                responseFuture.putResponse(null);
                log.warn("send a request command to channel <" + addr + "> failed.");
            }
        });
		//等待响应并获取响应结果
        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
        if (null == responseCommand) {
            if (responseFuture.isSendRequestOK()) {
                //省略异常代码
            } else {
                //省略异常代码
            }
        }
        return responseCommand;
    } finally {
        this.responseTable.remove(opaque);
    }
}
```

以上最后会通过channel将消息写入对端，然后通过`responseFuture.waitResponse`等待响应，并封装响应结果返回，然后在sendMessageSync方法中调用processSendResponse处理响应，此方法中主要工作就是封装发送结果SendResult并返回，这里源码略，可以自行查看。最终发送结果SendResult如果没问题就会返回生产者，结束发送流程，如果报错抛异常就会在sendDefaultImpl中进行重试。

以上就是消息同步发送的整体流程和源码，异步和单向消息在整体发送的流程上与同步类似，但是发送时和处理略有不同，下面我们回到上面提到的sendMessage，我们继续看如果是单向消息，那么就会调用invokeOneway方法，我们跟进去：

```java
@Override
public void invokeOneway(String addr, RemotingCommand request, long timeoutMillis) throws InterruptedException,
    RemotingConnectException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    //获取或创建channel
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            doBeforeRpcHooks(addr, request);
            //发送单向消息
            this.invokeOnewayImpl(channel, request, timeoutMillis);
        } catch (RemotingSendRequestException e) {
            //省略日志和异常代码
        }
    } else {
        //省略日志和异常代码
    }
}
```

下面我们进入到invokeOnewayImpl方法：

```java
public void invokeOnewayImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    request.markOnewayRPC();
    //用semaphore做流量控制，semaphoreOneway默认数量是65535
    //这里的流控原理很简单semaphore维护了一组计数信号量（counting semaphore），
    // 当能拿到锁说明还未到计数阈值，如果拿不到锁说明已经到达了阈值，这时就不能在发送单向消息了
    boolean acquired = this.semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        //这里用SemaphoreReleaseOnlyOnce包装一下，当release时，用cas保证只释放一次
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreOneway);
        try {
            //通过channel将消息写到对端
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    once.release();
                    if (!f.isSuccess()) {
                        //省略日志代码
                    }
                }
            });
        } catch (Exception e) {
            once.release();
            //省略日志代码和异常代码
        }
    } else {
        if (timeoutMillis <= 0) {
            throw new RemotingTooMuchRequestException("invokeOnewayImpl invoke too fast");
        } else {
            //省略日志代码和异常代码
        }
    }
}
```

这里单向消息用semaphore做了一个流量控制，然后如果在阈值内就直接将消息发到对端（broker）。这里实际上异步消息和单向消息都做了流控，默认阈值是65535，这里做流控是因为防止producer发送过多的消息（因为异步不需要等待broker返回）导致消息积压，所以需要控制生产者一端的发送速度。

> mq中场景之一就是异步消息发送做流控，防止消息发送过多打满网卡或对端。在rpc类中有semaphore成员。
>
> 在发送时会先获取信号量，然后再去发消息。释放时mq封装了个释放的类，通过cas保证只释放一次，从而保证释放的正确性。异步发送时就是在回调中释放的。

以上是单向消息的发送，下面我们继续来看异步消息，我们还是回到上面提到的sendMessage方法，当消息是异步发送时，会调用sendMessageAsync方法，我们跟进去：

```java
private void sendMessageAsync(
    final String addr,
    final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int retryTimesWhenSendFailed,
    final AtomicInteger times,
    final SendMessageContext context,
    final DefaultMQProducerImpl producer
) throws InterruptedException, RemotingException {
    //异步发送
    this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        public void operationComplete(ResponseFuture responseFuture) {
            //处理异步回调
            RemotingCommand response = responseFuture.getResponseCommand();
            //当用户没有传回调方法
            if (null == sendCallback && response != null) {

                try {
                    //构建异步发送结果
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);
                    if (context != null && sendResult != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }
                } catch (Throwable e) {
                }
                //更新故障隔离的FaultItem
                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                return;
            }
            //默认走这里，当用户传了回调方法
            if (response != null) {
                try {
                    //构建异步发送结果
                    SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);
                    assert sendResult != null;
                    if (context != null) {
                        context.setSendResult(sendResult);
                        context.getProducer().executeSendMessageHookAfter(context);
                    }

                    try {
                        //执行用户传的异步回调方法
                        sendCallback.onSuccess(sendResult);
                    } catch (Throwable e) {
                    }
					//当抛异常时更新故障隔离的FaultItem
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                } catch (Exception e) {
                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                    //然后调用onExceptionImpl方法，此方法内部直接调用用户传的异常回调方法
                    onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, e, context, false, producer);
                }
            } else {
                //处理发送失败，并重试发送异步消息，注意这里onExceptionImpl传入的是true，代表需要重试发送
                producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                if (!responseFuture.isSendRequestOK()) {
                    MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else if (responseFuture.isTimeout()) {
                    MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                        responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                } else {
                    MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                    onExceptionImpl(brokerName, msg, 0L, request, sendCallback, topicPublishInfo, instance,
                        retryTimesWhenSendFailed, times, ex, context, true, producer);
                }
            }
        }
    });
}
```

在此方法中会通过netty进行异步发送，当netty收到对端的响应后会执行我们传入的回调`InvokeCallback()`方法，在`InvokeCallback`方法中会构建异步发送结果，然后执行用户给我的传的回调方法。如果发送失败会进行重试（关于异步消息的重试会在6.3.6中介绍，这里略）。

> 这里不要弄混了，InvokeCallback是sendMessageAsync传给rpc端的回调，在消息发送结束后，rpc会回调InvokeCallback方法，如下图：
>
> netty客户端流水线会有一个handler处理消息的响应：
>
> ![image-20230403111155335](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403111155335.png)
>
> 此方法最终会从响应缓存中获取对应的请求，然后执行请求的回调方法，也就是我们这里的InvokeCallback方法
>
> ![image-20230403111340913](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403111340913.png)
>
> 而sendCallback是用户传进来的回调方法，如下图：
>
> ![image-20230403111850452](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403111850452.png)

上面研究完了发送回调，下面我们跟进invokeAsync，在invokeAsync中核心流程主要是调用了invokeAsyncImpl方法，我们跟进invokeAsyncImpl中，看看消息是如何发送的：

```java
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
    final InvokeCallback invokeCallback)
    throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    long beginStartTime = System.currentTimeMillis();
    final int opaque = request.getOpaque();
    //semaphore流控
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        //包装类，通过CAS保证只释放一次
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
        long costTime = System.currentTimeMillis() - beginStartTime;
        if (timeoutMillis < costTime) {
            once.release();
            throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
        }
        //创建ResponseFuture，这里包含着回调函数等信息
        final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
        //响应缓存中保存此请求的ResponseFuture
        this.responseTable.put(opaque, responseFuture);
        try {
            //通过channel将消息写到对端
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    }
                    requestFail(opaque);
                    log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } catch (Exception e) {
            //释放流控信号量
            responseFuture.release();
            log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
            throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
        }
    } else {
        if (timeoutMillis <= 0) {
            throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
        } else {
            //省略日志和异常代码
        }
    }
}
```

上面的方法中，最终会通过netty的channel将消息写到对端，然后netty会有一个handler监听响应，当broker发送相应回来时就会调用传入的InvokeCallback，这时就会执行后续的SendResult的相关处理并调用用户的回调方法了。

综上，上面就是消息发送到对端的整体流程。

### 6.3.6 消息发送的可靠性

生产者发送消息如果失败或抛异常，生产者会自动重试，以此来保证消息投递的高可靠，重试需要判断是否超时，未超时情况下同步消息重试三次，单向不重试，异步默认重试2次。

> 异步的重试在DefaultMQProducer#retryTimesWhenSendAsyncFailed属性中配置，默认两次。
>
> private int retryTimesWhenSendAsyncFailed = 2;

同步消息重试三次具体的代码在sendDefaultImpl中，如下图：

![image-20230402220921209](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402220921209.png)

![image-20230402222054320](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230402222054320.png)

以上是普通同步消息的可靠性的保证。当消息是单向消息时不重试，因为单向消息不需要保证可靠性，当消息是异步消息时，会在超时时间内不断重试，在sendMessageAsync方法中，如果消息发送失败，那么onExceptionImpl的needRetry会传入true，这时如果还未超时，那么就会重试，代码如下：

```java
private void onExceptionImpl(final String brokerName,
    final Message msg,
    final long timeoutMillis,
    final RemotingCommand request,
    final SendCallback sendCallback,
    final TopicPublishInfo topicPublishInfo,
    final MQClientInstance instance,
    final int timesTotal,
    final AtomicInteger curTimes,
    final Exception e,
    final SendMessageContext context,
    final boolean needRetry,
    final DefaultMQProducerImpl producer
) {
    //这里timesTotal默认是2次。
    
    int tmp = curTimes.incrementAndGet();
    if (needRetry && tmp <= timesTotal) {
        //当可重试，且未超过重试次数
        //默认会向同一个broker继续重试并发送
        String retryBrokerName = brokerName;//by default, it will send to the same broker
        if (topicPublishInfo != null) { //select one message queue accordingly, in order to determine which broker to send
            MessageQueue mqChosen = producer.selectOneMessageQueue(topicPublishInfo, brokerName);
            retryBrokerName = mqChosen.getBrokerName();
        }
        //获取对端地址
        String addr = instance.findBrokerAddressInPublish(retryBrokerName);
        log.info("async send msg by retry {} times. topic={}, brokerAddr={}, brokerName={}", tmp, msg.getTopic(), addr,
            retryBrokerName);
        try {
            request.setOpaque(RemotingCommand.createNewRequestId());
            //重新执行sendMessageAsync，发送异步消息
            sendMessageAsync(addr, retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance,
                timesTotal, curTimes, context, producer);
            //如果报错就递归调用自己，根据不同的错误决定需不需要重试
        } catch (InterruptedException e1) {
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                context, false, producer);
        } catch (RemotingConnectException e1) {
            producer.updateFaultItem(brokerName, 3000, true);
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                context, true, producer);
        } catch (RemotingTooMuchRequestException e1) {
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                context, false, producer);
        } catch (RemotingException e1) {
            producer.updateFaultItem(brokerName, 3000, true);
            onExceptionImpl(retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance, timesTotal, curTimes, e1,
                context, true, producer);
        }
    } else {
		//如果不重试，或者超过重试次数
        if (context != null) {
            context.setException(e);
            context.getProducer().executeSendMessageHookAfter(context);
        }

        try {
            //直接执行用户传入的onException方法
            sendCallback.onException(e);
        } catch (Exception ignored) {
        }
    }
}
```

以上就是全部普通消息的可靠性保证，但是有序消息的重试应该由业务来完成，RocketMQ不进行负责。

因为分区消息需要发送到某个特定broker的特定分区上，如果失败了那么重试失败的概率也很大，所以mq不负责有序消息的重试。在发送有序消息得地方作为入口，有序消息有分区选择器MessageQueueSelector。

在2.3小节中，我们使用了有序消息发送，我们来回顾以下有序消息的例子：

![image-20230403112333146](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403112333146.png)

> 注意，有序消息是同步发送的，在方法中可见：DefaultMQProducerImpl#send(org.apache.rocketmq.common.message.Message, org.apache.rocketmq.client.producer.MessageQueueSelector, java.lang.Object, long)

如上图，有序消息选择分区是需要我们自己写分区选择的逻辑的，我们跟进有序消息的发送方法，最终会来到sendSelectImpl方法中，在这里我们可以看到有序消息不会走rocketMQ的分区选择方法，而是直接调用我们选择器的select方法，如下：

```java
private SendResult sendSelectImpl(
    Message msg,
    MessageQueueSelector selector,//用户业务代码传入的选择器
    Object arg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback, final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    long beginStartTime = System.currentTimeMillis();
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer);

    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        try {
            //获取消费队列列表
            List<MessageQueue> messageQueueList =
                mQClientFactory.getMQAdminImpl().parsePublishMessageQueues(topicPublishInfo.getMessageQueueList());
            Message userMessage = MessageAccessor.cloneMessage(msg);
            String userTopic = NamespaceUtil.withoutNamespace(userMessage.getTopic(), mQClientFactory.getClientConfig().getNamespace());
            userMessage.setTopic(userTopic);

            //选择消费队列，这里是用自己的选择器进行选择
            mq = mQClientFactory.getClientConfig().queueWithNamespace(selector.select(messageQueueList, userMessage, arg));
        } catch (Throwable e) {
            throw new MQClientException("select message queue throwed exception.", e);
        }

        long costTime = System.currentTimeMillis() - beginStartTime;
        if (timeout < costTime) {
            throw new RemotingTooMuchRequestException("sendSelectImpl call timeout");
        }
        if (mq != null) {
            //发送消息，注意这里有序消息使用的是同步发送的模式，且没有重试机制，因为普通同步消息在这里会进行循环，在循环中调用sendKernelImpl方法，而有序消息只调用一次不进行重试。
            return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, null, timeout - costTime);
        } else {
            throw new MQClientException("select message queue return null.", null);
        }
    }

    validateNameServerSetting();
    throw new MQClientException("No route info for this topic, " + msg.getTopic(), null);
}
```

在上面的方法中我们可以看到，有序消息使用的是同步模式发送，且分区是自己选择的。由于有序消息的rocketMQ不会自动重试，所以如果想保证有序消息的可靠性，可以在业务代码里自行增加此业务。例如通过一个for循环，最多重试几次来实现。

实际上有序消息RocketMQ不进行重试的原因主要是这些消息只能发送某个特定的Broker上的某个特定的Queue中，如果发送失败，重试失败的可能依然很大，所以默认不进行重试。如果需要重试，需要业务方自己来做。

### 6.3.8 消息发送的整体流程

上面是生产者发送消息的整体流程，以下是我根据上面的源码和流程整理的消息发送的整体流程图：

 ![rocketmqproducersend20230403141600](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/rocketmqproducersend20230403141600.jpg)

## 6.4 Broker消息存储源码分析研读

broker消息存储流程主要包含五个部分：分别是RPC（Netty代码）层、Broker处理请求部分、Broker处理消息存储、文件映射层、写入磁盘。关于Netty部分的代码，在本章节暂略，下面我们直接从Broker处理请求的位置开始研究Broker的消息存储。

### 6.4.1 Broker处理消息请求

> 关于如何定位到处理消息请求的流程在`6.2.3番外`一节中已经介绍了，broker和namesrv处理请求部分本质都是Netty服务端，所以这里不再赘述

我们直接来到处理发送消息的请求的位置：

![image-20230403142555176](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403142555176.png)

我们跟进此方法（SendMessageProcessor#consumerSendMsgBack）在此方法中，首先会进行请求消息的校验与处理，这里主要是前置的检查和校验步骤，构造MessageExtBrokerInner对象、decode反序列化、存储消息构造Response返回对象等。校验和存储消息完成后就会将RemotingCommand对象返回给对端。此方法比较长，下面只的代码是核心流程源码：

![image-20230404085325840](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230404085325840.png)

在构建完MessageExtBrokerInner后，会通过brokerController获取 MessageStore，然后调用putMessage，进行消息的存储。存储完成后会返回一个消息存储结果PutMessageResult，然后根据结果构建返回值RemotingCommand，并进行响应的发送。

以上就是Brokerc处理消息请求的流程。

### 6.4.2 Broker处理消息存储

> 这一小节我们这里暂不关注也不深究文件映射的相关方法，只抓RocketMQ存储消息的主要流程。文件映射的方法将在下一小节6.4.3中再深入分析。

#### CommitLog写入

在Broker处理请求的消息时，会调用putMessage进行消息存储，下面我们跟进此方法，查看这部分流程：

此方法会先对消息做一些校验，我们这里略过，直接看核心流程。如下图：

![image-20230403150724479](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230403150724479.png)

在校验完消息后就会调用commitLog的putMessage方法进行消息的存储

> 在RocketMQ中对消息存储一共抽象了四个类，分别是：
>
> - DefaultMessageStore
> - CommitLog
> - ConsumerQueue
> - IndexFile 
>
> 后三个类分别对应RocketMQ的三个存储文件，这四个类的关系我们可以简单理解为:DefaultMessageStore内部包含了这三个类自身或者相关服务作为成员，使用时由DefaultMessageStore向外提供服务。

我们跟进此方法（`org.apache.rocketmq.store.CommitLog#putMessage`）：

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // Set the storage time
    msg.setStoreTimestamp(System.currentTimeMillis());
    // Set the message body BODY CRC (consider the most appropriate setting
    // on the client)
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
    // Back to Results
    AppendMessageResult result = null;

    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

    String topic = msg.getTopic();
    int queueId = msg.getQueueId();

    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
        || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        //如果不是事务消息或者是事务提交消息
        // Delay Delivery
        if (msg.getDelayTimeLevel() > 0) {
            //如果延迟等级大于0，设置了延时等级
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }
            //设置topic为SCHEDULE_TOPIC，设置queueId为延时等级
            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

            // Backup real topic, queueId 备份真实的topic和queueId
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
            //将SCHEDULE_TOPIC和延时等级设置到消息中
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    //省略部分代码。。。。

    long eclipsedTimeInLock = 0;

    MappedFile unlockMappedFile = null;
    //获取映射文件
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
    //上锁
    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;

        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);
        //如果映射文件为空或者满了，则重新创建新文件
        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }

        //追加保存消息，真正保存是在appendMessageCallback这个回调中
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:
                //如果消息保存到了文件末尾，则创建新文件重新保存消息
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        eclipsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        //解锁
        putMessageLock.unlock();
    }

    if (eclipsedTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", eclipsedTimeInLock, msg.getBody().length, result);
    }

    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }
    //设置消息存储结果
    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());
    //处理刷盘
    handleDiskFlush(result, putMessageResult, msg);
    //处理主从复制
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}
```

上述方法我已经标好注释，在这个方法中，首先会进行消息的预处理，具体可见上面我标注的代码注释这里不展开；然后会调用mappedFileQueue的`getLastMappedFile`方法获取映射文件；之后就是加锁，调用appendMessage方法将消息保存进映射文件CommitLog中（注意这时并没有落盘，而是保存到了文件映射的缓存中）；消息保存完之后会返回消息保存结果，然后会对结果进行处理（这里不展开）；然后会处理消息得刷盘和主从复制，最终将消息保存结果封装后返回此方法。

以上就是CommitLog的写入过程。

#### 消费分区索引和哈希索引写入

以上就是CommitLog消息主题保存的流程，但是实际上还有ConsumeQueue和IndexFile需要保存。我们回到Broker启动流程，在broker启动的过程中会启动很多存储相关的后台服务线程，其中有一个ReputMessageService就是处理消息分区和哈希索引的服务线程。启动的流程如下：

![image-20230404092339961](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230404092339961.png)

在messageStore启动时就会启动ReputMessageService：

![image-20230404092417377](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230404092417377.png)

> Broker组件初始化的时候，会启动很多存储相关的后台服务线程：
>
> - AllocateMappedFileService（MappedFile预分配服务线程）
> - ReputMessageService（消息分区和哈希索引服务线程）
> - HAService（Broker主从同步服务线程）
> - StoreStatsService（消息存储统计服务线程）
> - IndexService（索引文件服务线程）
> - CommitRealTimeService 缓存提交线程
> - FlushRealTimeService文件系统缓存刷新线程

我们来看一下ReputMessageService，首先这个类继承了ServiceThread：

```java
class ReputMessageService extends ServiceThread
```

当调用start方法时，会调用父类也就是ServiceThread的start方法：

```java
public void start() {
    log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
    if (!started.compareAndSet(false, true)) {
        return;
    }
    stopped = false;
    //初始化thread成员，将自己传入Thread
    this.thread = new Thread(this, getServiceName());
    //设置自己为后台线程
    this.thread.setDaemon(isDaemon);
    //启动自己
    this.thread.start();
}
```

在父类中有一个Thread成员，在start方法中会将自己传入，初始化线程并启动线程。由于ServiceThread实现了Runnable接口，所以可以传入Thread并启动，同时ServiceThread又是一个抽象类，所有run方法实现都是由子类自行实现，因此当我们调用`ReputMessageService#start`方法时实际上最终会启动后台线程并执行ReputMessageService的run方法体，我们来到此方法：

```java
@Override
public void run() {
    DefaultMessageStore.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            Thread.sleep(1);
            //执行doReput方法分发消息，每1ms都执行一次
            this.doReput();
        } catch (Exception e) {
            DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }

    DefaultMessageStore.log.info(this.getServiceName() + " service end");
}
```

此方法中主要就是执行doReput进行消息的分发，我们来到doReput方法，在doReput方法中维护了一个偏移量用于记录当前CommitLog中的消息处理到哪里了，如果有新消息就会调用doDispatch方法将新消息分发给消费分区索引和哈希索引的服务，下面我们来看一下此方法：

```java
private void doReput() {
    //比较reputFromOffset和文件名的偏移量
    //reputFromOffset是ReputMessageService内部维护的偏移量，表示当前commitLog的数据已经处理到哪了
    //commitLog.getMinOffset()获取的是fileFromOffset，这个是commitLog的文件名
    if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
        //当reputFromOffset小于fileFromOffset说明，当前commitLog已经保存到了下一个文件，这时需要重新设置一下reputFromOffset
        log.warn("The reputFromOffset={} is smaller than minPyOffset={}, this usually indicate that the dispatch behind too much and the commitlog has expired.",
            this.reputFromOffset, DefaultMessageStore.this.commitLog.getMinOffset());
        this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
    }
    //isCommitLogAvailable方法：比较reputFromOffset和CommitLog的有效数据位置
    //当reputFromOffset小于CommitLog的有效数据位置，说明有新消息。
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {

        if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
            && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
            break;
        }
        //从reputFromOffset获取数据
        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            try {
                this.reputFromOffset = result.getStartOffset();

                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    //获取消息的字节内容构建分发请求
                    DispatchRequest dispatchRequest =
                        DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                    int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();

                    if (dispatchRequest.isSuccess()) {
                        if (size > 0) {
                            //调用doDispatch方法分发请求
                            DefaultMessageStore.this.doDispatch(dispatchRequest);

                            if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                                && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) {
                                DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                    dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                    dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                    dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                            }
						//维护reputFromOffset
                            this.reputFromOffset += size;
                            readSize += size;
                            if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                                    .addAndGet(dispatchRequest.getMsgSize());
                            }
                        } else if (size == 0) {
                            this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                            readSize = result.getSize();
                        }
                    } else if (!dispatchRequest.isSuccess()) {
						//省略部分代码
                    }
                }
            } finally {
                result.release();
            }
        } else {
            doNext = false;
        }
    }
}
```

上面这段代码的意思我已经做了注解的标注，大概就是如果有新消息就调用doDispatch方法将新消息分发到其他处理服务上，当然我们这里主要关注处理消费分区索引和哈希索引的服务。我们跟进doDispatch：

```java
public void doDispatch(DispatchRequest req) {
    for (CommitLogDispatcher dispatcher : this.dispatcherList) {
        dispatcher.dispatch(req);
    }
}
```

这里会遍历所有的dispatcherList，然后进行消息分发，我们在这里主要关注两个服务，我们debug可以查看，如下图：

![image-20230404102040678](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230404102040678.png)

我们主要关注构建消费分区索引的CommitLogDispatcherBuildConsumeQueue，源码如下：

```java
class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {

    @Override
    public void dispatch(DispatchRequest request) {
        final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
        switch (tranType) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                //调用putMessagePositionInfo方法,此方法最终会根据DispatchRequest构建ConsumeQueue
                DefaultMessageStore.this.putMessagePositionInfo(request);
                break;
            case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                break;
        }
    }
}
```

和构建哈希索引的CommitLogDispatcherBuildIndex，源码如下：

```java
class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {

    @Override
    public void dispatch(DispatchRequest request) {
        if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
            //调用buildIndex方法，此方法最终会根据DispatchRequest构建IndexFile
            DefaultMessageStore.this.indexService.buildIndex(request);
        }
    }
}
```

以上就是RocketMQ处理消息存储的全部流程。

### 6.4.3 Broker文件映射层

broker的文件映射部分主要有两个核心类：MappedFileQueue 文件队列类和MappedFile 消息文件类。RocketMQ主要采用mmap和filechannel完成数据文件的读写。其中采用mmap内存映射文件的方式在mq中被封装为MappedFile类。

实际上对于每类大文件（CommitLog/ConsumerQueue/IndexFile）在存储时都会被分成固定大小的文件：

- CommitLog 1G	
- ConsumerQueue 5.7M（600W字节）	
- IndexFile 400M

每个CommitLog 、ConsumerQueue 文件的文件名为前面所有文件的字节大小数目，可以理解为第一个item的起始偏移量，从而实现了整个文件的串联。注意IndexFile是创建日期与其他两个不同。每一种类的单个文件均由MappedFile类提供读写操作服务。MappedFile类提供了顺序写/随机读、内存数据刷盘、内存清理等和文件相关的服务。多个文件，由 MapedFileQueue 文件队列，统一管理。

#### CommitLog的写入

下面我们来看一下看每个文件的消息写入过程。首先我们先来看一下CommitLog的文件写入过程：

在上面我们了解了`CommitLog#putMessage`方法，此方法中首先是通过mappedFileQueue的`getLastMappedFile`方法获取mappedFile映射文件，我们跟进此方法看一下是如何获取的映射文件：

```java
public MappedFile getLastMappedFile(final long startOffset, boolean needCreate) {
    long createOffset = -1;
    //先调用重载方法，这个没有参数的重载方法实际上是直接获取最新的映射文件
    MappedFile mappedFileLast = getLastMappedFile();

    //如果最新的映射文件为空，那么就计算应该从哪里开始创建
    if (mappedFileLast == null) {
        createOffset = startOffset - (startOffset % this.mappedFileSize);
    }

    //如果最新的映射文件不为空，但是满了，那么也应该计算从哪里开始创建
    if (mappedFileLast != null && mappedFileLast.isFull()) {
        createOffset = mappedFileLast.getFileFromOffset() + this.mappedFileSize;
    }

    if (createOffset != -1 && needCreate) {
        //当需要创建文件时，计算创建文件的路径
        String nextFilePath = this.storePath + File.separator + UtilAll.offset2FileName(createOffset);
        String nextNextFilePath = this.storePath + File.separator
            + UtilAll.offset2FileName(createOffset + this.mappedFileSize);
        MappedFile mappedFile = null;

        //创建mappedFile
        if (this.allocateMappedFileService != null) {
            mappedFile = this.allocateMappedFileService.putRequestAndReturnMappedFile(nextFilePath,
                nextNextFilePath, this.mappedFileSize);
        } else {
            try {
                mappedFile = new MappedFile(nextFilePath, this.mappedFileSize);
            } catch (IOException e) {
                log.error("create mappedFile exception", e);
            }
        }

        if (mappedFile != null) {
            if (this.mappedFiles.isEmpty()) {
                mappedFile.setFirstCreateInQueue(true);
            }
            //维护mappedFiles列表，这是个mappedFile的列表缓存
            this.mappedFiles.add(mappedFile);
        }

        return mappedFile;
    }

    return mappedFileLast;
}
```

在上面的方法中，默认是先获取最新的MappedFile，如果没有或者满了，那么就创建一个新的MappedFile并加入MappedFiles缓存中，方便下次获取最新的MappedFile。

在得到MappedFile之后，在6.4.2介绍的`CommitLog#putMessage`方法中就会通过`MappedFile#appendMessage`方法进行消息追加保存，我们这里跟进去详细看一下：

```java
public AppendMessageResult appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb) {
    return appendMessagesInner(msg, cb);
}
```

此方法会调用appendMessagesInner，我们继续跟进：

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    assert messageExt != null;
    assert cb != null;
    //获取上一个消息的写入位置 => 当前消息的位置
    int currentPos = this.wrotePosition.get();

    if (currentPos < this.fileSize) {
        //获取byteBuffer，如果二级缓存writeBuffer开始则获取二级缓存的切片，否则获取映射文件的切片
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        //设置byteBuffer的写指针的位置position
        byteBuffer.position(currentPos);
        AppendMessageResult result;
        if (messageExt instanceof MessageExtBrokerInner) {
            //调用回调函数，进行消息追加保存
            //参数分别是，文件的起始偏移量、我们的byteBuffer映射对象、该对象剩余空间（fileSize - currentPos）、以及消息的封装messageExt
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {
            //批量消息保存
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
        //更新消息写入位置
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```

此方法我已经标注好了注释，我们可以看到最终是通过我们传入的回调函数进行消息的保存，我们可以跟进doAppend方法，此方法最终会组装commitLog并写入如下图：

![image-20230405212212682](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230405212212682.png)

#### ConsumerQueue写入

我们从6.4.2继续分析，RocketMQ实际上会通过ReputMessageService后台服务类检测新消息的到来，并分发给不同的服务去处理消费分区索引和哈希索引。这里我们关注消费分区索引，ReputMessageService会将消息分发给CommitLogDispatcherBuildConsumeQueue类，在dispatch方法中会进行消费分区索引的处理，我们现在跟进此方法：

```java
class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {

    @Override
    public void dispatch(DispatchRequest request) {
        final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
        switch (tranType) {
            case MessageSysFlag.TRANSACTION_NOT_TYPE:
            case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                DefaultMessageStore.this.putMessagePositionInfo(request);
                break;
            case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
            case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                break;
        }
    }
}
```

我们可以看到对于普通消息和提交的事务消息将会去创建索引，我们跟进putMessagePositionInfo方法：

```java
public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
    //查找ConsumeQueue
    ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());
    //保存消费分区索引
    cq.putMessagePositionInfoWrapper(dispatchRequest);
}
```

这个方法先查出来对应的ConsumeQueue，然后通过这个ConsumeQueue去保存消费分区索引，我们先看一下如何查询ConsumeQueue，源码如下：

```java
public ConsumeQueue findConsumeQueue(String topic, int queueId) {
    //根据主题获取ConsumeQueue信息的Map
    ConcurrentMap<Integer, ConsumeQueue> map = consumeQueueTable.get(topic);
    if (null == map) {
        //如果没有当前主题的ConsumeQueue信息Map，那么将在这里维护
        ConcurrentMap<Integer, ConsumeQueue> newMap = new ConcurrentHashMap<Integer, ConsumeQueue>(128);
        ConcurrentMap<Integer, ConsumeQueue> oldMap = consumeQueueTable.putIfAbsent(topic, newMap);
        if (oldMap != null) {
            map = oldMap;
        } else {
            map = newMap;
        }
    }

    //根据queueId获取ConsumeQueue信息
    ConsumeQueue logic = map.get(queueId);
    if (null == logic) {
        //如果ConsumeQueue信息为空，则创建
        ConsumeQueue newLogic = new ConsumeQueue(
            topic,
            queueId,
            //获取ConsumeQueue存储路径：{user.home}/store
            StorePathConfigHelper.getStorePathConsumeQueue(this.messageStoreConfig.getStorePathRootDir()),
            this.getMessageStoreConfig().getMappedFileSizeConsumeQueue(),
            this);
        ConsumeQueue oldLogic = map.putIfAbsent(queueId, newLogic);
        if (oldLogic != null) {
            logic = oldLogic;
        } else {
            logic = newLogic;
        }
    }

    return logic;
}
```

实际上这个RocketMQ维护了一个内部缓存consumeQueueTable，然后会根据topic和queueId查找缓存的ConsumeQueue对象，如果没有就新建一个，最后返回这个ConsumeQueue对象。

在拿到这个对象后就会调用这个对象的putMessagePositionInfoWrapper方法将分发过来的包装好的dispatchRequest进行处理并保存消费分区索引，我们跟进此方法，在此方法中主要是调用了putMessagePositionInfo方法，将消息进行了保存，如下图：

![image-20230406100318583](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406100318583.png)

在putMessagePositionInfo最终通过`mappedFile.appendMessage`将消息分区索引写入mmap映射缓存。

#### IndexFile写入

我们从6.4.2继续分析，这里我们关注哈希索引，也就是IndexFile文件的处理，ReputMessageService会将消息分发给CommitLogDispatcherBuildIndex类，在dispatch方法中会进行消费分区索引的处理，下面我们跟进此方法：

```java
class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {

    @Override
    public void dispatch(DispatchRequest request) {
        if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
            DefaultMessageStore.this.indexService.buildIndex(request);
        }
    }
}
```

我们跟进buildIndex方法：

```java
public void buildIndex(DispatchRequest req) {
    //获取或创建IndexFile
    IndexFile indexFile = retryGetAndCreateIndexFile();
    if (indexFile != null) {
        long endPhyOffset = indexFile.getEndPhyOffset();
        //从请求中获取topic和keys
        DispatchRequest msg = req;
        String topic = msg.getTopic();
        String keys = msg.getKeys();
        if (msg.getCommitLogOffset() < endPhyOffset) {
            return;
        }
        //省略部分代码。。。
        if (req.getUniqKey() != null) {
            //消息是否设置了UNIQ_KEY属性，如果设置了就通过UNIQ_KEY构建哈希索引的Key
            //通过putKey保存indexFile
            indexFile = putKey(indexFile, msg, buildKey(topic, req.getUniqKey()));
            if (indexFile == null) {
                log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
                return;
            }
        }

        if (keys != null && keys.length() > 0) {
            String[] keyset = keys.split(MessageConst.KEY_SEPARATOR);
            for (int i = 0; i < keyset.length; i++) {
                String key = keyset[i];
                if (key.length() > 0) {
                    //默认通过topic和key构建哈希索引的Key
                    //通过putKey保存indexFile
                    indexFile = putKey(indexFile, msg, buildKey(topic, key));
                    if (indexFile == null) {
                        log.error("putKey error commitlog {} uniqkey {}", req.getCommitLogOffset(), req.getUniqKey());
                        return;
                    }
                }
            }
        }
    } else {
        log.error("build index error, stop building index");
    }
}
```

在此方法中最终通过putKey方法将哈希索引写入IndexFile，我们跟进此方法，此方法最终也是通过mappedByteBuffer映射缓存进行了保存，如下图

![image-20230406101332119](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406101332119.png)

### 6.4.4 Broker文件存储

在上一节，我们了解了RocketMQ会将消息写入到消息的对应的映射ByteBuffer中，由于这个ByteBuffer是mmap创建的文件映射缓存，所以实际上消息这时已经写入到了操作系统的page cahce中，但是要注意此时还没有写入到磁盘中。

是因为实际上mmap会将文件复制到内存中，我们操作时可以直接操作内存，达到操作文件的目的，但是实际上当把数据写入到文件中后实际上数据都在操作系统的内存中（这里实际上是将将PTE中的dirty bit设置为1，表示这一页是脏页），并没有刷到磁盘上，如果操作系统挂了，数据就丢失了。

那么如何保证数据不丢失呢？操作系统有两种方式保证数据刷盘，一是应用程序可以调用fsync这个系统调用来强制刷盘；二是操作系统有后台线程，定时刷盘。因为频繁调用fsync会影响性能，需要在性能和可靠性之间进行权衡。

下面我们了解一下定时刷盘：

定时刷盘有两种情况：脏页太多、脏页太久。刷新的策略是：首先，如果 pagecache 太多，到达阈值 dirty_background_ratio ，5%，会刷盘；其次，如果一个脏页的 pagecache 存在的时间太长，到达阈值dirty_expire_centisecs ，也会刷盘。

**系统一般在下面三种情况下回写dirty页:**

1. 定时方式: 定时回写是基于这样的原则:/proc/sys/vm/dirty_writeback_centisecs的值表示多长时间会启动回写线程,由这个定时启动的回写线程只回写在内存中为dirty时间超过(/proc/sys/vm/dirty_expire_centisecs / 100)秒的页(这个值默认是3000,也就是30秒),一般情况下dirty_writeback_centisecs的值是500,也就是5秒,所以默认情况下系统会5秒钟启动一次回写线程,把dirty时间超过30秒的页回写,要注意的是,这种方式启动的回写线程只回写超时的dirty页，不会回写没超时的dirty页,可以通过修改/proc中的这两个值，细节查看内核函数wb_kupdate。

2. 内存不足的时候: 这时并不将所有的dirty页写到磁盘,而是每次写大概1024个页面,直到空闲页面满足需求为止

3. 写操作时发现脏页超过一定比例: 当脏页占系统内存的比例过/proc/sys/vm/dirty_background_ratio 的时候,write系统调用会唤醒pdflush回写dirty page,直到脏页比例低于/proc/sys/vm/dirty_background_ratio,但write系统调用不会被阻塞,立即返回. 当脏页占系统内存的比例超/proc/sys/vm/dirty_ratio的时候, write系统调用会被被阻塞,主动回写dirty page,直到脏页比例低于/proc/sys/vm/dirty_ratio

这些都由 Linux 内核的后台线程执行。相关的控制参数有（以下参数来自我的CentOS7.5的云服务器）：

~~~sh
$ sysctl -a | grep dirty
vm.dirty_background_bytes = 0
vm.dirty_background_ratio = 10
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_ratio = 30
vm.dirty_writeback_centisecs = 500
~~~

- dirty_writeback_centisecs 表示多久唤醒一次刷新脏页的后台pdflush线程。这里的500表示５秒唤醒一次。指定多长时间清理脏数据的进程会唤醒一次，然后检查是否有缓存需要清理。

- dirty_expire_centisecs 表示脏数据多久会被刷新到磁盘上。这里的3000表示 30秒。指定脏数据能存活的时间。

- dirty_background_ratio 表示当脏页占总内存的的百分比超过这个值时，后台线程开始刷新脏页。

  比如内存大小为10G，配置该项值为90，意思是可以有10G*90%=9G的脏数据待在内存，超过9G才会由后台进程来清理（写入磁盘）

- dirty_ratio 当脏页占用的内存百分比超过此值时，**内核会阻塞掉写操作**，并开始刷新脏页。可以用脏数据填充的最大系统内存量，当系统到达此点时，必须将所有脏数据提交到磁盘，同时所有新的I/O块都会被阻塞，直到脏数据被写入磁盘。这通常是长I/O卡顿的原因，但这也是保证内存中不会存在过量脏数据的保护机制。


> 关于Linux的内核刷盘参数，作为Java开发人员只需要了解原理即可，如果想调整参数最好还是由专业的linux运维人员操作。

### 6.4.5 Broker 消息存储整理流程

根据上面的源码的学习，下面给出我对Broker 消息存储的整体的流程整理：

![brokersavemsg20230404161300](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/brokersavemsg20230404161300.jpg)

## 6.5 CommitLog的源码分析

> 在6.4一节中，详细学习了RocketMQ的消息存储的整体流程，下面6.5-6.7节将对三种文件CommitLog、ConsumeQueue和IndexFile的相关源码进行更细致的研读，了解一下RocketMQ的文件存储的源码细节。

### 6.5.1 CommitLog成员与方法

我们先来看一下CommitLog的核心成员和构造方法，这些成员会在消息存储的流程中使用到，这里先熟悉了解一下：

```java
public class CommitLog {
    //文件队列，通过这个来获取MappedFile
    protected final MappedFileQueue mappedFileQueue;
    //文件映射核心类
    protected final DefaultMessageStore defaultMessageStore;
    //刷盘线程
    private final FlushCommitLogService flushCommitLogService;

    //If TransientStorePool enabled, we must flush message to FileChannel at fixed periods
    //专用缓存（或者说是二级缓存）writebuffer提交线程
    private final FlushCommitLogService commitLogService;
    //追加保存消息的回调方法
    private final AppendMessageCallback appendMessageCallback;
    //topic对应的偏移量缓存
    protected HashMap<String/* topic-queueid */, Long/* offset */> topicQueueTable = new HashMap<String, Long>(1024);
    private final ThreadLocal<MessageExtBatchEncoder> batchEncoderThreadLocal;
    protected volatile long confirmOffset = -1L;

    private volatile long beginTimeInLock = 0;
    //锁成员
    protected final PutMessageLock putMessageLock;
    
    
    //构造函数
    public CommitLog(final DefaultMessageStore defaultMessageStore) {
        //初始化mappedFileQueue
        this.mappedFileQueue = new MappedFileQueue(defaultMessageStore.getMessageStoreConfig().getStorePathCommitLog(),
                                                   defaultMessageStore.getMessageStoreConfig().getMappedFileSizeCommitLog(), defaultMessageStore.getAllocateMappedFileService());
        //初始化defaultMessageStore
        this.defaultMessageStore = defaultMessageStore;

        //初始化刷盘线程
        if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            //GroupCommitService是同步刷盘等待线程
            this.flushCommitLogService = new GroupCommitService();
        } else {
            //FlushRealTimeService异步刷盘线程
            this.flushCommitLogService = new FlushRealTimeService();
        }
		//CommitRealTimeService特殊的，异步提交+刷盘，包含提交和异步刷盘逻辑，负责提交专用缓存（二级缓存），然后刷盘
        this.commitLogService = new CommitRealTimeService();
		//初始化回调方法
        this.appendMessageCallback = new DefaultAppendMessageCallback(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
        batchEncoderThreadLocal = new ThreadLocal<MessageExtBatchEncoder>() {
            @Override
            protected MessageExtBatchEncoder initialValue() {
                return new MessageExtBatchEncoder(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
            }
        };
        //初始化锁
        this.putMessageLock = defaultMessageStore.getMessageStoreConfig().isUseReentrantLockWhenPutMessage() ? new PutMessageReentrantLock() : new PutMessageSpinLock();

    }
    //省略其余代码。。。。。。
}
```

### 6.5.2 RocketMQ的定时任务原理与线程设计

下面我们来详细了解RocketMQ的刷盘线程及其父线程ServiceThread的设计，我们这里结合GroupCommitService同步刷盘等待线程和ServiceThread来看一下。

首先我们先来看一下ServiceThread，这个类是RocketMQ的服务的抽象父类，它继承了Runnable接口，同时将run方法下放到子类去实现。ServiceThread内部维护了一个线程，当调用start方法时，会初始化并将自己传入线程中，然后启动这个线程，这时就会去执行子类中的run方法。我们看源码：

```java
public abstract class ServiceThread implements Runnable {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.COMMON_LOGGER_NAME);

    //内部维护的线程
    private Thread thread;
	//是否是后台线程
    protected boolean isDaemon = false;
    protected volatile boolean stopped = false;
    
    //Make it able to restart the thread
    private final AtomicBoolean started = new AtomicBoolean(false);

    //省略其余代码。。。。。。

    public void start() {
        log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
        if (!started.compareAndSet(false, true)) {
            return;
        }
        stopped = false;
        //新建线程，并传入自己
        this.thread = new Thread(this, getServiceName());
        this.thread.setDaemon(isDaemon);
        //启动线程
        this.thread.start();
    }
    //ServiceThread没有run方法，run方法需要子类自行实现
}
```

在ServiceThread中最重要的两个模板方法就是wakeup和waitForRunning。我们先来看一下源码：

```java
//这时RocketMQ改造过的CountDownLatch，在原有的CountDownLatch上加了一个reset的特性
protected final CountDownLatch2 waitPoint = new CountDownLatch2(1);

protected volatile AtomicBoolean hasNotified = new AtomicBoolean(false);
public void wakeup() {
    //判断标志位没有被设置为false，即判断当前是否已经被通知过了
    if (hasNotified.compareAndSet(false, true)) {
        //如果没被通知过则调用countDown方法，由于只有一个计数，所以会唤醒下面的waitForRunning方法
        waitPoint.countDown(); // notify
    }
}

protected void waitForRunning(long interval) {
    //判断标志位没有被设置为true
    if (hasNotified.compareAndSet(true, false)) {
        //如果标志位被设置为true，说明线程被唤醒，这是会调用onWaitEnd模板方法
        this.onWaitEnd();
        return;
    }

    //如果线程未被唤醒，则调用reset方法重置计数器
    //entry to wait，重置CountDownLatch2，这里是将计数重置为1
    waitPoint.reset();

    try {
        //调用await方法阻塞等待
        waitPoint.await(interval, TimeUnit.MILLISECONDS);
    } catch (InterruptedException e) {
        log.error("Interrupted", e);
    } finally {
        hasNotified.set(false);
        this.onWaitEnd();
    }
}
```

以上就是ServiceThread的关键源码。

> 这里我们看一下CountDownLatch2这个RocketMQ改造过的CountDownLatch，实际上它在原有的CountDownLatch上加了一个reset的特性。这个reset方法本质上就是将计数器计数变回最开始的计数，在这个场景下就是1.
>
> ```java
> protected void reset() {
>     setState(startCount);
> }
> ```
>
> 我们下面看一下这个场景：
>
> 在waitForRunning方法中，如果标志位没有被设置为true，说明还没有被唤醒，这时会重置计数器，内部只是简单的将CountDownLatch的计数器重置，也就是说没有走AQS去唤醒所有的等待线程。这个操作在waitForRunning的使用场景下是可以的，但是在其他场景下是不可以的。
>
> 因为在waitForRunning中不会存在并发问题，只有线程本身会调用这个方法，这时等待队列中其实是没有线程的，所以这时可以简单的将状态重置。也就是说这个状态并不是共享的、可变的，并没有并发安全问题。但是正常的CountDownLatch的使用场景中需要唤醒其他的在等待队列中的线程，所以不可以重置状态。

下面我们结合GroupCommitService同步刷盘线程来看一下，我们来到这个子类的run方法：

```java
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            //调用waitForRunning方法，然后调用doCommit
            //实际上就是每隔10ms执行一次doCommit方法
            this.waitForRunning(10);
            this.doCommit();
        } catch (Exception e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }

    //下面代码可忽略，省略部分代码。。。。。。
}
```

实际上大部分的子类都是都是类似的工作原理，还比如ReputMessageService等等。通过这种设计可以使线程周期性的执行某个任务，同时还拥有了紧急执行该任务的能力。

以上就是RocketMQ定时任务原理与线程设计，在Broker中比如刷盘线程中使用的非常多。

### 6.5.3 CommitLog消息写入的流程

在6.4.2中，我们只关注了消息写入的核心流程，下面我们详细的了解一下`CommitLog#putMessage`方法。

#### 消息预处理

首先此方法会根据根据条件设置延迟消息。如果是非事务消息或者事务提交消息，就判断是否设置了延迟时间，从而去设置延迟级别：设置延迟消息的topic和queueId，然后备份真正的topic和queueId。代码如下图:

![image-20230406160756632](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406160756632.png)

#### 获取映射文件

然后，通过映射文件队列获取或创建最后一个映射文件：

![image-20230406161049693](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406161049693.png)

#### 加锁并写入消息

接下来会上锁，限制同一时间只有一个线程进行数据的put工作。这里有spinLock和ReentranLock两种实现，默认使用自旋锁：

```java
//默认useReentrantLockWhenPutMessage = false;
this.putMessageLock = defaultMessageStore.getMessageStoreConfig().isUseReentrantLockWhenPutMessage() ? new PutMessageReentrantLock() : new PutMessageSpinLock();
```

接下来就会写入消息，会调用appendMessage 方法进行消息的写入，最终在回调方法内进行消息写入。

> 再写入过程中会维护消息的起始全局偏移量wroteOffset（文件名（文件的第一个消息的位置）+ 内存相对位置（mappedFile的写入到的位置））。详情见下图的代码注释，如果看不清可以看下面的代码：
>
> ```java
> public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
>     assert messageExt != null;
>     assert cb != null;
>     //获取上一个消息的写入位置 => 当前消息的位置
>     int currentPos = this.wrotePosition.get();
> 
>     if (currentPos < this.fileSize) {
>         //获取byteBuffer，如果二级缓存writeBuffer开始则获取二级缓存的切片，否则获取映射文件的切片
>         ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
>         //设置byteBuffer的position，即设置这条新消息的开始写入的位置
>         byteBuffer.position(currentPos);
>         AppendMessageResult result;
>         if (messageExt instanceof MessageExtBrokerInner) {
>             //调用回调函数，进行消息追加保存
>             result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
>         } else if (messageExt instanceof MessageExtBatch) {
>             //批量消息保存
>             result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
>         } else {
>             return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
>         }
>         //更新消息写入位置
>         this.wrotePosition.addAndGet(result.getWroteBytes());
>         this.storeTimestamp = result.getStoreTimestamp();
>         return result;
>     }
>     //省略日志代码
>     return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
> }
> ```

流程如下图：

![image-20230406161852027](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406161852027.png)

在写入消息时如果文件剩余长度不够就返回不够的结果在doAppend方法中，如下图

![image-20230406162414266](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406162414266.png)

然后在putMessage中创建文件重新写入，代码如下：

![image-20230406162451723](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406162451723.png)

#### 处理消息刷盘

接下来消息已经处理完了，就会解锁，然后进行消息刷盘处理：

![image-20230406162617317](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230406162617317.png)

接下来我们来看一下刷盘的处理流程：

```java
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    // Synchronization flush 同步刷盘
    if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
        final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
        if (messageExt.isWaitStoreMsgOK()) {
            //消息的物理存储位置+当前消息的大小 = 目标刷盘位置
            GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
            //提交刷盘请求
            service.putRequest(request);
            //waitForFlush方法中会调用countDownLatch#await方法将当前线程阻塞，等待刷盘完成
            boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
            if (!flushOK) {
                log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                    + " client address: " + messageExt.getBornHostString());
                putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
            }
        } else {
            service.wakeup();
        }
    }
    // Asynchronous flush 异步刷盘
    else {
        //判断是否开启了二级缓存writeBuffer，默认未开启
        if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
            //开启了就唤醒异步刷盘线程
            flushCommitLogService.wakeup();
        } else {
            //否则就开启专用二级缓存的刷盘线程
            commitLogService.wakeup();
        }
    }
}
```

> 在上面的方法中构建同步刷盘请求GroupCommitRequest时传入的参数是从doAppand方法来的，如下图：
>
> ![image-20230407092422129](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230407092422129.png)

我们先来看一下同步刷盘的流程，首先同步刷盘会创建一个同步刷盘的请求GroupCommitRequest，然后调用同步刷盘服务GroupCommitService的putRequest方法将刷盘请求提交。我们来看一看这个同步刷盘服务，如下图：

![image-20230703160647885](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230703160647885.png)

这个类是ServiceThread类的实现类（代码如下），这个类在6.5.2学习过，作用是周期性的执行某个任务，同时还拥有了紧急执行该任务的能力。

```java
abstract class FlushCommitLogService extends ServiceThread {
    protected static final int RETRY_TIMES_OVER = 10;
}
```

刷盘服务GroupCommitService中维护了两个队列，这两个队列形成了特殊的生产者消费者模式，下面我们从run方法来看一下这两个队列的工作方式，因为这个类是ServiceThread类的实现类，所以我们来到run方法：

```java
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            this.waitForRunning(10);
            this.doCommit();
        } catch (Exception e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }

    // Under normal circumstances shutdown, wait for the arrival of the
    // request, and then flush
    try {
        Thread.sleep(10);
    } catch (InterruptedException e) {
        CommitLog.log.warn("GroupCommitService Exception, ", e);
    }

    synchronized (this) {
        this.swapRequests();
    }

    this.doCommit();

    CommitLog.log.info(this.getServiceName() + " service end");
}
```

可以看到GroupCommitService每10ms执行一次doCommit方法。我们进入这个方法：

```java
private void doCommit() {
    //上锁
    synchronized (this.requestsRead) {
        if (!this.requestsRead.isEmpty()) {
            //遍历读队列
            for (GroupCommitRequest req : this.requestsRead) {
                // There may be a message in the next file, so a maximum of
                // two times the flush
                boolean flushOK = false;
                //下一个文件中可能有消息，所以最多刷新两次
                for (int i = 0; i < 2 && !flushOK; i++) {
                    //判断是否完成了刷盘
                    flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();

                    if (!flushOK) {
                        //刷盘，最终调用fileChannel.force进行刷盘
                        CommitLog.this.mappedFileQueue.flush(0);
                    }
                }
                //唤醒客户端线程，这里是调用countDownLatch#countDown方法唤醒handleDiskFlush方法中的waitForFlush阻塞的线程。
                req.wakeupCustomer(flushOK);
            }

            long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
            if (storeTimestamp > 0) {
                CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
            }

            this.requestsRead.clear();
        } else {
            // Because of individual messages is set to not sync flush, it
            // will come to this process
            CommitLog.this.mappedFileQueue.flush(0);
        }
    }
}
```

在doCommit中我们可以看到会将requestsRead上锁，然后将requestsRead队列中的所有刷盘请求拿出来进行刷盘。为什么这里用的是requestsRead队列，而我们将请求放入requestsWrite队列呢？实际上RocketMQ这里用了读写分离的设计方式，在调用doCommit方法之前会先调用swapRequests方法，具体调用流程如下图：

![image-20230705093325802](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230705093325802.png)

```java
private void swapRequests() {
    List<GroupCommitRequest> tmp = this.requestsWrite;
    this.requestsWrite = this.requestsRead;
    this.requestsRead = tmp;
}
```

这里在doCommit方法前会将读写两个队列进行交换，通过这种方式可以实现RocketMQ的读写分离GroupCommitService 的刷盘操作是同步的，刷盘期间仍然会有大量的刷盘请求被提交进来，拆分成两个读写列表，请求提交到写列表，刷盘时处理读列表，刷盘结束交换列表，循环往复，两者可以同时进行。同时也可以减少锁的竞争。

我们继续看doCommit方法，在将读队列上锁后从交换过的读队列拿取刷盘请求，然后进行刷盘，刷盘后，唤醒handleDiskFlush中的waitForFlush方法阻塞的线程。同步刷盘的流程如下：

![image-20230706132020404](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230706132020404.png)

以上就是同步刷盘的源码和流程，下面我们来看异步的刷盘流程和源码，我们回到handleDiskFlush代码如下：

![image-20230706135959798](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230706135959798.png)

如果是异步刷盘，那么就判断是否开启了二级（专用）缓存，根据此配置唤醒异步刷盘线程或者唤醒专用二级缓存的刷盘线程。

异步刷盘线程和二级缓存的刷盘线程都是抽象类FlushCommitLogService的实现类，FlushCommitLogService的代码如下：

```java
abstract class FlushCommitLogService extends ServiceThread {
    protected static final int RETRY_TIMES_OVER = 10;
}
```

我们先看异步刷盘线程FlushRealTimeService，我们跟进run方法：

```java
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        //是否定时刷新，默认为实时
        boolean flushCommitLogTimed = CommitLog.this.defaultMessageStore.getMessageStoreConfig().isFlushCommitLogTimed();
        //CommitLog刷新间隔  将数据刷新到磁盘 默认500ms
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushIntervalCommitLog();
        //在刷新CommitLog时要刷新多少页 默认4
        int flushPhysicQueueLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogLeastPages();

        int flushPhysicQueueThoroughInterval =
            CommitLog.this.defaultMessageStore.getMessageStoreConfig().getFlushCommitLogThoroughInterval();

        boolean printFlushProgress = false;

        // Print flush progress
        long currentTimeMillis = System.currentTimeMillis();
        if (currentTimeMillis >= (this.lastFlushTimestamp + flushPhysicQueueThoroughInterval)) {
            this.lastFlushTimestamp = currentTimeMillis;
            flushPhysicQueueLeastPages = 0;
            printFlushProgress = (printTimes++ % 10) == 0;
        }

        try {
            if (flushCommitLogTimed) {
                //定时刷新就休眠当前线程
                Thread.sleep(interval);
            } else {
                //实时刷新就调用waitForRunning方法，因为此方法可以实时唤醒
                this.waitForRunning(interval);
            }

            if (printFlushProgress) {
                this.printFlushProgress();
            }

            long begin = System.currentTimeMillis();
            //刷盘
            CommitLog.this.mappedFileQueue.flush(flushPhysicQueueLeastPages);
            long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
            if (storeTimestamp > 0) {
                CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
            }
            long past = System.currentTimeMillis() - begin;
            if (past > 500) {
                log.info("Flush data to disk costs {} ms", past);
            }
        } catch (Throwable e) {
            CommitLog.log.warn(this.getServiceName() + " service has exception. ", e);
            this.printFlushProgress();
        }
    }

    // Normal shutdown, to ensure that all the flush before exit
    //正常关闭时 需要刷完盘在关闭
    boolean result = false;
    for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.flush(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
    }

    this.printFlushProgress();

    CommitLog.log.info(this.getServiceName() + " service end");
}
```

从上面的源码我们可以看到，默认异步刷盘是每500ms执行一次，且由于继承了ServiceThread同时拥有了可以被随时唤醒去执行刷盘的能力。此方法中刷盘也是使用的mappedFileQueue.flush方法。以上就是异步刷盘的源码流程。

下面我们再看看二级缓存CommitRealTimeService的刷盘线程的run方法：

```java
@Override
public void run() {
    CommitLog.log.info(this.getServiceName() + " service started");
    while (!this.isStopped()) {
        //CommitLog刷新间隔  将数据刷新到磁盘 默认200ms
        int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();
        //在刷新CommitLog时要刷新多少页 默认4
        int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();

        int commitDataThoroughInterval =
            CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();

        long begin = System.currentTimeMillis();
        if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
            this.lastCommitTimestamp = begin;
            commitDataLeastPages = 0;
        }

        try {
            //提交二级缓存
            boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
            long end = System.currentTimeMillis();
            if (!result) {
                this.lastCommitTimestamp = end; // result = false means some data committed.
                //now wake up flush thread.
                //唤醒异步刷盘线程 去进行刷盘
                flushCommitLogService.wakeup();
            }

            if (end - begin > 500) {
                log.info("Commit data to file costs {} ms", end - begin);
            }
            this.waitForRunning(interval);
        } catch (Throwable e) {
            CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
        }
    }

    boolean result = false;
    for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
        result = CommitLog.this.mappedFileQueue.commit(0);
        CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
    }
    CommitLog.log.info(this.getServiceName() + " service end");
}
```

这里的逻辑与异步刷盘类似，此类是每200ms执行一次，这里是先将二级缓存提交到文件系统缓存中，然后唤醒异步刷盘线程，将其刷盘到磁盘中。

以上就是消息刷盘的源码分析。处理完消息刷盘后就会处理消息的主从复制，

#### 处理消息的主从复制

我们进入到handleHA方法中，此方法源码如下：

```java
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
    if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
        HAService service = this.defaultMessageStore.getHaService();
        if (messageExt.isWaitStoreMsgOK()) {
            // Determine whether to wait
            if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                //构建请求，这里通过wroteOffset和当前消息的长度计算目标的刷盘位置
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                //然后将请求放入到HAService的请求队列中
                service.putRequest(request);
                service.getWaitNotifyObject().wakeupAll();
                //等待HAService执行完毕
                boolean flushOK =
                    request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                              + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                    //如果同步失败，设置消息存储结果为刷盘超时
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                }
            }
            // Slave problem
            else {
                // Tell the producer, slave not available
                putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
            }
        }
    }

}
```

在此方法中首先获取主从复制的服务 HAService，然后判断消息是否满足waitStoreMsgOk的标志，如果满足则构建请求GroupCommitRequest，然后将这个请求放入HAService 线程中的写入队列里，然后调用waitForFlush方法等待HAService 线程执行完毕（内部是CountDownLatch），如果没刷盘成功，就设置一个同步从节点失败的状态。 

### 6.5.4 同步刷盘与异步刷盘流程总结todo







 

MPSC无锁编程：

同步刷盘流程：

我们可以在BrokerController中找到发送消息的请求码注册的地方，找到使用了sendMessageExecutor线程池（核心和最大线程数是相同的且默认值是1）。所以处理消息的线程在处理完消息后会走到数据刷盘的流程，进到handleDiskFlush方法。在同步刷盘的情况下这个线程会将刷盘请求放到刷盘线程的写队列中，然后就会等待刷盘完成。消息的业务处理线程虽然默认是1，但是实际上可以配置多个，所以这里可以看作是MPSC模式。业务处理线程向同步刷盘线程提交刷盘请求，并将请求加入到写队列，这里消息的业务处理线程是生产者且可以是多个，然后刷盘线程是消费者且只有一个。

刷盘线程在放入请求（putRequest）时，还会根据serviceThread 的标志位唤醒消费者，也就是唤醒唯一的线程（通过CountDownLatch2的countDown方法）--》

这时业务线程会使用CountDownLatch正常的闭锁去等待同步刷盘线程完成。//与此同时由于putRequest唤醒了消费者（刷盘线程），所以刷盘线程run中的waitForRunning就会结束等待（这里结束后会调用钩子方法，交换读写队列）。然后会执行doCommit方法 --》

> 实际上这里用了读写分离的思想，但是4.6操作读队列时加锁了，可以看一下后面的代码，看看是否优化了

doCommit方法：当读队列不为空，进行刷盘（因为等待完成后读写队列交换，所以请求都在读队列中）-》遍历读队列 -》因为每个请求都记录了下一个消息的起始地址（见上），同时mq还有一个flushWhere判断到底刷到哪里了，所以根据这两个变量进行判断当前请求的消息是否已经刷盘完成了，如果没有刷盘成功，就调用flush方法去刷盘 -》flush：执行刷盘，然后计算新的位置，并更新flushWhere -》最后唤醒业务线程通过CountDownLatch



异步刷盘流程：跟同步刷盘线程同理，异步刷盘线程也是在CommitLog的构造函数中初始化的

run流程：获取配置参数并检查 --》判断是否是定时刷盘，是的话刷一次要睡眠（sleep）默认500ms，否则 --》

调用waitForRunning，可以等待500ms或者被wakeup唤醒 --》刷盘 根据偏移量找到映射文件，然后执行刷盘（最终调用force方法去落盘），之后计算新的位置并更新flushWhere --》更新时间戳 



### 6.5.5 文件预热+内存锁定

mmap最大的劣势就是会占用内存，当操作系统内存数量达到低水位时，就会触发LRU内存回收，这时就有可能回收RocketMQ的映射内存，当再次存放内存时，就会触发page fault。所以RocketMQ有两种方式解决此问题，一是文件预热+内存锁定，二是使用二级缓存。

我们先来看看RocketMQ如何使用文件预热和内存锁定的。

首先内存锁定使用了mlock方法，可以传一段用户空间地址和内存大小，这样就可以锁定内存，操作系统在回收时就不会回收这段内存，源码如下：

```java
#org.apache.rocketmq.store.MappedFile#mlock
public void mlock() {
    final long beginTime = System.currentTimeMillis();
    final long address = ((DirectBuffer) (this.mappedByteBuffer)).address();
    Pointer pointer = new Pointer(address);
    {
        int ret = LibC.INSTANCE.mlock(pointer, new NativeLong(this.fileSize));
        log.info("mlock {} {} {} ret = {} time consuming = {}", address, this.fileName, this.fileSize, ret, System.currentTimeMillis() - beginTime);
    }

    {
        int ret = LibC.INSTANCE.madvise(pointer, new NativeLong(this.fileSize), LibC.MADV_WILLNEED);
        log.info("madvise {} {} {} ret = {} time consuming = {}", address, this.fileName, this.fileSize, ret, System.currentTimeMillis() - beginTime);
    }
}
```

而这个mlock会在warmMappedFile方法中调用的。warmMappedFile方法中会根据OS的页大小进行遍历，在循环中给每一页都存放一个字节，完成对应的内存分配，然后在调用mlock方法锁定分配的这段内存。warmMappedFile方法的源码如下：

```java
public void warmMappedFile(FlushDiskType type, int pages) {
    long beginTime = System.currentTimeMillis();
    ByteBuffer byteBuffer = this.mappedByteBuffer.slice();
    int flush = 0;
    long time = System.currentTimeMillis();
    //MappedFile.OS_PAGE_SIZE 默认4K操作系统一页的大小 
    for (int i = 0, j = 0; i < this.fileSize; i += MappedFile.OS_PAGE_SIZE, j++) {
        //每一页都放入一个字节，达到预分配的目的
        byteBuffer.put(i, (byte) 0);
        // force flush when flush disk type is sync
        //如果是同步刷盘就将预分配的这页刷入磁盘中
        if (type == FlushDiskType.SYNC_FLUSH) {
            if ((i / OS_PAGE_SIZE) - (flush / OS_PAGE_SIZE) >= pages) {
                flush = i;
                mappedByteBuffer.force();
            }
        }

        // prevent gc 
        //Thread.sleep(0)使用native方法进入安全点，加快GC频率，防止长时间的GC
        if (j % 1000 == 0) {
            log.info("j={}, costTime={}", j, System.currentTimeMillis() - time);
            time = System.currentTimeMillis();
            try {
                Thread.sleep(0);
            } catch (InterruptedException e) {
                log.error("Interrupted", e);
            }
        }
    }

    // force flush when prepare load finished
    if (type == FlushDiskType.SYNC_FLUSH) {
        log.info("mapped file warm-up done, force to disk, mappedFile={}, costTime={}",
                 this.getFileName(), System.currentTimeMillis() - beginTime);
        mappedByteBuffer.force();
    }
    log.info("mapped file warm-up done. mappedFile={}, costTime={}", this.getFileName(),
             System.currentTimeMillis() - beginTime);
	//锁定内存
    this.mlock();
}
```

此方法功能是对每一页都放入一个字节达到内存预分配的目的（操作系统的一页shi）

warmMappedFile是在mmapOperation中调用的，mmapOperation在创建好了映射文件后，在满足条件时就会调用warmMappedFile进行文件预热（预写入）。这里满足的条件是，只有当映射文件大小大于CommitLog文件大小（1G），并且预热选项被开启时（默认不开启），才会调用warmMappedFile进行文件预热。

mmapOperation是在AllocateMappedFileService的run方法中调用的，AllocateMappedFileService这个线程类似刷盘线程，也是一个MPSC的线程，他会不断的根据放入的分配文件的请求去分配文件调用mmapOperation。

那么这个分配文件的请求是调用putRequestAndReturnMappedFile方法放入的。这个方法会构建AllocateRequestnexReq和nextNextReq对象（也就是下个CommlitLog和下下个CommitLog文件），并把对象放入请求队列中，在run中就会获取需要创建的任务，并创建文件。

那么这个putRequestAndReturnMappedFile实际上就是在getLastMappedFile方法中获取的。只有CommitLog会开启这个AllocateMappedFileService服务。ConsumeQueue不会开启



### 6.5.5 消息写入的二级缓存

二级缓存writeBuffer

TransientStorePool，专用缓存池。初始化时默认申请5个G的直接内存，然后锁定这些内存，最后将内存块加到缓存池中。

> 使用时需要打开开关，缓存池大小也可以配置，transientStorePoolEnable = true 
>
> transientStorePoolSize = 1

使用时通过borrowBuffer借用缓存（这个方法只有MappedFile初始化时会被调用），使用完调用returnBuffer归还缓存（在MappedFile的commit中被调用）。

专用缓存写入：appendMessageInner，如果writeBuffer不为空，那么回调方法doAppend传入的就是 writeBuffer这个专用缓存

专用缓存提交和刷盘：在MappedFile中有两个变量，一是flushedPosition，表示文件系统刷入磁盘的位置；还有一个变量是wrotePosition，表示文件系统缓存写入位置或二级缓存写入位置，在getReadPosition中，如果二级缓存没有开启就会返回，否则返回committedPosition（提交位置，表示专用缓存提交到文件系统缓存中）

专用缓存是通过CommitRealTimeService完成提交工作的，包含提交和异步刷盘的逻辑。

> 在transientStorePoolEnable=true时使用这种方式。二级缓存还有一些配置：
>
> commitIntervalCommitLog =200//提交到FileChannel的时间间隔，默认0.2s
>
> commitCommitLogLeastPages=4//每次提交至少提交多少页默认四个
>
> commitCommitLogThoroughInterval =200 //提交完成的时间间隔，默认0.2s

CommitRealTimeService --》run -》定时执行 -调用mappedFileQueue.commit提交//然后唤醒异步刷盘线程去刷盘。

commit -》最终调用映射文件的commit方法 -》commit0 -》获取上次的提交位置和这次的写入位置，得到这次需要提交的位置。 -》提交：先创建切片，然后将指针定到上一次提交的位置，设置limit为写入位置，然后将这段数据通过fileChannel写入，最后更新committedPosition



### 6.5.7 番外 写入消息时的读写分离思想

读写分离

在mq中使用读写双缓存队列来实现读写分离，读写分离带来的好处主要是在内部的同步刷盘请求可以不用加锁，提高并发。读写分离之前，写写和写读是需要加锁的，但是读写分离之后，写写需要加锁，写读就不需要加锁了。无锁编程优化，mpsc+读写分离进行了优化。



SpinLock锁

mq默认使用自旋锁进行数据put工作的同步，可重入锁涉及的线程唤醒与阻塞涉及到CPU切换，所以在线程停顿时间短的情况下，自旋锁小号的CPU资源比阻塞要低得多。内部是CAS的简单实现



### 6.5.8 消息读取的流程

消息读取的流程

消费时，Consumer是从ConsumeQueue中拉取数据，拉去的顺序从旧到新，在文件表示每一个ConsumeQueue都是顺序读，然后拉取完ConsumeQueue是没有数据的，里面是有一个CommitLog的引用，所以需要再次拉取CommitLog。流程主要是：

NettyRemotingAbstract#run -》 processRequest#PullMessageProcessor（在消费者一侧是DefaultMQPushConsumer） -》DefaultMessageStore#getMessage -》 ConsumeQueue#getIndexBuffer

getMessage有三个参数比较重要，分别是queueOffset（分区偏移量），queueId（消息的分区Id），topic，此方法流程：校验 -》findConsumeQueue（根据Topic和queueId定位ConsumeQueue）-》校验 -》 getIndexBuffer通过刚才查出来的分区队列和偏移量，找到对应的消息的索引数据（从索引位置到分区文件的末尾）-》遍历消息过滤（Broker每次最大过滤数是800条） -》按根据TagsHash快速匹配，判断tagsCode是否存在在SubscriptionData的codeSet集合内 -》满足条件进一步操作，根据offsetPy偏移量/物理长度，去读取CommitLog的文件映射缓冲区，通过commitLog的getMessage方法：先找到对应的映射文件，然后找到对应的映射 -》过滤，根据消息属性进行过滤（这个是如果使用SQL的话，就这样过滤） -》满足要求，加入到最终的Result。 -》返回给消费者。



消息读取性能优化：

消息的读取是随机读取，但是CommitLog的读取整体来看还是有序的读，因为消息总是都最新的，只要还在最新的page cache中，还是可以充分的利用PageCache的。正常情况下，基本都可以命中。如果读取的消息没有命中pagecache，那么就会产生较多的随机读取，会影响性能。这是可以通过换更好的磁盘，比如ssd，然后选择合适的IO调度算法进行性能提升。比如ssd可以采用noop磁盘调度算法。

## ——ConsumeQueue的源码分析

```java
private final DefaultMessageStore defaultMessageStore;
//映射文件队列
private final MappedFileQueue mappedFileQueue;
//topic
private final String topic;
//消息的queueId
private final int queueId;
//指定大小的花村，因为一个记录的大小是20byte
private final ByteBuffer byteBufferIndex;
//保存的路径
private final String storePath;
//映射文件的大小
private final int mappedFileSize;
//最后一个消息对应的物理偏移量
private long maxPhysicOffset = -1;
//最小的逻辑偏移量 在ConsumeQueue中的最小偏移量
private volatile long minLogicOffset = 0;
//ConsumeQueue的扩展文件，保存一些次重要的信息，比如消息存储时间等
private ConsumeQueueExt consumeQueueExt = null;
```

load方法：查找文件，加载文件并保存进写时复制队列

recover方法：。。。



消费分区索引写入的原理：

broker收到消息后，首先在commitLog中进行消息存储。同时在broker启动时会启动一个ReputMessageService线程，1ms运行一次，该任务会构建ConsumeQueue数据。

ReputMessageService维持一个commitLog文件中的消息偏移量reputFromOffset，根据该偏移量读取对应的消息，然后将其位置信息追加到ConsumeQueue文件中，完成一条消息的索引插入，然后更新reputFromOffset变量。

在ReputMessageService#run方法中 -》doReput -》判断，计算消息偏移量reputFromOffset小于消息正文的偏移量 -》继续，进行索引分发。首先查询出要分发的消息 -》遍历消息-》构建请求（获取消息内容，如果是延迟消息则计算延迟时间替换tagscode字段，消息内容存入请求） -》执行分发调用doDispatch（会走到CommitLogDIspatcherBuildConsumeQueue#dispatch -》 DefaultMessageStore#putMessagePositionInfio -》ConsumeQueue#putMessagePositionInfoWrapper-》写入消息）



消费分区索引读取的原理

查询时，首先根据逻辑偏移量定位到数据块，然后根据块中的内容定位到消息在CommitLog的起始位置和tag信息。consumequeue文件以及在该文件中的数据块 = `逻辑偏移*20 % consumequeue（30万）`

getIndexBuffer  ： 见上

## ——index的源码分析

indexFile哈希索引：主要是根据消息的业务Key进行消息的检索

consumequeue是类似数组的顺序索引文件，以一个单调递增的int数字为索引，所以结构很简单，在indexFile中，以msgId或者消息的key作为索引key，是一个hash索引结构。index存储路径：`$HOME\store\index\${时间戳}`。注意indexFile文件命名是时间戳。

indexFile包含：lndexHeader、 Hash 槽、 Hash 条目。其中hash槽通过取余得到每一个hash值的槽位，而每个槽位上保存着索引值位置，而通过位置就可以找到对应的Hash条目。就可以拿到索引值了。

![image-20230210151112082](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230210151112082.png)

其中每个条目会有最后四个字节用来解决hash冲突，它是存放上一个该槽位的条目，这样就能形成一个链表结构。也就是说每一个冲突的hash条目都会加入链表的最开始，然后存储上一个加进来的条目。类似Java的HashMap解决冲突那样。其实Hash槽（soltTable）+  条目链表（revert entry LinkedList反向链表）形成的结构可以理解为Java的HashMap。（反向链表的设计是因为这样越新的消息越有可能会被查找）

solt存放着当前solt下的最新的index的序号，index中存储了当前slot下、当前index的前一个index序号，这就把slot下的所有index连起来了。

由于indexHeader ，solt，entry都是固定大小的，所以使用时是这样计算的：

第n个solt在indexFile中的起始位置：`40+(n-1)*4`

第n个entry在indexFile中的起始位置：`40+5000000*4+(n-1)*20`

跟index相关的类有两个：IndexFile和IndexService。IndexService是对多个IndexFile类的封装。也是Index文件的最外层的操作类。类似CommitLog和ConsumeQueue



hash索引写入流程原理：

ReputMessageService#run -》doReput -》DefaultMessageStore#doDispatch -》CommitLogDIspatcherBuildIndex#dispatch -》DefaultMessageStore#putMessagePosotionInfo  -》IndexService#buildIndex（这里会构建hash索引）  -》 indexService#putKey  -》IndexFile#putKey

indexService#putKey：通过hash算出槽位位置，然后构建条目对象（条目是从1递增的）



根据mes key和时间范围查询index的索引的流程：

根据MessageKey查消息：我们拿到消息的msgId，然后计算hash值，然后对500w取余计算对应的slot序号 -》计算出该slot的文件位置。最后读取slot值（Index序号）

注意：根据mesKey查找消息是，还需要传入时间范围。首先是因为indexFile文件名就是一个时间戳，所以能更精确的定位到具体的indexFile文件，缩小查找的范围，减少查找时间。  同时如果不是业务产生的meskey，二是由producer生产的那么就会有可能重复，所以输入时间会减少重复的概率。

DefaultMessageStore#queryMessage -》queryOffset -》遍历所有的indexFile 调用selectPhyOffset，将详细正文的偏移量保存进一个List集合（phyOffsets）中  -》在selectPhyOffset中，就是核心的查找流程 -》首先根据Hash值计算槽位 slotPos-》然后根据`40+(n-1)*4`计算出slot在文件中的位置absSlotPos-》然后读取slot的值slotValue这里就是entry的编号 -》然后根据`40+5000000*4+(n-1)*20`计算出entry在文件中位置 -》然后读取entry的所有元素值 -》进行时间范围校验，将key 的hash值和传入的时间范围与index的keyhash值以及timeDiff值进行对比-》如果满足就加入结果集 -》一直遍历（根据entry的preIndexNo可以找到上一个entry），直到slot的indexLinkedList结束

## ——RocketMQ文件映射总结

主要有MappedFileQueue和MappedFile。

写的时候，主要在appendMessageInner中，拿到MappedByteBuffer或writeBuffer写入

读的时候，主要在selectMappedBuffer中



MappedByteBuffer：

rocketmq主要通过MappedByteBuffer对文件进行读写。通过文件映射对文件进行操作。优缺点：

优点：减少了操作系统在内核缓冲区和用户缓冲区之间来回数据拷贝的性能开销，也减少了进行的上下文切换。

缺点：占用大量内存，且如果内存不足可能会触发操作系统的内存回收。

writeBuffer：

专用的二级写入缓存。TransientStorePool。rocketmq到哪都创建一个MappedByteBuffer的内存缓存池，用来临时存储数据，数据先写入这块缓存，然后由commit线程定时将数据从内存复制到与目标物理文件对应的内存映射中。



MappedFile高性能写原理：

如果开启了二级缓存（TransientStorePoolEnable=true），内容就先存储在对外内存（临时缓存池），然后通过Commit线程将数据提交到fileChannel中，再通过flush线程将filechannel文件缓存持久化到磁盘中。

如果没开启二级缓存（TransientStorePoolEnable=false），则内容先存储在mappedfile中，再通过flush线程将mappedFile的数据持久化到磁盘中。

 

MappedFile高性能读原理：

selectMappedBuffer方法：读的时候这个方法会接受第n个消息的起始地址，然后读的时候会做一个切片，切片从传的位置开始一直到末尾，如果没开启二级缓存末尾就是wrotePosition，如果开启了二级缓存那么末尾就是CommittedPosition。



文件定期扫描（10s）和清理策略：

跟kafka一样，commitLog的内容在消费后是不会删除的。这样有两个好处，一是可以被多个重复的consumer group重复消费，只要修改了消费者组就可以重头开始消费，每个消费者组维护自己的offset。另一个好处是支持消息回溯，随时可以搜索某消息。但是这样做会导致占用磁盘越来越多，所以rocketmq会将commitLog、consumequeue 的这些过期文件进行删除，默认是超过72小时的文件。

会通过线程调用DefaultMessageStore.this.cleanFilesPeriodically去清理，分两种情况：

一是timeup，每天凌晨4点去删除这些文件。二是spacefull，磁盘使用空间超过75%，会开始批量清理文件.第三种情况就是磁盘空间超过85%或90%就立即清理磁盘。每批删除会删除10个文件。

## ——RPC层的源码分析

基础架构和使用：

RPC协议只规定了client和Server之间的点对点调用流程，包括stub、通信协RPC消息解析等部分。在实际应用中，还要考虑服务的高可用、负载均衡等问题。很多RPC框架指的是能够完成RPC调用的解决方案，除了点对点的RPC协议实现外，还包含服务的发现于注销、提供服务的Server的负载均衡，服务的高可用等。RPC框架大致有两种方向：一种是服务治理，一种是跨语言调用。

seata=rpc框架+分布式事务   rocketmq=rpc框架+消息存储和转发业务处理

rocketmq的通讯框架主要在remoting这个模块，mq是基于netty的rpc框架。

mq官方模块图（IDEA：RemotingService）

有两大类，一类是服务端（NettyRemotingServer），一类是客户端（NettyRemotingClient），他们将通用的方法都抽取到基类里面（NettyRemotingAbstract），比如消息发送。同时为了能兼容其他技术，mq还将服务端和客户端业务进行了抽象成了两个角色（RemotingServer和RemotingClient）。最上层的接口是RemotingService。



消息协议设计与编解码

mq的通信协议格式主要分为四部分：1.消息总长度 int 4字节  2. 序列化类型和消息头长度 int 4字节 第一个字节序列化长度，后三个字节消息头长度 3. 消息头数据 4. 消息主题数据

消息的编解码分别在NettyEncoder和NettyDecoder -》RemotingCommand的encode和decode方法

编码 encode：

。。。

解码decode，基于netty内置的长度域解码器：

。。。

头部数据RomotingCommand对象本身

body数据不同消息有不同的类，以心跳为例 RegisterBrokerBody

RomotingCommand：

- code 请求操作码在RequestCode中   ----  应答响应码0成功，非0各种错误
- language 请求方实现的语言 ---- 应答方实现的语言
- version  请求方程序的版本  ---- 应答方程序的版本
- opaque 相当于requestId，在同一个连接上的不同请求标识码，与响应消息对应 ---- 应答方直接返回
- flag  区分是普通rpc还是onewayRpc  ----区分是普通rpc还是onewayRpc   （位运算用法各个位意义不同）
- remark  自定义文本信息   ---- 自定义文本信息
- extFields  请求自定义扩展信息   --- 响应自定义扩展信息

异步消息的流程和原理

请求和响应的映射 responseTable

客户端发送异步消息： invokeAsync  -》创建获取通道 -》调用invokeAsyncImpl -》获取请求id -》构建ResponseFuture（这里会传入回调函数onvokeCallback），根据请求id将ResponseFuture放入responseTable -》通过netty的channel发送请求数据 -》监听器回调方法中处理结果。

> ResponseFuture中有一个RemotingCommand成员，这个是预留给服务端回复过来的响应报文的。

服务端处理异步消息：serverHandler -》processMessageReceived -》请求就调用processRequestCommand -》根据请求码找到处理器和处理器对应的线程池的Pair对象 -》 构造target（Runnable） -》根据target构造任务并提交线程池 -》执行target，调用processRequest方法 -》processRequest返回响应报文RemotingCommand -》设置请求id（opaque属性）-》通过netty将响应写回客户端

客户端处理响应的过程：processMessageReceived -》响应就调用processResponseCommand -》根据响应报文的请求id，到responseTable 中找到responseFuture -》如果responseFuture 中有回调（也就是是否是异步消息），有就执行回调 -》executeInvokeCallback -》获取异步回调的执行线程池 -》提交异步回调的任务 -》调用executeInvokeCallback -》调用invokeCallback的operationComplete方法（用cas保证只执行一次）

客户端响应缓存堆积处理：

responseTable的本地缓存可能会出现堆积的问题：比如发送消息时如果遇到异常情况，或者服务端没有返回resposne或者response在网络中丢失。这时就需要一个定时任务去做responseTable的清理回收。在RocketMQ客户端/服务端启动时会启动一个定时任务（1s）检查responseTable中的responseFuture ，判断是否已经返回并进行相应的处理。将超时很久的responseFuture 从本地缓存中删掉 --》scanResponseTable



同步消息的流程和原理

客户端发送请求：invokeSync -》 invokeSyncImpl -》构造ResponseFuture时callback为null -》发送请求后会调用responseFuture .waitResponse方法，调用countdownlatch的await方法阻塞等待响应。-》最终删除responseFuture 。

服务端处理请求：略

客户端处理响应：在processResponseCommand方法中 -》如果responseFuture 中没有回调（也就是同步消息），调用responseFuture .putResponse()方法和responseFuture .release()方法 -》 在putResponse中会调用countdownlatch锁的countDonn方法。唤醒请求的线程。



单向消息的流程和原理

客户端发送请求：invokeOneway -》invokeOnewayImpl  -》通过通道写入请求 。注意：单向消息不缓存responseFuture

服务端处理请求：processRequestCommand 中如果是单向消息不做响应回复



多线程Reactor模型设计

1+N+M1+M2的多线程Reactor模型。连接事件处理线程池、IO事件处理线程池、基础RPC线程池、业务处理线程池。

1：NettyBoss_%d：Reactor监听线程，处理新连接（accept）事件 。即 boss线程池

N（默认3）：NettyServerEPOLLSelector%d%d：处理IO读写事件，即Reactor线程池

M1（默认8）：NettyServerCodecThread_%d：做基础的RPC处理（所有netty流水线上的处理），即消息的编解码，即Worker线程池

M2（不同请求是不同的线程池）：NRemotingExecutorThread_%d：业务处理线程池



rocketmq的线程池：

以sendMessageExecutor（消息存储处理的线程池）为例：参数为核心线程数=最大线程数 =x ；线程存活时间60s；sendThreadPoolQueue等待队列（阻塞队列，长度10000）；线程工厂（自增编号，设置后台线程）。

这里没设置拒绝策略，所以是默认拒绝策略，在使用线程池时会捕获默认策略抛出的异常。

在低性能场景下，rocketmq还是使用了快捷创建线程池。

## ——Consumer的源码分析

消费者消费以组为单位，一个组可以包含多个消费者，每个消费者组可以订阅多个主题。消费者组之间有集群和广播模式两种消费模式：集群模式：主题下的同一条消息只允许被组内其中一个消费者消费。（一个主题下一个分区只允许被一个消费者消费）广播模式：主题下的同一条消息被集群内所有消费者消费。（一个主题下一个分区会被所有的消费者消费）

对于每个分区，broker和consumer之间的消息传输也有两种模式，拉模式，消费者端主动拉取消息请求。推模式，消息到达服务器后，推送给消费者。

rocketmq实现了这两种模式：推模式DefaultMQPushConsumer；拉模式DefaultMQPullConsumer。

rocketmq推模式是基于拉模式实现的，一个任务拉去完成后开始拉取下一个任务。是一种长轮询的机制。



rocketmq长轮询push模式：

push的方式就是Server收到消息后，主动将消息推送给客户端。但是push主动的推送也存在一定的问题，比如加大Server端的工作量，并且可能会造成客户端消息堆积（因为客户端不是高性能组件）。rocketmq对push模式进行了长轮询改造，本质也是轮询但是对轮询进行了优化。也就是说实际上DefaultMQPushConsumer不是原始的push模式，而是使用了长轮询，兼顾了push和pull两种模式的优点。具体的优化策略：

客户端不断轮询，循环想消息服务端发送拉取请求，如果有消息broker就会返回。但是客户端并不是定时轮询，而是收到响应之后在轮询。对于broker来说，如果有消息会立即返回，如果没有消息，Broker会保持客户端的连接，在一定时间内（5s）如果有消息，就会利用这个连接返回消息。

那如果没有消息的话，broker后续有两种方式处理：

一是定时任务，每隔5s轮询检查一次消息是否可达，一旦有消息到达就挂起线程，再次检查是否是自己感兴趣的消息，如果是则从CommitLog中获取消息，封装返回客户端。否则就挂起到超时，超时时间是消息拉取方在请求参数中封装的，PUSH模式是15s。如果15s之后没有新消息，就会返回通知消费者（通知一个PULL_NOT_FOUND）。pullRequestHoldService，每隔5s被唤醒，去尝试检测是否有新消息到来，才会给客户端响应，但是这样实时性比较差，所以Rockertmq还有另外一种机制，也就是第二种处理方式。

二是在消息到达时会进行请求的回复notifyMessageArriving。由建立分区索引的线程ReputMessageService进行请求的回复。

> 当然了，如果不开启长轮询push机制，那么默认会一秒检查一次，且当有新的消息到来时不会通知客户端。
>
> 可以使用longPollingEnable配置。默认就是开启长轮询push的

push虽然好处多多，但是是有局限性的，即保持住客户端的连接是占用资源的，一般是和客户端连接数可用的场景下



push客户端源码分析：

客户端有一个重要的线程即PullMessageService，run方法中有一个阻塞队列（pullRequestQueue），不断地从中获取任务。队列中基本上一个分区（队列）对应一个请求。

pullRequestQueue有三种情况会加入pullRequest：一是程序启动的时候，先向队列中放入负载均衡器（RebalanceService#start）RebalanceImpl。它会根据负载均衡算法计算出当前消费者要消费的队列，每个队列构建一个pullRequest，而后放入队列pullRequestQueue。二是每次请求消息之后，会把pullRequest重新放入队列，一边接下来继续请求，当收到PULL_NOT_FOUND后，也会加入到pullRequestQueue队列。三是程序运行过程中，负载均衡器检测到有新增长的队列，那么会把新增长的队列作为一个pullRequest放入队列。

pullMessageService#run -> DefaultMQPushConsumerImpl#pullMessage -> PullAPIWrapper#pullKernelImpl -> MQClientAPIImpl#pullMessage ->MQClientAPIImpl#pullMessageAsync



push的broker端源码分析：

消费者发送拉消息请求到broker，broker判断当前有没有新消息邮寄返回。没有就挂住请求（将请求放入PullRequestHoldService的pullRequestTable中），轮询（PullRequestHoldService定时任务扫描）等待，每5秒检查一次有没有新消息到了，如果有就返回结果，如果过了15s也会返回结果。同时在分区索引线程中，当收到分区索引创建完索引之后，会处理分区对应的被挂起的请求。

PullMessageProcessor -》处理校验  ——》这里主要处理四种类型，SUCCESS有消息则读取消息返回。PULL_NOT_FOUNT就将其加入到pullRequestTable中。

pullRequestTable的定期扫描检查主要是遍历pullRequestTable的ketSet，然后获取指定messageQueue下最大的offset，调用notifyMessageArriving -》将最大的offset和当前的offset作比较，确定是否有新消息。有就返回客户端。同样的如果超时也返回。除了这两种都不返回（这里是先复制清除，如果不满足条件再加回来）。



使用FileRegion提升性能：

> transferMesByHeap =false

PullMessageProcessor如果有新消息，那么读取消息时默认会调用readGetMessageResut方法比较低效，因为有很多次CPU复制，所以RocketMQ还提供了可配置的使用FileRegion提升性能。

> FileRegion是netty提供的接口，在DefaultFileRegion时调用nio的transferTo方法，然后这个transferTo底层是调用sendFile零复制方法的。

rocketmq这里借用了FileRegion接口（因为需要靠netty是发送数据），实际上是拿到channel直接将数据写进去。但这里要注意，只有消息正文是直接内存（所以属于JVM的零拷贝），消息头还是堆内存（是没有零拷贝优化的）。

> 这里默认不开启，个人感觉至少在4.6版本的优化不多，只是消息体采用了JVM层面的零拷贝优化。到时可以看看5.0的这里的优化。



pull模式原理：

使用时pull的客户端会将所有的分区全部拉下来，使用时负载均衡、消费位点等都需要业务自行处理。

使用DefaultMQPullConsumer的方式：1.先获取MessageQueues并遍历，然后进行分区的分配，如果有多个消费者，需要进行负载均衡的计算。2.对于每一个分区，需要维护offset消费位点（消费的偏移量），这需要用户自行保存3.根据不同的消息状态进行处理，一般比较重要的是FOUND（有新消息）和NO_NEW_MSG（没有新消息）



消费执行的核心流程：

这里只看长轮询推模式：主要有两个核心类

PullMessageService（客户端拉消息的类）核心是pullMessageQueue阻塞队列

ConsumeMessageService（消费接口，主要有两个实现类，一是普通并发实现类ConsumeMessageConcurrentlyService，内部有一个consumeExecutor是业务消费线程池（核心=最大=20个），二是有序消息实现类）

在拉到消息后会执行回调，如果有消息就会在处理时将消息提交到ConsumeMessageService中（调用submitConsumeRequest）。 -》

在ConsumeMessageService#run中 -》调用listener.consumeMessage（这是我们业务开发时自己设置的）-》得到消息的状态 -》 根据状态进行处理调用 ConsumeMessageConcurrentlyService.thos.processConsumeResult -》先获取ackIndex，如果消费成功，将ackIndex与拉到的消息总数量最对比，如果ackIndex大则ackIndex是最后一个消息的位置。然后根据ackIndex计算成功和失败的标志位。如果消费失败则ackIndex=-1。（这里其实主要就是用一个ackIndex看看消息是消费成功了还是失败了） -》根据集群或广播模式对消息进行后续处理：集群模式下将消息回发给broker。如果是广播模式，则对消息打日志报警，不给broker回发重试。



消费流量控制机制：

consumer是业务组件，当消息增加时很容易造成消息积压，所以使用processQueue进行流控。

Push模式消费者会从三个角度控制：1.获取了还未处理的消息个数（1000个）、消息总大小（100M）、offset（消息跨度2000）

processQueue可以理解为MessageQueue在消费端的快照，主要成员：

msgTreeMap消息容器、lockTreeMap读写锁、msgCount总消息数、queueOffsetMax（processQueue队列的最大偏移量）、dropped当前是否被丢弃、上一次拉去时间戳、上一次消费时间戳

会通过processQueue的putMessage方法将msgs迭代存入msgTreeMap（key消费位点，值MessageExt）

> TreeMap的key是有序的，默认会将key做排序，在getMaxSpan可以通过最大和最小获取消息的区间

限流的做法是，放弃本次拉取消息的动作，如果任意阈值超过设定的大小，就延迟一段时间在拉取消息，从而达到流量控制的目的。这里会将pullRequest放入线程池延时50s。 



客户端在平衡机制：

RebalanceService继承ServiceThread也是多生产者单消费者的服务。主要用于消费者的分区的再平衡机制的。当新增或者删除消费者时，就需要重新平衡。

触发再平衡的条件主要有三点：当有新的消费者加入消费者组；消费者组成员其中一个下线或异常；消费者拉取请求超时导致再平衡。

RebalanceService#run -》 MQClientInstance#doRebalance -》DefaultMQPullConsumerImpl#doRebalance ->RebalanceImpl #doRebalance  -> rebalanceByTopic ->根据广播和集群两种模式分别做处理：广播模式拿到topic的所有分区，然后去调用updateProcessQueueTableInRebalance；集群模式下拿到所有分区和所有消费者，然后根据负载均衡策略进行计算当前消费者需要哪些分区然后调用updateProcessQueueTableInRebalance去更新 -》updateProcessQueueTableInRebalance方法中会将之前的分区processQueue和新的分区做比对，移除老的分区，如果拉取过期了（broker超过了120s没返回）将processQueue分区删除。 -》然后遍历新的分区方案，创建processQueue，同时如果之前没有负责这个分区就创建拉取请求。，并放入pullRequestQueue。



客户端负载均衡机制：

负载均衡可以在服务端做也可以在客户端做，rocketmq、springcloud都是在客户端做的。

广播模式通常不需要负载均衡，一般都是集群模式需要：

消息队列的负载均衡，通常是一个消息队列在同一个时间只允许被一个消费者消费，一个消费者可以同时消费多个消费队列。也就是将同一个topic的所有消息队列合理的分配到同一个group下的多个消费者。

如果可以被多个消费者消费那么就会出现重复消费的问题。当然如果分区数大于消费者个数，可能会导致有些消费者无法消费。

负载均衡时，需要知道所有的分区和所有的消费者，调用strategy.allocate方法：默认使用平均分配策略。



平均分配策略，将消费队列平均分给消费者，首先将读队列分成消费者数目的份数，平均分给每个消费者，除不尽就将余数分给剩下的消费者。

轮询分配策略：每个消费者一次分配一个消费队列

就近机房分配策略（AllocateMachineRoomNearBy）：首先统计消费者与broker的所在机房，保证broker中的消息优先被同机房的消费者消费。如果机房中没有消费者，就由其他机房的消费者进行消费。同机房或跨机房的具体分配时可以指定具体的算法，比如平均分配。假设有三个机房，机房1和3存在消费者，2没有。那么机房1、3中的队列会分配给各自机房的消费者，机房2的队列会被消费者平均分配。





# 参考资料

- RocketMQ 源码 这里使用了4.6版本，注意学习源码切勿只看笔记，我在我其他的博客笔记中也写了一定要自行跟一边源码流程，自己下载看一遍源码，自己画图总结一遍，否则只是隔靴搔痒，学不到精髓。

- 《横扫全网：rocketmq工业级高可用架构原理与实操v2》尼恩。尼恩大大的课质量很高，本文也是跟着尼恩大大的课，加上自己反复研读、debug源码，画图总结写出来的。

- 《RocketMQ技术内幕：RocketMQ架构设计与实现原理（第2版）》

- [RocketMQ官方文档](https://rocketmq.apache.org/zh/docs/4.x/)

- [Windows安装RocketMQ](https://blog.csdn.net/qq_43849912/article/details/106351233)

- [springboot整合rocketmq](https://blog.csdn.net/qq_36737803/article/details/112261352)

- [Linux内核刷新机制](https://zhuanlan.zhihu.com/p/577116167)

- [RocketMQ自动创建Topic的原理](https://blog.csdn.net/qq_39408435/article/details/125209812)

- [RocketMQ preventGC原理](https://mp.weixin.qq.com/s/y0zt9a4WmwBE1DcDGtN6fA)

  > 原文章名称《没有几十年功力，写不出这一行“看似无用”的代码！！》