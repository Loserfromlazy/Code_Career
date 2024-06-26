# Java高并发原理篇HTTP原理

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷一尼恩编著、HTTP权威指南以及菜鸟等互联网资源

# 一、HTTP原理与Web服务器

## 1.1 高性能Web应用架构

> 术业有专攻，本小节的LVS、KeppAlived具体配置和运行，更多属于运维的工作，开发人员仅需要了解原理即可。

一般来说，高性能的IM应用还需要高性能的Web应用来进行配合。高并发、大流量的WEB应用，QPS在10万甚至千万每秒，所以使用高并发HTTP通信技术提升内部各节点的通信性能，对于提升分布式系统整体的吞吐量有着非常大的作用。

我们先来看10w级别的Web应用架构：

![image-20220805105906938](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805105906938.png)

这种架构重点是接入层和服务层。首先是服务层，微服务计数前通过tomcat集群向外提供服务实例，在微服务技术成为主流后，服务层主要是微服务实例提供服务，并由网关提供对外统一的访问服务。其次是接入层，主要是反向代理层，利用Nginx来做反向代理。

> - Nginx将客户端请求分发给上游多个Web服务，Nginx向外暴露一个IP，Nginx与Web服务使用内网访问。
> - Ngixn需要保证负载均衡，可通过Lua脚本进行动态伸缩，动态增加Web服务节点的能力
> - Nginx需要保证高可用，任一台Web服务挂了，都可以将流量迁移至其他节点。
>
> 由于Ngixn使用Reactor模式，处理大并发请求时，内存消耗很小，3w并发下，开启10个Ngixn进程才消耗150M内存。

Nginx同样需要保证高可用，所以可以使用Nginx+KeepAlived组合，具体如下：

1. 使用两台或以上Nginx组成集群，分别部署上KeppAlived，设置成相同的虚ip共下游访问
2. 当一台Nginx挂了，KeepAlived能探测到，并将流量自动迁移到另一台Nginx上

如果流量一直上升，则可以使用LVS+KeepAlived组合实现Nginx的可拓展比如百万级别的Web应用架构：

![image-20220805110518046](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805110518046.png)

这种架构重点在负载均衡层和客户端层：

在客户端层，需要在DNS服务器上使用负载均衡机制，DNS负载均衡很简单，属于运维层面的技术，就是在DNS服务器上配置多个A记录，在一个域名下添加多个IP，由DNS服务器进行多个IP的负载均衡，甚至可以按照就近原则，为用户返回最近的服务器IP地址。

DNS服务器负载均衡也有很多缺点：首先无法动态调整主机地址的权重，其次DNS通常会缓存查询响应，以便更迅速的向用户提供查询服务，如果某主机宕机，那么第一时间移除服务器IP也无济于事。

由于DNS负载均衡无法满足高可用，所以仅用作客户端层面的简单负载均衡，同时引入一个专门的负载均衡层，通过LVS+KeepAlived组合达到高可用和负载均衡的目的。

> LVS是Linux Virtual Server的简写，是一个虚拟的服务器集群系统。

LVS+KeepAlived具体方案如下：

1. 使用两台或以上LVS组成集群，分别部署上KeppAlived，设置成相同的虚拟IP（VIP）供下游访问。KeepAlived对LVS负载均衡调度器实现健康监控、热备切换，具体来说，对服务器中的各个节点进行健康检查，自动移除失效节点，恢复后在重新加入，从而保证LVS高可用
2. 在LVS系统上可配置多个接入层Nginx服务器集群，由LVS完成高速的请求分发和接入层的负载均衡。

LVS常常使用直接路由方式（DR）进行负载均衡，数据在分发过程中不修改IP地址，只修改mac地址，由于实际处理请求的真实物理IP地址和数据请求目的IP地址一致，所以响应数据包可以不需要通过LVS负载均衡服务器进行地址转换，而是直接返回给用户浏览器，避免LVS负载均衡服务器网卡带宽成为瓶颈。此种方式又称作三角传输模式，具体如下图所示

![image-20220805130115493](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805130115493.png)

使用三角传输模式的链路层负载均衡是目前大型网站使用最广泛的一种负载均衡手段，目前，LVS（Linux Virtual Server）是Linux平台上最好的三角传输模式软件负载均衡开源产品。

> LVS与Nginx区别？
>
> Nginx主要用于四层、七层负载均衡，平时Nginx进行Web应用的负载属于七层负载均衡，而LVS主要用于二层、四层负载均衡，但由于性能原因更多用于二层负载均衡。
>
> 二层、四层、七层负载均衡？
>
> 均属于OSI模型的概念。二层是指OSI模型的数据链路层主要根据报文中的链路层地址，比如MAC地址，在多个上游服务器选择一个RS（Real Server）真实服务器，然后进行报文转发和处理，实现负载均衡
>
> 四层是指OSI模型的传输层，主要是通过修改报文中的目标ip地址和端口，在多个上游TCP/UDP服务器之间选择一个RS，然后进行报文转发，实现负载均衡
>
> 七层是指OSI模型的应用层，主要根据报文中的应用层内容如HTTP协议URI、Cookie信息，虚拟主机Host名称等，在多个上游应用层服务器之间选择一个RS，然后进行报文转发，实现负载均衡

LVS的转发分为NAT模式（四层）和DR模式（二层），具体内容如下：

1. NAT：NAT是一种外网和内网地址映射的技术，是一种网络地址转换技术，NAT模式下网络数据包的进出都要经过LVS的处理，LVS作为RS的网关。

   > NAT 包括目标地址转换（DNAT）和源地址转换（SNAT）。当包到达 LVS 时，LVS需要做目标地址转换（DNAT）：将目标IP改为RS的IP，RS在接收到数据包以后，仿佛是客户端直接发给它的一样；RS处理完返回响应时，源IP是RS的IP，目标IP是客户端的IP，这时LVS 需要做做源地址转换（SNAT），将包的源地址改为VIP（对外的IP），这样这个包对客户端看起来就仿佛是LVS直接返回给它的

2. DR：DR也叫直接路由、三角传输模式。DR模式下需要LVS和RS集群绑定到同一个VIP（虚拟ip）上，与NAT不同的是，请求由LVS接收，处理有由RS直接返回给用户。

   > 一个请求过来时，LVS 只需要将网络帧的 MAC 地址修改为某一台 RS 的 MAC，该包就会被转发到相应的 RS 处理，注意此时的源 IP 和目标 IP 都没变，LVS 只是做了一下移花接木。RS 收到 LVS 转发来的包时，链路层发现 MAC 是自己的，到上面的网络层，发现 IP 也是自己的，于是这个包被合法地接受，RS 感知不到前面有 LVS 的存在。而当 RS 返回响应时，只要直接向源IP（即客户端的 IP）返回即可，不再经过LVS转发。这里有个系统运维的要点：RS的Loopback口和需要和LVS设备上存在着相同的VIP地址，这样响应才能直接返回到客户端。
   >
   > 在DR负载均衡模式下，数据在分发过程中不修改IP地址，只修改MAC地址，由于实际处理请求的真实物理IP地址和数据请求目的IP地址一致，所以不需要通过负载均衡服务器进行地址转换，其最大的优势为：可将响应数据包直接返回给用户浏览器，避免负载均衡服务器网卡带宽成为瓶颈，因此，DR模式具有较好的性能，是目前大型网站使用最广泛的一种负载均衡手段

## 1.2 HTTP应用层协议

HTTP是一个应用层的面向对象的协议，是一种请求-响应模式的协议，由于其简捷、快速的方式，适用于分布式超媒体信息系统，是互联网上应用最为广泛的一种网络协议。所有的WWW文件都必须遵守这个标准。1960年美国人Ted Nelson构思了一种通过计算机处理文本信息的方法，并称之为超文本（Hyper Text），这成为了HTTP超文本传输协议标准架构的发展根基。最终，万维网协会（World Wide Web Consortium）和互联网工程工作小组（Internet Engineering Task Force ）共同合作研究HTTP协议，最终发布了一系列的RFC文档，其中著名的RFC 2616定义了HTTP 1.1协议。

### 1.2.1 HTTP的请求URL

URL提供了一种定位因特网上任意资源的手段，但这些资源是可以通过各种不同的方案（比如HTTP、FTP、SMTP）来访问的，因此URL语法会随方案的不同而有所不同。大部分URL都遵循通用的URL语法，而且不同URL方案的风格和语法都有不少重叠。大多数URL方案的URL语法都建立在这个由9部分构成的通用格式上：

~~~
<scheme>://<user>:<password>@<host>:<port>/<path>; <params>?<query>#<frag>
~~~

几乎没有哪个URL包含了所有这些组件。URL最重要的3个部分是方案、主机和路径。见下表：

| 组件 | 描述                                                         | 默认值       |
| ---- | ------------------------------------------------------------ | ------------ |
| 方案 | 访问服务器获取资源时的协议                                   | 无           |
| 用户 | 某些方案访问资源时需要的用户名                               | 匿名         |
| 密码 | 用户名后面可能要包含的密码，中间用冒号分隔                   | E-mail地址   |
| 主机 | 资源宿主服务器的主机名或IP地址                               | 无           |
| 端口 | 资源宿主服务器正在监听的端口号，很多方案都有默认的端口号，比如HTTP的默认是80 | 每个方案特有 |
| 路径 | 服务器上资源的本地名，由一个斜杠将其与前面的URL组件分隔开来。 | 无           |
| 参数 | 某些方案会用这个组件来指定输入参数。参数为`名/值对`。URL中可以包含多个参数字段，他们之间以及路径用分号分隔 | 无           |
| 查询 | 某些方案会用这个组件传递参数已激活应用程序，查询组件内用没有通用格式，用字符？将其与URL其余部分分割开来 | 无           |
| 片段 | 一小片或一部分资源的名字，引用对象时，不会将frag字段传送给服务器，这个字段是在客户端内部使用的，通过#将其与URL其余部分分隔开来。 | 无           |

1. 方案:方案实际上是规定如何访问指定资源的主要标识符，它会告诉负责解析URL的应用程序应该使用什么协议。方案组件必须以一个字母符号开始，由第一个“:”符号将其与URL的其余部分分隔开来。方案名是大小写无关的，因此URL`http://www.demo.com`和`HTTP://www.demo.com`是等价的。

2. 主机与端口：主机组件标识了因特网上能够访问资源的宿主机器。可以用主机名（`www.demo.com`），或者IP地址来表示主机名。端口组件标识了服务器正在监听的网络端口。对下层使用了TCP协议的HTTP来说，默认端口号为80。

3. 用户和密码组件：很多服务器都要求输入用户名和密码才会允许用户访问数据。FTP服务器就是这样一个常见的实例。这里有几个例子：

   比如`ftp://ftp.demo.edu/pub/gnu`此例子没有用户或密码组件，只有标准的方案、主机和路径。如果某应用程序使用的URL方案要求输入用户名和密码，比如FTP，但用户没有提供，它通常会插入一个默认的用户名和密码anonymous（匿名用户）作为你的用户名，并发送一个默认的密码（Internet Explorer会发送IEUser, Netscape Navigator则会发送mozilla）。

   再比如`ftp://anonymous@ftp.prep.ai.mit.edu/pub/gnu`这个anonymous的用户名与主机组件组合在一起，看起来就像E-mail地址一样。字符“@”将用户和密码组件与URL的其余部分分隔开来。

   再来看`ftp://anonymous:my passwd@ftp.demo.edu/pub/gnu`此例子指定了用户名（anonymous）和密码（my passwd），两者之间由字符“:”分隔。

4. URL的路径组件：路径说明了资源位于服务器的什么地方。路径通常很像一个分级的文件系统路径。比如：`http://www.demo.com:80/seasonal/index-fall.html`这个URL中的路径为`/seasonal/index-fall.html`，很像UNIX文件系统中的文件系统路径。

   路径是服务器定位资源时所需的信息。可以用字符“/”将HTTP URL的路径组件划分成一些路径段（path segment）。每个路径段都有自己的参数（param）组件。

5. 参数：负责解析URL的应用程序需要这些协议参数来访问资源。否则，另一端的服务器可能就不会为请求提供服务，或者更糟糕的是，提供错误的服务。比如，像FTP这样的协议，有两种传输模式，二进制和文本形式。你肯定不希望以文本形式来传送二进制图片，这样的话，二进制图片可能会变得一团糟。

   为了向应用程序提供它们所需的输入参数，以便正确地与服务器进行交互，URL中有一个参数组件。这个组件就是URL中的名值对列表，由字符“; ”将其与URL的其余部分（以及各名值对）分隔开来。它们为应用程序提供了访问资源所需的所有附加信息。比如：`ftp://ftp.demo.edu/pub/gnu; type=d`在这个例子中，有一个参数type=d，参数名为type，值为d。

   HTTP URL的路径组件可以分成若干路径段。每段都可以有自己的参数。比如：`http://www.demo.com/hammers;sale=false/index.html;graphics=true`这个例子就有两个路径段，hammers和index.html。hammers路径段有参数sale，其值为false。index.html段有参数graphics，其值为true。

6. 查询字符串：很多资源，比如数据库服务，都是可以通过提问题或进行查询来缩小所请求资源类型范围的。

   我们来看一个URL：`http://www.demo.com/check.do?item=12731`这个URL的大部分都与我们见过的其他URL类似。只有问号（?）右边的内容是新出现的。这部分被称为查询（query）组件。URL的查询组件和标识网关资源的URL路径组件一起被发送给网关资源。基本上可以将网关当作访问其他应用程序的访问点。

   按照常规，很多网关都希望查询字符串以一系列“名/值”对的形式出现，名值对之间用字符“&”分隔。比如`http://www.demo.com/check.do?item=12731&color=blue`在这个例子中，查询组件有两个名/值对：item=12731和color=blue。

7. 片段：有些资源类型，比如HTML，除了资源级之外，还可以做进一步的划分。比如，对一个带有章节的大型文本文档来说，资源的URL会指向整个文本文档，但理想的情况是，能够指定资源中的那些章节。为了引用部分资源或资源的一个片段，URL支持使用片段（frag）组件来表示一个资源内部的片段。比如，URL可以指向HTML文档中一个特定的图片或小节。片段挂在URL的右手边，最前面有一个字符“#”。比如：`http://www.demo.com/tools.html#drills`在这个例子中，片段drills引用了Demo的Web服务器上页面/tools.html中的一个部分。这部分的名字叫做drills。

