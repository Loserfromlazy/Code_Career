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

### 1.3.1 流量削峰



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
  #namesrv的第一个节点
  rmqnamesrv-a:
    #镜像
    image: apacherocketmq/rocketmq:4.6.0
    #容器名称
    container_name: rmqnamesrv-a
    #端口映射
    ports:
      - 9876:9876
    restart: always
    #环境设置
    environment:
      TZ: Asia/Shanghai
      #设置工作目录
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    #挂载目录
    volumes:
      - /usr/local/soft/rocketmq/data/namesrv-a/logs:/opt/logs
      - /usr/local/soft/rocketmq/data/namesrv-a/store:/opt/store
    #启动namesrv的shell命令
    command: sh mqnamesrv
    networks:
      rmq:
        aliases:
          - rmqnamesrv-a
  #namesrv的第二个节点
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
      - /usr/local/soft/rocketmq/data/namesrv-b/logs:/opt/logs
      - /usr/local/soft/rocketmq/data/namesrv-b/store:/opt/store
    command: sh mqnamesrv
    networks:
      rmq:
        aliases:
          - rmqnamesrv-b
  #broker集群的第一个master节点
  rmqbroker-a:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqbroker-a
    ports:
     - 10911:10911
     - 10912:10912
    volumes:
      - /usr/local/soft/rocketmq/data/broker-a/logs:/opt/logs
      - /usr/local/soft/rocketmq/data/broker-a/store:/opt/store
      - /usr/local/soft/rocketmq/conf/brokera.conf:/opt/rocketmq-4.6.0/conf/broker.conf
    environment:
      TZ: Asia/Shanghai
      NAMESRV_ADDR: "rmqnamesrv-a:9876"
      JAVA_OPTS: "-Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx512m"
    #启动时
    command: sh mqbroker -c /opt/rocketmq-4.6.0/conf/broker.conf autoCreateTopicEnable=true &
    links:
      - rmqnamesrv-a:rmqnamesrv-a
      - rmqnamesrv-b:rmqnamesrv-b
    networks:
      rmq:
        aliases:
          - rmqbroker-a
  #broker集群的第二个master节点
  rmqbroker-b:
    image: apacherocketmq/rocketmq:4.6.0
    container_name: rmqbroker-b
    ports:
     - 10921:10911
     - 10922:10912
    volumes:
      - /usr/local/soft/rocketmq/data/broker-b/logs:/opt/logs
      - /usr/local/soft/rocketmq/data/broker-b/store:/opt/store
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
  #控制台
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
#docker容器内网络
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

## 2.5 批量消息

## 2.6 事务消息

## 2.7 rocketmq在springboot中的开发方式

# 三、RocketMQ的存储架构

# 四、RocketMQ的高可用

# 五、RocketMQ的高可靠

# 参考资料

- 《横扫全网：rocketmq工业级高可用架构原理与实操v2》尼恩
- [RocketMQ官方文档](https://rocketmq.apache.org/zh/docs/4.x/)
- [Windows安装RocketMQ](https://blog.csdn.net/qq_43849912/article/details/106351233)