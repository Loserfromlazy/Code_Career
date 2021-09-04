# Docker学习笔记

## 一、概述

Docker 是一个开源的应用容器引擎，基于Go语言并遵从 Apache2.0 协议开源。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

**docker架构**

Docker 包括三个基本概念:

- **镜像（Image）**：Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu16.04 最小系统的 root 文件系统。
- **容器（Container）**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
- **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程API来管理和创建Docker容器。

Docker 容器通过 Docker 镜像来创建。

容器与镜像的关系类似于面向对象编程中的对象与类。

![img](https://www.runoob.com/wp-content/uploads/2016/04/576507-docker1.png)

### 1.1 安装docker

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start
```

**配置镜像加速器**（以阿里云为例）

~~~
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["加速地址"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
~~~

> 加速地址请自行登录阿里云获取自己的专属地址

## 二、Docker命令（CentOS7）

### 2.1 Docker服务相关命令

~~~
systemctl start docker   docker服务启动
systemctl stop docker    docker服务停止
systemctl restart docker 重启docker服务
systemctl status docker  查看docker服务状态
systemctl enable docker  设置开机启动
~~~

### 2.2 Docker镜像相关命令

~~~
docker images            查看镜像列表
docker search xxx        搜索镜像eg：docker search redis
docker pull xxx          拉取镜像eg：docker pull redis:3.2 #可以加冒号指定版本
docker rmi xxx           删除镜像 #可以通过名字id和版本删除
~~~

> 搜索镜像参数：
>
> **NAME:** 镜像仓库源的名称
>
> **DESCRIPTION:** 镜像的描述
>
> **OFFICIAL:** 是否 docker 官方发布
>
> **stars:** 类似 Github 里面的 star，表示点赞、喜欢的意思。
>
> **AUTOMATED:** 自动构建。

### 2.3 Docker容器相关命令

~~~
docker ps               查看正在运行的容器
docker ps -a            查看所有容器
docker run 参数          创建并启动容器
#参数说明
-i 保持容器运行。通常与-t使用，加入it两个参数后，容器创建后自动进入，退出容器后，容器自动关闭
-t 为容器重新分配一个为输入终端
-d 以守护模式运行，创建一个容器在后台运行，使用docker exec进入容器，退出后容器不会关闭
-it一般称为交互式容器，-id一般称为守护式容器
--name 为创建的容器命名
docker exec xx           进入容器
docker rm xx             删除容器#需要容器停止状态
docker start xx          启动容器
docker inspect xx        查看容器信息
~~~

## 三、Docker容器的数据卷

### 3.1 数据卷概念

数据卷是宿主机中的一个目录或文件，通过挂载到容器实现绑定

当容器目录与数据卷目录绑定后，对方的修改会立即同步

一个数据卷可以被多个容器挂载

一个容器也可以挂载多个数据卷

**数据卷作用**

容器数据持久化

外部机器与容器相通信

容器之间数据交换

### 3.2 配置数据卷

~~~
#创建启动容器时，使用-v参数设置数据卷
docker run ...  -v 宿主机目录或文件:容器内目录或文件
~~~

> 目录必须是绝对路径
>
> 目录不存在会自动创建
>
> 可以挂载多个数据卷

### 3.3 数据卷容器

多容器数据交换有两种方式：挂载同一个数据卷或者使用数据卷容器

~~~
#创建启动数据卷容器，使用-v参数设置数据卷在宿主机上会自动创建目录可使用inspect查看
docker run -it --name=c3 -v /volume centos:7 /bin/bash
#创建启动c1、c2容器，使用--volumes-from 参数设置数据卷
docker run -it --name=c1 --volume-from c3 centos:7 /bin/bash
docker run -it --name=c2 --volume-from c3 centos:7 /bin/bash
~~~

## 四、docker应用部署

### 一、部署MySQL

1. 搜索mysql镜像

```shell
docker search mysql
```

2. 拉取mysql镜像

```shell
docker pull mysql:5.6
```

3. 创建容器，设置端口映射、目录映射

```shell
# 在/root目录下创建mysql目录用于存储mysql数据信息
mkdir ~/mysql
cd ~/mysql
```

```shell
docker run -id \
-p 3307:3306 \
--name=c_mysql \
-v $PWD/conf:/etc/mysql/conf.d \
-v $PWD/logs:/logs \
-v $PWD/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql:5.6
```

- 参数说明：
  - **-p 3307:3306**：将容器的 3306 端口映射到宿主机的 3307 端口。
  - **-v $PWD/conf:/etc/mysql/conf.d**：将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。配置目录
  - **-v $PWD/logs:/logs**：将主机当前目录下的 logs 目录挂载到容器的 /logs。日志目录
  - **-v $PWD/data:/var/lib/mysql** ：将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。数据目录
  - **-e MYSQL_ROOT_PASSWORD=123456：**初始化 root 用户的密码。



4. 进入容器，操作mysql

```shell
docker exec –it c_mysql /bin/bash
```

5. 使用外部机器连接容器中的mysql

![1573636765632](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/docker/1573636765632.png)


### 二、部署Tomcat

1. 搜索tomcat镜像

```shell
docker search tomcat
```

2. 拉取tomcat镜像

```shell
docker pull tomcat
```

3. 创建容器，设置端口映射、目录映射

```shell
# 在/root目录下创建tomcat目录用于存储tomcat数据信息
mkdir ~/tomcat
cd ~/tomcat
```

```shell
docker run -id --name=c_tomcat \
-p 8080:8080 \
-v $PWD:/usr/local/tomcat/webapps \
tomcat 
```

- 参数说明：

  - **-p 8080:8080：**将容器的8080端口映射到主机的8080端口

    **-v $PWD:/usr/local/tomcat/webapps：**将主机中当前目录挂载到容器的webapps



4. 使用外部机器访问tomcat

![1573649804623](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/docker/1573649804623.png)


### 三、部署Nginx

1. 搜索nginx镜像

```shell
docker search nginx
```

2. 拉取nginx镜像

```shell
docker pull nginx
```

3. 创建容器，设置端口映射、目录映射


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

### 四、部署Redis

1. 搜索redis镜像

```shell
docker search redis
```

2. 拉取redis镜像

```shell
docker pull redis:5.0
```

3. 创建容器，设置端口映射

```shell
docker run -id --name=c_redis -p 6379:6379 redis:5.0
```

4. 使用外部机器连接redis

```shell
./redis-cli.exe -h 192.168.149.135 -p 6379
```

