### 1.2.2 HTTP协议的报文

如果说HTTP是因特网的信使，那么HTTP报文就是它用来搬东西的包裹了。HTTP报文是在HTTP应用程序之间发送的数据块。这些数据块以一些文本形式的元信息（meta-information）开头，这些信息描述了报文的内容及含义，后面跟着可选的数据部分。这些报文在客户端、服务器和代理之间流动。术语“流入”、“流出”、“上游”及“下游”都是用来描述报文方向的。

> 不管是请求报文还是响应报文，所有报文都会向下游流动。所有报文的发送者都在接收者的上游。

HTTP报文由起始行（start line）、首部（header）块，以及可选的、包含数据的主体（body）三部分组成。

所有的HTTP报文都可以分为两类：请求报文（request message）和响应报文（response message）。

#### 报文的语法

所有的HTTP报文都可以分为两类：请求报文和响应报文。请求报文会向Web服务器请求一个动作。响应报文会将请求的结果返回给客户端。请求和响应报文的基本报文结构相同

请求报文格式：

~~~
<method> <request-URL> <version>
<headers>
<entity-body>
~~~

响应报文格式：

~~~
<version> <status> <reason-phrase>
<headers>
<entity-body>
~~~

- method（方法）

  客户端希望服务器对资源的动作，常见的有GET、POST

- 请求URL（request-URL）

  命名了资源的URL路径

- 版本（version）

  报文所使用的HTTP版本，格式：`HTTP/<major>.<minor>`例如：`HTTP /1.1`

- 状态码（status）

  三位数字用于描述请求过程中发生的情况

- 原因短语（reason-phrase）

  数字状态码的可读版本，包含行终止序列之前的所有文本

- 头部（header）

  可以有零个或多个首部，每个首部包含一个名字跟着一个冒号，跟着一个可选的空格，跟着一个值，最后是一个CRLF（空行）

- 主体部分（entity-body）

  实体的主体部分包含由一个任意数据组成的数据块。并不是所有的报文都包含主体部分

#### 报文的组成

**起始行**

所有的HTTP报文都以一个起始行作为开始。其中包括请求行和响应行。

请求报文请求服务器对资源进行一些操作。请求报文的起始行，或称为请求行，包含了一个方法和一个请求URL，这个方法描述了服务器应该执行的操作，请求URL描述了要对哪个资源执行这个方法。

响应报文承载了状态信息和操作产生的所有结果数据，将其返回给客户端。响应报文的起始行，或称为响应行，包含了响应报文使用的HTTP版本、数字状态码，以及描述操作状态的文本形式的原因短语。

请求的起始行以方法作为开始，常见的方法如下：

| 方法    | 描述                                               | 是否包含主体 |
| ------- | -------------------------------------------------- | ------------ |
| GET     | 从服务器获取一份文档                               | 否           |
| HEAD    | 只从服务器获取文档首部                             | 否           |
| POST    | 向服务器发送需要处理的数据                         | 是           |
| PUT     | 将请求的主体存储在服务器上                         | 是           |
| TRACE   | 瑞可能经过代理服务器传送到服务器上去的报文机型追踪 | 否           |
| OPTIONS | 决定可以在服务器上执行哪些方法                     | 否           |
| DELETE  | 从服务器上删除文档                                 | 否           |

方法是用来告诉服务器做什么事情的，状态码则用来告诉客户端，发生了什么事情。常见的状态码：

| 整体范围 | 已定义范围 | 分类       |
| -------- | ---------- | ---------- |
| 100-199  | 100-101    | 信息提示   |
| 200-299  | 200-206    | 成功       |
| 300-399  | 300-305    | 重定向     |
| 400-499  | 400-415    | 客户端错误 |
| 500-599  | 500-505    | 服务器错误 |

原因短语是响应起始行中的最后一个组件。它为状态码提供了文本形式的解释。比如，在行HTTP/1.0200 OK中，OK就是原因短语。原因短语和状态码是成对出现的。原因短语是状态码的可读版本，应用程序开发者将其传送给用户，用以说明在请求期间发生了什么情况

**首部**

跟在起始行后面的就是零个、一个或多个HTTP首部字段。HTTP首部字段向请求和响应报文中添加了一些附加信息。本质上来说，它们只是一些名/值对的列表。例如Content-length:19。具体的详细的首部信息可以自行查阅。

首部和方法配合工作，共同决定了客户端和服务器能做什么事情，首部可以主要分为五种类型：

- 通用首部

  这些是客户端和服务器都可以使用的通用首部。可以在客户端、服务器和其他应用程序之间提供一些非常有用的通用功能。比如，Date首部就是一个通用首部：

  ~~~
  Date: Tue, 3 Oct 1999 02:17:03 GMT
  ~~~

