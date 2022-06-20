# Nginx高并发编程

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Spring Cloud、Nginx高并发核心编程尼恩编著、以及互联网资源
>

# 一、Nginx/OpenResty详解

## 1.1 Nginx简介

Nginx有以下3个主要社区分支：

1. Nginx官方版本：更新迭代比较快，并且提供免费版本和商业版本。
2. Tengine：Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上针对大访问量网站的需求添加了很多高级功能和特性。Tengine的性能和稳定性已经在大型的网站（如淘宝网、天猫商城等）得到了很好的检验。它的最终目标是打造一个高效、稳定、安全和易用的Web平台。
3. OpenResty：2011年，中国人章亦春老师把LuaJIT VM嵌入Nginx中，实现了OpenResty这个高性能服务端解决方案。OpenResty是一个基于Nginx与Lua的高性能Web平台，其内部集成了大量精良的Lua库、第三方模块以及大多数的依赖项，用于方便地搭建能够处理超高并发、扩展性极高的动态Web应用、Web服务和动态网关。

> OpenResty的目标是让Web服务直接跑在Nginx服务内部，充分利用Nginx的非阻塞I/O模型，不仅对HTTP客户端请求，甚至对远程后端（诸如MySQL、PostgreSQL、Memcached以及Redis等）都进行一致的高性能响应。OpenResty通过汇聚各种设计精良的Nginx模块（主要由OpenResty团队自主开发）从而将Nginx有效地变成一个强大的通用Web应用平台，使得Web开发人员和系统工程师可以使用Lua脚本语言调动Nginx支持的各种C以及Lua模块，快速构造出足以胜任10KB乃至1000KB以上单机并发连接的高性能Web应用系统。

### 1.1.1 正向代理和反向代理

正向代理和反向代理的使用场景说明：

- 正向代理的主要场景是客户端。由于网络不通等物理原因，需要通过正向代理服务器这种中间转发环节顺利访问目标服务器。当然，也可以通过正向代理服务器对客户端某些详细信息进行一些伪装和改变。
- 反向代理的主要场景是服务端。服务提供方可以通过反向代理服务器轻松实现目标服务器的动态切换，实现多目标服务器的负载均衡等。

通俗点来说，正向代理（如Squid、Proxy）是对客户端的伪装，隐藏了客户端的IP、头部或者其他信息，服务器得到的是伪装过的客户端信息；反向代理（如Nginx）是对目标服务器的伪装，隐藏了目标服务器的IP、头部或者其他信息，客户端得到的是伪装过的目标服务器信息。

### 1.1.2 Nginx启动与停止

这里用windows平台和OpenResty举例：

首先下载OpenResty的压缩包，然后解压，最后双击nginx.exe即可启动，启动后浏览器输入localhost，如果出现以下页面则启动成功。（Nginx同理）

![image-20220616132152559](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220616132152559.png)

关于Nginx的在windows/linux平台更详细的安装过程和编写更方便的nginx/OpenResty脚本的过程，这里推荐尼恩的博客，写的很详细，这里就不再赘述：

