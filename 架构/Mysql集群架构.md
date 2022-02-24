# Mysql集群架构

转载请声明作者和出处！

本文如有错误，欢迎指正，感激不尽。

# 一、集群架构设计

## 1.1 设计理念

在集群架构设计时，主要遵从下面三个维度：

- 可用性
- 扩展性
- 一致性

## 1.2 可用性设计

- 站点高可用，冗余站点
- 服务高可用，冗余服务
- 数据高可用，冗余数据

**保证高可用的方法是冗余**。但是数据冗余带来的问题是数据一致性问题

实现高可用的方案有以下几种架构模式：

- 主从
  - 简单灵活，能满足多种需求。比较主流的用法，但是写操作高可用需要自行处理
- 主主
  - 互为主从，有双主双写、双主单写两种方式，建议使用双主单写

## 1.3 扩展性设计

扩展性主要围绕着读操作扩展和写操作扩展展开

- 扩展以提高读性能
  - 加从库
    - 简单易操作，方案成熟
    - 从库过多会引发主库性能损耗。建议不要作为长期的扩充方案，应该设法用良好的设计避免持续加从库来缓解读性能问题
  - 分库分表
    - 可以分为垂直拆分和水平拆分，垂直拆分可以缓解部分压力，水平拆分理论上可以无限扩展
- 如何扩展以提高写性能
  - 分库分表

## 1.4 一致性设计

一致性主要考虑集群中各数据库数据同步以及同步延迟问题，可以使用如下方法：

- 不使用从库
  - 扩展读性能问题需要单独考虑，否则容易出现系统瓶颈
- 增加访问路由层
  - 可以先得到主从同步最长时间t，在数据发生修改后的t时间内，先访问主库

# 二、主从模式

## 2.1、简介和准备

### 2.1.1 概念

mysql主从复制，主要是将主数据库的增删改查操作记录到二进制日志文件中，从库接收主库日志文件，根据最后一次更新的起始位置，同步复制到从数据库中，使得主从数据库保持一致。

MySQL可以从一个MySQL数据库服务器主节点复制到一个或多个从节点。MySQL 默认采用异步复制方式，这样从节点不用一直访问主服务器来更新自己的数据，从节点可以复制主数据库中的所有数据库，或者特定的数据库，或者特定的表

### 2.1.2 作用

- 高可用性：主数据库异常可切换到从数据库
- 负载均衡：实现读写分离
- 备份：日常备份

### 2.1.3 过程

![mysql主从复制](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/mysql%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6.png)

如上图，其中的Binary Log 是日志文件，主库将数据库的变更操作记录到Binlog日志文件

从库读取主库的Binlog日志文件信息写入到从库的Relay Log中继日志中

从库读取中继日志信息在从库中进行Replay,更新从库数据信息

## 2.2 实践

准备两个数据库，在`conf`文件夹下创建`my.cnf`配置文件

主库从库配置相同，但server-id不同

~~~mysql
[mysqld]
#[必须]启用二进制日志
log-bin=mysql-bin
#[必须]服务器唯一ID，默认是取IP最后一段
server-id=10
#binlog-do-db = xxxname 要同步的数据库名
#binlog-ignore-db = mysql 不同步mysql库和test库
#binlog-ignore-db = test
~~~

主库执行命令

~~~mysql
#创建同步账户并授权
create user 'copyUser'@'%' IDENTIFIED by 'copyUser';
GRANT REPLICATION SLAVE ON *.* TO 'copyUser'@'%';
flush PRIVILEGES;
#查看master状态
show master status
#查看二进制相关的配置项
show global VARIABLES like 'binlog%'
#查看server相关的配置项
show GLOBAL VARIABLES like 'server%'
#如果出现Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
ALTER USER 'copyUser'@'%' IDENTIFIED WITH mysql_native_password BY 'copyUser';
flush PRIVILEGES;
~~~

![image-20211029171220902](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211029171220902.png)

从库执行命令

~~~mysql
#设置master相关信息
CHANGE MASTER TO
master_host='ip地址',
master_user='copyUser',
master_password='copyUser',
master_port=3306,
master_log_file='mysql-bin.000001',
master_log_pos=2145;
#启动同步
start SLAVE;
#查看master状态
show slave status;
#stop SLAVE;可以使用此命令停止主从复制
~~~

![image-20211030105932129](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211030105932129.png)

测试是否成功：

主库创建数据库、表并插入数据

![image-20211030111312095](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211030111312095.png)

在从库中进行查看

![image-20211030111525074](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211030111525074.png)

## 2.3 取消主从复制

### 2.3.1 slave流程

停止slave    `mysql> stop slave`

清除slave信息

~~~
mysql>reset slave all;

# 可以通过以下命令查看当前状态
mysql> show slave status\G

Emptyset (0,00 sec)
~~~

### 2.3.2 master流程

清除master上主从信息    `mysql> reset master;`

如果想彻底清除主从的机制，可以修改配置文件，删除主从相关的配置项，然后重启mysql即可。

## 2.4 主从复制的问题与解决

mysql主从复制存在的问题：

- 主库宕机后，数据可能丢失
- 从库只有一个SQL Thread，主库写压力大，复制很可能延时

解决方法：

- 半同步复制---解决数据丢失的问题
- 并行复制----解决从库复制延迟的问题



