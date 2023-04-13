# MySql——InnoDB深入学习

> 参考文档：Mysql技术内幕InnoDB存储引擎第二版、高性能Mysql第三版及部分互联网资源

# 一、Mysql体系结构和存储引擎

## 1.1 MySQL体系结构

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

## 1.2 连接MySQL

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

## 1.3 InnoDB概述

> 以下内容来自百度百科

InnoDB，是MySQL的数据库引擎之一，现为MySQL的默认存储引擎，为[MySQL AB](https://baike.baidu.com/item/MySQL AB?fromModule=lemma_inlink)发布binary的标准之一。InnoDB由Innobase Oy公司所开发，2006年五月时由[甲骨文公司](https://baike.baidu.com/item/甲骨文公司/430115?fromModule=lemma_inlink)并购。与传统的[ISAM](https://baike.baidu.com/item/ISAM?fromModule=lemma_inlink)与[MyISAM](https://baike.baidu.com/item/MyISAM?fromModule=lemma_inlink)相比，InnoDB的最大特色就是支持了[ACID](https://baike.baidu.com/item/ACID/10738?fromModule=lemma_inlink)兼容的[事务](https://baike.baidu.com/item/事务/5945882?fromModule=lemma_inlink)（Transaction）功能，类似于[PostgreSQL](https://baike.baidu.com/item/PostgreSQL?fromModule=lemma_inlink)。

事务型数据库的首选引擎，支持ACID事务，支持行级锁定。InnoDB是为处理巨大数据量时的最大性能设计。InnoDB存储引擎完全与MySQL服务器整合，InnoDB存储引擎为在主内存中缓存数据和索引而维持它自己的缓冲池。InnoDB存储它的表&索引在一个表空间中，表空间可以包含数个文件（或原始磁盘分区）。这与MyISAM表不同，比如在MyISAM表中每个表被存在分离的文件中。InnoDB 表可以是任何尺寸，即使在文件尺寸被限制为2GB的操作系统上。InnoDB默认地被包含在MySQL二进制分发中。Windows Essentials installer使InnoDB成为Windows上MySQL的默认表。

InnoDB 给 MySQL 提供了具有[事务](https://baike.baidu.com/item/事务?fromModule=lemma_inlink)(transaction)、[回滚](https://baike.baidu.com/item/回滚?fromModule=lemma_inlink)(rollback)和崩溃修复能力(crash recovery capabilities)、多版本[并发控制](https://baike.baidu.com/item/并发控制?fromModule=lemma_inlink)(multi-versioned concurrency control)的事务安全(transaction-safe (ACID compliant))型表。InnoDB 提供了行级锁(locking on row level)，提供与 Oracle 类似的不加锁读取(non-locking read in SELECTs)。InnoDB锁定在行级并且也在[SELECT语句](https://baike.baidu.com/item/SELECT语句/15652895?fromModule=lemma_inlink)提供一个Oracle风格一致的非锁定读。这些特色增加了多用户部署和性能。没有在InnoDB中扩大锁定的需要，因为在InnoDB中行级锁定适合非常小的空间。InnoDB也支持FOREIGN KEY强制。在SQL查询中，你可以自由地将InnoDB类型的表与其它MySQL的表的类型[混合](https://baike.baidu.com/item/混合?fromModule=lemma_inlink)起来，甚至在同一个查询中也可以混合。这些特性均提高了多用户并发操作的性能表现。在InnoDB表中不需要扩大锁定(lock escalation)，因为 InnoDB 的行级锁定(row level locks)适宜非常小的空间。InnoDB 是 MySQL 上第一个提供[外键](https://baike.baidu.com/item/外键?fromModule=lemma_inlink)约束(FOREIGN KEY constraints)的表引擎。

## 1.4 InnoDB体系

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

   InnoDB中大量使用了AIO处理写请求，这样可以极大提高数据库性能。IO Thread的工作主要是负责IO请求的回调。一共有四种，分别是write、read、insert buffer和log io thread。linux平台的IO线程数量不能调整。我们可以通过`show engine innodb status/G;`来观察IO Thread，如下图：

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

   一般来说数据库缓冲池是通过LRU（最近最少使用算法）来进行管理的。最频繁使用的页在LRU的前端，最少使用的页在LRU的尾端。当缓冲池不能存放新读取的页时，应先释放LRU尾部的页。在InnoDB中，缓冲池页大小默认为16KB，且InnoDB对传统LRU算法进行了优化，加入了midpoint位置。即新读取的页，并不是放入LRU列表的首部，而是在midpoint位置。默认配置下，该位置在LRU列表长度的5/8处，midpoint由参数`innodb_old_blocks_pct`控制，如下：

   ![image-20221014135127892](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221014135127892.png)

   该值默认为37，也就是在LRU列表尾端的37%左右的位置，在InnoDB中，在midpoint之后的列表是old列表，在midpoint之前的列表是new列表，可以简单理解为活跃数据端。

   MySQL这么改进的原因主要是想解决缓存污染的问题。那么什么是缓存污染的问题呢？比如我有一条sql`select * from tb_XXX where name like"%张%"`，这条sql会扫描了大量的数据时，在缓冲池空间比较有限的情况下，可能会将缓冲池里的所有页都替换出去，导致大量热数据被淘汰了，等这些热数据又被再次访问的时候，由于缓存未命中，就会产生大量的磁盘 I/O，MySQL 性能就会急剧下降。而这些不常用的数据又占据着当前的缓冲池，造成缓存污染。

   而MySQL为了解决此问题又引入了另一个参数进一步管理LRU列表`innodb_old_blocks_time`，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。如果第二次的访问时间与第一次访问的时间在 1 秒内（默认值），那么该页就不会被从 old 区域升级到 new区域；如果第二次的访问时间与第一次访问的时间超过 1 秒，那么该页就会从 old 区域升级到 young 区域。这是为了防止某些不常用的页（如全表扫描的页）在短时间（如1秒内）内被多次访问，让系统误以为它是热数据从而将其放入了热端区域。

   LRU列表用来管理已经读取的页，但数据库刚启动时，LRU列表是空的，这时页都存放在Free列表中，当需要缓冲池分页时，首先从Free表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，当如LRU列表。否则淘汰LRU末尾的页。当页从old加入到new部分时，称为page made young，反之因为innodb_old_blocks_time导致没有移动到new 被称为page not made young，可以通过`SHOW ENGINE INNODB STATUS来观察`：

   ![image-20221014154035205](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221014154035205.png)

   > 这里还有个变量 Buffer poll hit rate 表示缓冲池命中率，一般该值不低于95%

   在LRU中的页没修改后，该页为脏页，这时数据库就会通过CHECKPOINT机制将数据刷新回磁盘。Flush列表中的页就是脏页，但是脏页即存在LRU列表又存在于Flush列表。

   InnoDB从1.0.x版本开始支持压缩页，即将16K的页压缩为1、2、4、8KB，对于非16K的页，是通过unzip_LRU管理的。

3. 重做日志缓冲

   InnoDB除了有缓冲池外，还有 重做日志缓冲(redo log buffer) 。lnnoDB首先将重做日志信息先放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件。一般默认为8M，可通过`innodb_log_buffer_size控制`：

   ![image-20221014155227355](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221014155227355.png)

   重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中

   - Master Thread 每一秒将重做日志缓冲刷新到重做日志文件；
   - 每个事务提交时 会将重做日志缓冲刷新到重做日志文件；
   - 当重做日志缓冲池剩余空间小于1/2 时， 重做日志缓冲刷新到重做日志文件。

4. 额外的内存池

   在InnoDB 存储引擎中，对内存的管理是通过一种称为 内存堆（heap）的方式 进行的。在对一些数据结构本身的内存进行分配时， 需要从额外的内存池中进行申请，当该区域的内存不够时，会从缓冲池中进行申请。例如，分配了缓冲池(innodb_buffer _pool), 但是每个缓冲池中的帧缓冲(frame buffer) 还有对应的缓冲控制对象(buffer control block), 这些对象记录了一些诸如LRU 、锁、等待等信息，而这个对象的内存需要从额外内存池中申请。因此，在申请了很大的InnoDB 缓冲池时，也应考虑相应地增加这个值。

## 1.5 Checkpoint技术

上面我们知道，页的操作首先会在缓冲池中完成，当sql语句修改了某一页的数据，那么此时页就是脏的，缓冲池就需要将数据刷新回磁盘。但是如果每次变化就刷新磁盘，那么IO开销是很大的，而且如果发生宕机，那么数据就会丢失，因此为了避免这些问题，事务数据库普遍采用了Write Ahead Log策略，即先写重做日志，在修改页，这样就可以通过重做日志来恢复数据。但重做日志不能无限增大、缓冲池也不能无限增大，因此出现了Checkpoint技术。检查点（Checkpoint）技术目的是可以解决以下问题：

- 缩短数据库恢复时间
- 缓冲池不够时，将脏页刷新回磁盘
- 重做日志不可用时，刷新脏页

有了检查点后，数据库宕机时就不需要恢复所有的重做日志，只需要恢复检查点后的重做日志即可。缓冲池不够时需要将尾部的页清除，若此时为脏页，就需要强制执行Checkpoint，将脏页刷回磁盘。而重做日志不可用是指重做日志一般都是循环使用，其可以被重用的部分是指这部分日志已经不再需要，可以被覆盖使用。因此需要强制产生Checkpoint，将缓冲池中的页刷新到当前重做日志的位置。

在InnoDB中，通过LSN来标记版本。LSN是8字节。在每个页、重做日志中都有LSN。

可以通过`show engine innodb status\G`命令查看

![image-20221018130741464](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221018130741464.png)

在innodb中，checkpoint发生的时间、条件和脏页的选择都非常复杂，而checkpoint所做的事无外乎是将缓冲池中的脏页刷回磁盘，不同之处在于每次刷多少页，从哪刷，什么时间刷新。在innodb中一般有两种checkpoint，分别是：

- Sharp Checkpoint
- Fuzzy Checkpoint

发生在数据库关闭时，将所有的脏页刷新回磁盘，这是默认的方式，参数为`innodb_fast_shutdown=1`。但如果运行时使用，则数据库可用性就会受到影响，因此在innodb中使用了fuzzy checkpoint，只刷新一部分脏页。一般可能会发生以下几种情况的Fuzzy Checkpoint：

- Master Thread Checkpoint：在Master Thread中发生的Checkpoint，以每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘，此过程是异步的。

- FLUSH_LRU_LIST Checkpoint：因为InnoDB要保证LRU列表中有差不多100个空闲页可使用。在1.1.x版本之前，需要检查LRU列表中是否有足够的可用空间，这个操作发生在用户查询线程，会阻塞用户查询操作，如果没有100个可用空闲页，Innodb就会将LRU列表尾端的页移除，如果这些需要被移除的页有脏页就需要checkpoint，因此被称为FLUSH_LRU_LIST Checkpoint。

  在MySQL5.6也就是InnoDB1.2.x版本开始后，这些检查放在了单独的Page Cleaner线程中执行，我们可以通过`innodb_lru_scan_depth`控制LRU列表中可用页的数量。

- Async/Sync Flush Checkpoint：指重做日志不可用的情况下，需要强制将一些页刷回磁盘，此时脏页是从脏页列表中选取的。

  这里我们将写入重做日志的LSN记为redo_lsn，将已经刷新回磁盘的LSN记为checkpoint_lsn，然后定义：

  checkpoint_age= redo_lsn-checkpoint_lsn

  然后定义：

  async_water_mark = 75% * total_redo_log_file_size

  sync_water_mark = 90% * total_redo_log_file_size

  如果每个重做日志文件大小为1G，且定义了两个，那么重做日志的总大小为2G，因此async_water_mark =1.5G，sync_water_mark =1.8G，因此当：

  - checkpoint_age<async_water_mark 时，不刷新任何脏页到磁盘
  - async_water_mark<checkpoint_age<sync_water_mark时，触发Async Flush从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark
  - checkpoint_age>sync_water_mark这种情况很少发生，除非重做日志台校，并且在进行类似Load Data的Bulk Insert操作。此时触发Sync Flush，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark

  因此Async/Sync Flush Checkpoint是为了保证重做日志的可用性，在MySQL5.6也就是InnoDB1.2.x版本开始后，这些操作放在了单独的Page Cleaner线程中执行，不会阻塞用户查询线程。

- Dirty Page too much Checkpoint：脏页数量太多，导致InnoDB存储引擎强制执行Checkpoint，目的总的来说还是为了缓冲池中有足够可用的页。可以由参数`innodb_max_dirty_pages_pct`控制。此值在1.0.x版本前默认90之后的版本默认75。此值表示当缓冲池中脏页数量占75%的时候强制执行Checkpoint。

# 二、索引与算法

InnoDB支持一下几种常见的索引：

- B+索引
- 全文索引
- 哈希索引

其中InnoDB存储引擎支持的哈希索引是自适应的，其会根据表的使用情况自动生成哈希索引，不能人为干预是否在一张表中生成哈希索引。B+索引是传统意义上的索引，这是目前关系型数据库中查找最为常用和最有效的索引。

索引可以让服务器快速定位到表的指定位置，也有一些其他的作用，比如B树索引，按照顺序存储数据，所以mysql可以用索引来做Order by和group by 。因为数据是有序的，所以B+树会将相关的列存储到一起。总结一下，索引共有三个优点：

- 索引大大减少了服务器需要扫描的数据量
- 索引可以帮助服务器避免排序和临时表
- 索引可以将随机IO变为顺序IO

Lahdenmaki在他的书中介绍了索引的三星系统，即索引将相关的记录放在一起就获得一星；如果索引中的数据顺序与查询中的排序顺序一致则获得二星；如果索引中的列包含了查询中需要的全部列则获得三星。

## 2.1 基础数据结构和算法

### 2.1.1 二分查找法

二分查找法(binary search)，用来查找有序数组中的某一记录，基本思想是将记录有序化，在查找的过程中先以中点作为比较对象，如果值小于中点，则在序列的左半部分，否则是右半部分，一次查找可以将区间缩小一半。Java实现如下：

```java
/**
 * 二分法查找数据
 */
public int binarySearch(long value){
    int middle=0;
    int low=0;//第一位索引
    int pow=elements;//最后一位索引
    while(true){
        middle=(pow+low)/2;
        if(arr[middle]==value){
            return middle;
        }else if(low >pow){//如果第一位索引大于最后一位的索引，代表搜索结束，没有找到该值
            return -1;
        }else{
            if(arr[middle]>value){
	//如果中间的值比查的值大，说明待查值在前面，所以将最后一位的索引改为中间的索引的前一位
                pow=middle-1;
            }else{
    //如果中间的值比查的值小，说明待查值在后面，所以将第一位的索引改为中间的索引的后一位
                low=middle+1;
            }
        }
    }
}
```

### 2.1.2 二叉查找树和平衡二叉树

二叉排序树（Binary Sort Tree），又称二叉查找树Binary Search Tree），亦称二叉搜索树。是数据结构中的一类。在一般情况下，查询效率比链表结构要高。

二叉排序树是一棵空树，或者是具有下列性质的二叉树

1. 若左子树不空，则左子树上所有结点的值均小于它的根结点的值；
2. 若右子树不空，则右子树上所有结点的值均大于它的根结点的值；
3. 左、右子树也分别为二叉排序树；
4. 没有键值相等的结点。

二叉查找树可以任意的构造，也就是说也可以构造成以下的样子：

![img](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_data/erchaquedian.png)

这样的二叉查找树的效率就很低，因此想最大性能的构建二叉查找树就需要这棵树是平衡的，平衡树是二叉搜索树和堆合并构成的数据结构，它是一 棵空树或它的左右两个子树的高度差的绝对值不超过1。

因此我们需要构建平衡二叉树，它又被称为AVL树，平衡二叉树，再新增删除节点时需要进行左旋或右旋操作，因此对一颗平衡树的维护是有一定的开销的。

## 2.2 B+树

> 这里推荐B+树生成网站[B+ Tree](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

B+树是为磁盘或其他直接存取设备设计的一种平衡查找树。它是B树的扩展，因为B树允许在内部节点和叶子节点都存储键值和数据，这使得B树插入特定节点的能力被降低了。而B+树所有的记录节点都是按照键值的大小顺序存放在同一层的叶子节点上，且各个叶子节点通过指针进行连接，如下图（图片来自MySQL技术内幕）：

![image-20221020142141912](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221020142141912.png)

### 2.2.1 B+树的插入

- 首先执行搜索操作，查找新节点该存放的位置
- 如果叶子节点的数据页未满，则直接插入该节点
- 否则将中间节点放入父节点，然后再插入新节点
- 重复以上步骤，直到父节点达到条件（即父节点未满）

我们举个例子，如下图，假设我们每页最大数量为4，变化如下：

![image-20221020144011829](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221020144011829.png)

### 2.2.2 B+树的删除

- 如果在叶子节点上，则直接删除
- 如果删除的键值在父节点上，则删除后将右兄弟节点补充进父节点中
- 如果删除后的填充因子小于50%，那么就需要进行合并

我们举个例子，如下图，假设我们每页最大数量为4，变化如下：

![image-20221020150300855](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221020150300855.png)

## 2.3 B+树索引

B+树索引本质上就是B+树在数据库中的实现，但是B+树索引在数据库中有一个特点就是高扇出性，因此在数据库中，B+树的高度一般在2-4层，也就是说查找某一键值只需要2-4次IO，当前机械硬盘每秒100次IO，因此意味着查询时间只需要0.02-0.04秒。

数据库中的B+树索引可以分为聚集索引和辅助索引，两种内部都是B+树，即高度平衡的，叶子节点存放着所有的数据，聚集索引和辅助索引不同的是，B+Tree的叶子节点存放主键索引值和行记录就属于聚簇索引；如果索引值和行记录分开存放就属于非聚簇索引。

> InnoDB的表要求必须要有聚簇索引：
>
> - 如果表定义了主键，则主键索引就是聚簇索引
> - 如果表没有定义主键，则第一个非空unique列作为聚簇索引
> - 否则InnoDB会从建一个隐藏的row-id作为聚簇索引

B+树索引通常意味着所有的值都是按照顺序存储的，并且每一个叶子页到根的距离相同，结构如下（图片来自高性能MySQL第三版）：

![image-20221024140801244](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024140801244.png)

B+树的叶子节点比较特别，它们的指针指向的是被索引的数据，而不是其他的节点页。

### 2.3.1 B+树索引的分裂

B+树索引的分裂并不总是从页的中间记录开始的，这样可能会导致空间的浪费，加入我们要插入`1 2 3 4 5 6`插入顺序是自增进行的，且插入第7条数据时就要分裂这时会将4作为分裂点得到以下两个页：

![image-20221024141952195](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024141952195.png)

由于插入是顺序的，因此第一个页将不会再有记录插入，导致空间浪费，而第二个页又会再次分裂：

![image-20221024142033686](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024142033686.png)

这个问题可以在MySQL的[Bug #67718](https://bugs.mysql.com/bug.php?id=67718)中查看关于此Bug的详细描述，以及如何重现此问题。

MySQL是如何优化的呢？

InnoDB的Page Header中有以下几个部分来保存插入的顺序信息：

- **PAGE_LAST_INSERT**：最后插入记录的位置。
- **PAGE_DIRECTION**：记录插入的方向。假如新插入的一条记录的主键值比上一条记录的主键值大，我们说这条记录的插入方向是右边，反之则是左边。用来表示最后一条记录插入方向的状态就是`PAGE_DIRECTION`。
- **PAGE_N_DIRECTION**：假设连续几次插入新记录的方向都是一致的，InnoDBhi把沿着同一个方向插入记录的条数记下来，这个条数就用`PAGE_N_DIRECTION`这个状态表示。当然，如果最后一条记录的插入方向改变了的话，这个状态的值会被清零重新统计。

通过这些信息，InnoDB存储引擎可以决定向左还是向右进行分裂，同时决定将分裂点记录为哪个。若插入是随机的，则取页的中间记录作为分裂点的记录。

> InnoDB引擎插入数据时时，首先需要进行定位，**定位到的记录为待插入记录的前一条记录**。

若往同一方向进行插入的记录条数为`5`，并且目前已经定位到的记录之后还有`3`条记录，则分裂点的记录为定位到的记录后的第三条记录，否则分裂点记录就是待插入的记录。（这里的`5`和`3`是固定的，与具体例子无关）。分别举例子如下（例子来自Mysql技术内幕InnoDB存储引擎第二版）：

1. 定位到的记录之后还有`3`条记录

   ![image-20221024155048696](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024155048696.png)

2. 分裂点就为插入记录本身，向右分裂后仅插入记录本身

   ![image-20221024155627879](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024155627879.png)

我们用现行的mysql版本（我的版本是mysql8）重复官方的bug复现步骤（[Bug #67718](https://bugs.mysql.com/bug.php?id=67718)），结果如下：

![image-20221024160033037](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024160033037.png)

可以发现最后插入的三条记录并没有单独分裂，而是在一个页中。当然这样做虽然避免了空间浪费，但是增加了索引插入时的性能。如果在最极端的情况下，新增的数据定位到了第一个叶子结点，插入后，叶子结点达到最大数量，然后分裂出最后的一个元素插入到后一个叶子结点。假设后面的叶子结点都是已经满额的状态，那么这个插入会导致所有的叶子结点都发生一次分裂，且所有父结点的数据都要重新调整。这样的效率简直是灾难性的。**当然这个问题也人为的进行避免，那就是使用自增索引。因为自增索引永远都只会修改和新增最右的叶子结点，而不用修改原有叶子结点。**

### 2.3.2 Cardinality值

如果想查看一个表的索引可以通过 `show index from table_name;`查看：

![image-20221024165704644](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221024165704644.png)

其中各列的意义如下：

- Table：索引所在的表名
- Non_unique：非唯一的索引，0表示必须唯一。
- Key_name：索引的名字
- Seq_in_index：索引中该列的位置
- Column_name：索引列的名称
- Collation：列以什么方式存储在索引中，值可以是A或NULL。B+树索引总是A，即排序的。如果是Heap存储引擎且建立Hash索引，这里就是NULL；
- Cardinality：表示索引中唯一值的数据的估计值
- Sub_part：是否是列的部分被索引。比如只对某列的前100个字符进行索引
- Packed：关键字如何被压缩
- Null：是否索引的列含有NULL值。
- Index_type：索引的类型》InnoDB只支持B+树索引
- Comment注释

对于什么时候添加B+树索引，一般来说在低选择性的字段上加比较有意义。比如对于性别字段它的取值范围很小，这就是低选择性。如何查看索引的高选择性呢？可以通过Cardinality值，它表示索引中不重复记录数量的预估值。在实际使用时Cardnality应该尽可能接近1，如果非常小，用户应考虑是否有必要创建这个索引。

Cardnality的统计是放在存储引擎层进行的。在生产环境中，索引的更新操作是非常频繁的，所以数据库对于Cardnality的统计是通过采样来完成的。在InnoDB中，Cardnality的统计发生在INSERT和UPDATE中，InnoDB的更新Cardnality的策略是：

- 表中1/16的数据发生了变化

- `stat_modified_counter`>2000000000 

  在InnoDB中有一个计数器`stat_modified_counter`用来表示发生变化的次数，这是因为如果表中某一行数据频繁的进行更新，这时表中的数据并没有增加，所以增加此计数器用来表示发生变化的次数。

在InnoDB中是通过采样的方式进行Cardinality的值进行统计和更新的。默认InnoDb对8个叶子节点进行采样，过程如下：

- 取得B+树索引中的叶子节点的数量，记作A
- 随机取得B+树索引的8个叶子节点，统计每个页不同的记录的个数，记作P1，P2.....P8。
- 根据采样信息给出Cardinality的预估值：`Cardinality=(P1+P2+......+P8)*A /8`

因为8个叶子节点是随机的所以Cardinality的值每次可能是不同的。我们可以使用`SHOW INDEX
 FROM Table_Name`触发MYSQL对Cardinality的值的统计，如下图：

![image-20230413091914344](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413091914344.png)

## 2.4 哈希索引

## 2.5 高性能索引策略

### 2.5.1 使用独立的列

如果查询中的列不是独立的，那么MySQL不会使用索引，独立的列的意思是索引列不能是表达式的一部分，也不能是函数的参数，如下我们有一个`test_user`表，其中有一个`login_error_count`列，这个列上有索引，现在有以下两个sql：

~~~mysql
SELECT id,`name` from test_user where login_error_count + 1 = 4;//不走索引
SELECT id,`name` from test_user where login_error_count = 4;//走索引
~~~

其中`login_error_count + 1 = 4`这个条件mysql无法解析此方程式，这是用户行为所以无法走索引。下面我们用explain分析一下：

![image-20230413100455822](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413100455822.png)

再看一下另一个：

![image-20230413100519191](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413100519191.png)

我们写SQL时应该**习惯简化WHERE条件，应该始终将索引列单独放在比较符号的另一侧**。

下面是另一个常见的错误：

~~~mysql
SELECT ... from test_user where TO_DAYS(CURRENT_DATE)-TO_DAYS(date_col) <=10;
~~~

### 2.5.2 前缀索引和索引选择性

有时候可能会需要索引很长的字符列，这会让索引变得又大又慢，有一种策略是使用2.4中的模拟hash索引，还有一种就是可以使用下面的前缀索引。其实**前缀索引就是索引字符串前面的部分字符，这样可以节约索引空间，提高索引效率，但是这样也会降低索引的选择性**。索引的选择性是指不重复的索引值和数据表的记录总数（#T）的比值，范围从`1/#T`到1之间。索引的选择性越高，擦汗寻的效率就越高，唯一索引的选择性是1，这也是最好的索引选择性。

在创建索引时，需要选择足够长的前缀以保证较高的选择性，但又不能太长，我们下面举个例子：

首先我们用以下语句建立测试数据集：

> 注意sakila库是一个MySQL官方提供的模拟电影出租厅信息管理系统的示例数据库，下载地址：[sakila库](http://downloads.mysql.com/docs/sakila-db.zip)

~~~mysql
create table sakila.city_demo(city VARCHAR(50) not null);

insert into sakila.city_demo(city) SELECT city from sakila.city;
#执行5次
insert into sakila.city_demo(city) SELECT city from sakila.city_demo;

update sakila.city_demo set city = (select city from sakila.city ORDER BY RAND() LIMIT 1)
~~~

然后我们先查询出来最常见的城市列表：

~~~mysql
#查询最常见的city list
select count(*) as cnt,city from city_demo group by city order by cnt desc LIMIT 10;
~~~

![image-20230413104815216](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413104815216.png)

然后我们可以从3开始试，逐渐增加前缀长度，直到前缀的选择性接近上面完整的选择性：

~~~mysql
#查询
select count(*) as cnt,LEFT(city,3) as pref from city_demo group by pref order by cnt desc LIMIT 10;
select count(*) as cnt,LEFT(city,7) as pref from city_demo group by pref order by cnt desc LIMIT 10;
~~~

前缀为3结果如下：

![image-20230413110006720](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413110006720.png)

直到试到前缀为7：

![image-20230413110101905](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413110101905.png)

当然也有另一种方式就是计算选择性，如下：

~~~
#计算完整列的选择性
SELECT count(DISTINCT city)/count(*) from city_demo
#计算不同前缀长度的选择性
SELECT count(DISTINCT left(city,3))/count(*) as select3,
count(DISTINCT left(city,4))/count(*) as select4,
count(DISTINCT left(city,5))/count(*) as select5,
count(DISTINCT left(city,6))/count(*) as select6,
count(DISTINCT left(city,7))/count(*) as select7
from city_demo
~~~

完整的选择性结果：

![image-20230413110210160](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413110210160.png)

不同前缀长度的选择性：

![image-20230413110237586](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413110237586.png)

既然我们找到了合适的前缀的长度，我们可以使用：

~~~mysql
alert table city_demo add key(city(7));
~~~

创建前缀索引。注意前缀索引有缺点，即mysql无法用前缀索引做order by 和 group by，也无法使用前缀索引做覆盖扫描。前缀索引比较常见的场景就是很长的十六进制唯一ID。

### 2.5.3 多列索引

### 2.5.4 合适的索引列顺序

### 2.5.5 聚集索引

### 2.5.6 覆盖索引

### 2.5.7 索引和锁

### 2.5.8 MRR优化

### 2.5.9 ICP优化

# 三、锁

## 3.1 锁概述

锁是数据库系统区别于文件系统的关键特性。锁机制用于管理对共享资源的并发访问。InnoDB会在行级别上对表数据上锁，同时也会在数据库内部其他多个地方使用锁，从而允许对多种不同资源提供并发访问，例如操作缓冲池中的LRU列表。尽管数据库系统大都类似，但是不同数据库的锁实现大多是不同的。在SQL语法方面，由于SQL标准所以大差不差，但是对于锁在Oracle、sql server、innoDb中实现都是完全不同。其中InnoDB与Oracle数据库类似，提供了一致性的非锁定读、行级锁的支持。行级锁没有额外的开销，且可以得到并发性和一致性。

在学习锁之前还应该区别数据库的lock和latch的概念，在数据库中这俩都可以被称为锁，但是含义是不同的，这里我们主要关注lock。

- latch一般称为闩锁（轻量级锁），其要求锁定时间非常短，若持续的时间长，则应用的性能会非常差。在InnoDB 存储引擎中，latch 又可以分为 mutex(互斥量)和 rwlock(读写锁)。其目的是用来保证并发线操作临界资源的正确性，并且通常没有死锁检测的机制。
- lock 的对象是事务，用来锁定的是数据库中的对象，如表、页、行。并且一般 lock的对象仅在事务 commit 或 rollback 后进行释放(不同事务隔离级别释放的时间可能不同)。与大多数数据库中一样，lock是有死锁机制的。

下表显示了 lock 与latch 的不同：

|          | lock                                                  | latch                                                        |
| -------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| 对象     | 事务                                                  | 线程                                                         |
| 保护     | 数据库内容                                            | 内存数据结构                                                 |
| 持续时间 | 整个事务过程                                          | 临界资源                                                     |
| 模式     | 行级锁、表锁、意向锁                                  | 读写锁、互斥锁                                               |
| 死锁     | 通过waits-for graph、time out等机制进行死锁检测与处理 | 无死锁检测与处理机制。仅通过应用程序加锁的顺序保证无死锁的情况发生 |
| 存在于   | Lock Manager的哈希表中                                | 每个数据结构的对象中                                         |

对于InnoDB中的latch，可以通过`SHOW ENGINE INNODB MUTEX`查看，数据如下：

~~~
Type	Name	Status
InnoDB	rwlock: fil0fil.cc:3013	waits=1
InnoDB	sum rwlock: buf0buf.cc:781	waits=20
~~~

> 其中列type总是InnoDB；列name是latch的信息及其源码位置；列status比较复杂可能会包含os_waits（操作系统的等待的次数）等信息。

对于InnoDB中的lock，可以通过`SHOW ENGINE INNODB STATUS`和information_schema下的表INNODB_TRX、INNODB_LOCKS、INNODB_LOCKS_WAITS查看锁的信息。

## 3.2 InnoDB中的Lock

### 3.2.1 锁的类型

InnoDB实现了下面两种标准的行级锁：

- 共享锁 S Lock，允许事务读一行数据
- 排他锁 X Lock，允许事务删除或更新一行数据

如果一个事务（记作T1）已经获取了行r的共享锁，那么另外的事务 T2 可以立即获得行r的共享锁，因为读取并没有改变行r 的数据，这种情况称为锁兼容 (Lock Compatible)。但若有其他的事务 T3 获得行r的排他锁，则其必须待事务 T1、T2释放行r的共享锁，这种情况称为锁不兼容。俩锁的兼容如下表：

|      | X      | S      |
| ---- | ------ | ------ |
| X    | 不兼容 | 不兼容 |
| S    | 不兼容 | 兼容   |

X锁与任何锁都不兼容，S锁只与S锁兼容。此外InnoDB支持多粒度锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在，为了支持在不同粒度上进行加锁操作，InmDB 存储引擎支持一种额外的锁方式，称之为意向锁 (Intention Lock)。意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度(fine granularity)上进行加锁，如下图（参考Mysql技术内幕InnoDB存储引擎第二版绘制）：

![image-20230413144816179](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413144816179.png)

若将上锁的对象当作一个树，对最下层的对象（最细粒度）上锁需要先对粗粒度的对象上锁，比如上图中对记录r上X锁,那么需要分别对数据库A、表、页上意向锁IX，最后对记录上X锁。如果其中任一部分导致等待，那么该操作需要粗粒度锁的完成。eg：对记录r加X锁之前，已经有事务加了S表锁，也就是表1已经有S锁。这时事务需要对记录r在表1上加IX锁，由于不兼容，需要等待S表锁的操作完成。

InnoDB的意向锁就是表级别的锁。设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型，共有两种意向锁：

- 意向共享锁 IS Lock，事务想获得一张表中某几行的共享锁
- 意向排他锁 IX Lock，事务想获得一张表中某几行的排他锁

由于ImnnoDB存储引擎支持的是行级别的锁，因此意向锁其实不会阻塞除全表扫以外的任何请求。故表级意向锁与行级锁的兼容性如下：

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | 兼容   | 兼容   | 兼容   | 不兼容 |
| IX   | 兼容   | 兼容   | 不兼容 | 不兼容 |
| S    | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

在InnoDB 1.0版本前，用户只能通过`SHOW ENGINE INNODB STATUS`，`SHOW FULL PROCESSLIST`等查看当前数据库的锁的请求，然后再判断事务所的情况。从1.0开始，在information_schema下添加了表INNODB_TRX、INNODB_LOCKS、INNODB_LOCKS_WAITS。通过这三张表可以简单监控当前事务并分析可能存在的锁问题。表INNODB_TRX的定义如下：

~~~mysql
CREATE TEMPORARY TABLE `INNODB_TRX` (
  `trx_id` varchar(18) NOT NULL DEFAULT '' COMMENT 'InnoDB存储引擎内部的唯一事务ID',
  `trx_state` varchar(13) NOT NULL DEFAULT '' COMMENT '当前事务的状态',
  `trx_started` datetime NOT NULL DEFAULT '0000-00-00 00:00:00' COMMENT '事务的开始时间',
  `trx_requested_lock_id` varchar(105) DEFAULT NULL COMMENT '等待事务的锁id。比如trx_state为LOCK——WAIT，该值代表当前事务等待之前事务占用锁资源的ID。若trx_state不是LOCK_WAIT，该值为null',
  `trx_wait_started` datetime DEFAULT NULL COMMENT '事务等待开始的时间',
  `trx_weight` bigint(21) unsigned NOT NULL DEFAULT '0' COMMENT '事务权重，反映了一个事务修改和锁住的行数，当发生死锁需要回滚时，InnoDB会选择该值最小的进行回滚',
  `trx_mysql_thread_id` bigint(21) unsigned NOT NULL DEFAULT '0' COMMENT 'mysql中的线程ID，SHOW PROCESSLIST显示的结果',
  `trx_query` varchar(1024) DEFAULT NULL COMMENT '事务允许的SQL语句',
  `trx_operation_state` varchar(64) DEFAULT NULL,
  `trx_tables_in_use` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_tables_locked` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_lock_structs` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_lock_memory_bytes` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_rows_locked` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_rows_modified` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_concurrency_tickets` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_isolation_level` varchar(16) NOT NULL DEFAULT '',
  `trx_unique_checks` int(1) NOT NULL DEFAULT '0',
  `trx_foreign_key_checks` int(1) NOT NULL DEFAULT '0',
  `trx_last_foreign_key_error` varchar(256) DEFAULT NULL,
  `trx_adaptive_hash_latched` int(1) NOT NULL DEFAULT '0',
  `trx_adaptive_hash_timeout` bigint(21) unsigned NOT NULL DEFAULT '0',
  `trx_is_read_only` int(1) NOT NULL DEFAULT '0',
  `trx_autocommit_non_locking` int(1) NOT NULL DEFAULT '0'
) ENGINE=MEMORY DEFAULT CHARSET=utf8;
~~~

表INNODB_LOCKS结构如下图（图片来自Mysql技术内幕InnoDB存储引擎第二版）：

![image-20230413154337824](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413154337824.png)

> innodb_locks表在8.0.13版本中由performance_schema.data_locks表所代替

表INNODB_LOCKS_WAITS结构如下图（图片来自Mysql技术内幕InnoDB存储引擎第二版）：

![image-20230413154451680](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230413154451680.png)

> INNODB_LOCKS_WAITS表在8.0.13版本中由performance_schema.data_locks_waits表所代替

### 3.2.2 一致性非锁定读

> **事务的隔离级别：**
>
> Read uncommitted：未提交读，任何读问题都解决不了
>
> Read committed：已读提交，解决脏读，但是不可重复读和虚读有可能发生（Oracle）
>
> Repeatable read：重复读，解决脏读和不可重复读，但是虚读有可能发生（Mysql）
>
> Serializable：解决所有读问题
>
> MySQL的默认隔离级别是：REPEATABLE READ



