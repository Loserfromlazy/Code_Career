# rabbitMQ学习笔记

消息中间件（消息队列）是分布式系统中重要的组件，主要解决应用耦合，异步消息，流量削锋等问题实现高性能，高可用，可伸缩和最终一致性[架构] 使用较多的消息队列有ActiveMQ，RabbitMQ，ZeroMQ，Kafka，MetaMQ，RocketMQ

## 1. 概述

rabbitMQ是由Erlang语言开发的的AMQP的开源实现

AMQP：高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

RabbitMQ 最初起源于金融系统，用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

## 2. 架构和主要概念

## 3. 安装与启动

1. 下载并安装Eralng

2. 下载并安装rabbitmq，不要安装在中文和空格目录下

3. 进入安装目录的sbin目录输入命令安装管理界面（插件）

   > rabbitmq-plugins enable rabbitmq_management

4. 重新启动rabbitmq的服务

5. 输入http://127.0.0.1:15672即可看到登陆界面，默认账号密码为guest

## 4. 发送与接收消息

### 4.1 直接模式Direct

我们将消息发送给唯一一个节点时使用这种模式

![img](https://img2018.cnblogs.com/blog/1538609/201907/1538609-20190720105736817-253615143.png)

1. 创建队列queue.test

2. 代码实现消息生产者

   导入依赖

   ~~~xml
   <dependency>
               <groupId>org.springframework.amqp</groupId>
               <artifactId>spring-rabbit</artifactId>
               <version>2.1.4.RELEASE</version>
           </dependency>
   ~~~

   编写配置文件

   ~~~java
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   	   xmlns:rabbit="http://www.springframework.org/schema/rabbit"
   	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                             http://www.springframework.org/schema/rabbit http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">
   	<!--连接工厂-->
   	<rabbit:connection-factory id="connectionFactory" host="127.0.0.1" port="5672" username="guest" password="guest" publisher-confirms="true"/>
   	<rabbit:template id="rabbitTemplate" connection-factory="connectionFactory" />	
   </beans>
   ~~~

   编写测试代码

   ~~~java
   public static void main(String[] args) {
           ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext-rabbitmq-producer.xml");
           RabbitTemplate rabbitTemplate = (RabbitTemplate) context.getBean("rabbitTemplate");
           rabbitTemplate.convertAndSend("","queue.test","直接模式测试");
           ((ClassPathXmlApplicationContext) context ).close();
       }
   ~~~

3. 代码实现消息消费者

   编写消息监听类

   ~~~java
   /**
    * 消息监听类
    */
   public class messageConsumer implements MessageListener {
       @Override
       public void onMessage(Message message) {
           System.out.println("接收消息"+new String(message.getBody()));
       }
   }
   ~~~

   编写配置文件

   ~~~xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   	   xmlns:rabbit="http://www.springframework.org/schema/rabbit"
   	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                             http://www.springframework.org/schema/rabbit http://www.springframework.org/schema/rabbit/spring-rabbit.xsd">
   	<!--连接工厂-->
   	<rabbit:connection-factory id="connectionFactory" host="127.0.0.1" port="5672" username="guest" password="guest" publisher-confirms="true"/>
   	<!--队列-->
   	<rabbit:queue name="queue.test" durable="true" exclusive="false" auto-delete="false" />
   	<!--消费者监听类-->
   	<bean id="messageConsumer" class="com.yhr.demo.messageConsumer"></bean>
   	<!--设置监听容器-->
   	<rabbit:listener-container connection-factory="connectionFactory" acknowledge="auto" >
   		<rabbit:listener queue-names="queue.test" ref="messageConsumer"/>
   	</rabbit:listener-container>
   </beans>
   ~~~

   编写测试代码

   ```java
   public static void main(String[] args) {    
       ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext-rabbitmq-consumer.xml");
   }
   ```

   执行即可看到上一步发送的消息，且此进程会一直运行，保持监听

**当有多个消费者同时监听一个队列，只有一个消费者可以接受到消息**

**在发送消息时，制定routingKey时如果没有此队列那么就会进行丢弃处理**

### 4.2 分列模式Fanout

将消息发送给多个队列使用这种模式

![img](https://img-blog.csdnimg.cn/20190416163519756.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIyNTk2OTMx,size_16,color_FFFFFF,t_70)

1. 创建队列queue.test1和queue.test2,创建交换器exchange.fanout_test，并绑定queue.test1和queue.test2

2. 代码实现消息生产者

   ~~~java
   public static void main(String[] args) {
           ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext-rabbitmq-producer.xml");
           RabbitTemplate rabbitTemplate = (RabbitTemplate) context.getBean("rabbitTemplate");
           rabbitTemplate.convertAndSend("exchanges.fanout_test","","分列模式测试");
           ((ClassPathXmlApplicationContext) context ).close();
       }
   ~~~

3. 代码实现消息消费者

   修改配置

   ~~~xml
   <!--队列-->
   	<rabbit:queue name="queue.test1" durable="true" exclusive="false" auto-delete="false" />
   	<rabbit:queue name="queue.test2" durable="true" exclusive="false" auto-delete="false" />
   	<!--消费者监听类-->
   	<bean id="messageConsumer1" class="com.yhr.demo.messageConsumer1"></bean>
   	<bean id="messageConsumer2" class="com.yhr.demo.messageConsumer2"></bean>
   	<!--设置监听容器-->
   	<rabbit:listener-container connection-factory="connectionFactory" acknowledge="auto" >
   		<rabbit:listener queue-names="queue.test1" ref="messageConsumer1"/>
   		<rabbit:listener queue-names="queue.test2" ref="messageConsumer2"/>
   	</rabbit:listener-container>
   ~~~

   执行上面的消费者的测试代码。

### 4.3通过配置文件创建队列和交换器

~~~xml
<!--rabbitAdmin -->
	<rabbit:admin connection-factory="connectionFactory"></rabbit:admin>
	<!--创建队列-->
	<rabbit:queue name="queue.test1"></rabbit:queue>
	<rabbit:queue name="queue.test2"></rabbit:queue>

	<!--创建交换器-->
	<rabbit:fanout-exchange name="exchanges.fanout_test">
		<rabbit:bindings>
			<rabbit:binding queue="queue.test1"></rabbit:binding>
			<rabbit:binding queue="queue.test2"></rabbit:binding>
		</rabbit:bindings>
	</rabbit:fanout-exchange>
~~~

