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

# 三、自定义Tomcat

[项目地址](https://github.com/Loserfromlazy/My_Tomcat)

自定义Tomcat是可以作为一个服务器软件提供服务的，就是说我们可以使用浏览器客户端发送http请求，My_Tomcat可以接收请求并处理，处理成功后可以返回浏览器客户端。

我们分步来实现，首先我们请求http://localhost:8080,返回一个固定的Hello World字符串，然后封装Request和Response，返回html静态文件即静态资源，最后我们使My_Tomcat可以请求动态资源。

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

```java
public class HttpProtocolUtil {

    public static String getHttpHeader200(long contentLength){
        return "HTTP/1.1 200 OK \n" +
                "Content-Type:text/html \n" +
                "Content-Length: "+contentLength+"\n" +
                "\r\n";
    }

    public static String getHttpHeader404(){
        String str404="<h1>404 not found</h1>";
        return "HTTP/1.1 404 NOT Found \n" +
                "Content-Type:text/html \n" +
                "Content-Length: "+str404.getBytes(StandardCharsets.UTF_8).length+"\n" +
                "\r\n" +str404;
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
    ServerSocket serverSocket = new ServerSocket(port);
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



