# Tomcat学习笔记

转载请声明作者和出处！！！

本文如有错误欢迎指正，感激不尽。

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

其实，可以认为整个Tomcat就是一个Catalina实例，Tomcat启动的时候会初始化这个实例，Catalina实例通过加载server.xml完成其他实例的创建，创建并管理Server，Server创建并管理多个服务，每个服务又可以有多个Connector和一个Container，各个组件介绍如下：

![TomcatServer20220112](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/TomcatServer20220112.png)

1. Catalina

   负责解析Tomcat的配置文件server.xml，以此来创建服务器server组件并进行管理

2. Server

   服务器表示整个Catalina Servlet容器以及其他组件，负责组装并启动Servlet引擎，Tomcat连接器。Server通过实现Lifecycle接口，提供一种优雅的启动和关闭整个系统的方式

3. Service

   服务是Server内部的组件，一个Server包含多个Service。它将若干个Connector组件绑定到一个Container

4. Container

   容器，负责处理用户的Servlet请求，并返回对象给web用户的模块

Container组件有以下几种具体的组件，分别是Engine、Host、Context和Wrapper，这4个组件是父子关系：

1. Engine

   表示整个Catalina的Servlet引擎，用来管理多个虚拟站点，一个Service最多只能有一个Engine，但一个引擎可以包含多个Host

2. Host

   代表一个虚拟主机，或者一个站点，可以给Tomcat配置多个虚拟主机地址，而一个虚拟主机可包含多个Context

3. Context

   表示一个Web应用程序，一个Web应用可包含多个Wrapper

4. Wrapper

   表示一个Servlet，Wrapper作为容器中的最底层不能包含子容器

# 二、Tomcat核心配置

## 2.1 配置详情

上述的组件都可以在conf/server.xml文件中体现，Tomcat作为服务器的配置主要是在server.xml的配置，里面包含了Servlet容器的相关配置，即Catalina的配置。具体配置内容如下：

**`server.xml`**

~~~xml
<!--port：关闭服务器的监听端口
	shutdown：关闭服务器的指令字符串
-->
<Server port="8005" shutdown="SHUTDOWN">
  <!-- 以日志的形式输出服务器、操作系统、JVM版本信息-->
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <!-- 加载和销毁APR，如果找不到APR库就会输出日志，并不影响Tomcat启动-->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!-- 避免JRE内存泄漏问题-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <!-- 加载和销毁全局命名服务-->
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <!-- 在Context停止时重建Executor池中的线程，避免ThreadLocal相关内存泄漏-->
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
   <!--定义了全局命名服务 -->
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!--
	定义一个Service服务，一个Server可以有多个Service
   -->
    <Service name="Catalina">
        <!--内容见下面-->
    </Service>
</Server>
~~~

**service标签及内部具体组件**

~~~xml
<!-- 用于创建Service实例，默认使用 org.apahce.catalina.core.StandardService
	默认情况下，仅指定Service的名称即name="Catalina"
	service子标签：
	Listener（用于为Service添加生命周期监听器）
	Executor（用于配置共享线程池）
	Connector（用于配置Service包含的链接器）
	Engine（用于配置Service中链接器对应的Servlet容器引擎）
-->
<Service name="Catalina">

        <!--
默认情况下，Service 并未添加共享线程池配置。 如果我们想添加⼀个线程池， 可以在<Service> 下添加如下配置：
name：线程池名称，⽤于 Connector中指定
namePrefix：所创建的每个线程的名称前缀，⼀个单独的线程名称为
namePrefix+threadNumber
maxThreads：池中最⼤线程数
minSpareThreads：活跃线程数，也就是核⼼池线程数，这些线程不会被销毁，会⼀直存在
maxIdleTime：线程空闲时间，超过该时间后，空闲线程会被销毁，默认值为6000（1分钟），单位毫秒
maxQueueSize：在被执⾏前最⼤线程排队数⽬，默认为Int的最⼤值，也就是⼴义的⽆限。除⾮特殊情况，这个值不需要更改，否则会有请求不会被处理的情况发⽣
prestartminSpareThreads：启动线程池时是否启动 minSpareThreads部分线程。默认值为false，即不启动
threadPriority：线程池中线程优先级，默认值为5，值从1到10
className：线程池实现类，未指定情况下，默认实现类org.apache.catalina.core.StandardThreadExecutor如果想使⽤⾃定义线程池⾸先需要实现org.apache.catalina.Executor接⼝
		-->

    <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>

    
    <!-- 默认情况下，server.xml 配置了两个链接器，⼀个⽀持HTTP协议，⼀个⽀持AJP协议，⼤多数情况下，我们并不需要新增链接器配置，只是根据需要对已有链接器进⾏优化 
    -->
    <!-- 
port：端⼝号，Connector ⽤于创建服务端Socket 并进⾏监听，以等待客户端请求链接。如果该属性设为0，Tomcat将会随机选择⼀个可⽤的端⼝号给当前Connector 使⽤

protocol：当前Connector ⽀持的访问协议。 默认为 HTTP/1.1 ， 并采⽤⾃动切换机制选择⼀个基于JAVA NIO的链接器或者基于本地APR的链接器（根据本地是否含有Tomcat的本地库判定）

connectionTimeOut:Connector 接收链接后的等待超时时间， 单位为 毫秒。 -1 表示不超时。

redirectPort：当前Connector 不⽀持SSL请求， 接收到了⼀个请求， 并且也符合security-constraint 约束，
需要SSL传输，Catalina⾃动将请求重定向到指定的端⼝。

executor：指定共享线程池的名称， 也可以通过maxThreads、minSpareThreads 等属性配置内部线程池。