- 请求首部

  请求首部是请求报文特有的。它们为服务器提供了一些额外信息，比如客户端希望接收什么类型的数据,例如下面Accept首部就用来告知服务器客户端会接受与其请求相符的任意媒体类型:

  ~~~
  Accept： */*
  ~~~

- 响应首部

  响应报文有自己的首部集，以便为客户端提供信息（比如，客户端在与哪种类型的服务器进行交互）。例如下面Server首部就用来告知客户端它在与openresty服务器进行交互

  ~~~
  server: openresty
  ~~~

- 实体首部

  实体首部指的是用于应对实体主体部分的首部。比如，可以用实体首部来说明实体主体部分的数据类型。例如下面Content-Type首部告知应用程序，数据是以utf-8字符集表示的HTML文档

  ~~~
  Content-Type: text/html; charset=utf-8
  ~~~

- 扩展首部

  扩展首部是非标准的首部，由应用程序开发者创建，但还未添加到已批准的HTTP规范中去。即使不知道这些扩展首部的含义，HTTP程序也要接受它们并对其进行转发。

**主体**

HTTP报文的第三部分是可选的实体主体部分。实体的主体是HTTP报文的负荷。就是HTTP要传输的内容。HTTP报文可以承载很多类型的数字数据：图片、视频、HTML文档、软件应用程序、信用卡事务、电子邮件等。

## 1.3 HTTP协议的演进

> 本小节大部分内容均是Java高并发编程卷一和百度等资料中复制过来的。

HTTP1.1之前，由于无状态等特点，每次请求都需要通过TCP三次握手四次挥手，和服务器重新建立连接。比如某个客户端在短时间多次请求同一个资源，但是服务器并不能区分是否已经响应过用户的请求，所以每次请求都需要重新响应。为了节省资源消耗，HTTP协议也进行了发展和改进：

| 版本     | 产生时间 | 内容                                                         | 发展状况         |
| -------- | -------- | ------------------------------------------------------------ | ---------------- |
| HTTP/0.9 | 1991     | 不涉及数据包传输，规定客户端和服务器之间通信格式，只能GET请求 | 没有作为正式标准 |
| HTTP/1.0 | 1996     | 传输内容格式不限制，增加PUT、PATCH、HEAD、OPTIONS、DELETE    | 正式作为标准     |
| HTTP/1.1 | 1997     | 持久连接（长连接）、节约带宽、HOST域、管道机制、分块传输协议 | 2015年前广泛使用 |
| HTTP/2.0 | 2015     | 多路复用、服务器推送、头信息压缩、二进制协议等等             | 逐渐覆盖市场     |

### HTTP 1.0

HTTP一共有三个版本的演进，第一个版本为0.9但仅支持GET请求，且不支持协议头，因此也支持纯文本。

第二个版本是HTTP1.0。增加了以下特性：

1. 请求和响应头
2. 响应对象以一个响应状态行开始
3. 响应对象不只限于超文本
4. 支持GET、HEAD、POST
5. 支持长连接（以扩展的方式支持）、缓存及身份认证
6. 请求行必须在尾部添加协议版本，且必须包含头消息

HTTP 1.0版本使用Content-Type字段来表示客户端请求服务端的数据是什么格式，服务端使用Content-Type来表示具体的响应体中的媒体类型信息，比如`Content-Type: text/html`。媒体类型（MediaType），全称为互联网媒体类型（Internet Media Type），也叫做MIME（多用途互联网邮件扩展）类型。具体的常用类型请自行查阅相关资料。MIME（多用途互联网邮件扩展）类型，每个值包括一级类型和二级类型，之间用斜杠分隔。除了预定义的类型，厂商也可以自定义类型：`application/vnd.debian.binary-package`。MIME还可以在尾部使用分号，添加参数，比如：`Content-Type: text/html;charset=utf-8`

HTTP1.0工作方式是每次发送一个请求就需要一个TCP连接，当服务器响应后就会关闭这次连接，下一次请求再次建立TCP连接，这点和HTTP0.9相同。但是这样传输性能比较差为了解决这个问题，有些浏览器在请求时，对HTTP 1.0版本进行了扩展，增加了一个非标准的Connection头部字段，如果要对传输层的HTTP连接进行复用，Connection头部的值如下：`Connection: keep-alive`这个头部字段要求服务器不要关闭TCP连接，以便其他HTTP请求复用，同样服务器需要回应这个字段`Connection: keep-alive`如果连接的两端都有“Connection: keep-alive”头部，则一个可以复用的TCP连接就建立了，直到客户端或服务器主动关闭连接。但是，Connection不是标准字段，不同服务端实现的行为可能不一致，因此并不是提高传输性能的最终解决办法。

### HTTP 1.1

Http1.1版本引入了很多关键技术，主要包括：持久连接、管道机制、分块传输编码、字节范围（Range）请求等等。以下是每个新技术的介绍：

HTTP1.1版本的最大变化就是引入了持久连接（长连接），即下层的TCP连接默认不关闭，可以被多个请求复用，而且报文不用声明Connection: keep-alive头部值。在连接建立后，客户端和服务器端都可以进行通信检测，如果发现对方在一段时间没有活动，就可以主动关闭TCP连接。不过，相对规范的做法是，客户端在最后一个请求时，发送带Connection:close请求头的HTTP报文。对于同一个域名（带端口），大多数浏览器允许同时建立6个持久连接，这些持久连接在降低了传输延迟的同时，也提高了带宽的利用率。

HTTP 1.1版本加入了管道机制，在同一个TCP连接里，允许多个请求同时发送，增加了并发性，进一步改善了HTTP协议的效率；举例来说，客户端需要请求两个资源。以前的做法是，在同一个TCP连接里面，先发送A请求，然后等待服务器做出回应，收到后再发出B请求。管道机制则是允许浏览器同时发出A请求和B请求，但是服务器还是按照顺序，先回应A请求，完成后再回应B请求。

HTTP1.1在客户端请求头中新增了Host字段，用来指定服务器的域名。在1.0版本中协议认为每台服务器都绑定唯一的ip地址，因此URL没有主机名，但随着虚拟主机的发展，在一台服务器上可以存在多个虚拟主机，并且共享同一个ip地址甚至端口号，而有了Host字段，就可以将请求发送到同一台服务器上的不同网站。在1.1版本请求消息如果没有Host字段，很多服务器会报400的错误。

HTTP 1.1版本加入了一个新的状态码100（Continue），服务端通过该响应码告知客户端继续发送后面的请求。例如，客户端事先发送一个只带令牌的Authorization头域而不带Body的请求，如果服务器因为权限拒绝了请求，就回送响应码401（Unauthorized）；如果服务器通过权限校验而接收此请求，就回送响应码100，客户端就可以继续发送带实体的完整请求了。

HTTP 1.1版本加入了一些Cache的新特性，当缓存对象的Age超过Expire时，缓存对象变为Stale对象之后，HTTP 1.0版本会直接抛弃Stale对象，HTTP 1.1版本则可以不需要直接抛弃Cache中的Stale对象，而是与源服务器进行重新激活（revalidation）操作

HTTP 1.1版本支持传送内容的一部分，也就是“字节范围请求”。当客户端已经拥有的请求资源的一部分后，只需要跟服务器请求另外的部分资源即可。“字节范围请求”是支持文件断点续传的基础。“字节范围请求”是通过Range头部实现的，HTTP 1.0版本每次传送文件都是只能从文件头开始，即0字节处开始。而在HTTP 1.1版本中，客户端通过`Range:bytes=XX`的请求头部值，表示要求服务器从文件的“XX”字节处开始传送，也就是断点续传。其对应的部分内容的响应码不是200，而是使用专门的响应码206（Partial Content）

> 关于断点续传，我在我自己的笔记上[Java爬虫第2.4节](https://www.cnblogs.com/yhr520/p/15495801.html)中写了一些用HttpClient写的分段下载的案例，所有的分段下载，断点续传本质上都是通过range字段对文件进行拆分。

HTTP 1.1版本支持分块（Chunked）传输编码。分块传输编码（Chunked transfer encoding）是一种新数据传输机制，允许服务端将数据分成多个部分发送到客户端。普通的服务端响应，会将响应数据的长度通过Content-Length字段告诉客户端。但是，使用Content-Length字段的前提条件是，服务器发送回应之前，必须知道回应的数据长度。而对于一些很耗时的动态操作来说，这意味着，服务器要等到所有操作完成，才能发送数据，显然这样的效率不高。更好的处理方法是，产生一块数据，就发送一块，采用“流模式”发送取代“缓存模式”发送。因此，HTTP 1.1版本规定请求或者响应报文可以不使用Content-Length字段告知长度，而使用分块传输编码（Chunked transfer encoding）字段。只要请求或回应的头部有Transfer Encoding字段，就表明数据将由数量未定的数据块组成。`Transfer-Encoding: chunked`每个分块报文的非空的数据块之前，会有一个16进制的数值，表示当前块的长度。最后是一个大小为0的块，就表示本次回应的数据发送完了。分块传输编码（Chunked transfer encoding）的具体传输规则为：

1. 在头部加入 Transfer-Encoding: chunked 之后，就代表这个报文采用了分块编码。这时，报文中的实体需要改为用一系列分块来传输。
2. 每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括分块长度后面的结尾CRLF(\r\n)的长度，也不包括分块数据后面的结尾CRLF(\r\n)的长度。
3. 最后一个分块的长度值必须为 0，对应的分块数据没有内容，表示所有的Body数据传输完成

下面举一个例子：

~~~
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked
25
This is the data in the first chunk
1C
and this is the second one
3
con
8
sequence
0
~~~

上面示例中的20、1C、3、8、0都是16进制的分片内容净长度，且后面的报文不需要HTTP头部，因为1.1版本的连接是持久的，接收端最后通过一个长度为0的分片，表示当前的Body在这里结束即可。

### HTTP 2.0

HTTP协议的2.0版本是一个二进制协议，二进制更易于Frame(帧、数据包)的传输。HTTP 1.x版本在应用层以纯文本的形式进行通信，而HTTP 2.0将所有的传输信息分割为更小的消息和数据帧，并对它们采用二进制格式编码。这样，客户端和服务端都需要引入新的二进制编码和解码的机制。

HTTP2的协议有很多Frame，最重要的两个是Data帧和Headers帧，在1.0中的报文首部会被封装在Headers帧中，而请求体会被封装在Data帧中。HTTP2.0并没有改变1.0的语义，而是去改进传输性能，实现低延迟和高吞吐量，那么如何实现二进制传输呢？就是通过在应用层和传输层之间增加一个二进制分帧层，并进行二进制编码。HTTP2.0都在一个连接上完成，这个连接可以承载任意数量的双向传输流。HTTP2.0主要有首部压缩、多路复用、并行双向传输、服务端推送等特点。

HTTP2.0协议在客户端和服务器端使用“首部（请求头）表”来跟踪和存储之前发送的请求头键-值对，对于相同的数据，不再通过每次请求和响应发送；通信期间几乎不会改变的通用键-值对的值(如用户代理等)，所以请求头只需发送一次即可。此时所有首部都自动使用之前请求发送的首部。一旦请求的首部发生变化了，那么只需要Headers帧里发送变化了首部，将新增或修改的首部帧会被追加到“首部表”即可。首部表中的键值对，在HTTP/2.0协议的TCP连接存续期内始终存在，由客户端和服务器共同渐进地更新

HTTP2.0协议的多路复用，指的对多资源的请求可以在一个TCP连接上完成。HTTP2.0协议把HTTP协议通信的基本单位缩小为一个一个的帧，这些帧对应着逻辑流中的消息，并行地在同一个TCP连接上双向交换消息。HTTP性能的关键在于低延迟而不是带宽利用率低。大多数HTTP连接的时间都很短，数据传输是突发性的，但是，TCP传输只有在长连接并且传输大块数据时，其效率才是最高的。HTTP2.0协议通过让所有数据流共用同一个连接，可以更有效地让TCP连接高带宽，也能真正的服务于HTTP的性能提升。多资源单链接的多路复用方式，在服务器和网络传输的层面都得到了好处：比如可以减少服务链接压力，内存占用少了，连接吞吐量大了；由于 TCP 连接减少而使网络拥塞状况得以改观；TCP慢启动时间减少，拥塞和丢包恢复速度更快。

在HTTP2.0协议中，客户端和服务器可以把HTTP消息分解为互不依赖的帧，然后乱序发送，最后再在另一端把它们重新组合起来。注意，同一连接上有多个不同方向的数据流在传输。客户端可以一边乱序发送消息流，也可以一边接收者服务器的响应流，而服务器同理。把HTTP消息分解为独立的帧，双向交错发送，然后在另一端重新组装，是HTTP/2.0最重要的一项增强。该机制会在整个Web技术栈中引发一系列连锁反应, 从而带来巨大的性能提升，大致的原因是：可以并行交错地发送请求，请求之间互不影响；可以并行交错地发送响应，响应之间互不干扰；只使用一个连接即可并行发送多个请求和响应；消除不必要的延迟，从而减少页面加载的时间。

在HTTP2.0协议中，新增的一个强大的新功能，就是服务器可以对一个客户端请求发送多个响应。或者说服务器还可以额外向客户端推送资源，而无需客户端明确地请求。在服务端主动推送这一点上，HTTP2.0协议和WebSocket协议有点儿类似。使用HTTP2.0协议的前提是需要Web服务器和浏览器双方都支持，才可以启用HTTP2.0协议，如果任何一端不匹配，则会回退到HTTP/1.1协议。

## 1.4 基于Netty实现简单的Web服务器

Netty天生是异步事件驱动的架构，在性能和可靠性上都十分优异，非常适合作为WEB服务器使用，相比于传统的Tomcat的Web容器，基于Netty的Web服务器更小巧灵活且定制性更强。我们这里基于Netty实现的Web服务器非常简单就是将HTTP请求的请求头，请求体等内容回显出来。

此简单web服务器的Netty流水线如下图：

![image-20220811124935470](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811124935470.png)

1.4.1-1.4.3为netty关于Http的相关知识介绍。

### 1.4.1 Netty处理HTTP请求

通常HTTP协议通信过程中，客户端和服务器的交互过程如下：

1. 客户端向服务器端发送HTTP请求
2. 服务端对HTTP请求进行解析
3. 服务端发送HTTP响应报文
4. 客户端解析HTTP响应的应用层协议内容

在Netty中已经内置了这些编解码的处理器，大致如下：

- HttpRequestDecoder：HTTP请求编码器，这是一个入站处理器，间接的继承了ByteToMessageDecoder，将ByteBuf缓冲区解码成HttpRequest和HttpContent实例。并且解码时会处理好分块类型和固定长度类型的HTTP请求报文。

- HttpResponseEncoder：响应编码器，是一个出站处理器，把代表响应HttpResponse和HttpContent内容实例编码成ByteBuf。

- HttpServerCodec：编解码器。

- HttpObjectAggregator：HttpObject实例聚合器，这是入站处理器，通过此类，可以把HttpMessage头部实例和一个或多个HttpContent内容实例最终聚合成FullHttpRequest。

  ![image-20220811140542612](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220811140542612.png)

  > HttpRequest、HttpContent、FullHttpRequest等都是HttpObject的子类。

  在请求请Request Body处理过程中，会涉及到Content-Length和Trunked两种类型的请求体，但是其处理差异被HttpRequestDecoder协议解码器所屏蔽，它们的最终出站对象是一致的，当HttpObjectAggregator发现有入站包为LastHttpContent实例入站时，代表HTTP请求数据协议解析完成，此时，会将所收到的全部HttpObject实例，封装一个FullHttpRequest整体请求实例。通过聚合器HttpObjectAggregator处理之后，输出的都是FullHttpRequest实例。然后可以通过该FullHttpRequest实例获取到所有与HTTP请求的所有内容。

  > 这里的Thunked简单理解是一个系列，里面有很多块，该怎么解析。
  >
  > 而chunked简单理解是里面的小块怎么解析。这里仅作简单了解，不深入研究。

- QueryStringDecoder：把HTTP请求URI分成Path路径和Key-Value参数键值对，同一次请求该解码器仅能使用一次。

**综上，整体来说，如果想对HTTP请求报文进行读取，只要在流水线上配置好两个内置处理器HttpRequestDecoder和HttpObjectAggregator即可。**

### 1.4.2 Netty的报文解码

通过内置处理器HttpRequestDecoder和HttpObjectAggregator对HTTP请求报文进行解码后，会将HTTP请求封装成一个FullHttpRequest实例，然后发送给下一站。

Netty内置的HTTP请求报文对应的类主要有以下几个：

1. FullHttpRequest：包含整个HTP请求的信息，包含对HttpRequest首部和HttpContent请求体的组合
2. HttpRequest：请求首部，主要包含对HTTP请求行和请求头Header的封装
3. HttpContent：是对Http请求体的封装，本质上就是一个ByteBuf缓冲区实例。如果ByteBuf长度固定而请求体又太长的话那么就会产生多个HttpContent对象，解码时最后一个返回LastHttpContent，表示对请求体的解码已经完成。
4. HttpMethod：对HTTP请求方法的封装
5. HttpVersion：对HTTP版本的封装，该类定义了HTTP/1.0和HTTP/1.1两个协议版本
6. HttpHeaders：包含对HTTP报文请求头Header的封装和相关操作。

各个类与HTTP协议的对应如下图（图片来自于Java高并发核心编程卷一）：

![image-20220818155441217](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220818155441217.png)

一般来说HTTP字节流可能会被分成多个BuyeBuf包，一般有几种策略处理拆包：

1. 定长分包策略：接收端和发送端都按照固定长度进行收发
2. 长度域分包策略：使用LengthFieldBasedFrameDecoder接收，发送端先发送4个字节表示消息的长度
3. 分隔符分隔，使用LineBasedFrameDecoder解码器通过换行进行拆包，或者通过特定字符。

Netty结合了2和3策略完成了HTTP报文的拆包，请求头使用分隔符分包策略，以特定分隔符`\r\n`进行拆包，对于请求体使用长度域分包策略，按照请求体的内容长度进行拆包。具体来说就是这样：

1. 处理HTTP请求行，因为请求行的边界时CRLF，所以读到CRLF则表示请求行已经完成读取
2. 处理请求头，因为Header边界时CRLF，所以读到一个CRLF是一个请求头读取完成，如果连续两个CRLF则表示全部请求头已完成读取。
3. 处理请求体，一般按照Content-Length处理，如果没有此请求头，则表示是Chunked块编码报文，具体解析方式可以参考Thunked协议

> 为了减少内存复制，Netty使用了组合缓冲区(CompositeByteBuf)，比如FullHttpMessage实现类，内部就是一个CompositeByteBuf组合缓冲区实例，该组合缓冲区会将HttpRequest内部的ByteBuf、HttpContent内部的ByteBuf都组合在一起，作为最终的HTTP报文缓冲区，从而避免数据拷贝。如下图（图片来自Java高并发核心编程卷一）：
>
> ![image-20220818164245710](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220818164245710.png)

### 1.4.3 Netty的HTTP响应

Netty的HTTP响应的处理流程，只需要在流水线装配HttpResponseEncoder编码器即可，该编码器是一个出站处理器，输入的是FullHttpResponse，输出的是ByteBuf。且该编码器按照HTTP协议对入站FullHttpResponse实例的响应行、响应头、响应体进行序列化，通过Header去判断是否含有Content-Length头或Trunked头，然后将响应体按照长度规则进行序列化。

### 1.4.4 回显业务处理器

经过上面的介绍，其实我们这个简单的web服务器，本质上我们只需要将Netty为我们处理好并传过来的FullHttpRequest进行处理，将需要回显的内容提取出来，然后封装成FullHttpResponse传到出站处理器即可，剩下的工作netty都会帮我们完成。

首先需要编写业务处理器，主要思路就是拿到请求uri、请求方法、请求头以及请求参数的信息，代码如下：

```java
public class EchoHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        if (!request.decoderResult().isSuccess()) {
            EchoUtil.sendError(ctx, HttpResponseStatus.BAD_REQUEST);
            return;
        }
        EchoUtil.cacheHttp(ctx, request);
        Map<String, Object> result = new HashMap<>();
        result.put("request_uri", request.uri());
        result.put("request_method", request.method().toString());
        HttpHeaders headers = request.headers();
        Map<String, Object> headerMap = new HashMap<>();
        for (Map.Entry<String, String> next : headers.entries()) {
            headerMap.put(next.getKey(), next.getValue());
        }
        result.put("request_header", headerMap);
        Map<String, Object> params = getParams(request);
        result.put("request_params", params);
        if (HttpMethod.POST.equals(request.method())) {
            Map<String, Object> data = getPostData(request);
            result.put("request_data", data);
        }
        String jsonString = JSONObject.toJSONString(result);
        EchoUtil.sendJsonContent(ctx,jsonString);
    }

    /**
     * 获取Post参数
     */
    private Map<String, Object> getPostData(FullHttpRequest request) {
        Map<String, Object> data = null;
        String contentType = request.headers().get("Content-Type").trim();
        if (contentType.contains("application/x-www-form-urlencoded")) {
            data = getFormBody(request);
        } else if (contentType.contains("multipart/form-data")) {
            data = getFormBody(request);
        } else if (contentType.contains("application/json")) {
            data = getFormJson(request);
        } else if (contentType.contains("text/plain")) {
            ByteBuf content = request.content();
            byte[] bytes = new byte[content.readableBytes()];
            content.readBytes(bytes);
            String text = new String(bytes, StandardCharsets.UTF_8);
            data = new HashMap<>();
            data.put("text", text);
        }
        return data;
    }

    /**
     * 获取uri参数
     */
    private Map<String, Object> getParams(FullHttpRequest request) {
        Map<String, Object> paramsMap = new HashMap<>();
        //把uri的参数分割成key-value形式
        QueryStringDecoder decoder = new QueryStringDecoder(request.uri());
        Map<String, List<String>> parameters = decoder.parameters();
        for (Map.Entry<String, List<String>> entry : parameters.entrySet()) {
            paramsMap.put(entry.getKey(), entry.getValue().get(0));
        }
        return paramsMap;
    }

    /**
     * 获取form表单数据
     */
    private Map<String, Object> getFormBody(FullHttpRequest request) {
        Map<String, Object> data = new HashMap<>();
        try {
            HttpPostRequestDecoder decoder = new HttpPostRequestDecoder(new DefaultHttpDataFactory(DefaultHttpDataFactory.MINSIZE), request, StandardCharsets.UTF_8);
            List<InterfaceHttpData> bodyData = decoder.getBodyHttpDatas();
            if (bodyData == null || bodyData.isEmpty()) {
                if (request.content().isReadable()) {
                    String json = request.content().toString(StandardCharsets.UTF_8);
                    data.put("body", json);
                }
                return data;
            }
            for (InterfaceHttpData bodyDatum : bodyData) {
                if (bodyDatum.getHttpDataType() == InterfaceHttpData.HttpDataType.Attribute) {
                    MixedAttribute attribute = (MixedAttribute) bodyDatum;
                    data.put(attribute.getName(), attribute.getValue());
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return data;
    }

    /**
     * 获取json数据
     */
    private Map<String, Object> getFormJson(FullHttpRequest request) {
        Map<String, Object> params = new HashMap<>();

        ByteBuf content = request.content();
        byte[] bytes = new byte[content.readableBytes()];
        content.readBytes(bytes);
        String strContent = new String(bytes, StandardCharsets.UTF_8);

        JSONObject jsonObject = JSONObject.parseObject(strContent);
        for (Object key : jsonObject.keySet())
        {
            params.put(key.toString(), jsonObject.get(key));
        }

        return params;
    }
}
```

然后附带的工具类代码如下：

```java
public class EchoUtil {
    public static final AttributeKey<HttpVersion> VERSION_KEY =
            AttributeKey.valueOf("VERSION");
    public static final AttributeKey<Boolean> KEEP_ALIVE_KEY =
            AttributeKey.valueOf("KEEP_ALIVE_KEY");

    /**
     * 通过channel缓存Http协议版本，以及是否是长连接
     */
    public static void cacheHttp(ChannelHandlerContext context, FullHttpRequest request) {
        if (context.channel().attr(KEEP_ALIVE_KEY).get() == null) {
            context.channel().attr(VERSION_KEY).set(request.protocolVersion());
            boolean keepAlive = HttpUtil.isKeepAlive(request);
            context.channel().attr(KEEP_ALIVE_KEY).set(keepAlive);
        }
    }

    public static void sendError(ChannelHandlerContext context, HttpResponseStatus status) {
        FullHttpResponse response = new DefaultFullHttpResponse(getHttpVersion(context), status, Unpooled.copiedBuffer("Fail" + status + "\r\n", StandardCharsets.UTF_8));
        response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain;charset=UTF-8");
        sendAndCleanupConnection(context,response);
    }

    private static HttpVersion getHttpVersion(ChannelHandlerContext ctx) {
        HttpVersion version;
        if (isHTTP1_0(ctx)) {
            version = HttpVersion.HTTP_1_0;
        } else {
            version = HttpVersion.HTTP_1_1;
        }
        return version;
    }

    public static boolean isHTTP1_0(ChannelHandlerContext ctx) {

        HttpVersion version =
                ctx.channel().attr(VERSION_KEY).get();
        if (null == version) {
            return false;
        }
        return version.equals(HttpVersion.HTTP_1_0);
    }

    /**
     * 发送响应
     */
    public static void sendAndCleanupConnection(ChannelHandlerContext ctx, FullHttpResponse response) {
        boolean keepAlive = ctx.channel().attr(KEEP_ALIVE_KEY).get();
        HttpUtil.setContentLength(response, response.content().readableBytes());
        if (!keepAlive) {
            // 如果不是长连接，设置 connection:close 头部
            response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaders.Values.CLOSE);
        } else if (isHTTP1_0(ctx)) {
            // 如果是1.0版本的长连接，设置 connection:keep-alive 头部
            response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderNames.KEEP_ALIVE);
        }
        //发送内容
        ChannelFuture channelFuture = ctx.channel().writeAndFlush(response);

        if (!keepAlive) {
            // 如果不是长连接，发送完成之后，关闭连接
            channelFuture.addListener(ChannelFutureListener.CLOSE);
        }
    }

    /**
     * 发送Json数据
     */
    public static void sendJsonContent(ChannelHandlerContext ctx, String json){
        HttpVersion version = getHttpVersion(ctx);
        FullHttpResponse response = new DefaultFullHttpResponse(
                version, HttpResponseStatus.OK, Unpooled.copiedBuffer(json, StandardCharsets.UTF_8));
        response.headers().set(CONTENT_TYPE, "application/json; charset=UTF-8");
        sendAndCleanupConnection(ctx, response);
    }
}
```

最后完成服务端启动类的编写即可：

```java
@Slf4j
public class EchoServer {

