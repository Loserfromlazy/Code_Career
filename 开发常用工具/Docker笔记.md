# Docker学习笔记

## 一、概述

Docker 是一个开源的应用容器引擎，基于Go语言并遵从 Apache2.0 协议开源。当人们说“Docker”时，他们通常是指 Docker Engine，它是一个客户端 - 服务器应用程序，由 Docker守护进程、一个REST API指定与守护进程交互的接口、和一个命令行接口（CLI）与守护进程通信（通过封装REST API）。Docker Engine 从 CLI 中接受docker 命令，例如 docker run 、docker ps 来列出正在运行的容器、docker images 来列出镜像，等等。

Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。

**docker架构**

Docker 基本组成:

- **Docker主机（Host）**：安装了Docker程序的机器
- **Docker镜像（Image）**：Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu16.04 最小系统的 root 文件系统。
- **Docker容器（Container）**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
- **Docker仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程API来管理和创建Docker容器。

Docker 容器通过 Docker 镜像来创建。

容器与镜像的关系类似于面向对象编程中的对象与类。

![img](https://www.runoob.com/wp-content/uploads/2016/04/576507-docker1.png)

### 1.1 安装docker

```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
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

Docker命令结构图如下：

![image-20221107102131523](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221107102131523.png)

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
#常用参数说明
-i 保持容器运行。通常与-t使用，加入it两个 参数后，容器创建后自动进入，退出容器后，容器自动关闭
-t 为容器重新分配一个为输入终端
-d 以守护模式运行，创建一个容器在后台运行，使用docker exec进入容器，退出后容器不会关闭
-it一般称为交互式容器，-id一般称为守护式容器
--name 为创建的容器命名
-p 指定端口映射
-v 绑定数据卷
docker exec xx           进入容器
docker rm xx             删除容器#需要容器停止状态
docker start xx          启动容器
docker inspect xx        查看容器信息
docker stop xx -t        停止容器 -t10s内没停止则进行强制停止
docker logs 参数 xx
#参数说明：
-f 跟踪日志输出
--tail 仅列出最新n条容器日志
~~~

## 三、Docker容器的数据卷

### 3.1 数据卷概念

数据卷是宿主机中的一个目录或文件，通过挂载到容器实现绑定。当我们使用docker容器时，会产生一系列的数据文件，这些数据文件在我们删除docker容器时是会消失的，但是某些内容我们不想让他消失。docker将应用和运行环境打包成容器发布，我们希望在运行过程中产生的部分数据是持久化的，并且实现数据共享。简单的理解，可以将docker容器数据卷看成u盘，它存在一个或多个容器中，它的特点是：

1. 数据卷可以在容器中共享或重用数据
2. 数据卷的更改可以立即生效
3. 数据卷的更改不会包含在镜像的更新中
4. 数据卷会一直存在，即使容器被删除
5. 数据卷的生命周期一直持续到没有容器使用它为止。

容器数据主要有两种方式去管理：一是使用数据卷直接映射到本地主机环境，二是使用特定的容器去维护数据卷

**数据卷类型**

- 宿主机数据卷：直接在宿主机文件系统中，但是容器可以访问
- 命名的数据卷：磁盘上Docker管理的数据卷，但是这个数据卷有名字
- 匿名数据卷：磁盘上Docker管理的数据卷，但没有名字由docker管理

一般推荐使用宿主机数据卷，且数据卷都在宿主机文件系统中，只是后两种在docker管理的目录下，一般是`/var/lib/docker/volumes`

**数据覆盖问题**

- 如果挂载一个空的数据卷到容器中的一个非空目录中，那么这个目录下的文件会被复制到数据卷中
- 如果挂载一个非空的数据卷到容器中的一个目录中，那么容器中的目录会显示数据卷中的数据。如果原来容器中的目录有数据，那么原始数据会被隐藏掉。

### 3.2 配置数据卷

注意：挂载数据卷时最好通过run而不是通过create/start，因为create/start创建启动容器后，在挂载数据卷非常麻烦，要修改很多配置文件。docker官网推荐尽量进行目录挂载。

~~~
#创建启动容器时，使用-v参数设置数据卷
docker run ...  -v 宿主机目录或文件:容器内目录或文件
#容器的目录权限可以通过ro(readonly)只读或rw(readwrite)读写进行控制
docker run ...  -v 宿主机目录或文件:容器内目录或文件:ro
docker run ...  -v 宿主机目录或文件:容器内目录或文件:rw
~~~

> 我们在管理数据时还可以用cp命令进行管理，docker cp命令用于容器与宿主机之间的数据拷贝，语法：
>
> 宿主机文件复制到容器内
>
> docker cp 参数 SRC_PATH CONTAINER:DEST_PATH
>
> 容器内文件复制到宿主机
>
> docker cp 参数 CONTAINER:SRC_PATH  DEST_PATH
>
> 举个例子：1.nginx宿主机index.html复制并覆盖容器内的index.html页面
>
> docker cp /data/index.html nginx:/usr/share/nginx/html/index.html
>
> 2.nginx容器内配置文件复制到宿主机
>
> docker cp nginx:/etc/nginx/nginx/conf /data

### 3.3 数据卷容器

如果用户需要在多个容器之间共享一些持续更新的数据，最简单的方式是使用数据卷容器。数据卷容器也是一个容器，但是它的目的是专门用来提供数据卷供其他容器挂载

> 多容器数据交换有两种方式：挂载同一个数据卷或者使用数据卷容器
>

~~~
#创建启动数据卷容器，使用-v参数设置数据卷在宿主机上会自动创建目录可使用inspect查看
docker run -it --name=c3 -v /volume centos:7 /bin/bash
#创建启动c1、c2容器，使用--volumes-from 参数设置数据卷
docker run -it --name=c1 --volume-from c3 centos:7 /bin/bash
docker run -it --name=c2 --volume-from c3 centos:7 /bin/bash
~~~

## 四、docker应用部署案例

### 1 部署MySQL

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
-p 3306:3306 \
--name=mysql \
-v $PWD/conf:/etc/mysql/conf.d \
-v $PWD/logs:/logs \
-v $PWD/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
mysql:5.6
```

- 参数说明：
  > - **-p 3306:3306**：将容器的 3306 端口映射到宿主机的 3306 端口。
  >
  > - **-v $PWD/conf:/etc/mysql/conf.d**：将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。配置目录
  >
  > - **-v $PWD/logs:/logs**：将主机当前目录下的 logs 目录挂载到容器的 /logs。日志目录
  >
  > - **-v $PWD/data:/var/lib/mysql** ：将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。数据目录
  >
  > - **-e MYSQL_ROOT_PASSWORD=123456：**初始化 root 用户的密码。
  >
  > - -v : 挂载宿主机目录和 docker容器中的目录，前面是宿主机目录，后面是容器内部目录
  >
  >   -d : 后台运行容器
  >
  >   -p 映射容器端口号和宿主机端口号
  >
  >   -e 环境参数，MYSQL_ROOT_PASSWORD设置root用户的密码

4. 进入容器，操作mysql

```shell
docker exec -it mysql /bin/bash
```

`ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';`

5. 使用外部机器连接容器中的mysql

![1573636765632](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/docker/1573636765632.png)


### 2 部署Tomcat

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


### 3 部署Nginx

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
#user  nobody;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;
    #gzip  on;
    server {
        listen       80;
        server_name  localhost;
	    root html;
    }
}

