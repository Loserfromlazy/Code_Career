# Mysql主从复制

## 一、简介和准备

### 1.1 概念

mysql主从复制，主要是将主数据库的增删改查操作记录到二进制日志文件中，从库接收主库日志文件，根据最后一次更新的起始位置，同步复制到从数据库中，使得主从数据库保持一致。

### 1.2 作用

- 高可用性：主数据库异常可切换到从数据库
- 负载均衡：实现读写分离
- 备份：日常备份

### 1.3 过程

![mysql主从复制](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/mysql%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6.png)

如上图，其中的Binary Log 是

## 二、实践

主库从库配置相同，但server-id不同

~~~
[mysqld]
#[必须]启用二进制日志
log-bin=mysql-bin
#[必须]服务器唯一ID，默认是取IP最后一段
server-id=10
~~~

主库

~~~mysql
#创建同步账户并授权
create user 'copyUser'@'%' IDENTIFIED by 'copyUser';
grant replication slave on *.* to 'copyUser'@'%';
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

从库

~~~mysql
#设置master相关信息
CHANGE MASTER TO
master_host='ip地址',
master_user='cpoyUser',
master_password='cpoyUser',
master_port=3306,
master_log_file='mysql-bin.000001',
master_log_pos=2145;
#启动同步
start SLAVE;
#查看master状态
show slave status;
~~~

很长，这里只截一部分

![image-20211029171332488](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211029171332488.png)