URIEncoding:⽤于指定编码URI的字符编码， Tomcat8.x版本默认的编码为 UTF-8 , Tomcat7.x版本默认为ISO-
8859-1 -->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" disableUploadTimeout="true" useBodyEncodingForURI="true" URIEncoding="UTF-8" />
<!-- Define an AJP 1.3 Connector on port 8009 -->
    <Connector protocol="AJP/1.3"
               port="8009"
               redirectPort="8443"  URIEncoding="UTF-8"/>
    
<!--name： ⽤于指定Engine 的名称， 默认为Catalina
defaultHost：默认使⽤的虚拟主机名称， 当客户端请求指向的主机⽆效时， 将交由默认的虚拟主机处
理， 默认为localhost-->
    <Engine name="Catalina" defaultHost="localhost">

      <Realm className="org.apache.catalina.realm.LockOutRealm">

        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
<!--
Host用于配置主机
name： ⽤Host 的名称,即Engine指定的名称
appBase：网站资源信息的、存放位置，默认是webapps
unpackWARs：是否解压war包
autoDeploy：是否自动部署
-->
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
		<!--Context 配置一个Web应用
			docBase：web应用的路径，可以是磁盘上的绝对路径，也可以是相对于appBase的相对路径
			path：Web应用的访问路径，如果我们Host名为localhost， 则该web应⽤访问的根路径为：http://localhost:8080/demo
		-->
          <Context docBase="..." path="/demo"/>
      </Host>
    </Engine>
  </Service>
~~~

## 2.2 案例演示

server.xml中大部分配置保持默认即可，我们这里对host标签和context标签的配置进行演示：

我们首先在host中增加两个网址，用于配置host标签，我这里之前弄eureka时添加过，所以就不添加了，如下：

127.0.0.1       CloudEurekaServerA
127.0.0.1       CloudEurekaServerB

然后在server.xml中，修改host配置：

~~~xml
      <Host name="CloudEurekaServerA"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
	  <Host name="CloudEurekaServerB"  appBase="webapps2"
            unpackWARs="true" autoDeploy="true">

        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
~~~

然后我们复制一份webapps命名为webapps2，修改两个中的ROOT下的index.jsp（随便改，能区分即可）便于查看，然后我们分别访问两个host，结果如下：

![image-20220112141730338](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220112141730338.png)

![image-20220112141749175](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220112141749175.png)

我们下面在host中增加Context标签，我们可以将webapps复制到其他磁盘中，然后修改index.jsp，使用绝对路径配置（相对路径也可以，这里是为了演示方便），配置如下：

~~~xml
<Host name="CloudEurekaServerA"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
		<Context docBase="D:\webapps\ROOT" path="/demo" />
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
~~~

结果如下，这样我们就可以通过加`/路径名`来跳转到别的项目中：

![image-20220112142553665](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220112142553665.png)

# 三、手写简单Tomcat

[项目地址](https://github.com/Loserfromlazy/My_Tomcat)

自定义Tomcat是可以作为一个服务器软件提供服务的，就是说我们可以使用浏览器客户端发送http请求，My_Tomcat可以接收请求并处理，处理成功后可以返回浏览器客户端。主要是简单的理解tomcat的主要工作流程。

我们分步来实现，首先我们请求http://localhost:8080,返回一个固定的Hello World字符串，然后封装Request和Response，返回html静态文件即静态资源，最后我们使My_Tomcat可以请求动态资源。

我写的时候每一步都用了BIO和NIO两种方式，用的时候注释掉一种即可。这里只展示主要代码，具体工具类代码请自行github查看。

## 3.1 返回固定字符串 

第一步我们先让我们的项目能监听8080端口，并返回信息：

我们新建项目然后创建Boostrap类，类信息如下，主要思路是通过监听8080端口然后返回信息，在返回信息时需要将HTTP请求体进行拼接我写在了HttpProtocolUtil类中：

![image-20220115122657579](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220115122657579.png)

```java
public class BootStrap {
    //暂时写死，可以修改到xml文件中进行读取
    private int port = 8080;
    public int getPort() {
        return port;
    }
    public void setPort(int port) {
        this.port = port;
    }
    /**
     * MYTomcat程序init和启动
     *
     * @author Loserfromlazy
     * @date 2022/1/12 15:35
     */
    public void start() throws IOException {
        //===================BIO
        /*ServerSocket serverSocket = new ServerSocket(port);
        System.out.println("My_Tomcat Listening on port 8080");
        while (true){
            Socket socket = serverSocket.accept();
            OutputStream outputStream = socket.getOutputStream();
            String data = "Hello,World!";
            String responseText = HttpProtocolUtil.getHttpHeader200(data.getBytes(StandardCharsets.UTF_8).length) + data;
            outputStream.write(responseText.getBytes(StandardCharsets.UTF_8));
            socket.close();
        }*/
        //==================NIO
        ServerSocketChannel serverSocketChannel= ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            if (selector.select(2000)==0){
                continue;
            }
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                if (key.isAcceptable()){
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ);
                    String data = "Hello,World!";
                    String responseText = HttpProtocolUtil.getHttpHeader200(data.getBytes(StandardCharsets.UTF_8).length) + data;
                    ByteBuffer buffer = ByteBuffer.wrap(responseText.getBytes(StandardCharsets.UTF_8));
                    socketChannel.write(buffer);
                    iterator.remove();
                }
            }
        }
    }
    /**
     * MYTomcat程序入口
     *
     * @author Loserfromlazy
     * @date 2022/1/12 15:24
     */
    public static void main(String[] args) {
        BootStrap bootStrap = new BootStrap();
        try {
            bootStrap.start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 3.2 返回静态资源

第二步，我们可以封装请求的Request和Response对象，然后返回静态资源

> 这里我将上一步的Bootstrap改为Bootstrap1，然后新建一个Bootstrap，用于保存每一步的代码，后面也会这么做。

在进行封装前，我们可以查看HTTP请求头信息，可以使用以下方法查看或者F12：

```java
public void start() throws IOException {
    //===================BIO
    /*ServerSocket serverSocket = new ServerSocket(port);
    System.out.println("My_Tomcat Listening on port 8080");
    while (true) {
        Socket socket = serverSocket.accept();
        InputStream inputStream = socket.getInputStream();
        int count = 0;
        while (count == 0) {
            count = inputStream.available();
        }
        byte [] bytes = new byte[count];
        inputStream.read(bytes);
        System.out.println(new String(bytes));
        socket.close();
    }*/
    //==================NIO
        ServerSocketChannel serverSocketChannel= ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        System.out.println("My_Tomcat Listening on port 8080");
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            if (selector.select(2000)==0){
                continue;
            }
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                if (key.isAcceptable()){
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ);
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int count = 0;
                    while (count==0){
                        count = socketChannel.read(buffer);
                    }
                    buffer.flip();
                    System.out.println(new String(buffer.array()));
                    iterator.remove();
                }
            }
        }
}
```

输出信息：

~~~
GET / HTTP/1.1
Host: localhost:8080
Connection: keep-alive
Cache-Control: max-age=0
sec-ch-ua: " Not;A Brand";v="99", "Google Chrome";v="97", "Chromium";v="97"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: Idea-3b85319b=6df6ddd1-3436-43bd-a172-56742a301d72
~~~

拿到了请求头，我们就可以封装Request，我们这里就封装请求方法和URL。

**Request对象**

封装Request对象的思路就是将HTTP头中的信息保存在Requset对象中

```java
public class Request {

