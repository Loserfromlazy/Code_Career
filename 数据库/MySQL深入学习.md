# MySql深入学习

> 参考文档：Mysql技术内幕InnoDB存储引擎第二版及部分互联网资源

## 一、Mysql体系结构和存储引擎

### 1.1 MySQL体系结构

我们先明白两个概念，数据库和实例。数据库是物理上的操作系统文件或其他文件的集合。在MySQL中数据库文件可以是frm等格式结尾的文件。而实例在MySQL中是与数据库一对一的。  

MySQL是一个单进程多线程架构的数据库，MySQL实例在系统上就是一个进程。当启动MySQL实例时，MySQL会读取配置文件，根据配置文件的参数来启动数据库实例，如果没有配置文件会按照默认参数来启动实例，

> 可以使用mysql --help |grep my.cnf 来查看当前MySQL数据库实例启动时在哪些位置查找配置文件

MySQL的体系结构如下：

![image-20220726171551519](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220726171551519.png)

由上图可知MySQL主要有以下组件组成：

- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓冲组件
- 插件式存储引擎
- 物理文件

MySQL数据库最重要的特点就是插件式的表存储引擎，MySQL插件式的存储引擎提供了一系列标准的管理和服务支持。**存储引擎是基于表的，而不是数据库。**

MySQL存储引擎有很多，现在最主流的就是InnoDB引擎，本文也将主要对此引擎进行学习。

### 1.2 连接MySQL

连接MySQL本质上是一个连接进程和数据库实例进行通讯，本质上是进程通信。

1. TCP/IP通信

   这是任何平台都提供的方式，也是用的最多的方式，一般二者在两台服务器上，通过TCP/IP进行网络连接，比如：

   ![image-20221011132031331](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011132031331.png)

   在通过TCP/IP连接MySQL实例时，MySQL会先检索mysql数据库下的user表，来判断当前客户端ip是否允许连接到mysql实例（注意mysql8 密码不再是password字段）：

   ![image-20221011132628783](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011132628783.png)

   除此之外，这个表还给出了该用户的操作权限等。

2. 命名管道通信

   使用命名管道方式连接 MySQL 只适合在部分 Windows 系统下用来连接本机的 MySQL ，性能可比一般的TCP/IP方式提升30%~50%。MySQL默认不启用命名管道连接方式，需要在配置文件中启用`–enable-named-pipe`选项。在MySQL4.1之后的版本中，MySQL还提供了共享内存的连接方式，需要在配置文件中添加`–shared-memory`，在连接时，MySQL客户端还必须使用`–protocol=memory`。

3. UNIX域套接字

   同理，在linux和unix环境下，如果客户端和服务端在同一台服务器上，用户可以在配置文件中指定套接字文件的路径，如`–socket=/tmp/mysql.sock`。当数据库实例启动后，用户可以通过下列命令进行UNIX域套接字文件的查找：`mysql> SHOW VARIABLES LIKE 'socket';`在知道了UNIX域套接字文件的路径后，就可以使用该方式进行连接了：`mysql -u [用户名] -S [socket文件路径]`

### 1.3 InnoDB概述

> 以下内容来自百度百科