> [windows平台安装和脚本编写](https://www.cnblogs.com/crazymakercircle/p/12111283.html)
>
> [linux平台安装和脚本编写](https://www.cnblogs.com/crazymakercircle/p/12115651.html)

## 1.2 Nginx原理

### 1.2.1 Reactor模式

关于Reactor模式，我们在上一篇学习NIO和Netty中，有学到过。这是一种高性能的事件驱动模式，具体可以在[NIO和Netty的学习笔记]()中进行查看。

正是由于 Nginx 使用了高性能的 Reactor 模式，因此是目前并发能力很高的 Web 服务器之一，成为迄今为止使用广泛的工业级 Web 服务器。当然，Nginx 也解决了著名的网络读写的 C10K问题。什么是 C10K 问题呢？网络服务在处理数以万计的客户端连接时，往往出现效率低下甚至完全瘫痪，这类问题就被称为 C10K 问题。

### 1.2.2 Nginx的两类进程

nginx有两种启动方式：一种是单进程启动，还有一种是多进程启动。在单进程启动时，系统中只有一个进程，该进程即当Master角色，又当Worker角色。在多进程启动时，系统中有且只有一个master进程，至少有一个worker进程。如下图：

![image-20220616142258489](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220616142258489.png)

nginx启动后会以守护进程的方式在后台运行，其后台进程有两类，一类是Master进程，一类是Worker进程。

Master管理进程的工作主要有以下两点：

1. 负责调度Worker工作进程，比如加载配置、启动工作进程、接收外界信号、向Worker发送信号、监控Worker进程的运行状态等。
2. 负责创建监听套接字，交由Worker进程进行连接监听。

而Worker工作进程主要用来处理网络事件当一个 Worker 进程在接收一条连接通道之后，就开始读取请求、解析请求、处理请求，处理完成产生数据后，再返回给客户端，最后断开连接通道。

各个Worker进程之间是对等且相互独立的，它们同等竞争来自客户端的请求，一个请求只可能在一个 Worker 进程中处理。这都是典型的 Reactor 模型中 Worker 进程（或者线程）的职能。

如果启动了多个 Worker 进程，那么每个 Worker 子进程独自尝试接收已连接的 Socket 监听通道，accept 操作默认会上锁，优先使用操作系统的共享内存原子锁，如果操作系统不支持，就使用文件上锁。

Worker 进程的接收操作也可以不使用锁，在多个进程同时接收时，当一个连接进来的时候多个工作进程同时被唤起，则会导致惊群问题。而在上锁的场景下，只会有一个 Worker阻塞在 accept 上，其他的进程会因为不能获取锁而阻塞，所以上锁的场景不存在惊群问题。

### 1.2.3 Nginx的模块化设计

Nginx服务器被分解为多个模块，各个模块遵循高内聚、低耦合的原则，每个模块聚焦一个功能，高度模块化是Nginx的架构基础。

在Nginx中，一个模块包括一系列命令，和这些命令的处理函数。Nginx的Worker进程在执行过程中会通过配置文件的配置定位到对应模块的某个命令，然后调用命令对应的处理函数完成处理。Nginx的Worker进程会首先调用Core核心模块，这个模块内部会维护一个循环，主要包括事件的收集、分发、处理。Core模块负责执行网络请求处理的基础操作，比如网络读写、存储读写、内容传输、外出过滤以及请求发往上游服务器等。

Nginx的Core模块是启动时一定会加载的，其余模块只有在解析配置时遇到了才会加载对应的模块。Core模块为其他模块构建了基本的运行时环境。当然除了Core，Nginx还有其他比如Event、Conf、HTTP等一系列模块，Nginx的模块结构如下图（图片来自于Spring Cloud、Nginx高并发核心编程一书）：

![image-20220616143140075](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220616143140075.png)

主要模块：

1. Core核心模块：必不可少，提供了错误日志记录、配置文件解析、Reactor事件驱动、进程管理等核心功能。
2. 标准HTTP模块：提供解析HTTP协议的相关功能，比如端口配置、网页编码配置、HTTP响应头设置等
3. 可选HTTP模块：主要用于扩展标准的HTTP功能，让Nginx能处理一些特殊的服务，比如Flash多媒体、网络传输压缩、SSL等
4. 邮件服务模块：主要用于支持Nginx的邮件服务，包括PoP3、IMAP、SMTP的支持。
5. 第三方模块：用于拓展Nginx服务器的功能，定制自定义功能，比如支持JSON、Lua等。

### 1.2.4 Nginx配置文件的结构

一个Nginx功能模块包含一系列命令以及对应的处理函数。而Nginx会根据配置文件中的指令就知道对应到哪个模块的哪个命令，然后调用对应的处理函数来处理。

一个Nginx配置文件包含若干配置项，每个配置项由配置指令和指令参数两部分组成，可以简单认为配置项就是一个键值对：

![image-20220616145413824](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220616145413824.png)

Nginx配置文件中的配置指令如果包含空格，就需要用单引号和双引号引起来。指令参数如果是简单的字符串就以分号结束，如果是复杂的多行字符串，配置项就用花括号`{}`括起来。

Nginx配置文件的配置块大致有5种，其结构如下：

~~~nginx
...              #全局块

events {         #events块
   ...
}

http      #http块
{
    ...   #http全局块
    server        #server块
    { 
        ...       #server全局块
        location [PATTERN]   #location块
        {
            ...
        }
        location [PATTERN] 
        {
            ...
        }
    }
    server
    {
      ...
    }
    ...     #http全局块
}
mail #mail服务配置块
{
    ...#email相关协议配置
}
~~~

1. **全局块**：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
2. **events块**：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
3. **http块**：可以嵌套多个server，http块是配置最频繁的部分，常见有：虚拟主机的配置，监听端口的配置，请求转发，反向代理，负载均衡配置等。
4. **server块**：配置虚拟主机的相关参数，一个http中可以有多个server。
5. **location块**：配置请求的路由，以及各种页面的处理情况。
6. **mail块：**Nginx 为 email 相关协议（如 SMTP/IMAP/POP3）提供反向代理时，mail 服务配置块负责配置一些相关的配置项

### 1.2.5 Nginx请求处理流程

Nginx中HTTP请求的处理流程可以分为4步：

1. 读取解析请求行
2. 读取解析请求头
3. 多阶段处理，执行handler处理器列表
4. 将结果返回给客户端

多阶段处理时Nginx处理HTTP请求流程中重要的一步，Nginx将请求划分为11个阶段，在完成第一步和第二步后，Nginx将请求封装到请求结构体中ngx_http_request中，然后进入到第三步。

![image-20220617152852910](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220617152852910.png)

1. post-read阶段：完成读取解析请求头和请求行后，首先就是这个阶段，注册在这个阶段的处理器不多，标准模块的ngx_realip处理器就在这个模块，而ngx_realip的作用是改写请求的来源地址。

   > ngx_realip：当Nginx处理的请求经过了某个正向代理服务器的转发后，ip地址可能就不是真实IP地址了，而是变成下游服务器的IP，解决办法之一就是将原来的IP地址放到HTTP请求头中，Nginx获取后传给Nginx的上游服务器，ngx_realip就是这个作用。
   >
   > 比如以下配置：将正向代理服务器192.168.0.100的所有请求的IP地址修改为请求头X-MY-IP所指定的值，然后放在$remote_addr内置标准变量中。
   >
   > ~~~nginx
   > server{
   >     listen 8080;
   >     set_real_ip_from 192.168.0.100;
   >     real_ip_header X-MY-IP;
   >     location /test {
   >         echo "from: $remote_addr"
   >     }
   > }
   > ~~~

2. server-rewrite

   这个阶段是server块中的请求地址重写阶段，在进行UIR与location路由规则匹配之前可以修改请求的URI地址。大部分在server块中的配置项都在这个阶段运行，比如：

   ~~~nginx
   server{
       listen 8080;
       set $a hello;#在server-rewrite阶段运行
       location /test {
           set $b "$a world";
           echo $b;
       }
       set $b hello;#在server-rewrite阶段运行
   }
   ~~~

   两个变量赋值直接写在server配置块中，因此他们就运行在server-rewrite阶段。

3. find-config

   这个阶段叫配置查找阶段，主要功能是根据URL地址去匹配location路由表达式。此阶段由Nginx HTTP Core模块全部负责，并完成当前请求URL和location配置块之间的配对工作，这个阶段不支持 Nginx 模块注册处理程序。

   在`find-config`阶段之前，客户端请求并没有与任何location配置块相关联,在此之前的 post-read 和 server-rewrite 阶段来说，只有 server 配置块以及更外层作用域中的配置项才会起作用，location 配置块中的配置项不起作用。

4. rewrite

   由于Nginx在find-config阶段完成了当前请求和location的匹配，因此从rewrite阶段开始，location配置块中的指令就起作用了。

   这个阶段也叫请求地址重写阶段，rewrite 阶段也叫请求地址重写阶段，注册在 rewrite 阶段的指令首先是 ngx_rewrite 模块的指令，比如 break、if、return、rewrite、set 等。其次，第三方ngx_lua 模块中的 set_by_lua 指令和rewrite_by_lua 指令也能在此阶段注册。

5. post-rewrite

   请求地址URI重写提交阶段，防止修改递归修改URI造成死循环（一个请求执行 10 次就会被 Nginx 认定为死循环）该阶段只能由 Nginx HTTP Core（ngx_http_core_module）模块实现。

6. preaccess

   访问权限检查准备阶段，控制访问频率的 ngx_limit_req 模块和限制并发度的 ngx_limit_zone模块的相关指令就注册在此阶段。

7. access

   在访问权限检查阶段，配置指令大多是执行访问控制类型的任务，比如检查用户的访问权限、检查用户的来源IP地址是否合法等。在此阶段注册的指令有：HTTP标准模块ngx_http_access_module 的指令、第三方 ngx_auth_request 模块的指令、第三方 ngx_lua 模块的access_by_lua 指令等。

   > 比如，deny和allow指令属于ngx_http_access_module模块，使用示例如下：
   >
   > ~~~nginx
   > server{
   >     #...
   >     #拒绝全部
   >     location = /denyall {
   >         dent all;
   >     }
   >     #允许来源ip属于192.168.0.0/24网段或127.0.0.1的请求，其他来源的ip全部拒绝
   >     location =  /allowsome {
   >         allow 192.168.0.0/24;
   >         allow 127.0.0.1;
   >         deny all;
   >         echo "this is ok";
   >     }
   >     #...
   > }
   > ~~~
   >
   > 如果同一个location配置了多个allow/deny配置项，access阶段的配置项之间是按照配置的先后顺序匹配的，匹配成功一个跳出。在上面的例子中，如果客户端的ip是127.0.0.1，则匹配到` allow 127.0.0.1;`配置项后就不在匹配后面的`deny all;`，也就是说不会被拒绝。如果这些配置项的指令来自不同的模块，则每个模块会执行一个访问控制类型的指令。
   >
   > echo 指令用于返回内容，在 location 上下文中，该指令注册在 content 生产阶段。由于 echo 指令不是注册在 access 阶段，因此在 access 阶段不执行该指令的配置项。

8. post-access

   访问权限检查提交阶段。如果请求不被允许访问Nginx服务器，该阶段负责向用户返回错误响应。在access阶段可能存在多个访问控制模块的指令注册，post-access阶段的satisfy配置指令可以用于控制它们之间的协作方式。比如：

   ~~~nginx
   location =  /satisfy-demo {
           satisfy any;
           access_by_lua "ngx.exit(ngx.OK)";
           deny all;
           echo "this is ok";
       }
   ~~~

   在这个例子中，deny指令属于HTTP标准模块的ngx_http_access_module访问控制模块，而access_by_lua指令属于第三方ngx_lua模块，两个模块都有自己的计算结果，需要最终进行结果统一。

   而这个统一工作由satisfy指令负责：

   - 逻辑或操作

     具体的配置项为`satisfy any;`,表示访问控制模块A、B、C或更多只要其中任意一个通过验证就算通过

   - 逻辑与操作

     具体的配置项为`satisfy all;`,表示访问控制模块A、B、C或更多全部模块都通过验证才算通过

9. try-files

   如果HTTP请求访问静态资源文件，那么try-files配置项可以使这个请求按顺序访问多个静态资源文件，直到某个静态文件符合选取条件。这个阶段不支持Nginx模块注册处理程序，只有一个标准配置指令`try-files`。

   try-files 指令接收两个以上任意数量的参数，每个参数都指定了一个 URI，Nginx 会在 try-files阶段依次把前 N-1 个参数映射为文件系统上的对象（文件或者目录），然后检查这些对象是否存在。若 Nginx 发现某个文件系统对象存在，则查找成功，进而在 try-files 阶段把当前请求的 URI改写为该对象所对应的参数 URI（但不会包含末尾的斜杠字符，也不会发生“内部跳转”）。如果前 N-1 个参数所对应的文件系统对象都不存在，try-files 阶段就会立即发起“内部跳转”，跳转到最后一个参数（第 N 个参数）所指定的 URI。

   这里举个例子：

   ~~~nginx
   root /var/www;    #root指令把查找文件的根目录配置为/var/www/
   location = /try_files-demo {
       try_files /foo /bar  /last;
   }
   # 对应到前面 try_files的最后一个URI
   location /last{
       echo "uri: $uri";
   }
   ~~~

   这里 try-files 会在文件系统查找前两个参数对应的文件 /var/www/foo 和 /var/www/bar 所对应的文件是否存在。如果不存在，此时 Nginx 就会在 try-files 阶段发起到最后一个参数所指定的URI（/last）的内部跳转。

10. content

    大部分 HTTP 模块会介入内容产生阶段，是所有请求处理阶段中重要的阶段。Nginx 的 echo指令、第三方 ngx_lua 模块的 content_by_lua 指令都注册在此阶段。

    > 这里要注意的是，每一个 location 只能有一个“内容处理程序”，因此，当在 location 中同时使用多个模块的 content 阶段指令时，只有一个模块能成功注册成为“内容处理器”。例如 echo和 content_by_lua 同时注册，最终只会有一个生效，但具体是哪一个生效，结果是不稳定的。

11. log

    日志模块处理阶段，记录日志。

Nginx 将一个HTTP请求分为11个处理阶段，这样做让每个HTTP模块可以只专注于完成一个独立、简单的功能。而一个请求的完整处理过程由多个 HTTP 模块共同合作完成，可以极大地提高多个模块合作的协同性、可测试性和可扩展性。

Nginx 请求处理的 11 个阶段中，有些阶段是必备的，有些阶段是可选的，各个阶段可以允许多个模块的指令同时注册。但是，find-config、post-rewrite、post-access、try-files 四个阶段是不允许其他模块的处理指令注册的，它们仅注册了 HTTP 框架自身实现的几个固定的方法。同一个阶段内的指令，Nginx 会按照各个指令的上下文顺序执行对应的 handler 处理器方法。

## 1.3 Nginx的基础配置

### 1.3.1 events事件驱动配置



## 1.4 location路由规则匹配



## 1.5 Nginx的rewrite模块



## 1.6 反向代理和负载均衡

