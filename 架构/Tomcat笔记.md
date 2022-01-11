# Tomcat学习笔记

# 一、Tomcat概述

## 1.1 浏览器访问流程

我们先从一个http请求的流程开始

![bs流程20220110](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/bs%E6%B5%81%E7%A8%8B20220110.png)

浏览器访问服务器使用的是HTTP协议，HTTP协议是应用层协议，数据传输是TCP/IP协议。我们的tomcat就是一个HTTP服务器。

## 1.2 tomcat总体架构

上面我们的请求最终会走到HTTP服务器中，而我们的tomcat也是一个http服务器，因为tomcat能够接收并处理http请求。

当请求走到tomcat后，接收到请求之后把请求交给Servlet容器来处理，Servlet 容器通过Servlet接口调⽤业务类。Servlet接口和Servlet容器这⼀整套内容叫作**Servlet规范**。因此tomcat既有http服务器的功能又按照Servlet规范实现了Servlet容器，所以tomcat又是一个Servlet容器。

**tomcat处理流程**

当请求到来时：

1. HTTP服务器会把请求信息使用ServletRequest对象封装起来
2. Servlet容器拿到请求后，根据URL和Service的对应关系，定位到到具体的Servlet中
3. 如果Servlet还没有被加载，则通过反射创建这个Servlet，并调用Servlet的init方法来完成初始化。
4. 接着调用对应的业务类方法处理请求，然后将结果用ServletResponse封装
5. 把ServletResponse对象返回给HTTP服务器，HTTP服务器会把相应发送给客户端。

![tomcat请求流程](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/tomcat%E8%AF%B7%E6%B1%82%E6%B5%81%E7%A8%8B.png)

 **tomcat总体架构**

通过上面的流程我们发现tomcat有两个很重要的功能，一个是与客户端交互，进行socket通信，将字节流和Request/Response进行转换另一个就是Servlet容器业务处理逻辑。

![tomcat架构20220110](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/tomcat%E6%9E%B6%E6%9E%8420220110.png)

tomcat设计了两个核心组件：连接器（Connector）和容器（Container）来完成这两个功能。

下面我们介绍这两个组件。

## 1.3 连接器组件Coyote

Coyote是tomcat中连接器的组件名称，客户端通过Coyote和服务器建立连接，发送请求并接受响应，同时它使Catalina与具体的请求协议和IO操作完全解耦。Coyote负责的时具体协议（应用层）和IO（传输层）的相关内容，它将Socket输入转换封装为Request对象，进一步封装后交由Catalina容器进行处理，处理完请求后，Catalina通过Coyote提供的Response对象将结果写入输出流。

![coyote20220111](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/coyote20220111.png)

Tomcat Coyote支持的协议：

| 应用层协议 | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| HTTP/1.1   | 大部分web应用采用的协议，也是tomcat默认协议                  |
| AJP        | 用于和WX集成（如Apache），以实现对静态资源的优化以及集群部署，当前支持AJP/1.3 |
| HTTP/2     | HTTP2.0大幅度提升了Web性能。自8.5以及9.0版本之后支持         |

Tomcat Coyote支持的IO模型：

| IO模型 | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| NIO    | 非阻塞IO，采用Java NIO类库实现                               |
| NIO2   | 异步IO，采用JDK7最新的NIO2类库实现                           |
| APR    | 采用Apache可移植运行库实现，是C/C++编写的本地库。如果选择该方案，需要单独安装APR库 |

在tomcat8.0前，默认采用BIO，之后改为NIO。⽆论 NIO、NIO2 还是 APR， 在性能⽅⾯均优于以往的BIO。 如果采⽤APR， 甚⾄可以达到 Apache HTTP Server 的影响性能。

**Coyote内部组件**

![Coyoto组件20220111](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Coyoto%E7%BB%84%E4%BB%B620220111.png)

| 组件            | 作用描述                                                     |
| --------------- | ------------------------------------------------------------ |
| EndPoint        | EndPoint是Coyote的通信端点，即通信监听的端口，是具体Socket接收和发送处理器，是传输层的对象，因此EndPoint用来实现TCP/IP协议的 |
| Processor       | Processor是Coyote协议处理接口，如果说Endpoint是用来实现TCP/IP协议的，那么Processor用来实现HTTP协议，Processor接收来自EndPoint的Socket，读取字节流解析成Tomcat Request和Response对象，并通过Adapter将其提交到容器处理，Processor是对应用层协议的抽象。 |
| ProtocolHandler | Coyote协议接口，通过EndPoint和Processor，实现针对具体协议的处理能力。Tomcat按照协议和IO提供了6个实现类：AjpNioProtocol，AjpAprProtocol，AjpNio2Protocol，Http11NioProtocol，Http11Nio2Protocol，Http11AprProtocol |
| Adapter         | 由于协议不同，客户端发过来的消息也不同，tomcat定义了自己的Request类，来封装这些请求信息。ProtocolHandler接口负责解析请求并生成Tomcat Request类。但是这个Tomcat Request对象不是标准的ServletRequest。所以引入了CoyoteAdapter，这是适配器模式的经典应用，连接器调用CoyoteAdapter的service方法，传入Tomcat Request对象，然后CoyoteAdapter将Tomcat Request对象转换成Servlet Request对象然后调用容器。 |

## 1.4 Servlet容器Catalina

Tomcat是由一系列可配置(conf/server.xml)的组件构成的web容器，而Catalina是Tomcat的servlet容器。从另一个角度，Tomcat本质就是一款Servlet容器，因为Catalina才是Tomcat的核心，其他模块都是为了Catalina支撑的。比如Coyote提供链接通信，Jasper提供JSP引擎，Naming提供JBDI服务，Juli提供日志服务。

其实，可以认为整个Tomcat就是一个Catalina实例，Tomcat启动的时候会初始化这个实例，Catalina实例通过加载server.xml完成其他实例的创建，创建并管理Server，Server创建并管理多个服务，每个服务又可以有多个Connector和一个Contanier，各个组件