InnoDB，是MySQL的数据库引擎之一，现为MySQL的默认存储引擎，为[MySQL AB](https://baike.baidu.com/item/MySQL AB?fromModule=lemma_inlink)发布binary的标准之一。InnoDB由Innobase Oy公司所开发，2006年五月时由[甲骨文公司](https://baike.baidu.com/item/甲骨文公司/430115?fromModule=lemma_inlink)并购。与传统的[ISAM](https://baike.baidu.com/item/ISAM?fromModule=lemma_inlink)与[MyISAM](https://baike.baidu.com/item/MyISAM?fromModule=lemma_inlink)相比，InnoDB的最大特色就是支持了[ACID](https://baike.baidu.com/item/ACID/10738?fromModule=lemma_inlink)兼容的[事务](https://baike.baidu.com/item/事务/5945882?fromModule=lemma_inlink)（Transaction）功能，类似于[PostgreSQL](https://baike.baidu.com/item/PostgreSQL?fromModule=lemma_inlink)。

事务型数据库的首选引擎，支持ACID事务，支持行级锁定。InnoDB是为处理巨大数据量时的最大性能设计。InnoDB存储引擎完全与MySQL服务器整合，InnoDB存储引擎为在主内存中缓存数据和索引而维持它自己的缓冲池。InnoDB存储它的表&索引在一个表空间中，表空间可以包含数个文件（或原始磁盘分区）。这与MyISAM表不同，比如在MyISAM表中每个表被存在分离的文件中。InnoDB 表可以是任何尺寸，即使在文件尺寸被限制为2GB的操作系统上。InnoDB默认地被包含在MySQL二进制分发中。Windows Essentials installer使InnoDB成为Windows上MySQL的默认表。

InnoDB 给 MySQL 提供了具有[事务](https://baike.baidu.com/item/事务?fromModule=lemma_inlink)(transaction)、[回滚](https://baike.baidu.com/item/回滚?fromModule=lemma_inlink)(rollback)和崩溃修复能力(crash recovery capabilities)、多版本[并发控制](https://baike.baidu.com/item/并发控制?fromModule=lemma_inlink)(multi-versioned concurrency control)的事务安全(transaction-safe (ACID compliant))型表。InnoDB 提供了行级锁(locking on row level)，提供与 Oracle 类似的不加锁读取(non-locking read in SELECTs)。InnoDB锁定在行级并且也在[SELECT语句](https://baike.baidu.com/item/SELECT语句/15652895?fromModule=lemma_inlink)提供一个Oracle风格一致的非锁定读。这些特色增加了多用户部署和性能。没有在InnoDB中扩大锁定的需要，因为在InnoDB中行级锁定适合非常小的空间。InnoDB也支持FOREIGN KEY强制。在SQL查询中，你可以自由地将InnoDB类型的表与其它MySQL的表的类型[混合](https://baike.baidu.com/item/混合?fromModule=lemma_inlink)起来，甚至在同一个查询中也可以混合。这些特性均提高了多用户并发操作的性能表现。在InnoDB表中不需要扩大锁定(lock escalation)，因为 InnoDB 的行级锁定(row level locks)适宜非常小的空间。InnoDB 是 MySQL 上第一个提供[外键](https://baike.baidu.com/item/外键?fromModule=lemma_inlink)约束(FOREIGN KEY constraints)的表引擎。

### 1.4 InnoDB体系概述

InnoDb存储引擎有多个内存块，这些内存块组成了一个大的内存池，负责以下工作：

- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据，方便快速读取，同时在对磁盘文件的数据修改之前在这里进行缓存
- 重做日志缓冲等

架构图如下（图片来自Mysql技术内幕InnoDB存储引擎）：

![image-20221011133753889](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011133753889.png)

**后台线程**

后台线程主要作用是负责刷新内存池中的数据，保证内存缓存是最近的数据，并将已修改的数据文件刷新到磁盘文件中，同时保证数据库发生异常时能恢复到正常情况。在InnoDB中不同线程负责不同的工作：

1. Master Thread

   这是一个核心线程，负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲、UNDO页的回收等。

2. IO Thread

   InnoDB中大量使用了AIO处理写请求，这样可以极大提高数据库性能。IO Thread的工作主要是负责IO请求的回调。一共唷四种，分别是write、read、insert buffer和log io thread。linux平台的IO线程数量不能调整。我们可以通过`show engine innodb status/G;`来观察IO Thread，如下图：

   ![image-20221011135708833](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011135708833.png)

3. 事务被提交之后，其使用的undolog可能不再需要，因此PurgeThread就会来回收已经使用并分配的undo页。在InnoDB1.1之后，purge可以独立到单独的线程中进行，以提高CPU使用率并提升存储引擎的性能。

   ![image-20221011140217518](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011140217518.png)

4. Page Cleaner Thread是在InnoDB1.2.x中引进的，其作用是将脏页刷新操作放到单独的线程中完成，目的是为了减轻Master Thread的工作和对用户查询线程的阻塞，提高性能。

**内存**

InnoDB内存数据对象如下：

![image-20221011141403838](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011141403838.png)

1. 缓冲池

   InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。但是由于CPU速度和磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池记录来提高数据库的的整体性能。缓冲池就是一块内存区域，在读取页的操作时，先将从磁盘中读取到的页存放在缓冲池中（也叫做将页FIX在缓冲池中），下一次读取时先判断该页是否在缓冲池中，如果在缓冲池中被命中，直接读取该页；否则读取磁盘上的页。在修改页内数据时，先修改缓冲池的页，然后以一定的频率刷新到磁盘上。这里刷新回去是通过checkpoint技术实现的。缓冲池可以通过`innodb_buffer_pool_size`进行设置。下图查看缓冲池大小。

   ![image-20221011141143654](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011141143654.png)

   可以通过命令`show engine innodb status/G;`查看缓冲池状态。

   ![image-20221011144233136](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011144233136.png)

   在配置文件中可以通过innodb_buffer_pool_instances设置大于1就可以得到多个缓冲池实例。在mysql5.6以后可以使用如下方式查看缓冲池的使用状态：

   ![image-20221011150046800](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221011150046800.png)

2. LRU List、FreeList和Flush List

   一般来说数据库缓冲池是通过LRU（最近最少使用算法）来进行管理的。最频繁使用的页在LRU的前端，最少使用的页在LRU的尾端。当缓冲池不能存放新读取的页时，应先释放LRU尾部的页。在InnoDB中，缓冲池页大小默认为16KB，且InnoDB对传统LRU算法进行了优化，加入了midpoint位置。

3. 重做日志缓冲

4. 额外的内存池

### 1.5 Checkpoint技术

### 1.6 Master Thread工作方式

### 1.7 InnoDB关键特性



