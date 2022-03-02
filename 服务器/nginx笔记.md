# Nginx学习笔记

转载请声明作者和出处！

本文如有错误，欢迎指正，感激不尽。

## 一、简介

Nginx是lgor Sysoev为俄罗斯访问量第二的rambler.ru站点设计开发的。从2004年发布至今，凭借开源的力量，已经接近成熟与完善。

Nginx功能丰富，可作为HTTP服务器，也可作为反向代理服务器，邮件服务器。支持FastCGI、SSL、Virtual Host、URL Rewrite、Gzip等功能。并且支持很多第三方的模块扩展。

### 1.1 常用功能

1. http代理，反向代理

   Nginx在做反向代理时，提供性能稳定，并且能够提供配置灵活的转发功能。Nginx可以根据不同的正则匹配，采取不同的转发策略，比如图片文件结尾的走文件服务器，动态页面走web服务器，只要你正则写的没问题，又有相对应的服务器解决方案，你就可以随心所欲的玩。并且Nginx对返回结果进行错误页跳转，异常判断等。如果被分发的服务器存在异常，他可以将请求重新转发给另外一台服务器，然后自动去除异常服务器。

   正反向代理区别：

   > 一般的正向代理，比如翻墙，任何可以连接到该代理服务器的软件，就可以通过代理访问任何的其他服务器，然后把数据返回给客户端，这里代理服务器只对客户端负责。而反向代理的话，如果他反向代理了两个服务，那么之后客户端访问这两个服务器的时候，该代理服务器才会给它代理，也就是说，这里的代理服务器只对该代理服务器所代理的服务器负责。这也就是为啥要起这样的名字。
   >
   > 
   >
   > 总结：一个对客户端负责，一个对所代理的服务器负责。一个正，一个反。

2. 负载均衡

   Nginx提供的负载均衡策略有2种：内置策略和扩展策略。内置策略为轮询，加权轮询，Ip hash。扩展策略，可以自行实现任意策略。

   内置负载均衡策略实现：

   轮询：

   ![轮询负载均衡](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E8%BD%AE%E8%AF%A2%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.jpg)

   加权

   ![加权负载均衡](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E5%8A%A0%E6%9D%83%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1.jpg)

   ip hash

   ![ip_hash算法](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/ip_hash%E7%AE%97%E6%B3%95.jpg)

3. web缓存

   Nginx可以对不同的文件做不同的缓存处理，配置灵活，并且支持FastCGI_Cache，主要用于对FastCGI的动态程序进行缓存。配合着第三方的ngx_cache_purge，对制定的URL缓存内容可以的进行增删管理。
   
4. 静态资源部署

### 1.2 Docker上安装nignx

1. 搜索nginx镜像

```shell
docker search nginx
```

2. 拉取nginx镜像

```shell
docker pull nginx
```

3. 创建容器，设置端口映射、目录映射（目录请自行修改）


```shell
# 在/root目录下创建nginx目录用于存储nginx数据信息
mkdir ~/nginx
cd ~/nginx
mkdir conf
cd conf
# 在~/nginx/conf/下创建nginx.conf文件,粘贴下面内容
vim nginx.conf
```

```shell
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}


```


```shell
docker run -id --name=c_nginx \
-p 80:80 \
-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf \
-v $PWD/logs:/var/log/nginx \
-v $PWD/html:/usr/share/nginx/html \
nginx
```

- 参数说明：
  - **-p 80:80**：将容器的 80端口映射到宿主机的 80 端口。
  - **-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf**：将主机当前目录下的 /conf/nginx.conf 挂载到容器的 :/etc/nginx/nginx.conf。配置目录
  - **-v $PWD/logs:/var/log/nginx**：将主机当前目录下的 logs 目录挂载到容器的/var/log/nginx。日志目录

4. 使用外部机器访问nginx

![1573652396669](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/docker/1573650577429.png)

> 若浏览器访问后报403错误，则可能是html目录缺少index.html

### 1.3 CentOS上安装nginx

安装nginx的环境

~~~
yum install gcc-c++
yum install -y pcre pcre-devel
yum install -y zlib zlib-devel
yum install -y openssl openssl-devel
~~~

官网下载nginx安装包然后上传到服务器上

使用命令`tar -zxvf nginx-1.20.2.tar.gz `解压

进入到**解压后的目录**

使用`./configure`命令

然后使用`make`命令编译，然后使用`make install`

完毕之后在/usr/local/下会产⽣⼀个nginx⽬录至此安装完毕

可以进入到nginx的sbin目录中`./nginx`启动nginx

`./nginx -s stop`终止nginx

`./nginx -s reload` 重新加载配置文件

### 1.4 nginx的目录结构

~~~
/var/nignx
|--conf   存放配置文件
|--html   存放nginx默认站点的目录index.html/error.html
|--logs   日志
~~~

## 二、Nginx配置文件

nginx服务器的配置主要在nginx.conf，默认的配置文件内容如下

~~~nginx
#user  nobody;
#worker进程数量，通常设置为和cpu数量相等
worker_processes  1;

#全局错误日志和pid文件位置
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    #单个worker进程的最大并发连接数
    worker_connections  1024;
}