    private String method;//例如GET POST
    private String url;//例如 /index.html

    public String getMethod() {
        return method;
    }

    public void setMethod(String method) {
        this.method = method;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    private InputStream inputStream;//根据传入的inputStream解析请求头
    private SocketChannel socketChannel;////根据传入的SocketChannel解析请求头(NIO)

    public Request() {
    }

    public Request(SocketChannel socketChannel) throws IOException {
        this.socketChannel = socketChannel;
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int count = 0;
        while (count==0){
            count = socketChannel.read(buffer);
        }
        buffer.flip();
        String httpHeaderStr = new String(buffer.array());//http头协议
        String[] split = httpHeaderStr.split("\\n");
        String firstLine = split[0];//协议第一行
        String[] firstLineItem = firstLine.split(" ");
        this.method =firstLineItem[0];
        this.url = firstLineItem[1];
        System.out.println("method==>"+this.method+";url==>"+this.url);
    }

    public Request(InputStream inputStream) throws IOException {
        this.inputStream = inputStream;

        int count = 0;
        while (count == 0) {
            count = inputStream.available();
        }
        byte [] bytes = new byte[count];
        inputStream.read(bytes);
        String httpHeaderStr = new String(bytes);//http头协议
        String[] split = httpHeaderStr.split("\\n");
        String firstLine = split[0];//协议第一行
        String[] firstLineItem = firstLine.split(" ");
        this.method =firstLineItem[0];
        this.url = firstLineItem[1];
        System.out.println("method==>"+this.method+";url==>"+this.url);

    }
}
```

**Response对象**

封装Response对象的思路就是通过构造的输出流输出我们需要输出的内容。

```java
public class Response {

    private OutputStream outputStream;

    private SocketChannel socketChannel;