```

运行命令：


```shell
docker run -id --name=nginx \
-p 80:80 \
-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf \
-v $PWD/logs:/var/log/nginx \
-v $PWD/html:/etc/nginx/html/ \
nginx
```

- 参数说明：
  - **-p 80:80**：将容器的 80端口映射到宿主机的 80 端口。
  - **-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf**：将主机当前目录下的 /conf/nginx.conf 挂载到容器的 :/etc/nginx/nginx.conf。配置目录
  - **-v $PWD/logs:/var/log/nginx**：将主机当前目录下的 logs 目录挂载到容器的/var/log/nginx。日志目录

4. 使用外部机器访问nginx

![1573652396669](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/docker/1573650577429.png)

### 4 部署Redis

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

### 5 部署ES、Kibana

> - https://www.elastic.co/guide/en/elasticsearch/reference/8.6/advanced-configuration.html#set-jvm-options
> - https://blog.csdn.net/qq_39314099/article/details/105532460

1. 拉取镜像

   ```sh
   docker pull docker.elastic.co/elasticsearch/elasticsearch:7.14.0
   ```

2. 创建启动容器

   

## 五、虚拟化技术介绍

## 六、Dockerfile

Dockerfile 是一个用来构建镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。

Docker创建镜像的方式一共有三种：

1. 基于已有的镜像创建
2. 基于Dockerfile创建
3. 基于本地模板导入

### 6.1 docker镜像原理

操作系统组成部分：

进程调度子系统，进程通信子系统，内存管理子系统，设备管理子系统，文件管理子系统，网络通信子系统，作业控制子系统

Linux文件系统有bootfs和rootfs两部分组成

> bootfs：包含bootloader（引导加载程序）和 kernel（内核）
> rootfs： root文件系统，包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件
> 不同的linux发行版，bootfs基本一样，而rootfs不同，如ubuntu，centos等

dokcer镜像是由特殊的文件系统叠加而成，最底层是bootfs，并使用宿主机的bootfs

第二层是root文件系统rootfs基础镜像，成为baseimage

然后在往上可以叠加其他镜像文件，统一文件系统技术能够将不同层整合成一个文件系统，为这些层提供了一个统一的视角，这样就隐藏了多层的存在，在用户看来只有一个文件系统。一个镜像可以放在另一个镜像上面，位于下面的镜像成为父镜像，最底层的镜像称为基础镜像。

当一个镜像启动容器时，docker会在最顶层加载一个读写文件系统作为容器

### 6.2 docker镜像制作

**基于已有的镜像制作**

docker commit命令：从容器创建成一个新镜像

~~~
docker commit 参数 容器id 镜像名称:版本号
#参数说明
-a 提交的镜像作者
-c 使用dockerfile指令创建镜像
-m 提交时的文字说明
-p commit时将容器暂停
docker save -o 压缩文件名称 镜像名称:版本号
docker load -i 压缩文件名称
~~~

举个例子，基于已经运行的nginx容器制作新镜像：

比如我们现在运行的nginx镜像如下：

![image-20221107133613608](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221107133613608.png)

然后我们使用命令：

`docker commit -a "loserfromlazy" -m "test"  nginx testnginx:v1`如下图，新镜像已经制作完成：

![image-20221107133901551](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221107133901551.png)

**Dockerfile制作镜像**

dockerfile是一个文本文件，包含了一条条指令，每一条指令构建一层镜像，最终构建出一个新的镜像。对于开发人员，可以为开发团队提供一个完全一致的开发环境。对于测试人员：可以直接拿开发时所构建的镜像或者通过Dockerfile文件构建一个新的镜像开始工作了。对于运维人员：在部署时，可以实现应用的无缝移植。

Dockerfile分为四部分：基础镜像信息、维护者信息、 镜像操作指令和容器启动时执行指令。

在编写好dockerfile后就可以使用docker build命令制作镜像了，语法如下：

`docker build 参数 Path|URL`

其中常用参数如下：

- --build-arg=[]  设置镜像创建时的变量
- -f 指定要使用的Dockerfile的路径
- --rm 设置镜像成功后删除中间容器
- --tag  镜像的名字及标签，通常是name:tag或name的格式

dockerfile常用字段：

| 关键字      | 作用                     | 备注                                                         |
| ----------- | ------------------------ | ------------------------------------------------------------ |
| FROM        | 指定父镜像               | 指定dockerfile基于哪个image构建                              |
| MAINTAINER  | 作者信息                 | 用来标明这个dockerfile谁写的                                 |
| LABEL       | 标签                     | 用来标明dockerfile的标签 可以使用Label代替Maintainer 最终都是在docker image基本信息中可以查看 |
| RUN         | 执行命令                 | 执行一段命令 默认是/bin/sh 格式: RUN command 或者 RUN ["command" , "param1","param2"] |
| CMD         | 容器启动命令             | 提供启动容器时候的默认命令 和ENTRYPOINT配合使用.格式 CMD command param1 param2 或者 CMD ["command" , "param1","param2"] |
| ENTRYPOINT  | 入口                     | 一般在制作一些执行就关闭的容器中会使用                       |
| COPY        | 复制文件                 | build的时候复制文件到image中                                 |
| ADD         | 添加文件                 | build的时候添加文件到image中 不仅仅局限于当前build上下文 可以来源于远程服务 |
| ENV         | 环境变量                 | 指定build时候的环境变量 可以在启动的容器的时候 通过-e覆盖 格式ENV name=value |
| ARG         | 构建参数                 | 构建参数 只在构建的时候使用的参数 如果有ENV 那么ENV的相同名字的值始终覆盖arg的参数 |
| VOLUME      | 定义外部可以挂载的数据卷 | 指定build的image那些目录可以启动的时候挂载到文件系统中 启动容器的时候使用 -v 绑定 格式 VOLUME ["目录"] |
| EXPOSE      | 暴露端口                 | 定义容器运行的时候监听的端口 启动容器的使用-p来绑定暴露端口 格式: EXPOSE 8080 或者 EXPOSE 8080/udp |
| WORKDIR     | 工作目录                 | 指定容器内部的工作目录 如果没有创建则自动创建 如果指定/ 使用的是绝对地址 如果不是/开头那么是在上一条workdir的路径的相对路径 |
| USER        | 指定执行用户             | 指定build或者启动的时候 用户 在RUN CMD ENTRYPONT执行的时候的用户 |
| HEALTHCHECK | 健康检查                 | 指定监测当前容器的健康监测的命令 基本上没用 因为很多时候 应用本身有健康监测机制 |
| ONBUILD     | 触发器                   | 当存在ONBUILD关键字的镜像作为基础镜像的时候 当执行FROM完成之后 会执行 ONBUILD的命令 但是不影响当前镜像 用处也不怎么大 |
| STOPSIGNAL  | 发送信号量到宿主机       | 该STOPSIGNAL指令设置将发送到容器的系统调用信号以退出。       |
| SHELL       | 指定执行脚本的shell      | 指定RUN CMD ENTRYPOINT 执行命令的时候 使用的shell            |

### 6.3 案例

**案例一：自定义Centos7镜像**

自定义centos7镜像需求：

1.默认登录路径为/usr

2.可以使用vim

~~~dockerfile
FROM centos:7
MAINTAINER Loserfromlazy <loserfromlazy@163.com>
RUN yun install -y vim
WORKDIR /usr
CMD /bin/bash
~~~

`docker  build -f dockerfile的文件路径 -t 镜像名:版本`

**案例二：发布SpringBoot项目**

定义dockerfile发布springboot项目

~~~dockerfile
FROM java:8
MAINTAINER Loserfromlazy <loserfromlazy@163.com>
ADD springboot.jar app.jar
CMD java -jar app.jar
~~~

`docker build -f  dockerfile的文件路径 -t 镜像名:版本`

**案例三：MySQL镜像修改时区**

```dockerfile
FROM mysql:8.0.18
MAINTAINER mysql from date UTC by Asia/Shanghai "loserfromlazy@163.com"
ENV TZ Asia/Shanghai
```

`docker build --rm -t tmysql:8.0.18 .`这里使用`.`说明在当前目录下有名称为Dockerfile的文件，使用它去创建镜像

![image-20221107181833104](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221107181833104.png)

## 七、Docker-compose

在实际生产环境中，一个应用往往由许多服务构成，而 docker 的最佳实践是一个容器只运行一个进程，因此运行多个微服务就要运行多个容器。多个容器协同工作需要一个有效的工具来管理他们，定义这些容器如何相互关联。compose 应运而生。compose 是用来定义和运行一个或多个容器(通常都是多个)运行和应用的工具。使用 compose 可以简化容器镜像的构建以及容器的运行。

compose 使用 YAML 文件来定义多容器之间的关系。一个 docker-compose up 就可以把完整的应用跑起来。 本质上， compose 把 YAML 文件解析成 docker 命令的参数，然后调用相应的 docker 命令行接口，从而将应用以容器化的方式管理起来。它通过解析容器间的依赖关系顺序地启动容器。而容器间的依赖关系由 YAML 文件中的 links 标记指定。

### 7.1 安装

通过以下命令安装，如不想用2.2.2版本请自行替换，最新发行的版本地址：https://github.com/docker/compose/releases

`sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`

下载完成后使用命令给权限`sudo chmod +x /usr/local/bin/docker-compose`

然后进行验证`docker-compose version`

### 7.2 yml配置文件及常用指令

Docker Compose 使用 YAML 文件来定义多服务的应用。YAML 是 JSON 的一个子集，因此也可以使用JSON。Docker Compose 默认使用文件名 docker-compose.yml。当然，也可以使用 -f 参数指定具体文件。

Docker Compose 的yaml文件包含四个1级key：

- version：必须指定，而且总是位于文件第一行，它定义了compose文件格式（主要是API）的版本。它并不是定义Docker Compose或Docker引擎的版本号。
- services：用于定义不同的应用服务。比如我们可以定义一个mysql数据库服务，或者定义一个微服务。Dockers Compose会将每个服务部署在各自的容器中。
- networks：用于指引Docker创建新的网络。默认情况下，Docker Compose会创建bridge网络，这是一种单主机网络，只能实现同一主机上容器的连接，当然，也可以用driver属性指定不同的网络类型。
- volumes：用于指引Docker来创建新的卷

这里给出除了上面四个一级key以外，其他常用的key：

- image：指定容器运行的镜像，会从docker hub上下载并构建镜像。

- build：指定构建镜像上下文路径，在该路径下有Dockerfile文件，根据该文件构建镜像。

- container_name：指定自定义容器名称，而不是生成的默认名称。

- hostname：hostname 声明用于服务容器的自定义主机名。

- restart：重启策略

  - no：是默认的重启策略，在任何情况下都不会重启容器。
  - always：容器总是重新启动。
  - on-failure：在容器非正常退出时（退出状态非0），才会重启容器。
  - unless-stopped：在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

- ports：暴露容器端口。注意端口映射决不能与 network_mode: host 一起使用，这样做一定会导致运行时错误。

- volumes：也可以做非一级key，将主机的数据卷或着文件挂载到容器里。

- environment：添加环境变量。您可以使用数组或字典、任何布尔值，布尔值需要用引号引起来，以确保 YML 解析器不会将其转换为 True 或 False。例子：

  ```yaml
  environment:
    RACK_ENV: development
    SHOW: 'true'
  ```

- ulimits：覆盖容器默认的 ulimit。

- networks：配置容器连接的网络，引用顶级 networks 下的条目。

- depends_on：设置依赖关系。例子：

  ```yaml
  version: "3.7"
  services:
    web:
      build: .
      depends_on:
        - db
        - redis
    redis:
      image: redis
    db:
      image: postgres
  ```

  - docker-compose up ：以依赖性顺序启动服务。在上面例子中，先启动 db 和 redis ，才会启动 web。
  - docker-compose up SERVICE ：自动包含 SERVICE 的依赖项。在上面例子中，docker-compose up web 还将创建并启动 db 和 redis。
  - docker-compose stop ：按依赖关系顺序停止服务。在上面例子中，web 在 db 和 redis 之前停止。

关于docker compose的更多命令可以自行去查看[官方文档](https://docs.docker.com/compose/compose-file/)，这里也贴出菜鸟教程的地址[docker-compose](https://www.runoob.com/docker/docker-compose.html)

举个例子，例子来自官网：

创建Dockerfile：

~~~dockerfile
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY . .
CMD ["flask", "run"]
~~~

编写Docker compose：

~~~yaml
# yaml 配置
version: '3' #版本
services:
  web:
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
~~~

这个例子中有两个服务：web 和 redis。

- **web**：该 web 服务使用从 Dockerfile 当前目录中构建的镜像。然后，它将容器和主机绑定到暴露的端口 5000。
- **redis**：该 redis 服务使用 Docker Hub 的公共 Redis 映像。

### 7.3 Dockers Compose常用命令

这里以nginx举例：

~~~
#构建建启动nignx容器
docker-compose up -d nginx                     

#进入nginx容器中
docker-compose exec nginx bash            

#将会停止UP命令启动的容器，并删除容器
docker-compose down                             

#显示所有容器
docker-compose ps                                   

#重新启动nginx容器
docker-compose restart nginx                   

#构建镜像
docker-compose build nginx      

#不带缓存的构建
docker-compose build --no-cache nginx 

#查看nginx的日志
docker-compose logs  nginx                      

#查看nginx的实时日志
docker-compose logs -f nginx                   

#验证（docker-compose.yml）文件配置，
#当配置正确时，不输出任何内容，当文件配置错误，输出错误信息
docker-compose config  -q                        

#以json的形式输出nginx的docker日志
docker-compose events --json nginx       

#暂停nignx容器
docker-compose pause nginx                 

#恢复ningx容器
docker-compose unpause nginx             

#删除容器
docker-compose rm nginx                       

#停止nignx容器
docker-compose stop nginx                    

#启动nignx容器
docker-compose start nginx   
~~~

## 八、推送镜像到中央仓库

中央仓库地址如下，如果没有账号的需要先注册账号。免费版的只能创建公开仓库。[官网](https://hub.docker.com/)

下面我以我自己的mit6.s081-gdb镜像为例进行推送演示：

1. 注册账号，这一步略

2. 使用docker login命令登录

   ![image-20221216104420878](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221216104420878.png)

3. 本地镜像打标记

   ![image-20221216105241598](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221216105241598.png)

4. 推送镜像

   使用`docker push loserfromlazy/mit6.s081-gdb`推送镜像，这里的loserfromlazy/mit6.s081-gdb是我自己的本地镜像名，可以使用`docker images`查看。

   ![image-20221216104338297](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221216104338297.png)

到此镜像就已经推送到中央仓库了，可以在自己账号下进行查看：

![image-20221216105321793](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221216105321793.png)
