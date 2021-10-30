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

## 三、取消主从复制

### 3.1 slave流程

停止slave    `mysql> stop slave`

清除slave信息

~~~
mysql>reset slave all;

# 可以通过以下命令查看当前状态
mysql> show slave status\G

Emptyset (0,00 sec)
~~~

### 3.2 master流程

清除master上主从信息    `mysql> reset master;`

如果想彻底清除主从的机制，可以修改配置文件，删除主从相关的配置项，然后重启mysql即可。