    public Response(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    public Response() {
    }

    public Response(OutputStream outputStream) {
        this.outputStream = outputStream;
    }

    /**
     * 通过url获取静态资源的绝对路径，根据绝对路径获取静态文件，然后使用输出流输出
     *
     * @author Yuhaoran
     * @date 2022/1/17 13:23
     */
    public void outPutHtml(String url) throws IOException {
        String absolutePath = StaticResourceUtil.getAbsolutePath(url);
        File file = new File(absolutePath);
        if (file.exists() && file.isFile()){
            FileInputStream inputStream = new FileInputStream(file);
            //输出文件内容
            int count = 0;
            while (count == 0) {
                count = inputStream.available();
            }
            int written = 0;
            int byteSize = 1024;
            byte[] bytes = new byte[byteSize];
            outputStream.write(HttpProtocolUtil.getHttpHeader200(count).getBytes(StandardCharsets.UTF_8));
            while (written < count) {
                if (written + byteSize > count) {
                    bytes = new byte[count - written];
                }
                inputStream.read(bytes);
                outputStream.write(bytes);
                outputStream.flush();
                written += byteSize;
            }
        }else {
            //输出404
            outputStream.write(HttpProtocolUtil.getHttpHeader404().getBytes(StandardCharsets.UTF_8));
        }
    }

    public void outPutHtmlByChannel(String url) throws IOException {
        String absolutePath = StaticResourceUtil.getAbsolutePath(url);
        File file = new File(absolutePath);
        if (file.exists() && file.isFile()){
            //输出文件内容
            FileInputStream inputStream = new FileInputStream(file);
            FileChannel fileChannel = inputStream.getChannel();
            ByteBuffer buffer = ByteBuffer.allocate((int) file.length());
            fileChannel.read(buffer);
            buffer.flip();
            byte[] bytes = HttpProtocolUtil.getHttpHeader200(buffer.capacity()).getBytes(StandardCharsets.UTF_8);
            ByteBuffer responseBuffer = ByteBuffer.wrap(bytes);
            socketChannel.write(responseBuffer);
            socketChannel.write(buffer);
        }else {
            //输出404
            byte[] bytes = HttpProtocolUtil.getHttpHeader404().getBytes(StandardCharsets.UTF_8);
            ByteBuffer buffer = ByteBuffer.wrap(bytes);
            socketChannel.write(buffer);
        }
    }

    public void outPut(String contect) throws IOException {
        outputStream.write(contect.getBytes(StandardCharsets.UTF_8));
    }

    public void outPutByChannel(String contect) throws IOException {
        ByteBuffer buffer = ByteBuffer.wrap(contect.getBytes(StandardCharsets.UTF_8));
        socketChannel.write(buffer);
    }
}
```

封装完成后我们就可以在主方法中使用：

```java
public class BootStrap2 {
    //暂时写死，可以修改到xml文件中进行读取
    private int port = 8080;
    public int getPort() {
        return port;
    }
    public void setPort(int port) {
        this.port = port;
    }
    /**
     * MYTomcat程序init和启动
     *
     * @author Loserfromlazy
     * @date 2022/1/15 12:38
     */
    public void start() throws IOException {
        //===================BIO
//        ServerSocket serverSocket = new ServerSocket(port);
//        System.out.println("My_Tomcat Listening on port 8080");
//        while (true) {
//            Socket socket = serverSocket.accept();
//            InputStream inputStream = socket.getInputStream();
//            OutputStream outputStream = socket.getOutputStream();
//            Request request = new Request(inputStream);
//            Response response = new Response(outputStream);
//            response.outPutHtml(request.getUrl());
//            socket.close();
//        }
        //==================NIO
        ServerSocketChannel serverSocketChannel= ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        System.out.println("My_Tomcat Listening on port 8080");
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            if (selector.select(2000)==0){
                continue;
            }
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                if (key.isAcceptable()){
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ);
                    System.out.println(socketChannel.getRemoteAddress()+"发来请求");
                    Request request = new Request(socketChannel);
                    Response response = new Response(socketChannel);
                    response.outPutHtmlByChannel(request.getUrl());
                    socketChannel.close();
                    iterator.remove();
                }
            }
        }
    }
    /**
     * MYTomcat程序入口
     *
     * @author Loserfromlazy
     * @date 2022/1/12 15:24
     */
    public static void main(String[] args) {
        BootStrap2 bootStrap = new BootStrap2();
        try {
            bootStrap.start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 3.3 返回动态资源

> 关于Xpath表达式可以在菜鸟上看一下语法很简单https://www.runoob.com/xpath/xpath-syntax.html

动态资源就需要我们进行对Servlet的封装，代码如下，首先需要定义一下Servlet规范，如下面的Servlet接口，然后封装一个具体的HttpServlet，最后我们自己编写Servlet继承HttpServlet写自己的业务逻辑：

```java
public interface Servlet {
    void init() throws Exception;

    void destory() throws Exception;

    void service(Request request,Response response) throws Exception;
}
```

```java
public abstract class HttpServlet implements Servlet{
    public abstract void doGet(Request request,Response response);

    public abstract void doPost(Request request,Response response);

    @Override
    public void service(Request request,Response response) throws Exception {
        if ("GET".equals(request.getMethod())){
            doGet(request,response);
        }else{
            doPost(request,response);
        }
    }
}
```

```java
public class MyServlet extends HttpServlet{
    @Override
    public void doGet(Request request, Response response) {
        String content = "<h1>Hello My_Tomcat GET Method!</h1>";
        try {
            //BIO
            //response.outPut(HttpProtocolUtil.getHttpHeader200(content.getBytes(StandardCharsets.UTF_8).length)+content);
            //NIO
            response.outPutByChannel(HttpProtocolUtil.getHttpHeader200(content.getBytes(StandardCharsets.UTF_8).length)+content);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void doPost(Request request, Response response) {
        String content = "<h1>Hello My_Tomcat POST Method!</h1>";
        try {
            //BIO
            //response.outPut(HttpProtocolUtil.getHttpHeader200(content.getBytes(StandardCharsets.UTF_8).length)+content);
            //NIO
            response.outPutByChannel(HttpProtocolUtil.getHttpHeader200(content.getBytes(StandardCharsets.UTF_8).length)+content);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void init() throws Exception {

    }
    @Override
    public void destory() throws Exception {

    }
}
```

有了Servlet我们还需要web.xml,这里我们参考之前的Servlet即可：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<web-app>

    <servlet>
        <servlet-name>myServlet</servlet-name>
        <servlet-class>server.MyServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>myServlet</servlet-name>
        <url-pattern>/my</url-pattern>
    </servlet-mapping>

</web-app>
```

最后在主方法中使用即可：

```java
public class BootStrap {
    //暂时写死，可以修改到xml文件中进行读取
    private int port = 8080;
    public int getPort() {
        return port;
    }
    public void setPort(int port) {
        this.port = port;
    }
    /**
     * MYTomcat程序init和启动
     *
     * @author Loserfromlazy
     * @date 2022/1/15 12:38
     */
    public void start() throws Exception {
        //加载web.xml
        loadServlet();
        //===================BIO
        /*ServerSocket serverSocket = new ServerSocket(port);
        System.out.println("My_Tomcat Listening on port 8080");
        while (true) {
            Socket socket = serverSocket.accept();
            InputStream inputStream = socket.getInputStream();
            OutputStream outputStream = socket.getOutputStream();
            Request request = new Request(inputStream);
            Response response = new Response(outputStream);
            if (servletMap.get(request.getUrl())==null){
                response.outPutHtml(request.getUrl());
            }else {
                HttpServlet httpServlet = servletMap.get(request.getUrl());
                httpServlet.service(request,response);
            }

            socket.close();
        }*/
        //==================NIO
        ServerSocketChannel serverSocketChannel= ServerSocketChannel.open();
        serverSocketChannel.bind(new InetSocketAddress(8080));
        serverSocketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        System.out.println("My_Tomcat Listening on port 8080");
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true){
            if (selector.select(2000)==0){
                continue;
            }
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                if (key.isAcceptable()){
                    SocketChannel socketChannel = serverSocketChannel.accept();
                    socketChannel.configureBlocking(false);
                    socketChannel.register(selector,SelectionKey.OP_READ);
                    System.out.println(socketChannel.getRemoteAddress()+"发来请求");
                    Request request = new Request(socketChannel);
                    Response response = new Response(socketChannel);
                    if (servletMap.get(request.getUrl())==null){
                        response.outPutHtmlByChannel(request.getUrl());
                    }else {
                        HttpServlet httpServlet = servletMap.get(request.getUrl());
                        httpServlet.service(request,response);
                    }
                    socketChannel.close();
                    iterator.remove();
                }
            }
        }
    }

    private Map<String,HttpServlet> servletMap = new HashMap<>();
    /**
     * 加载解析web.xml
     * @author Loserfromlazy
     * @date 2022/1/18 13:18
     */
    private void loadServlet() {
        InputStream resourceAsStream = this.getClass().getClassLoader().getResourceAsStream("web.xml");
        SAXReader saxReader = new SAXReader();
        try {
            Document document = saxReader.read(resourceAsStream);
            Element rootElement = document.getRootElement();

            List<Element> selectNodes = rootElement.selectNodes("//servlet");
            for (Element element : selectNodes) {
                //获取servlet-name
                Node servletNameNode = element.selectSingleNode("servlet-name");
                String servletName = servletNameNode.getStringValue();
                //获取servlet-class
                Node servletClassNode = element.selectSingleNode("servlet-class");
                String servletClass = servletClassNode.getStringValue();

                //根据servlet-name找到servlet-class
                Node servletMapping = rootElement.selectSingleNode("/web-app/servlet-mapping[servlet-name = '" + servletName + "']");
                Node urlPatternNode = servletMapping.selectSingleNode("url-pattern");
                String urlPattern = urlPatternNode.getStringValue();

                //存入map
                servletMap.put(urlPattern, (HttpServlet) Class.forName(servletClass).newInstance());
            }
        }catch (Exception e){
            e.printStackTrace();
        }

    }
    /**
     * MYTomcat程序入口
     *
     * @author Loserfromlazy
     * @date 2022/1/12 15:24
     */
    public static void main(String[] args) {
        BootStrap bootStrap = new BootStrap();
        try {
            bootStrap.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 3.4 使用线程池的方式改造

我们只需要创建一个线程池和一个线程类来包装我们的执行逻辑，然后将线程提交到线程池即可。

这里以NIO的线程类为例：

```java
public class RequestProcess implements Runnable{
    private SocketChannel socketChannel;
    private Map<String,HttpServlet> servletMap = new HashMap<>();
    private Selector selector;

    public RequestProcess(SocketChannel socketChannel, Map<String, HttpServlet> servletMap, Selector selector) {
        this.socketChannel = socketChannel;
        this.servletMap = servletMap;
        this.selector = selector;
    }

    @Override
    public void run() {
        try {
            socketChannel.configureBlocking(false);
            socketChannel.register(selector, SelectionKey.OP_READ);
            System.out.println(socketChannel.getRemoteAddress()+"发来请求");
            Request request = new Request(socketChannel);
            Response response = new Response(socketChannel);
            if (servletMap.get(request.getUrl())==null){
                response.outPutHtmlByChannel(request.getUrl());
            }else {
                HttpServlet httpServlet = servletMap.get(request.getUrl());
                httpServlet.service(request,response);
            }
            socketChannel.close();
        }catch (Exception e){
            e.printStackTrace();
        }

    }
}
```

# 四、Tomcat源码

## 4.1 源码构建

官方文档指定使用ant进行构建，但是使用maven也是能构建的。因为本质上就是一个Java项目。

我们首先在官网上下载tomcat的源码，我的源码版本是8.5.75，下载apache-tomcat-8.5.75-src.zip即可下载完成后将zip包解压。解压后创建一个source文件夹，将conf和webapps文件夹拷贝到source文件夹下：

![image-20220216134840537](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216134840537.png)

然后在解压后的文件夹下创建pom.xml文件：

![image-20220216134921017](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216134921017.png)

pom文件内容如下：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
		http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	 <groupId>org.apache.tomcat</groupId>
	 <artifactId>apache-tomcat-8.5.75-src</artifactId>
	 <name>Tomcat8.5</name>
	 <version>8.5</version>
  <properties>
    <!-- 设置项目编码 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  </properties>

	 <build>
		 <!--指定源目录-->
		 <finalName>Tomcat8.5</finalName>
		 <sourceDirectory>java</sourceDirectory>
		 <resources>
			 <resource>
				<directory>java</directory>
			 </resource>
		 </resources>
		 <plugins>
		 <!--引入编译插件-->
			 <plugin>
				 <groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>3.1</version>
				 <configuration>
					<encoding>UTF-8</encoding>
					<source>8</source>
					<target>8</target>
				 </configuration>
			 </plugin>
		 </plugins>
	 </build>

	<!--tomcat 依赖的基础包-->
	 <dependencies>
		 <dependency>
			 <groupId>org.easymock</groupId>
			 <artifactId>easymock</artifactId>
			 <version>3.4</version>
		 </dependency>
		 <dependency>
			 <groupId>ant</groupId>
			 <artifactId>ant</artifactId>
			 <version>1.7.0</version>
		 </dependency>
		 <dependency>
			 <groupId>wsdl4j</groupId>
			 <artifactId>wsdl4j</artifactId>
			 <version>1.6.2</version>
		 </dependency>
		 <dependency>
			 <groupId>javax.xml</groupId>
			 <artifactId>jaxrpc</artifactId>
			 <version>1.1</version>
		 </dependency>
		 <dependency>
			 <groupId>org.eclipse.jdt.core.compiler</groupId>
			 <artifactId>ecj</artifactId>
			 <version>4.5.1</version>
		 </dependency>
		 <dependency>
			 <groupId>javax.xml.soap</groupId>
			 <artifactId>javax.xml.soap-api</artifactId>
			 <version>1.4.0</version>
		 </dependency>
	 </dependencies>
</project>
~~~

然后打开idea导入这个工程，idea会自动识别为maven工程。导入工程后启动BootStrap类，启动前加上启动参数：

> 添加启动参数的方法如下：
>
> 打开idea的启动类的配置编辑
>
> ![image-20220216135214702](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216135214702.png)
>
> 然后再下图的VM options中编写，如果没有这个文本框，就需要在下图右上角的这个Modify options中添加VM options然后在编写。
>
> ![image-20220216135401645](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216135401645.png)

VM参数如下：参数主要是用来指定tomcat的日志路径和产品目录路径。

> Bootstrap启动的时候使用了两个系统变量catalina.home和catalina.base，从官网和源码中的注释可以知道这两者的区别主要是：catalina.home是Tomcat产品的安装目录，而catalina.base是tomcat启动过程中需要读取的各种配置及日志的根目录。默认情况下catalina.base是和catalina.home是相同的。
>

~~~
-Dcatalina.home=D:\tomcat\apache-tomcat-8.5.75-src\source
-Dcatalina.base=D:\tomcat\apache-tomcat-8.5.75-src\source
-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
-Djava.util.logging.config.file=D:\tomcat\apache-tomcat-8.5.75-src\source\conf\logging.properties
~~~

然后启动BootStrap类，打开localhost:8080，正常弹出tomcat首页即可。

![image-20220216143150933](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216143150933.png)

## 4.2 源码构建的问题

以下是构件中编译遇到的各种问题：

### 4.2.1 JSP报错

如果打开8080弹出JSP报错（PS：我的是JSP报错加乱码），如下图：

![image-20220216142507101](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216142507101.png)

则需要手动添加JSP的编译器，在ContextConfig中的configureStart()方法中增加JSP编译器，如下图：

![image-20220216142645786](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216142645786.png)

重启后就可以解决JSP报错的问题。

### 4.2.2 tomcat编译乱码问题

如果也想我一样有日志乱码问题，首先可以试试网上的各种改UTF-8编码的操作，这里就不做赘述。如果不行就需要改源码解决，过程如下：

但是我改了7、8个地方改成UTF-8都不行，所以我对日志进行了Debug，过程如下：

首先看控制台的乱码如下图：

![image-20220216143320555](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216143320555.png)

然后在最开始出现乱码的地方，通过全局搜索找到这个VersionLoggerListener.log

![image-20220216143427884](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216143427884.png)

然后打上断点开始进行debug调试，一路跟代码，发现在getString方法中的bundle.getString(key)这一句执行完后，乱码就会出现。如下图：

![image-20220216143916714](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216143916714.png)

点进bundle.getString(key)，发现是JDK中ResourceBundle的方法.这个类是用来做国际化的。

最后在stackoverflow上找到了，如下图，Java9以上属性文件默认是UTF-8，但是Java8及以下默认是使用ISO-8859-1的，我的jdk版本就是8所以日志会有乱码问题。

![image-20220216145522949](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216145522949.png)

解决方法是：修改方法，手动进行转码（改的时候注意有很多getString，别改错了）：

![image-20220216151134899](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216151134899.png)

重启后发现第一个乱码问题已解决。

下一个乱码问题也是如此，最后跟到下面的地方，跟上面同理，手动进行转码

![image-20220216151251087](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220216151251087.png)

乱码问题解决。

> PS：我改完这两个地方就没有乱码了

## 4.3 源码流程分析

> 建议自己跟一编源码，光看别人的源码分析是看不懂的，别人的源码分析只能提供分析思路

### 4.3.1 tomcat启动流程

首先我们在startip.bat或startup.sh中可以找到这个脚本会拉起catalina.bat或catalina.sh的脚本，如下图

![image-20220218152901720](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218152901720.png)

然后再catalina.bat或catalina.sh中会发现这个脚本会启动BootStrap类，也就是main方法，然后启动时会带一个start的参数，如下图：

![image-20220218153057792](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218153057792.png)

然后我们进入BootStrap的main，具体的流程图（流程图是自己边Debug边画的只有大概的流程）在下面。

进入main方法后会先进入init方法

![image-20220218153804154](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218153804154.png)

这里会创建一个Catalina的实例

![image-20220218153849250](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218153849250.png)

然后回到main方法，继续往下走，我们会发现，会执行load()方法，然后执行start()方法。load方法主要是tomcat中的各种组件初始化，start方法是各种组件依次启动。

![image-20220218154210449](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218154210449.png)

我们先跟进load()方法,这里会调用Catalina的load方法：

![image-20220218154748322](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218154748322.png)

我们进入Catalina的load方法中，这个方法中主要关注的代码如下，它会拉取Server的init方法：

```java
getServer().init();
```

我们点进init方法，会进入到LifeCycle的init方法，LifeCycle这个接口主要就是定义了整体生命周期，我们进入LifecycleBase的init方法中（默认进这个类，可以自行debug确认）

![image-20220218155155040](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218155155040.png)

进入后会走到initInternal()方法中，这个方法是抽象方法，debug跟进会进入到对应的实现方法中，因为我们调用的是Server.init()所以会进入StandardServer的initInternal()中。

![image-20220218155644956](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218155644956.png)

我们进入该方法，这个方法中主要关注下面的代码：

```java
// Initialize our defined Services
for (Service service : services) {
    service.init();
}
```

service.init()会拉起Service的init方法，我们点进init(),会发现会回到LifeCycle的init方法，然后跟上面一样会进入到StandardService的initInternal()方法中，如下图

![image-20220218155407675](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220218155407675.png)

最后会按照流程图一步一步走下去，会把Tomcat的组件逐级init。然后会回到BootStrap的start()方法中，start()方法与init()方法流程差不多，这里就不再赘述。

Tomcat启动的整体流程图：

![tomcat启动流程](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/tomcat%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png)

### 4.3.2 Tomcat请求处理流程

**部署项目到源码**

如果需要查看Tomcat的请求流程，我们就需要有效的能进入我们Tomcat源代码工程的URL。所以我们可以新建一个JavaWeb工程，然后将这个JavaWeb工程部署到我们的Tomcat源码项目中。

首先我们先新建一个webdemo工程，然后创建一个Servlet,如下：

![image-20220221094659286](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221094659286.png)

```java
@WebServlet(name = "webdemo",value = "/web/webdemo")
public class DemoServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("进入了GET方法");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        System.out.println("进入了POST方法");
    }
}
```

然后我们将webdemo工程部署到tomcat中启动（注意是Tomcat不是我们的Tomcat源码工程），启动后会生成文件夹，如下图：

![image-20220221094826667](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221094826667.png)

然后我们将Tomcat生成的这个webdemo文件夹拷贝到我们的Tomcat源码工程中：

![image-20220221095019428](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221095019428.png)

然后就可以在源码中打断点查看Tomcat的请求流程了。

以上面的工程为例，请求地址是`http://localhost:8080/webdemo/web/webdemo`,Tomcat处理的大体流程如下图：

![Tomcat请求流程20220221](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/Tomcat%E8%AF%B7%E6%B1%82%E6%B5%81%E7%A8%8B20220221.png)

**Tomcat请求流程**

首先，我们需要找到断点的入口，我们在NioEndPoint中的Poller类的run方法中打上断点。

> 为什么在NioEndPoint中的Poller类的run方法中打上断点？
>
> Tomcat是从Connector接收请求的,在上面的启动流程源码分析中，如果debug跟代码的话，最后会发现在init流程中会执行socket的bind()方法在start流程中会执行socket的accept()方法。这两个方法都是在NioEndpoint中的，如下图：
>
> ![image-20220221150926632](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221150926632.png)
>
> 而在NioEndpoint的startInternal()方法中，在执行accept之前，会创建轮询器，
>
> ![image-20220221160351307](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221160351307.png)
>
> 我们进入到轮询器类中，发现其继承了Runnable，所以我们找到run方法，会发现，此方法会监听socket通道中的事件，因此这个轮询线程就是为了轮询监听socket通道事件的。因此我们在监听到事件之后的处理方法即`processKey(sk, socketWrapper);`上打断点开始debug。如下图：
>
> ![image-20220221160819369](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220221160819369.png)

下面我们debug运行BootStrap然后浏览器发送一个请求，开始查看请求处理流程：

> 这里只是演示请求处理流程，建议自己跟一遍源码。

首先我们看到请求确实来到了processKey方法，如下图：

![image-20220222104226663](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222104226663.png)

我们跟进这个方法，在processKey中有一个processSocket方法这个方法会继续处理我们的socket和key的信息，我们跟进这个processSocket方法。

![image-20220222105709562](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222105709562.png)

进入之后我们会发现进入到了NioEndPoint的抽象父类的方法中，然后我们会发现在这里会用线程池执行一个线程，因此我们需要关注这个线程的run方法。

![image-20220222105953264](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222105953264.png)

我们进入到SocketProcessorBase类中的run方法，代码如下：

```java
@Override
public final void run() {
    synchronized (socketWrapper) {
        // It is possible that processing may be triggered for read and
        // write at the same time. The sync above makes sure that processing
        // does not occur in parallel. The test below ensures that if the
        // first event to be processed results in the socket being closed,
        // the subsequent events are not processed.
        if (socketWrapper.isClosed()) {
            return;
        }
        doRun();
    }
}
```

我们在doRun()打上断点，让程序运行到这里。（因为是线程池执行这个线程，所以可以跳过来）

然后我们继续跟进doRun()方法，在这个方法中，我们会发现他会执行handler的process方法。

![image-20220222110701615](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222110701615.png)

进入process方法之后，在这个方法中会得到一个Processor对象，然后调用Processor对象的process方法，如下图：

![image-20220222111046555](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222111046555.png)

继续跟，会进入到AbstractProcessorLight中（这个类是一个轻量级的抽象处理器实现，旨在作为从轻量级升级处理器到HTTP/AJP处理器的所有处理器实现的基础。）也就是说在这里会找到具体的处理器Processor去处理。

我们在这个方法中能找到service方法如下，这个方法会继续处理我们我们第一步传过来的socketWrapper对象。

```java
state = service(socketWrapper);
```

我们跟进这个方法，果然进入到了具体的处理器中，也就是会进入到具体的Http1.1协议处理器的service方法中：

![image-20220222111732184](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222111732184.png)

进入到service方法后，这个方法会将我们传过来的request和response对象转换成标准的HttpServlet对象，然后调用容器，getContainer方法会拿到service关联的Engine。

这里会通过获取容器的管道然后调用管道中的第一个Value的invoke方法，如下图：

![image-20220222125009002](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222125009002.png)

我们跟进invoke方法：（此时会进入StandardEngineValve类） 此方法中会调用host，从host中管道取第一个Value执行invoke方法，去处理这个http请求。**管道Pipeline其实就是一些按顺序执行Value#invoke()方法的对应的类的封装**

> Pipeline是一个处理管道的标准实现，它将调用一系列已配置为按顺序调用的阀门。此实现可用于任何类型的容器。Pipeline里面是Value接口的各种实现类。
>
> **getFirst方法：**
>
> 在调用pipeline的getFirst()方法时，如果first上有Value就返回该Value，否则就返回basic上的Value。
>
> ```java
> protected Valve basic = null;
> protected Container container = null;
> protected Valve first = null;
> @Override
> public Valve getFirst() {
>     if (first != null) {
>         return first;
>     }
> 
>     return basic;
> }
> ```
>
> basic和first都是Value，Value是一个接口，下面有很多实现类比如StandardEngineValve
>
> basic其实就是它本身的标准Value实现类：
>
> ![image-20220222143447158](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222143447158.png)

```java
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

    // Select the Host to be used for this Request
    Host host = request.getHost();
    if (host == null) {
        // HTTP 0.9 or HTTP 1.0 request without a host when no default host
        // is defined.
        // Don't overwrite an existing error
        if (!response.isError()) {
            response.sendError(404);
        }
        return;
    }
    if (request.isAsyncSupported()) {
        request.setAsyncSupported(host.getPipeline().isAsyncSupported());
    }

    // Ask this Host to process this request
    host.getPipeline().getFirst().invoke(request, response);
}
```

> PS：在获取`host.getPipeline().getFirst().invoke(request, response);`
>
> host的pipeline的顺序是AbstractAccessLogValve->ErrorReportValve ->StandardHostValve。host的pipeline的basic时StandardHostValve，如图：
>
> ![image-20220222142458756](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222142458756.png)

我们继续跟进host的invoke，接下来会走到AbstractAccessLogValve的invoke方法然后调用`getNext().invoke(request, response);`

我们继续跟invoke会走到ErrorReportValve的invoke方法，此方法中继续调用`getNext().invoke(request, response);`

我们继续跟，此时会进入StandardHostValve类，然后会从context中拿管道调用invoke。

```java
if (!response.isErrorReportRequired()) {
    context.getPipeline().getFirst().invoke(request, response);
}
```

我们继续跟,进入到AuthenticatorBase#invoke()方法然后调用`getNext().invoke(request, response);`

我们继续跟invoke，会进入到StandardContextValve#invoke()方法

这里会从wrapper中获取管道然后执行

```java
wrapper.getPipeline().getFirst().invoke(request, response);
```

跟进invoke会进入StandardWrapperValve#invoke()方法

在这里会创建一个过滤器链，代码如下：

> 关于过滤器链的注解：filterchain的实现，用于管理针对特定请求的一组过滤器的执行。当一组定义的过滤器全部执行完毕后，对doFilter()的下一次调用将执行servlet的service()方法本身。

```java
ApplicationFilterChain filterChain =
        ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
```

上面代码往下几行，创建完之后会执行这个过滤器链：

```java
filterChain.doFilter
    (request.getRequest(), response.getResponse());
```

我们跟进doFilter方法：一直往下跟最后会执行service方法

```java
servlet.service(request, response);
```

> 从connector调用容器到servlet.service的过程：
>
> ![image-20220222132651362](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222132651362.png)

这个方法就是我们的Servlet的的service，然后就会取执行我们自己的Servlet方法了。此时看servlet，就是上面我们部署的webdome项目中的servlet，也就是说tomcat一步一步的找到了我们的Servlet并执行它。

![image-20220222132014991](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222132014991.png)

整体流程：

![tomcat处理请求流程20220222](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/tomcat%E5%A4%84%E7%90%86%E8%AF%B7%E6%B1%82%E6%B5%81%E7%A8%8B20220222.png)

# 五、Tomcat对HTTPS的支持

https需要证书，我们可以用java自带的工具生成自己的测试用的证书。

~~~shell
keytool -genkey -alias loserfromlazy -keyalg RSA -keystore miyao.keystore
~~~

- -alias 别名
- -keyalg 加密方式
- -keystore  密钥库名称
- ...其它具体参数请自行查阅

![image-20220222170641664](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222170641664.png)

这里注意名字与姓氏需要与域名保持一致，之后就会在当前目录下生成miyao.keystore文件

然后在tomcat的server.xml中配置

~~~xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="conf/miyao.keystore"
						 certificateKeystorePassword="123456"
                         type="RSA" />
        </SSLHostConfig>
    </Connector>
~~~

然后浏览器访问即可（注意需要忽略浏览器的安全警告）：

![image-20220222172111497](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220222172111497.png)

# 六、Tomcat调优

Tomcat本身也是一个Java项目，所以调优分为JVM调优和Tomcat自身调优

## 6.1 JVM调优

jvm调优只需要我们将jvm参数放到Catalina.bat或Catalina.sh中即可。



## 6.2 Tomcat自身调优