http {
    #引入mime类型的定义文件
    include       mime.types;
    default_type  application/octet-stream;

    #设置日志格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #链接超时时间
    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
~~~

### 2.1 nginx文件结构

nginx.conf默认有三大块 全局块、events块、http块

http块中可以配置多个server块，每个server块又可以配置多个location块

```nginx
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
```

> - 1、**全局块**：配置影响nginx全局的指令。一般有运行nginx服务器的用户组，nginx进程pid存放路径，日志存放路径，配置文件引入，允许生成worker process数等。
> - 2、**events块**：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网路连接，开启多个网络连接序列化等。
> - 3、**http块**：可以嵌套多个server，http块是配置最频繁的部分，常见有：虚拟主机的配置，监听端口的配置，请求转发，反向代理，负载均衡配置等。
> - 4、**server块**：配置虚拟主机的相关参数，一个http中可以有多个server。
> - 5、**location块**：配置请求的路由，以及各种页面的处理情况。

### 2.2 全局块

```
user www;                              #Nginx进程所使用的用户
worker_processes 1;                    #Nginx运行的work进程数量(建议与CPU数量一致或auto)
error_log /log/nginx/error.log  warn   #Nginx错误日志存放路径
#日志级别debug|info|notice|warn|error|crit|alert|emerg 建议不要设置info以下级别
pid /var/run/nginx.pid                 #Nginx服务运行后产生的pid进程号
```

### 2.3 event块

```
events {            
	accrpt_mutex on;          #可选on/off
    worker_connections 1024;  #每个worker进程支持的最大连接数
    use epool;                #事件驱动模型, epoll默认,可选值select/poll/epoll/kqueue等
    multi_accept on;          #可选on/off

}
```

> **accrpt_mutex**:这个配置主要可以用来解决常说的"惊群"问题。大致意思是在某一个时刻，客户端发来一个请求连接，Nginx后台是以多进程的工作模式，也就是说有多个worker进程会被同时唤醒，但是最终只会有一个进程可以获取到连接，如果每次唤醒的进程数目太多，就会影响Nginx的整体性能。如果将上述值设置为on(开启状态)，将会对多个Nginx进程接收连接进行序列号，一个个来唤醒接收，就防止了多个进程对连接的争抢。
>
> **multi_accept:**如果multi_accept被禁止了，nginx一个工作进程只能同时接受一个新的连接。否则，一个工作进程可以同时接受所有的新连接

### 2.4 http块

http块也可以包括全局块、server块

**http全局块指令包括文件引入、MIMR-TYPE定义、日志自定义，连接超时时间、单链接请求数上线等。**

***设置MIME-TYPE***

浏览器中可以显示的内容有HTML、XML、GIF等种类繁多的文件、媒体等资源，浏览器为了区分这些资源，就需要使用MIME-Type。所以说MIME Type是网络资源的媒体类型。Nginx作为web服务器，也需要能够识别前端请求的资源类型。

在nginx的配置文件中默认有两行配置：

~~~
include  mime.types;
default_type  application/octet-stream
~~~

default_type:用来配置Nginx响应前端请求默认的MIME类型。

***自定义服务配置***

Nginx中日志的类型分为access.log和error.log

access.log:用来记录用户所有的访问请求。

error.log:记录nginx本身运行时的错误信息，不会记录用户的访问请求。

Nginx服务器支持对服务日志的格式、大小、输出等进行设置，需要使用到两个指令，分别是access_log和log_format指令。

access_log:用来设置用户访问日志的相关属性。log_format:用来指定日志的输出格式。

***sendfile***

用来设置Nginx服务器是否使用sendfifile()传输文件，该属性可以大大提高Nginx处理静态资源的性能

***keepalive_timeout***

用来设置长连接的超时时间

> 我们都知道HTTP是一种无状态协议，客户端向服务端发送一个TCP请求， 服务端响应完毕后断开连接。 
>
> 如何客户端向服务端发送多个请求，每个请求都需要重新创建一次连接， 效率相对来说比较多，使用keepalive模式，可以告诉服务器端在处理完 一个请求后保持这个TCP连接的打开状态，若接收到来自这个客户端的其他请求，服务端就会利用这个未被关闭的连接，而不需要重新创建一个新连接，提升效率，但是这个连接也不能一直保持，这样的话，连接如果过多，也会是服务端的性能下降，这个时候就需要我们进行设置其的超时时间。 

***keepalive_requests***

用来设置一个keep-alive连接使用的次数

Server块和locaton块比较复杂，此处仅给出示例简单了解即可。

***http块配置示例：***

~~~
//公共的配置定义在http{}
http {  //http层开始
...    
    //使用Server配置网站, 每个Server{}代表一个网站(简称虚拟主机)
    server {
        listen       80;        //监听端口, 默认80
        server_name  localhost; //提供服务的域名或主机名
        access_log host.access.log  //访问日志
        //控制网站访问路径
        location / {
            root   /usr/share/nginx/html;   //存放网站代码路径
            index  index.html index.htm;    //服务器返回的默认页面文件
        }
        //指定错误代码, 统一定义错误页面, 错误代码重定向到新的Locaiton
        error_page   500 502 503 504  /50x.html;
    }
    ...
    //第二个虚拟主机配置
    server {
    ...
    }
    
    include /etc/nginx/conf.d/*.conf;  //包含/etc/nginx/conf.d/目录下所有以.conf结尾的文件

}   //http层结束
~~~

## 三、nginx反向代理和负载均衡

反向代理代理的是服务器，如下图：

![反向代理20220301](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%8620220301.png)

现在我们模拟反向代理，我们在服务器上部署两个SpringBoot项目（等同于部署两个Tomcat），然后使用nginx进行代理，具体如下图：

![反代负载需求20220301](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/%E5%8F%8D%E4%BB%A3%E8%B4%9F%E8%BD%BD%E9%9C%80%E6%B1%8220220301.png)

SpringBoot项目简单创建一个，然后实现`/aaa`和`/bbb`地址的Controller即可。

可以使用`nohup java -jar xxx.jar >/dev/null 2>&1 &`来后台运行SpringBoot的jar包。