    private static final Integer port = 8999;

    public static void main(String[] args) {
        start();
    }

    private static void start() {
        // 创建连接监听reactor 线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // 创建连接处理 reactor 线程组
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap
                    .group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new HttpRequestDecoder());
                            ch.pipeline().addLast(new HttpResponseEncoder());
                            ch.pipeline().addLast(new HttpObjectAggregator(65535));
                            ch.pipeline().addLast(new EchoHandler());
                        }
                    });
            Channel channel = bootstrap.bind(port).sync().channel();
            log.info("服务已经启动，正在监听{}端口", port);
            channel.closeFuture().sync();
        } catch (Exception e) {
            log.error("发生异常，错误信息{}", e.getMessage());
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }

    }
}
```

### 1.4.5 测试

我们可以使用postman进行测试，可以测试GET请求、POST请求及POST请求的各种参数类型。

![image-20220822140231349](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220822140231349.png)

# 二、高并发HTTP的核心原理

## 2.1 HTTP复用场景

客户端和服务器之间，使用HTTP短连接对用户体验和性能影响不大，单个客户端和服务器通信不频繁场景下，短连接性能还是很高的。但是由于微服务技术的发展，分布式应用内部会进行大量的、高频率的内部RPC调用或HTTP通信，频繁的建立和拆除连接，会降低整体的效率和性能，这时就需要HTTP复用技术。在Java分布式应用的架构下，涉及到HTTP连接复用的高并发场景大概有几种：

1. 反向代理Nginx和Java WEB服务之间的HTTP高并发通信

   传统的Nginx+Tomcat架构，一般Tomcat并发上升后，会引入Nginx进行反向代理和负载均衡。在此架构下Nginx和Tomcat之间进行请求转发时，对速度和性能要求很高，这时需要TCP连接通道具备可复用的能力，以提升响应效率和并发能力。

2. 微服务网关与微服务Provider之间的HTTP高并发通信

   在使用Nginx+Spring Cloud架构时，外部网关(Nginx)和内部网关(GateWay)以及内部网关和微服务之间都存在着反向代理。同时这些角色之间的HTTP通信和传输对性能要求较高，所以这部分也需要HTTP的传输层通道具备可复用的能力，以提高高并发的能力。

3. 微服务Provider之间的RPC或HTTP高并发通信

   在微服务架构中，微服务实例之间的RPC远程调用也需要HTTP协议，因此也需要其HTTP下层TCP传输层的连接通道复用，提升相应效率和高并发能力。

4. Java通过HTTP客户端访问REST接口服务

   在实际开发中，Java应用通常会涉及到对ESB企业服务总线注册的REST接口服务，或者Java应用会涉及到对其他的独立REST接口服务的访问。一般情况下，Java应用会使用本地HTTP客户端发起请求，从而获得REST访问结果。本地HTTP客户端和远程REST接口服务之间，需要进行频繁的HTTP通信，因此Java客户端与REST接口服务之间的HTTP通信也需要下层TCP传输层的连接通道具备可复用的能力，以提升请求效率和速度。

## 2.2 传输层TCP协议

### 2.2.1 TCP/IP协议的分层模型

国际标准化组织为了使网络应用更普及，推出了OSI参考模型。其含义是为所有的公司提供同一个规范来控制网络，所有的公司都遵循规范，网络就能互联了。OSI定义了七层框架，每一层实现了自己的功能和协议，并完成与相邻层的接口通信：

| 框架       | 协议                                   |
| ---------- | -------------------------------------- |
| 应用层     | HTTP、SMTP、FTP、Telnet、SSH等等       |
| 表示层     | XDR、ASN.1、SMB、NCP等等               |
| 会话层     | ASAP、SSH、RPC、Sockets等等            |
| 传输层     | TCP、UDP、TLS、SCTP、ATP等等           |
| 网络层     | IP、ICMP、IGMP、RIP等等                |
| 数据链路层 | 以太网、令牌环、HDLC、ISDN、802.11等等 |
| 物理层     | 网线、光缆、无线电等等                 |

TCP/IP协议是互联网最基本的协议，在一定程度上参考了OSI模型，OSI一共有七层，从下到上分别是物理层、数据链路层、网络层、运输层、会话层、表示层和应用层，这显然有些复杂，因此在TCP/IP协议中七层被简化成了四层，TCP/IP与OSI对应关系如下：

![image-20220820220154076](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220820220154076.png)

- TCP/IP协议的应用层

  应用层包括所有和应用程序协同工作、并利用基础网络交换应用程序的业务数据的协议。该层协议提供的服务能直接支持用户应用，比如HTTP万维网服务、FTP文件传输、SMTP电子邮件、SSH安全远程登录、DNS域名解析等许多协议。

- TCP/IP协议的传输层

  传输层解决了端到端的可靠性问题，能保证数据准确的可靠的到达目的地，甚至保证数据按照正确的顺序到达目的地。传输层功能主要如下：

  - 为端对端连接提供传输服务；
  - 这种传输分为可靠（典型的TCP）和不可靠（UDP）
  - 为端对端连接提供流量控制、差错控制、Qos服务质量管理等服务

  传输层主要有TCP传输控制协议和UDP用户数据报协议。TCP是一个面向连接的可靠的传输协议，他提供一种字节流，能保证数据据完整、无损并且按顺序到达。TCP尽量连续不断地测试网络的负载并且控制发送数据的速度以避免网络过载，并试图将数据按照规定的顺序发送。而UDP协议是一个无连接的数据报协议，是一个“尽力传递”和“不可靠”协议，不会对数据包是否已经到达目的地进行检查，并且不保证数据包按顺序到达。总体来说，TCP协议传输效率低，但可靠性强；UDP协议传输效率高，但可靠性略低，适用于传输可靠性要求不高、体量小的数据（比如QQ聊天数据）。

- TCP/IP协议的网络层

  TCP/IP协议网络层的作用是在复杂的网络环境中为要发送的数据报找到一个合适的路径进行传输。网络层负责将数据传输到目标地址，目标地址可以是多个网络通过路由器连接而成的某一个地址。网络层负责寻找合适的路径到达对方计算机，并把数据帧传送给对方，网络层还可以实现拥塞控制、网际互连等功能。网络层协议的代表包括：ICMP、IP、IGMP等。

- TCP/IP协议的链路层

  链路层有时也称作数据链路层或网络接口层，用来处理连接网络的硬件部分。该层既包括操作系统硬件的设备驱动、NIC（网卡）、光纤等物理可见部分，还包括连接器等一切传输媒介。在这一层，数据的传输单位为比特。其主要协议有ARP、RARP等。

### 2.2.2 TCP报文传输原理

TCP/IP进行网络通信时，数据包会按照分层顺序与对方通信，发送端从应用层向下走，接收端从链路层向上走。从客户端的数据，每一帧的顺序都是：应用层->运输层->网络层->链路层->链路层->网络层->运输层->应用层，如下图：

![image-20220820222030902](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220820222030902.png)

数据在通过互联网传输时需要添加标识，否则数据会乱，所以数据发送时都会加上特定标识，这个过程就叫做数据的封装，在数据使用的时候在去掉特定标识，这一过程叫分用，如上图。

在**传输层封装**时，IP首部会标识处理数据的协议类型，无论TCP还是UDP都用一个16位端口号来表示不同的应用程序，并且将源端口号和目标端口号存入报文的首部。在**网络层封装**时IP首部会标识处理数据的协议类型，或者说标识出网络层数据帧所携带的上层数据类型，如TCP、UDP、ICMP、IP、IGMP等等。 具体会在IP首部中存入一个长度为8位的数值，称作协议域： 1表示为ICMP协议、2表示为IGMP协议、6表示为TCP协议、17表示为UDP等等。IP首部还会标识发送方地址（源IP）和接收方地址（目标IP）。在**链路层封装**时，网络接口分别要发送和接收IP、ARP和RARP等多种不同协议的报文，因此也必须在以太网的帧首部中加入某种形式的标识，以指明所处理的协议类型，为此，以太网的报文帧的首部也有一个16位的类型域，标识出以太网数据帧所携带的上层数据类型，如IPv4、ARP、IPV6、PPPoE等等。

总体上数据封装和分用就是发送端每一层增加首部，接收端每一层删除首部。数据包在不同物理网络之间的传输过程中，网络层会通过路由器去对不同的网络之间的数据包进行存储、分组转发处理。物理网络之间通过路由器进行互连，随着增加不同类型的物理网络，可能会有很多个路由器，但是对于应用层来说仍然是一样的，TCP协议栈为大家屏蔽了物理层的复杂性。

### 2.2.3 TCP协议的报文格式

整体TCP/IP协议栈，共同配合一起解决数据如何通过许许多多个点对点通路，顺利传输到达目的地。一个点对点通路被称为一“跳”（hop），通过TCP/IP协议栈，网络成员能够在许多“跳”的基础上建立相互的数据通路。TCP/IP的数据帧格式如下图中的TCP段所示（图片来自HTTP权威指南）：

![image-20220225091241338](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220225091241338.png)

我们这里介绍一下上图中TCP段数据帧字段的含义。

- 源端口：源端口号表示报文的发送端口，占16位。源端口和源IP地址组合起来，可以标识报文的发送地址。

- 目的端口：目的端口号表示报文的接收端口，占16位。目的端口和目的IP地址相结合，可以标识报文的接收地址。TCP协议是基于IP协议的基础上传输的，TCP报文中的源端口号+源IP，与TCP报文中的目的端口号+目的IP一起，组合起来唯一性的确定一条TCP连接。

- 序号：序号占32位，发起方发送数据时都需要标记序号，序号的语义与SYN控制标志的值有关：

  - 当SYN=1时，当前为连接建立阶段，此时的序号为初始序号ISN，通过算法来随机生成序号
  - 当SYN=0时，是数据传输正式开始时，第一个报文的序号为 ISN + 1，后面的报文的序号，为前一个报文的SN值+TCP报文的净荷字节数(不包含TCP头)。比如，如果发送端发送的一个TCP帧的净荷为12byte，序号为5，则发送端接着发送的下一个数据包的时候，序号的值应该设置为5+12=17

  在数据传输过程中，TCP协议通过序号对上层提供有序的数据流。发送端可以用序号来跟踪发送的数据量；接收端可以用序号识别出重复接收到的TCP包，从而丢弃重复包；对于乱序的数据包，接收端也可以依靠序号对其进行排序。

- 确认序号：确认序号标识了报文接收端期望接收的字节序列。如果设置了ACK控制位，确认序号的值表示一个准备接收的包的序列码，注意，它所指向的是准备接收的包，也就是下一个期望接收的包的序列码。具体的ACK值如下图（图片来自Java高并发卷一）正常传输部分所示：

  ![image-20220822144053789](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220822144053789.png)

  在上图的左边部分，Server第1个ACK包的ACK Number值为1001，是通过Client第1个包的SN+包净荷=1+1000计算得到，表示期望第2个Client包的SN序号为1001；Server第2个ACK包的ACK Number值为2001，为Client第2个包的SN+包净荷=2001，表示期望第3个Server包的SN为2001，以此类推。

  如果发生错误，假设Server在处理Client的第二个发送包异常，Server仍然回复一个ACK Number值为1001的确认包，则Client的第二个数据包需要重复发送，具体的ACK值如上图右边的正常传输部分所示。

  只有控制标志的ACK标志为1时，数据帧中的确认序号ACK Number才有效。TCP协议规定，连接建立后，所有发送的报文的ACK必须为1，也就是建立连接后，所有报文的确认序号有效。如果是SYN类型的报文，其ACK标志为0，故没有确认序号。

  > 注意：ACK在这里是确认序号，在标志位中也有ACK

- 头部长度：该字段占用4位，用来表示TCP报文首部的长度，单位是4bit位。其值所表示的并不是字节数，而是头部的所含有的32bit的数目（或者倍数），或者4个字节的倍数，所以TCP头部最多可以有60字节(4*15=60)。没有任何选项字段的TCP头部长度为20字节，所以其头部长度为5，可以通过20/4=5计算得到。

- 保留：头部长度后面预留的字段长度为6位，作为保留字段

- 控制标志：控制标志共6个bit位，具体的标志位为：URG、ACK、PSH、RST、SYN、FIN。6个标志位的说明，如下图所示：

  ![image-20220822145245845](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220822145245845.png)

  在连接建立的三次握手过程中，若只是单个SYN置位，表示的只是建立连接请求。如果SYN和ACK同时置位为1，表示的建立连接之后的响应。

- 窗口大小：长度为16位，共2个字节。此字段用来进行流量控制。流量控制的单位为字节数，这个值是本端期望一次接收的字节数。

- TCP校验和：长度为16位，共2个字节。对整个TCP报文段，即TCP头部和TCP数据进行校验和计算，接收端用于对收到的数据包进行验证。

- 紧急指针：长度为16位，2个字节。它是一个偏移量，和SN序号值相加表示紧急数据最后一个字节的序号。

总体来说，TCP协议的可靠性，主要通过以下几点来保障：

1. 应用数据分割成TCP认为最适合发送的数据块。这部分是通过MSS（最大数据包长度）选项来控制的，通常这种机制也被称为一种协商机制，MSS规定了TCP传往另一端的最大数据块的长度。值得注意的是，MSS只能出现在SYN报文段中，若一方不接收来自另一方的MSS值，则MSS就定为536字节。一般来讲，MSS值还是越大越好，这样可以提高网络的利用率。
2. 重传机制。设置定时器，等待确认包，如果定时器超时还没有收到确认包，则报文重传。
3. 对首部和数据进行校验。
4. 接收端对收到的数据进行排序，然后交给应用层。
5. 接收端丢弃重复的数据。
6. TCP还提供流量控制，主要是通过滑动窗口来实现流量控制

### 2.2.4 TCP的三次握手

在两个设备要建立连接发送数据之前，双方都需要做一些准备工作

1. 最开始客户端和服务器处于CLOSE状态。客户端主动打开连接，服务器被动打开连接。TCP服务器创建传输控制块TCB，时刻接收客户端的连接请求，此时服务器进入LISTEN状态

2. 一次握手

   Client将标志位SYN置为1，随机产生一个值seq=J（seq是确认序号），并将该数据包发送给Server， Client进入SYN_SENT状态，等待Server确认。

3. 二次握手

   Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位 SYN和ACK都置为1，ack=J+1（这里的ack是确认序号，ACK是标志位），随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求 ，Server进入SYN_RCVD状态。

4. 三次握手

   Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK 置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以 开始传输数据了。

   ![TCP三次握手20220225](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/TCP%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B20220225.png)

### 2.2.5 TCP的四次挥手

终止TCP连接，就是指断开一个TCP连接时，需要客户端和服务端总共发送4个包以确认连接的断开。

- **第一次挥手**：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入 FIN_WAIT_1状态
- **第二次挥手**：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
- **第三次挥手**：在发送完成ACK报文后，Server还可以继续完成业务数据的发送，待剩余数据发送完成后，或者CLOSE-WAIT（关闭等待）截止后，Server会向Client发送一个FIN+ACK结束响应报文，表示Server的数据都发送完了，然后Server进入LAST_ACK状态。
- **第四次挥手**：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。Client进入TIME_WAIT状态后在等待2MSL的时间后，如果期间没有收到其他报文，则证明对方已经正常关闭，Client连接最终关闭。

![TCP四次挥手20220225](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/TCP%E5%9B%9B%E6%AC%A1%E6%8C%A5%E6%89%8B20220225.png)

> 为什么挥手是四次？因为服务端在LISTEN状态下，收到建立连接请求的SYN报文后，把ACK和SYN放在一个报文里 发送给客户端。而关闭连接时，当收到对方的FIN报文时，仅仅表示对方不再发送数据了但是还能接收数据，己方也未必全部数据都发送给对方了，所以己方可以立即close，也可以发送一些数据给对方后，再发送FIN报文给对方来表示同意现在关闭连接，因此，己方ACK和FIN一般都会分开发送。

## 2.3 TCP连接状态

TCP连接的11种状态，具体如下：

1. LISTEN：表示服务器端的某个ServerSocket监听连接处于监听状态，可以接受客户端的连接。
2. SYN_SENT ：这个状态与SYN_RCVD 状态相呼应，当客户端Socket连接的底层开始执行connect(…)方法发起连接请求是，本地连接会进入到SYN_SENT 状态，并发送SYN报文，并等待服务端的发送三次握手中的SYN+ACK报文。SYN_SENT状态表示客户端连接已发送SYN报文。
3. SYN_RCVD ：表示服务端ServerSocket接收到了来自客户端连接的SYN报文。在正常情况下，这个状态是ServerSocket连接在建立TCP连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用netstat指令很难看到这种状态，除非故意写一个监测程序，将三次TCP握手过程中最后一个ACK报文不予发送。当TCP连接处于此状态时，再收到客户端的ACK报文，它就会进入到ESTABLISHED状态。
4. ESTABLISHED ：表示TCP连接已经成功建立
5. FIN_WAIT_1 ：当连接处于ESTABLISHED状态时，想主动关闭连接，主动关闭方会调用底层的close(…)方法，要求主动关闭连接，此时主动断开方进入到FIN_WAIT_1状态。而当对方回应ACK报文后，则主动方进入到FIN_WAIT_2 状态。当然在实际的正常情况下，无论对方处于任何种情况下，都应该马上回应ACK报文，所以FIN_WAIT_1状态一般是比较难见到的，而FIN_WAIT_2状态有时仍可以用netstat指令看到。
6. FIN_WAIT_2 ：主动断开方处于FIN_WAIT_1状态后，如果收到对方的ACK报文，主动方会进入FIN_WAIT_2状态，此状态下的双向通道处于半连接（半开）状态，即被动方还可以传递数据过来，但主动方不可再发送数据出去。需要注意的是，FIN_WAIT_2是没有超时的（不像TIME_WAIT状态），这种状态下如果对方不发送FIN+ACK关闭响应（不配合完成4次挥手过程），那 FIN_WAIT_2状态将一直保持，该连接会一直被占用，资源不会被释放，越来越多的处于FIN_WAIT_2状态的半连接堆积，会导致操作系统内核崩溃。
7. TIME_WAIT ：该状态表示主动断开方已收到了对方的FIN+ACK关闭响应，并发送出了ACK报文。 TIME_WAIT状态下的主动方TCP连接会等待2MSL的时间，然后回到CLOSED 状态。如果FIN_WAIT_1状态下，收到了对方同时带FIN+ACK关闭响应报文，可以直接进入到TIME_WAIT状态，而无须经过FIN_WAIT_2状态。这种情况下，四次挥手变成三次挥手了。
8. CLOSING ：这种状态在实际情况中应该很少见，属于一种比较罕见的例外状态。正常情况下，当一方发送FIN报文后，理论上应该先收到（或同时收到）对方的ACK报文，再收到对方的FIN+ACK关闭响应报文。但是CLOSING状态表示一方发送FIN报文后，并没有收到对方的ACK报文，相反，却也收到了对方的FIN报文。什么情况下会出现此种情况呢？当双方几乎在同时close()双向连接时，就出现了双方同时发送FIN报文的情况，这是就会出现CLOSING状态，表示双方都正在关闭SOCKET连接。
9. CLOSE_WAIT ：表示正在等待关闭。在主动断开方调用close(…)方法关闭一个连接后，主动方会发送FIN报文给被动方，被动方在收到之后会回应一个ACK报文给主动方，回复完成之后，被动方的TCP连接则进入到CLOSE_WAIT状态。接下来呢，被动方需要检查是否还有数据要发送给主动方，如果没有的话，意味着被动方也就可以关闭连接了，此时给主动方发送FIN+ACK报文，即关闭自己到对方这个方向的连接。简单地说，当连接处于CLOSE_WAIT状态下，可以继续传输数据，传输完成之后关闭连接。
10. LAST_ACK ：当被动断开方发送完FIN+ACK确认断开之后，就处于LAST_ACK状态，等待主动断开方的最后一个ACK报文。当收到对方的ACK报文后，当被动关闭方也就可以进入到CLOSED可用状态了。
11. CLOSED：关闭状态或者初始状态，表示TCP连接是“关闭着的”或“未打开的”状态，或者说连接是可用的

> Netstat是一个的非常有用的工具，可以用于查看路由表、实际的网络连接、甚至每一个网络接口设备的状态信息。举个例子，使用netstat -ant指令查看当前Linux系统中所有的TCP/IP网络的连接信息。

## 2.4 HTTP长连接

HTTP长连接和HTTP短连接，指的是传输层的TCP连接是否被多次使用。默认情况下，HTTP1.0是短连接，每次请求后都会主动释放TCP连接。在短连接的场景下，如果想保持在线状态，客户端需要不断的向服务器发起请求，通常的做法是每隔一段固定的时间向服务器发送一个心跳请求，服务器向客户端回复，表示已经知道客户端在线，如果服务端长时间收不到请求，则认为客户端下线，如果客户端长时间收不到服务器的回复，则认为网络断开。

但是高并发场景下，一是性能较差，每次请求都需要TCP三次握手、四次挥手。二是很容易出现端口被占满，因为主动断开方的系统需要等两个MSL，TCP才会关闭，在高并发下，如果服务器主动断开，端口资源全部在TIME_WAIT状态而无法分配端口资源，很容易出现端口被占满。

因此高并发下，HTTP短连接肯定是不行的，而长连接就出现了。HTTP长连接也叫持久连接，指的是TCP连接建立后，TCP层不在释放，供应用层反复使用。HTTP长连接的特点是：首先性能较高，不需要重复建立TCP连接或者关闭TCP连接；其次TCP数据传输连接基本上不会出现CLOSE_WAIT和TIME_WAIT的问题，系统资源的使用效率会大大提升。

当然HTTP长连接也有缺点：一般需要一个连接池来对可供复用的TCP长连接进行管理和监测。常见的数据库连接池、HTTP连接池，本质上都属于TCP连接池。

### 2.4.1 在不同协议下的HTTP长连接

在HTTP1.0很多浏览器都对HTTP协议进行了扩展，那就是Keep-Alive扩展协议，该扩展作为HTTP1.0版本的补充实验性持久连接协议出现，实现了HTTP长连接的建立和使用。Keep-Alive协议中，如果客户端在首部加上`Connection:Keep-Alive`请求头，则表示请求服务端将传输层TCP连接默认保持在打开状态，如果服务端同意将这条TCP连接保持在打开状态，就会在响应中包含同样的首部，如果没有包含则客户端会认为服务端不支持keep-alive扩展协议。

当然此协议也存在问题，首先该协议不是扩展协议，客户端必须携带`Connection:Keep-Alive`请求头才能使用，而且处于客户端和服务端中间的反向代如果不支持也无法使用。

HTTP1.0的Keep-alive是应用层的扩展协议，与TCP的Keepalive不同，TCP的Keepalive是Socket连接的一个可选项，用于Scoket连接的保活，新建Socket时，可以设置SO_KEEPALIVE套接字可选项打开保活机制。

HTTP1.1并没有使用HTTP1.0的KeepAlive扩展协议，而是自己实现了连接复用方案。HTTP1.1默认使用长连接，需要显示关闭连接，在报文首部加上`Connection:Close`请求头就可以关闭了。当然，不发送`Connection:Close`请求头，不意味着服务器承诺TCP连接永远保持打开，空闲的TCP连接也可以被客户端与服务端关闭。

## 2.5 服务端HTTP长连接技术

### 2.5.1 Tomcat的长连接配置

服务器端Tomcat的长连接配置主要分为两种场景：

1. 单独部署的Tomcat

   对于独立的Tomcat，其长连接配置是通过修改Tomcat配置文件中Connector的配置完成的。一个使用HTTP长连接的Connector连接器的配置示例大致如下：

   ~~~xml
   <Connector port="8080" protocol="HTTP/1.1"
              connectionTimeout="20000"
              redirectPort="8443" 
              URIEncoding="UTF-8" 
              
              keepAliveTimeout="15000"
              maxKeepAliveRequests="-1"
              maxConnections="3000"
              maxThreads="1000"
              maxIdleTime="300000"
              maxSpareThreads="200"
              acceptCount="100"
              enableLookups="false"
              />
   ~~~

   上面的配置中主要有三个关于长连接的配置：

   1. keepAliveTimeout，此选项为TCP的连接保持时长，单位ms。假如在keepAliveTimeout的时间内一直有请求，那么该连接会一直被保持。
   2. maxKeepAliveRequests，此选项为长连接最大支持的请求数，超过数量的连接将被关闭，关闭时Tomcat会返回一个`Connection:close`响应头给客户端。值为-1表示没有最大请求限制，值为1表示将会禁用HTTP长连接。
   3. maxConnections，此选项为Tomcat任意时刻能接收和处理的最大连接数，如果其值被设为-1，则表示连接不受限制。由于Linux的内核默认限制了单进程最大打开文件句柄数为1024，因此，如果此配置项的值超过1024，则相应的需要对Linux系统的单进程最大打开文件句柄数限制进行修改。

   > 使用长连接意味着，一个TCP连接在当前请求结束后，如果没有新的请求到来，Socket连接不会立马释放，而是等keepAliveTimeout到期之后才被释放，如果一个高负载的Tomcat服务器建立的很多长连接，将无法继续建立新的连接，无法为新的客户端提供服务。所以，对于Tomcat长连接的配置需要慎重，错误的参数可能导致严重的性能问题。

2. 内嵌部署的Tomcat

   针对内嵌式Tomcat，其长连接配置可以通过一个自动配置类完成。在自动配置类中，可以配置一个TomcatServletWebServerFactory容器工厂Bean实例，SpringBoot将通过该工厂实例，在运行时获取内嵌式Tomcat容器实例。内置tomcat配置长连接方式（配置方式来自[官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.embedded-container.customizing.programmatic)）：

   ```java
   @Component
   public class MyTomcatWebServerFactoryCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
   
       @Override
       public void customize(TomcatServletWebServerFactory server) {
           server.addConnectorCustomizers((connector) -> {
               Http11NioProtocol protocolHandler = (Http11NioProtocol) connector.getProtocolHandler();
               //设置keepAliveTimeout，长连接保持时间
               protocolHandler.setKeepAliveTimeout(60000);
               //当客户端发送超过1000个会话则强制关闭socket连接
               protocolHandler.setMaxKeepAliveRequests(1000);
               //设置最大连接数
               protocolHandler.setMaxConnections(3000);
               connector.setAsyncTimeout(Duration.ofSeconds(20).toMillis());
           });
       }
   
   }
   ```

### 2.5.2 Nginx的长连接配置

反向代理的Nginx存在两种角色，对于下游客户端来说，Nginx是服务端；对于上游的WEB服务来说，Nginx是客户端。Ngixn作为服务端时，其长连接配置如下：

~~~nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    

    sendfile        on;
	#长连接保持时长
    keepalive_timeout  65s;
	#长连接最大处理请求数
	keepalive_requests 1000;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;
		
		#长连接保持时长
		keepalive_timeout  10s;
		#长连接最大处理请求数
		keepalive_requests 10;
		
		location / {
			root html;
		}
    }

}
~~~

1. keepalive_requests指令用于设置一个长连接上可服务的最大请求数量，nginx默认100。一个长连接建立之后，Nginx就会为这个连接设置一个计数器，记录这个长连接上已经接收并处理的客户端请求的数量。如果达到这个参数设置的最大值时，则Nginx会强行关闭这个长连接，让客户端重新建立新的长连接。
2. keepalive_timeout用于设置客户端的长连接在服务器端保持的最长时间，如果设置为0则禁用长连接。

### 2.5.3 服务端长连接的注意事项

1. 当单个客户端请求数较多时

   当客户端不是普通用户而是下游的代理服务器时，客户端数量是很少的，但是单个客户端的请求数是非常多的。

   在此场景下，选项keepalive_timeout可以配置一个较大的值，但是对于Nginx来说，不能对单个连接的处理请求数不做限制，必须定期关闭连接，才能释放每个连接的所分配的内存。由于过大的请求数会导致内存占用过大，因此不建议keepalive_requests设置过大，当然也不能不设置。

2. 当单个客户端请求数较少时

   当客户但是普通用户时，比如浏览器等，其单个客户端请求数是有限的，如果我们配置为以下方式：

   ~~~nginx
   #长连接保持时长
   keepalive_timeout  65s;
   #长连接最大处理请求数
   keepalive_requests 1000;
   ~~~

   那么大量的长连接由于达不到1000的请求数量，导致一直在空闲等待，需要等到65s结束后才能被关闭，所以需要减少最大请求处理数和长连接保持时长：

   ~~~nginx
   #长连接保持时长
   keepalive_timeout  10s;
   #长连接最大处理请求数
   keepalive_requests 100;
   ~~~

   但是也不能配置的太极端，比如keepalive_requests如果配置成10，就会造成大量连接被关闭。比如假如有100个用户，单客户端每秒100个请求，如果keepalive_requests配置成10，所以每秒100个请求就需要10个连接，如果100个用户就是1000个连接。

## 2.6 客户端HTTP长连接技术

### 2.6.1 HttpURLConnection

这是Java内置的HTTP通信技术。Java默认是支持长连接的，当我们完成读取响应体，jdk的http处理程序会尝试清理连接，如果成功就将其放入连接缓存中，以便将来的HTTP请求重用。具体请参见[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/net/http-keepalive.html)。我们简单编写一个案例：

```java
public static void httpUrlConnectionFun() {
    HttpURLConnection connection = null;
    InputStream inputStream = null;
    try {
        URL url = new URL("http://IP地址:端口号");
        connection = (HttpURLConnection) url.openConnection();
        connection.setRequestMethod("GET");
        connection.setRequestProperty("Accept","application/json");
        connection.connect();
        if (connection.getResponseCode() != 200){
            throw new RuntimeException("请求失败");
        }
        System.out.println(connection.getResponseMessage());
        inputStream = connection.getInputStream();
        byte [] bytes = new byte[1024];
        int length =-1;
        StringBuilder result = new StringBuilder();
        while ((length =inputStream.read(bytes))!=-1){
            result.append(new String(bytes,0,length));
        }
        System.out.println(result);
    }catch (Exception e){
        e.printStackTrace();
    }finally {
        try {
            inputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
        connection.disconnect();
    }
}
```

然后我们将1.4的回显web服务器放到linux服务器上运行，然后我们用这段代码进行测试：

```java
public static void main(String[] args) throws Exception {
    int count = 10000;
    ExecutorService service = Executors.newFixedThreadPool(10);
    while (--count >0){
        service.submit(TestHTTP::httpUrlConnectionFun);
    }
    Thread.sleep(Integer.MAX_VALUE);
}
```

然后我们在linux服务器上通过netstat命令查看当前连接状态。命令如下：

`while [ 1 -eq 1 ]; do netstat -anpt|grep 端口号;sleep 2;echo; done`

> 这里在shell编程中 -eq是等于的意思。上面命令基本是每隔两秒输出我们输入的端口号的网络连接状态。

### 2.6.2 Nginx承担客户端角色

Nginx有自己的类似客户端TCP连接池的连接管理组件，对于池中单个TCP连接可以通过upstream区域的keepalive指令完成，如下：

~~~nginx
upstream test {
    server 192.168.0.1:8080;
    server 192.168.0.2:8080;
    #连接池里面最大的空闲连接数量
    keepalive 300; 

}
server {

    location / {
        proxy_pass http://test;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;              # 设置http版本为1.1
        proxy_set_header Connection "";      # 设置Connection为长连接（默认为no）
    }
}
~~~

我们在使用时还需要指定http版本为1.1，并且为了保证反向代理到其上游之间的TCP连接能复用请求头Connection为空，需要将下游客户端发送过来的HTTP请求头“Connection:close”重置掉，将其值重置成空白字符串。

# 三、WebSocket原理

## 3.1 WebSocket简介

WebSocket是一种全双工通信的协议，其通信在TCP连接上进行，所以属于应用层协议。WebSocket使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在WebSocket编程中，浏览器和服务器只需要完成一次升级握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

### 3.1.1 Ajax短轮询和LongPoll长轮询原理

在WebSocket技术之前，浏览器和服务器之间的双向通信主要分为Ajax短轮询和LongPoll长轮询。

Ajax短轮询即浏览器周期性的向服务器发起Http请求，浏览器通过HTTP1.1的持久连接（一次连接，多次请求）可以在建立连接后发起多次异步请求。Ajax短轮询原理十分简单，让浏览器几秒一个请求，询问服务器是否有新信息。这种模式缺点很明显，即需要不断向服务器发送请求，HTTP请求每次都带上很长的请求头，造成服务器带宽和CPU资源浪费。

Long Poll长轮询在原理上跟Ajax短轮询差不多，都是采用轮询的方式，不过采取的是服务端阻塞模型，轮询过程中，服务器端在收到浏览器的请求后，如果暂时没有消息推给浏览器，服务端就会一直阻塞，直到服务端有消息才会响应给客户端，收到响应后开启下一次轮询请求。

无论是Ajax短轮询还是Long Poll长轮询，都不是最好的双向通信方式，都需要很多资源。而WebSocket则不同，该协议只需要经过一次HTTP 请求，就可以做到源源不断的信息传送了，当传输协议完成HTTP到WebSocket协议升级后，服务端就可以主动推送信息给客户端，高效率的实现双向通信。

### 3.1.2 WebSocket与HTTP关系

WebSocket的最大特点就是，是全双工通信，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息。WebSocket与HTTP之间的关系是：WebSocket其实是一个新协议，通信过程与跟HTTP协议基本没有关系，只是为了兼容现有浏览器，所以在握手阶段使用了HTTP协议，如下图：

![image-20220823212831804](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220823212831804.png)

WebSocket与HTTP协议都处于TCP/IP协议栈的应用层，都是TCP/IP协议的子集。HTTP协议是单向通信协议，只有客户端发起HTTP请求，服务端才会返回数据；而WebSocket协议是双向通信协议，在建立连接之后，客户端和服务器都可以主动向对方发送或接受数据。WebSocket的通信连接建立的前提需要借助HTTP协议，完成通信连接建立之后，通信连接上的双向通信就与HTTP协议无关了。

## 3.2 Netty处理WebSocket

WebSocket协议中大致包含了五种类型的数据帧，与之对应Netty中包含了它们的封装类型，这些都是WebSocketFrame的子类，如下：

- BinaryWebSocketFrame：封装二进制数据的WebSocketFrame数据帧
- TextWebSocketFrame：封装文本数据的WebSocketFrame数据帧
- CloseWebSocketFrame：表示一个Close结束请求，数据帧中包含结束的状态和原因，此帧是控制帧
- ContinuationWebSocketFrame：当发送的内容多于一个数据帧时，消息被拆分成多个WebSocketFrame数据帧发送，剩余的内容就由此数据帧发送。ContinuationWebSocketFrame可以发送后续的文本或二进制数据帧
- PingWebSocketFrame：Ping、Pong是WebSocket通信中的心跳帧，用来保证客户端是在线的，一般来说只有服务端给客户端发送Ping、然后客户端回复Pong，表明自己仍在线。PingWebSocketFrame属于控制帧，其对应协议报文中的操作码opcode值为0x9
- PongWebSocketFrame：PongWebSocketFrame是对PingWebSocketFrame的响应帧，也是控制帧，对应的协议报文中的操作码opcode值为0xA

在Netty中内置的WebSocket的处理器主要如下：

- WebSocketServerProtocolHandler：负责协议开始升级时的请求处理，也就是开启握手处理。另外，在协议升级握手完成后的WebSocket通信过程中，此处理器还负责对Close、Ping、Pong进行处理。
- WebSocketServerProtocolHandshakeHandler：此处理器负责进行协议升级握手处理，在握手完成后，此处理器会触发HANDSHAKE_COMPLETE用户事件，表示握手完成。
- WebSocketFrameEncoder：WebSocketFrame数据帧编码器，负责WebSocket数据帧编码。在握手时，针对不同的WebSocket协议版本，握手处理器会在流水线上装配对应的编码器子表。
- WebSocketFrameDecoder：WebSocketFrame数据帧解码器，负责WebSocket数据帧解码。在握手时，针对不同的WebSocket协议版本，握手处理器会在流水线上装配对应的解码器子表。

其中WebSocketServerProtocolHandler是非常关键的处理器，负责开始升级握手和控制帧的处理，可以理解为握手处理器。握手完成后，双方的通信协议会从HTTP升级到WebSocket协议，老的HTTP协议处理器会被该握手处理器替换掉，新的与WebSocket相关的解码器会被成功添加到流水线上。

## 3.3 WebSocket示例

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <input type="text" id="in"><button onclick="send()">发送</button>
    <br/>
    <textarea id = "response"></textarea>
</body>
<script>
webSocket = new WebSocket("ws://127.0.0.1:9092/ws","echo");
webSocket.onmessage = function(event){
    var result = document.getElementById("response");
    result.value = result.value +'\n' +event.data;
}
webSocket.onopen = function(event){
    var result = document.getElementById("response");
    result.value = "连接已经开启";
}
webSocket.omclose = function(event){
    var result = document.getElementById("response");
    result.value = "连接已经断开";
}
function send(){
    var message = document.getElementById("in");
    if(webSocket.readyState == WebSocket.OPEN){
        console.log(message.value)
        webSocket.send(message.value)
    }else{
        alert("连接未打开")
    }
}
</script>
</html>
~~~

```java
@Component
@Slf4j
public class ServerStart implements CommandLineRunner {

    private static int PORT;

    @Value("${websocket.port}")
    public void setPort(int port) {
        PORT = port;
    }

    static class MyInitializer extends ChannelInitializer<NioSocketChannel> {

        @Override
        protected void initChannel(NioSocketChannel channel) throws Exception {
            channel.pipeline().addLast(new HttpRequestDecoder());
            channel.pipeline().addLast(new HttpResponseEncoder());
            channel.pipeline().addLast(new HttpObjectAggregator(65535));
            //设置websocket服务监听URL为/ws，子协议为echo，最大WebSocket传输帧为1024*10
            //当服务端收到客户端的URL为/ws，子协议为echo的HTTP连接时，自动升级为WebSocket协议
            channel.pipeline().addLast(new WebSocketServerProtocolHandler("/ws","echo",true,1024*10));
            channel.pipeline().addLast(new EchoHandler());
            channel.pipeline().addLast(new MyWebSocketHandler());
        }
    }

    private final ServerBootstrap serverBootStrap = new ServerBootstrap();
    private final EventLoopGroup bossGroup = new NioEventLoopGroup();
    private final EventLoopGroup workerGroup = new NioEventLoopGroup();

    @Override
    public void run(String... args) throws Exception {
        try {
            ServerBootstrap bootstrap = serverBootStrap
                    .group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new MyInitializer());
            Channel channel = bootstrap.bind(PORT).sync().channel();
            log.info("WebSocket回显服务启动，端口号{}",PORT);
            channel.closeFuture().sync();
        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

```java
@Slf4j
public class MyWebSocketHandler extends SimpleChannelInboundHandler<WebSocketFrame> {
    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, WebSocketFrame webSocketFrame) throws Exception {
        //Ping和Pong已经被WebSocketServerProtocolHandler处理过了，我们只需要处理文本和二进制数据帧即可
        if (webSocketFrame instanceof TextWebSocketFrame){
            String text = ((TextWebSocketFrame) webSocketFrame).text();
            log.info("收到数据{}",text);
            String result = new Date() +"："+ text;
            TextWebSocketFrame resultFrame = new TextWebSocketFrame(result);
            channelHandlerContext.channel().writeAndFlush(resultFrame);
        }else if (webSocketFrame instanceof BinaryWebSocketFrame){
            log.error("这里暂不处理二进制数据");
        }
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof WebSocketServerProtocolHandler.HandshakeComplete){
            ctx.pipeline().remove(EchoHandler.class);
            log.info("握手成功");
            log.info("新的WebSocket客户端加入,通道为{}",ctx.channel());
        }
        super.userEventTriggered(ctx, evt);
    }
}
```

## 3.4 WebSocket原理

WebSocket是基于Http协议进行握手和协议升级的。且WebSocket是应用层的协议，是TCP/IP协议的子集，该协议是通过HTTP/1.1协议的101状态码完成握手，也就是说，WebSocket协议需要先借助HTTP协议，在服务端返回101状态码之后才可以进行WenSocket的全双工通信，协议升级之后就和HTTP协议没有关系了。客户端发起握手时，需要向WebSocket的监控URL发起GET请求，WebSocket握手和响应的报文如下：

![image-20221017170439084](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221017170439084.png)

实际上，握手的报文就是HTTP报文，但是根据WebSocket协议规定，创建WebSocket握手请求必须是一个HTTP请求，请求的方法必须是GET，并且HTTP协议的版本不可以低于1.1。同时握手请求HTTP报文需要携带一些WebSocket协议规范约定的请求头：

- Sec-WebSocket-Key请求头

  该请求头的值是一个Base64编码的值，这个是客户端浏览器随机生成的，服务端从请求(HTTP的请求头)信息中提取Sec-WebSocket-Key，服务端会对此值进行加密，之后会将加密结果响应给客户端。WebSocket协议规范约定，握手报文必须包含Sec-WebSocket-Key请求头。

- Upgrade请求头

  WebSocket协议规范约定，握手请求报文必须包含Upgrade请求头，并且此请求头的值必须包含"websocket"。

- Connection请求头

  WebSocket协议规范约定，握手报文必须包含Connection请求头，并且此请求头的值必须包含"Upgrade"。

- Sec-WebSocket-Version请求头

  WebSocket协议规范约定，握手报文必须包含Sec-WebSocket-Version请求头，其值若为13，表示客户端支持WebSocket的协议版本号为13，该版本为目前最为常用的协议版本，其余版本号包括00、07、08等。

- Sec-WebSocket-Protocol请求头

  Sec-WebSocket-Protocol则是表示通信使用的子协议，这属于用户自定义的协议名称，只要与服务端的子协议名称保持一致即可，否则会握手失败

服务端接收请求后启动服务端升级握手的机制，进行握手检查，如果客户端能够满足WebSocket通信的要求，握手处理器会向客户端发送Switching Protocols（转换协议）HTTP响应报文（见上图）。该响应报文的响应状态码为101，表示服务端同意客户端协议升级请求，并将协议从HTTP转换为WebSocket协议。响应报文中所涉及到的比较重要的响应头：

- Upgrade响应头

  响应报文中Upgrade头值为“websocket”，服务端通过该头告诉客户端，即将升级的通信协议是WebSocket协议，而不是其他的协议。

- Sec-WebSocket-Accept响应头

  服务端通过Sec-WebSocket-Accept响应头去确认客户端的Sec-WebSocket-Key，该响应头的值为加密过后的、客户端握手请求中的Sec-WebSocket-Key值。客户端收到报文后，会对改值进行校验，只有当握手请求的Sec-WebSocket-Key值经过固定算法加密后的结果和响应头里的Sec-WebSocket-Accept的值保持一致，该连接才会被认可建立。

- Sec-WebSocket-Protocol请求头

  Sec-WebSocket-Protocol则是表示最终使用的子协议，这属于应用程序的自定义协议

客户端收到服务端响应之后，如果校验通过，则握手成功，WebSocket连接建立，双向通信便可以开始了。握手完成之后，客户端与服务端之间的通信，不再是HTTP协议，而是WebSocket协议。WebSocket协议的报文如下图，图片来源网络：

![image-20221017171039334](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221017171039334.png)

WebSocket协议的通信报文是二进制格式，大致包含以下字段：

- FIN，占用1bit，如果其值为1，表示该帧是消息的最后一帧，如果是0，则不是消息的最后一帧。
- RSV1-3，默认是0 (必须是0)，除非有扩展定义了非零值的意义。
- Opcode，WebSocket的操作码，占4bit位。其值决定了后续如何解析Data Payload。具体如下：
         0x00 denotes a continuation frame
         0x01 表示一个text frame
         0x02 表示一个binary frame
         0x03 ~~ 0x07 are reserved for further non-control frames,为将来的非控制消息片段保留测操作码
         0x08 表示连接关闭
         0x09 表示 ping (心跳检测相关)
         0x0a 表示 pong (心跳检测相关)
         0x0b ~~ 0x0f are reserved for further control frames,为将来的控制消息片段保留的操作码
- Mask，一个bit位（值为1时抓包会显示为true），表示是否要对数据载荷进行掩码操作。客户端向服务端发送数据时，需要对数据进行掩码操作；从服务端向客户端发送数据时，不需要对数据进行掩码操作，如果服务端接收到的数据没有进行掩码操作，服务器需要断开连接。所有的客户端发送到服务端的数据帧，Mask值都是1。
- Payload len，通信报文中Payload data的长度。
- Masking-key，掩码键，如果Mask值为1，需要用这个掩码键来对数据进行反掩码，才能获取到真实的通信数据。为了避免被网络代理服务器误认为是HTTP请求，从而招致代理服务器被恶意脚本的攻击，WebSocket客户端必需掩码所有送给服务器的数据帧。
- Payload data，帧真正要发送的数据，可以是任意长度，但尽管理论上帧的大小没有限制，但发送的数据不能太大，否则会导致无法高效利用网络带宽。

# 四、SSL/TLS核心原理

## 4.1 SSL/TLS简介

HTTP协议中，信息是明文传输的，因此就有了HTTPS协议。HTTPS就是在HTTP协议中加入了SSL/TLS协议，SSL/TLS依靠证书来验证服务端身份，并为浏览器和服务端之间的通信加密。使用HTTPS的主要目的是提供对网站的服务端的身份认证，同时保护交换数据的隐私与完整性。

那么什么是SSL/TLS协议呢？SSL全拼是“Secure Sockets Layer”，中文可翻译为安全套接层，它是由Netscape公司设计和研发的安全传输技术。它是在传输通信协议（TCP/IP）上实现的一种安全协议，采用公开密钥技术。TLS是SSL的升级版，是更新更安全的SSL协议版本。TLS由两层组成：TLS记录协议和TLS握手协议。1999年，SSL 因为应用广泛，已经成为互联网上的事实标准。IETF就在那年把SSL标准化，完成标准化之后，SSL协议名称被改为TLS（Transport Layer Security)，中文叫做“传输层安全协议”。

SSL/TLS位于应用层和传输层之间，除HTTP外，它可以为任何基于TCP传输层以上的应用层协议提供安全性保障。在理论上SSL/TLS属于传输层，但是在实现上他表现在应用层上。目前使用最广泛的是SSL3.0、TLS1.0(SSL3.1)、TLS1.1(SSL3.2)、TLS1.2(SSL3.3)这四个协议。SSL/TLS的版本如下：

| 版本   | 发布时间 | 说明                                                         |
| ------ | -------- | ------------------------------------------------------------ |
| SSL1.0 |          | 存在漏洞，未发布                                             |
| SSL2.0 | 1995     | 网景公司发布了SSL2.0版本                                     |
| SSL3.0 | 1996     | 网景公司重新设计了SSL3.0版本。IETF通过互联网标准RFC6101文件发表了SSL3.0版本 |
| TLS1.0 | 1999     | IETF通过互联网标准RFC2246文件发表了TLS1.0版本，TLS1.0与SSL3.0差异微小 |
| TLS1.1 | 2006     | IETF通过互联网标准RFC4346文件发表了TLS1.1版本                |
| TLS1.2 | 2008     | IETF通过互联网标准RFC5246文件发表了TLS1.2版本                |
| TLS1.3 | 2018     | IETF通过互联网标准RFC8446文件发表了TLS1.3版本                |

## 4.2 加密算法原理

### 4.2.1 散列单向加密

散列算法比较简单，就是将待加密的信息，生成一个固定大小的字符串。

> 比如字符串通过MD5加密之后是32个字符

常见的常用的散列算法有MD5、SHA1、SHA-512等。散列转换过程是不可逆的，也就是说一旦数据被转换成散列值，原始数据就无法恢复。散列加密在用户注册和密码存储时非常有用，因为服务端存储的是密码的哈希值而非明文。这样即使数据库被攻击，也不会造成用户密码泄露。当用户下次登录时，会对用户的登入密码（明文）使用相同的散列算法（哈希函数）进行散列，并将散列结果与来自数据库的哈希密码进行匹配，如果它是相同的，用户将登录成功，否则用户将登录失败。

MD5曾广泛用于安全领域，但现在由于安全性问题，主要用于文件完整性校验。除了MD5，Java还提供了SHA1、SHA256、SHA512等散列摘要函数的实现。除了在算法上有些差异之外，这些散列函数的主要不同在于摘要长度，MD5的生成的摘要是128位，SHA1生成的摘要是160位，SHA256生成的摘要是256位，SHA512生成的摘要长度是512位。

例子如下：

```java
package com.learn.test;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class HashingExample {

    // 生成散列摘要的方法
    public static String generateHash(String data, String algorithm) throws NoSuchAlgorithmException {
        MessageDigest digest = MessageDigest.getInstance(algorithm);
        byte[] hash = digest.digest(data.getBytes());
        return bytesToHex(hash);
    }

    // 将字节数组转换为十六进制字符串
    private static String bytesToHex(byte[] hash) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : hash) {
            String hex = Integer.toHexString(0xff & b);
            if(hex.length() == 1) hexString.append('0');
            hexString.append(hex);
        }
        return hexString.toString();
    }

    // 验证散列摘要的方法
    public static boolean verifyHash(String data, String originalHash, String algorithm) throws NoSuchAlgorithmException {
        String newHash = generateHash(data, algorithm);
        return newHash.equalsIgnoreCase(originalHash);
    }

    public static void main(String[] args) {
        try {
            String originalData = "Hello World";
            String[] algorithms = {"MD5", "SHA-1", "SHA-256", "SHA-512"};

            for (String algorithm : algorithms) {
                // 生成散列摘要
                String hash = generateHash(originalData, algorithm);
                System.out.println(algorithm + " Generated Hash: " + hash);

                // 验证散列摘要
                boolean isVerified = verifyHash(originalData, hash, algorithm);
                System.out.println(algorithm + " Verification: " + isVerified + "\n");
            }

        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
    }
}

```

运行结果：

```
MD5 Generated Hash: b10a8db164e0754105b7a99be72e3fe5
MD5 Verification: true

SHA-1 Generated Hash: 0a4d55a8d778e5022fab701977c5d840bbc486d0
SHA-1 Verification: true

SHA-256 Generated Hash: a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
SHA-256 Verification: true

SHA-512 Generated Hash: 2c74fd17edafd80e8447b0d46741ee243b7eb74dd2149a0ab1b9246fb30382f27e853d8585719e0e67cbda0daa8f51671064615d645ae27acb15bfb1447f459b
SHA-512 Verification: true
```

### 4.2.2 对称加密算法

对称加密算法是一种加密和解密使用相同密钥的加密方法。这意味着加密和解密双方必须共享同一个密钥，并且保护这个密钥不被第三方获取。对称加密算法通常比非对称加密算法更快，适用于加密大量数据的场景。常见的对称加密算法包括DES、AES、3DES等。

DES（Data Encryption Standard）是一种较早的对称加密算法，虽然因其密钥长度较短（56位密钥）而被认为不够安全，但它在教学和了解对称加密的基本概念中仍然是一个很好的例子。案例如下：

```java
package com.learn.test;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;

public class DesExample {

    public static void main(String[] args) {
        try {
            // 生成DES密钥
            KeyGenerator keyGenerator = KeyGenerator.getInstance("DES");
            keyGenerator.init(56); // 指定密钥长度为56位
            SecretKey secretKey = keyGenerator.generateKey();
            byte[] keyBytes = secretKey.getEncoded();

            // 创建DES密钥规范
            SecretKeySpec keySpec = new SecretKeySpec(keyBytes, "DES");

            // 原始数据
            String data = "Hello, DES!";
            System.out.println("Original Data: " + data);

            // 加密数据
            Cipher cipher = Cipher.getInstance("DES/ECB/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, keySpec);
            byte[] encryptedData = cipher.doFinal(data.getBytes());
            System.out.println("Encrypted Data: " + new String(encryptedData));

            // 解密数据
            cipher.init(Cipher.DECRYPT_MODE, keySpec);
            byte[] decryptedData = cipher.doFinal(encryptedData);
            System.out.println("Decrypted Data: " + new String(decryptedData));

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

### 4.2.3 非对称加密算法

非对称加密算法，也称为公钥加密算法，使用一对密钥：公钥和私钥。如果公钥用于加密数据，则私钥用于解密。如果私钥用于加密数据，则公钥用于解密。由于加密和解密使用的是不同的密钥，这种机制允许公钥可以公开分享，而私钥保持私密。非对称加密常用于安全的数据传输、数字签名和身份验证。RSA是最早和最广泛使用的非对称加密算法之一。

Java使用RSA的例子：

```java
package com.learn.test;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.NoSuchAlgorithmException;
import java.security.PrivateKey;
import java.security.PublicKey;

import javax.crypto.Cipher;

public class RsaExample {

    public static void main(String[] args) {
        try {
            // 1. 生成RSA密钥对
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048); // 密钥长度推荐为2048位
            KeyPair keyPair = keyPairGenerator.generateKeyPair();
            PublicKey publicKey = keyPair.getPublic();
            PrivateKey privateKey = keyPair.getPrivate();

            // 2. 加密数据
            Cipher cipher = Cipher.getInstance("RSA");
            cipher.init(Cipher.ENCRYPT_MODE, publicKey);
            String originalData = "Hello, RSA!";
            byte[] encryptedData = cipher.doFinal(originalData.getBytes());
            System.out.println("Encrypted Data: " + bytesToHex(encryptedData));

            // 3. 解密数据
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            byte[] decryptedData = cipher.doFinal(encryptedData);
            System.out.println("Decrypted Data: " + new String(decryptedData));

        } catch (NoSuchAlgorithmException | javax.crypto.NoSuchPaddingException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 将字节数组转换为十六进制字符串的辅助方法
    private static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            String hex = Integer.toHexString(0xff & b);
            if (hex.length() == 1) hexString.append('0');
            hexString.append(hex);
        }
        return hexString.toString();
    }
}
```

非对称加密算法包含两种密钥，其中的公钥本来是公开的，这就不需要像对称加密算法那样将私钥给对方，对方解密时使用公开的公钥就行，这就大大提高了加密算法的安全性。退一步说，即使不法之徒获知了非对称加密算法的公钥，甚至获知了加密算法的源码，只要没有获取公钥对应的私钥，也是无法进行解密的。

### 4.2.4 数字签名

数字签名是一种用于验证数字信息完整性和来源认证的技术，它使接收者能够确认信息确实来自签名者，并且自签名之后未被篡改。数字签名广泛应用于软件分发、电子文档签署、在线交易等领域，是确保数字信息安全的关键技术之一。

在非对称加密算法中，发送方A通过接收方B的的公钥将数据加密后，将加密后的密文发给接收方B，B利用私钥解密就能得到需要的数据。这里有一个问题，那就是接收方B公钥是公开的，B如何检查发送方A的身份。

一种比较简单的方式就是发送方A利用A自己的私钥进行加密，然后B在利用A的公钥进行解密，由于私钥只有A知道，只要接收方解密成功，就可以确定消息来自A而不是其他地方。

签名过程涉及以下几个步骤：

1. **创建消息的散列（哈希）**：首先，对原始消息使用散列（哈希）函数生成一个固定长度的散列值（消息摘要）。这个散列值作为消息的唯一表示，任何对消息的微小改动都会导致散列值发生显著变化。
2. **加密散列值**：然后，发送方使用自己的私钥对散列值进行加密，生成数字签名。私钥的保密性确保了签名的独特性和不可伪造性。
3. **发送消息和数字签名**：发送方将原始消息和数字签名一起发送给接收方。

接收方收到消息和数字签名后，进行以下步骤验证签名：

1. **使用相同的散列函数生成散列值**：接收方对收到的原始消息应用相同的散列函数，生成一个散列值。
2. **使用公钥解密数字签名**：接收方使用发送方的公钥对数字签名进行解密，得到一个散列值。公钥的使用确保了只有对应的私钥签名者才能生成的签名。
3. **比较散列值**：接收方比较自己生成的散列值和解密得到的散列值。如果两者相同，说明消息在传输过程中未被篡改，且确认消息确实来自声称的发送者。

数字签名的特点和优势主要有以下几点：

- **认证**：数字签名帮助接收者验证消息的真实来源，确保消息是由预期的发送者签名的。
- **完整性**：通过比较散列值，接收者可以确认消息自签名后未被修改，保证了消息的完整性。
- **不可否认**：数字签名是签名者特有的，发送者不能否认他们签名的消息，提供了电子交易的法律依据。

数字签名的安全性和有效性依赖于私钥的保密性，以及所用散列函数和加密算法的强度。在实际应用中，通常会用证书（由可信第三方颁发的公钥证书）来进一步增强数字签名机制的信任度。

Java规范要求JDK版本提供的数字签名实现：

| Java规范      | 描述                                              |
| ------------- | ------------------------------------------------- |
| SHA1withDSA   | 使用SHA1算法生成摘要，使用DSA算法进行摘要加密。   |
| SHA1withRSA   | 使用SHA1算法生成摘要，使用RSA算法进行摘要加密。   |
| SHA256withRSA | 使用SHA256算法生成摘要，使用RSA算法进行摘要加密。 |

Java如何生成RSA密钥对，使用私钥对数据进行签名，以及使用公钥验证签名的案例如下：

```java
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.Signature;

public class DigitalSignatureExample {

    public static void main(String[] args) {
        try {
            // 生成RSA密钥对
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048);
            KeyPair keyPair = keyPairGenerator.generateKeyPair();
            PublicKey publicKey = keyPair.getPublic();
            PrivateKey privateKey = keyPair.getPrivate();

            // 创建要签名的数据
            String data = "Hello, this is a message to sign";
            byte[] dataBytes = data.getBytes();

            // 创建Signature实例并初始化为签名模式
            Signature signature = Signature.getInstance("SHA512withRSA");
            signature.initSign(privateKey);
            signature.update(dataBytes);
            byte[] digitalSignature = signature.sign();

            System.out.println("Digital Signature: " + bytesToHex(digitalSignature));

            // 验证签名
            signature.initVerify(publicKey);
            signature.update(dataBytes);
            boolean verified = signature.verify(digitalSignature);
            System.out.println("Signature verified: " + verified);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 将字节数组转换为十六进制字符串的辅助方法
    private static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            String hex = Integer.toHexString(0xff & b);
            if (hex.length() == 1) hexString.append('0');
            hexString.append(hex);
        }
        return hexString.toString();
    }
}
```



## 4.3 SSL/TLS运行过程

SSL/TLS基本流程如下：

1. 客户端向服务端索要验证公钥
2. 双方生成对话密钥
3. 双方采用对话密钥进行加密通信

前两步一般称为握手阶段，每个TLS连接都会以握手开始。握手阶段涉及到四次通信，且握手阶段通信是明文的，握手过程客户端和服务端的四个流程如下：

1. 交换各自支持的加密套件和参数，经过协商后，双方就加密套件和参数达成一致
2. 验证对方（主要是服务端）的证书，或是用其他方式进行服务端身份验证
3. 对将用于保护会话的共享主密钥达成一致
4. 验证握手消息是否被第三方修改

整体流程如下图：

![image-20240407165500714](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20240407165500714.png)

### 4.3.1 第一阶段握手

TCP三次握手建立传输层连接后，通信双方需要交换各自支持的加密套件和参数，经过协商后，目的是使双方的加密套件和参数保持一致。SSL/TLS握手的第一阶段工作为：客户端发一个Client Hello报文给服务端，并且第一个阶段只有这一个数据帧。Client Hello中主要包含以下信息：

1. 客户端支持的SSL/TLS协议版本
2. 客户端生成的随机数，这里成为Random_C
3. 客户端支持的签名算法、加密方法、摘要算法
4. 客户端支持的压缩方法

通过抓包软件抓取的信息如下：

![image-20221019161101679](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221019161101679.png)

### 4.3.2 第二阶段握手

SSL/TLS握手的第二阶段工作为服务端对客户端的Client Hello请求进行响应，在收到客户端请求后，服务端向客户端发出回应，这个阶段的服务端回应帧一般包含四个回复帧：Server Hello帧、Certificate帧、Server Key Exchange帧和Server Hello Done帧。我们依次来看一下：

1. Server Hello

   Server Hello中主要包含：回复服务端使用的加密通信协议版本，比如TLS1.2；回复一个服务端生成的随机数，这是握手阶段的第二个随机数记为Random_S,后面用于对话密钥的生成；回复确认使用的加密方法；回复服务端的证书。通过抓包软件抓取的信息如下：

   ![image-20221019163335770](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221019163335770.png)

2. Certificate

   Certificate帧用于返回服务端证书，该证书中含有服务端的证书清单（包括服务端公钥），用于身份验证和密钥协商。服务器一般不需要验证客户端的身份，但如果要求较高还是会验证的。只要服务端需要验证客户端的身份，服务端会发一个Certificate Request证书请求给客户端。

3. Server Key Exchange

   Sever Key Exchange帧的目的是携带密钥交换的额外数据，其消息内容对于不同的协商算法套件都会存在差异。在某些场景中，服务端不需要发送Server Key Exchange握手消息。如果在Server Hello消息中使用DHE/ECDHE非对称密钥协商算法来进行SSL握手，将发送该类型握手消息。对于使用RSA算法的SSL握手，不会发送该类型握手消息。

4. Server Hello Done

   Sever Hello Done帧是第二阶段的最后一个帧，标记服务端对客户端的Client Hello请求帧的所有响应报文发送完毕，Sever Hello Done帧的长度为0。

Certificate、Server Key Exchange、Server Hello Done的抓包信息如下：

![image-20221019164146545](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221019164146545.png)

### 4.2.3 第三阶段握手

SSL/TLS握手的第三阶段工作为：客户端进行回应，这个阶段一般包含三个数据帧Client Key Exchange、Change Cipher Spec、Encrypted Handshake。客户端收到二阶段的服务端回应后，首先验证服务端证书，如果证书不是可信机构或证书域名与实际不一致或证书过期，就会向访问者显示一个警告，由其选择是否还要继续通信。

如果证书可用，客户端就会从证书中取出服务端的公钥然后发送以下三种信息：

1. 一个随机数，用服务端公钥进行加密。这是第三个随机数，又记为Pre-master key。至此客户端和服务端都有了三个随机数，接着双方就用事先商定的加密算法，各自生成本次会话的会话密钥
2. 编码改变通知，表示随后的信息都用双方商定的加密算法和密钥加密后发送。
3. 客户端握手结束通知，表示客户端握手阶段结束

> 服务端的证书信息会包含公钥，客户端进行证书校验的流程大概为客户端随机生成一串数字，然后用服务端的公钥加密后发给服务端，然后服务端会用私钥进行解密，然后返回给客户端，客户端将其与原文比较，如果一致则说明对端就是证书拥有者。

我们分别来看一下这三个数据帧：

![image-20221020090833149](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221020090833149.png)

### 4.2.4 第四阶段握手

第四阶段服务端会进行最后的回应。在收到第三个随机数后，服务端会生成会话密钥，然后发送Change Cipher Spec帧（服务端编码改变通知）和Encrypted Handshake Message帧（服务端握手结束通知）、

![image-20221020091238622](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221020091238622.png)

## 4.4 Keytool工具

> 未完结

### 4.4.1 数字证书

SSL/TSL在加密传输的开始，客户端会通过ClientHello帧获得服务端的公钥，这个过程可能会被第三方劫持。

当客户端的ClientHello帧被劫持时，服务端发送到客户端的公钥，也会被第三方截获，然后第三方自已会伪造一对密钥（包含公钥和私钥），并将伪造的公钥发送给客户端。当服务端发送数据给客户端的时候，第三方也会将信息进行劫持，用一开始截获的公钥进行解密后，然后使用自已的私钥将数据再一次加密后发送给客户端，而客户端收到后使用第三方（劫持方）的公钥去解密。反过来也是如此，当客户端发送数据给服务端时，报文亦会被劫持方截取和转发，并且，整个截取和转发的过程是对于客户端和服务端都是透明和不可见的，但信息却被泄露了。

![image-20240403164644883](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20240403164644883.png)

为了防止这种情况，数字证书就出现了，数字证书就是互联网通讯中标志通讯各方身份信息的一串数字，它是由权威机构——CA机构（Certificate Authority、认证中心）发行的，人们可以在网上用它来识别对方的身份。

数字证书发行的流程一般为：用户先产生密钥对，并将公钥和身份信息提供给CA机构。认证中心在核实身份后，将执行一些必要的步骤，以确信请求确实由用户提交的，然后，认证中心将发给用户一个数字证书。一个证书中含有三个部分：证书内容、散列算法、加密密文。该证书内包含服务端的个人信息和公钥信息：加密密文为证书内容通过散列算法计算出摘要之后，然后使用CA机构的私钥进行非对称加密后的密文，加密密文也可以理解成为CA机构自已的数字签名。当客户端发起请求时，服务端将该数字证书发送给客户端，客户端首先需要对证书进行验证，具体的方法为：通过CA机构提供的公钥对服务端的证书的数字签名（加密密文）进行解密，以获得服务端证书的内容摘要（散列值），同时将证书内容使用相同的散列算法获取摘要，比对两个摘要，如果两者相等，说明证书中的公钥仍然是服务端原始公钥而没有被第三方篡改，则说明服务端证书没问题，说明服务端并没有被劫持。

数字证书的格式普遍采用的是X.509国际标准，X.509是一种进行身份认证的行业安全标准，在该标准中，用户可生成一段信息及其摘要(亦称作信息“指纹”），并用专用密钥对摘要加密以形成签名，接收者用发送者的公共密钥对签名解密，并将之与收到的信息“指纹”进行比较，以确定其真实性。

SSL/TLS中存储密钥和证书的文件格式大致如下：

- `.jks`

  `.jks`文件表示Java密钥存储仓库（JavaKeyStore），这种格式是Java的专利，表示一个密钥库，可以同时容纳多个公钥和私钥。Java的Keytool工具能直接生成"jks"格式文件，可以将"pfx"格式文件转为"jks"格式文件。

- `.keystore`

  `.keystore`文件其实跟"jks"基本是一样的，是默认生成的密钥存储库格式。

- `.cer`

  `.cer`俗称数字证书文件，该数字证书文件中只包含了公钥以及证书拥有者和颁发者的消息，数字证书文件肯定不会有私钥。“cer"格式文件既可以是BASE64编码的文本文件，也可以是DER编码的二进制文件。

  可以通过Java的Keytool工具，将"cer”证书文件导入到密钥存储仓库（如“jks”格式文件），或者从密钥存储仓库导出证书文件

- `.truststore`

  `.truststore`表示信任证书存储库，它仅仅包含了被信任的通信对方的公钥。

- `.pfx`

  `.pfx`也称为证书文件，是包含了公钥和私钥的二进制格式的证书文件，一般供客户端浏览器使用。与".cer"格式文件不同，“pfx"格式的数字证书是包含有私钥的，而"cer"格式的数字证书里面只有公钥。当然，“pfx"格式文件一般有密码保护，不输入密码是解不了密的。

### 4.4.2 keytool工具管理证书

除了从CA获取证书以外，还可以通过工具生成自签名证书。CA机构证书是需要费用的，一般是正式的项目或者生产环境需要（比如微信小程序不能使用自签名证书而需要CA证书）

Java中管理和生成自签名证书工具为Keytool。Keytool是Java中自带的工具，该工具将密钥（key）和证书（certificates）存在一个格式为"keystore”（或jks）的文件中，然后可以导出自签发的数字证书。在JDK安装过程中，Keytool工具已经解压到对应的JDK的/bin目录中，其可执行文件名称为keytool.exe。

## 4.5 Java程序管理密钥和证书

## 4.6 OIO中使用SSL/TLS

## 4.7 单向和双向认证

## 4.8 Netty中使用SSL/TLS