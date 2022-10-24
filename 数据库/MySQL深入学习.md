# MySql深入学习

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

### 2.3 B+树索引

B+树索引本质上就是B+树在数据库中的实现，但是B+树索引在数据库中有一个特点就是高扇出性，因此在数据库中，B+树的高度一般在2-4层，也就是说查找某一键值只需要2-4次IO，当前机械硬盘每秒100次IO，因此意味着查询时间只需要0.02-0.04秒。

数据库中的B+树索引可以分为聚集索引和辅助索引，两种内部都是B+树，即高度平衡的，叶子节点存放着所有的数据，聚集索引和辅助索引不同的是，聚集索引叶子节点存放了一整行的信息。

