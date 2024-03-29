# MySql——InnoDB深入学习

> **转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。**
>
> 参考资料见最后一章
>
> **所有例子均是本人亲自上机后，将代码或结果复制回来的。请勿盗图**

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

   一般来说数据库缓冲池是通过LRU（最近最少使用算法）来进行管理的。最频繁使用的页在LRU的前端，最少使用的页在LRU的尾端。当缓冲池不能存放新读取的页时，应先释放LRU尾部的页。**在InnoDB中，缓冲池页大小默认为16KB**，且InnoDB对传统LRU算法进行了优化，加入了midpoint位置。即新读取的页，并不是放入LRU列表的首部，而是在midpoint位置。默认配置下，该位置在LRU列表长度的5/8处，midpoint由参数`innodb_old_blocks_pct`控制，如下：

   ![image-20221014135127892](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221014135127892.png)

   该值默认为37，也就是在LRU列表尾端的37%左右的位置，在InnoDB中，在midpoint之后的列表是old列表，在midpoint之前的列表是new列表，可以简单理解为活跃数据端。

   MySQL这么改进的原因主要是想解决缓存污染的问题。那么什么是缓存污染的问题呢？比如我有一条sql`select * from tb_XXX where name like"%张%"`，这条sql会扫描了大量的数据时，在缓冲池空间比较有限的情况下，可能会将缓冲池里的所有页都替换出去，导致大量热数据被淘汰了，等这些热数据又被再次访问的时候，由于缓存未命中，就会产生大量的磁盘 I/O，MySQL 性能就会急剧下降。而这些不常用的数据又占据着当前的缓冲池，造成缓存污染。

   而MySQL为了解决此问题又引入了另一个参数进一步管理LRU列表`innodb_old_blocks_time`，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。如果第二次的访问时间与第一次访问的时间在 1 秒内（默认值），那么该页就不会被从 old 区域升级到 new区域；如果第二次的访问时间与第一次访问的时间超过 1 秒，那么该页就会从 old 区域升级到 young 区域。这是为了防止某些不常用的页（如全表扫描的页）在短时间（如1秒内）内被多次访问，让系统误以为它是热数据从而将其放入了热端区域。

   LRU列表用来管理已经读取的页，但数据库刚启动时，LRU列表是空的，这时页都存放在Free列表中，当需要缓冲池分页时，首先从Free表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，加入LRU列表。否则淘汰LRU末尾的页。当页从old加入到new部分时，称为page made young，反之因为innodb_old_blocks_time导致没有移动到new 被称为page not made young，可以通过`SHOW ENGINE INNODB STATUS来观察`：

   ![image-20221014154035205](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20221014154035205.png)

   > 这里还有个变量 Buffer poll hit rate 表示缓冲池命中率，一般该值不低于95%

   在LRU中的页被修改后，该页为脏页，这时数据库就会通过CHECKPOINT机制将数据刷新回磁盘。Flush列表中的页就是脏页，但是脏页即存在LRU列表又存在于Flush列表。

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

对于什么时候添加B+树索引，一般来说在低选择性的字段上加比较有意义。比如对于性别字段它的取值范围很小，这就是低选择性。如何查看索引的高选择性呢？可以通过Cardinality值，它表示索引中不重复记录数量的预估值。在实际使用时`Cardnality/n_rows_in_table`的值应该尽可能接近1，如果非常小，用户应考虑是否有必要创建这个索引。

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

### 2.3.3 聚集索引

聚集索引并不是单独的索引类型，而是一种数据存储方式，当表中有聚集索引时，它的数据行实际上存放在索引的叶子页中，聚集主要表示数据行和相邻的键值紧凑的存储在一起。因为无法同时把数据行存放在两个不同的地方，所以一个表只能有一个聚集索引。在多数情况下，查询优化器倾向于采用聚集索引，因为聚集索引在B+树索引的叶子节点上能直接找到数据，而且由于定义了数据的逻辑顺序，聚集索引能特别快的针对范围值的查询。查询优化器能快速发现某一段范围的数据页需要扫描。聚集索引的数据分布如下图（图片来自《高性能Mysql第三版》）：

![image-20230419092503587](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230419092503587.png)

注意，聚集索引的存储并不是物理上连续的，而是逻辑上连续的：

- 一是页通过双向链表链接，页按照主键顺序排序
- 二是每个页中的记录也是通过双向链表维护的，物理存储上可以不按照主键存储

聚集索引有一些非常重要的优点：

- 可以把相关数据保存在一起。例如实现email时可以通过用户ID来聚集数据，这样只需要从磁盘读取少量的数据页就能获取每个用户的全部邮件，如果没有聚集索引，则每封邮件都可能导致一次IO
- 数据访问更快。聚簇索引将索引和数据保存在同一个 B-Tree 中，因此从聚簇索引中获取数据通常比在非聚簇索引中查找要快
- 使用覆盖索引扫描的查询可以直接使用页节点中的主键值

如果设计表时能充分利用聚集索引，就能极大的提升性能，当然聚集索引也有缺点：

- 聚簇数据最大限度地提高了 I/0 密集型应用的性能，但如果数据全部都放在内存中则访问的顺序就没那么重要了，聚簇索引也就没什么优势了
- 插入速度严重依赖于插入顺序。按照主键的顺序插入数据到InnoDB表中是速度最快的方式。但如果不是按照主键顺序加载数据，那么在加载完成后最好使用OPTIMIZE TABLE命今重新组织一下表。
- 更新聚簇索引列的代价很高，因为会强制InnoDB 将每个被更新的动到新的位置。
- 基于聚簇索引的表在插人新行，或者主键被更新导致需要移动行的时候，可能面临B+树分裂的问题。（见2.3.1小节）。当行的主键值要求必须将这一行插到某个已满的页中时，存储引擎会将该页分裂成两个页面来容纳该行，这就是一次页分裂操作。页分裂会导致表占用更多的磁盘空间。
- 聚簇索引可能导致全表扫描变慢，尤其是行比较稀疏，或者由于页分裂导致数据存储不连续的时候。
- 二辅助索引(非聚族索引)可能比想象的要更大，因为在二级索引的叶子节点包含了引用行的主键列。

当然由于辅助索引的叶子节点保存的不是指向物理位置的指针，也是该数据行的主键值，所以通过辅助索引，存储引擎需要在辅助索引的叶子节点拿到对应的主键值，然后根据这个值去聚集索引查找对应的行，这里就是两次B+树查找。当然对于InnoDB，自适应哈希索引能减少这样的重复工作。

### 2.3.4 辅助索引

对于辅助索引（Secondary Index，非聚集索引）来说，叶子节点并不并不包含行记录的全部数据。叶子节点除了包含键值以外，每个叶子节点中的索引行中还包含了一个书签(bookmark)。该书签用来告诉 InnoDB 存引擎哪里可以找到与索引相对应的行数据。由于InnoDB 存储引擎表是索引组织表，因此 InnoDB 存储引擎的辅助索引的书签就是相应行数据的聚集索引键。下图是辅助索引和聚集索引的关系（图片来自《Mysql技术内幕InnoDB存储引擎第二版》）：

![image-20230419093816587](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230419093816587.png)

辅助索引的存在不影响聚集索引的组织，所以每张表上可以有多个辅助索引。当通过辅助索引查找数据时，存储引擎需要在辅助索引的叶子节点拿到对应的主键值，然后根据这个值去聚集索引查找对应的行记录。

## 2.4 哈希索引

在InnoDB中，对于哈希算法的实现是自适应哈希索引，且自适应哈希索引是InnoDB自己控制的，DBA不可以干预。可以通过参数`innodb_adaptive_hash_index`来禁用或启动此特性，默认开启。可以通过`SHOW ENGINE INNODB STATUS`看到当前哈希索引的使用状况。

InnoDB中使用链表实现冲突机制，哈希函数采用触发散列方式。

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

有时候可能会需要索引很长的字符列，这会让索引变得又大又慢，有一种策略是使用2.4中的模拟hash索引，还有一种就是可以使用下面的前缀索引。其实前缀索引就是索引字符串前面的部分字符，这样可以节约索引空间，提高索引效率，但是这样也会降低索引的选择性。**索引的选择性是指不重复的索引值和数据表的记录总数（#T）的比值，范围从`1/#T`到1之间。**索引的选择性越高，查询的效率就越高，唯一索引的选择性是1，这也是最好的索引选择性。

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

~~~mysql
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

> 注意这里

### 2.5.3 联合索引

> Lahdenmaki在他的书中介绍了索引的三星系统，即索引将相关的记录放在一起就获得一星；如果索引中的数据顺序与查询中的排序顺序一致则获得二星；如果索引中的列包含了查询中需要的全部列则获得三星。

关于索引一个常见的错误就是为每个列创建独立的索引，或者按照错误得顺序创建多列索引：

~~~mysql
create table t (
	c1 int,
    c2 int,
    c3 int,
    key(c1),
    key(c2),
    key(c3)
);
~~~

上面这种方式就是“把WHERE条件里的列都加上索引”这种模糊的建议导致的。实际上这个建议是错误得，这样一来最好的情况下也只能是一星索引，性能可能照最优得索引差几个数量级，有时如果无法设计一个三星索引，不如忽略WHERE子句，集中精力优化索引列得顺序，或者创建一个全覆盖索引。

**实际上在多个列上建立独立的单列索引大部分情况下并不能提高 MySQL 的查询性能**。MySQL5.0 及以后版本引入了一种叫“索引合并”(index merge)的策略，一定程度上可以使用表上的多个单列索引来定位指定的行。更早版本的 MySQL 只能使用其中某一个单列索引，这种情况下没有哪一个独立的单列索引是非常有效的。例如，表 film actor 在字段 film_id 和actor id 上各有一个单列索引。但对于下面这个查询 WHERE 条件，这两个单列索引都不是好的选择（下面的例子来自《高性能Mysql第三版》）：

~~~mysql
#film_actor是mysql官方示例库sakila中的表
select film_id,actor_id from film_actor是mysql官方示例库sakila中的表 where actor_id =1 OR film_id =1;
#union
select film_id,actor_id from film_actor where actor_id =1 
UNION ALL
explain select film_id,actor_id from film_actor where film_id =1 and actor_id <> 1;
~~~

在老的MYSQL中，这个查询会使用全表扫描，除非改成union的方式(见上面)，但是在5.0 及以后版本会使用索引合并，本质上就是两个索引扫描的联合，我们可以通过explain进行查看：

![image-20230418100937721](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418100937721.png)

如上面的例子，mysql会使用这个技术对复杂查询进行优化，所以某些sql在extra中还可以看到嵌套操作。索引合并是一种优化的结果，实际上说明了表的索引建的很糟糕：

- 当出现服务器对多个索引做相交操作时 (通常有多个 AND 条件)，通常意味着需要个包含所有相关列的多列索引，而不是多个独立的单列索引。
- 当服务器需要对多个索引做联合操作时(通常有多个 0R条件)，通常需要耗费大量CPU 和内存资源在算法的缓存、排序和合并操作上。特别是当其中有些索引的选择性不高，需要合并扫描返回的大量数据的时候。
- 更重要的是，优化器不会把这些计算到“查询成本”(cost)中，优化器只关心随机页面读取。这会使得查询的成本被“低估”,导致该执行计划还不如直接走全表扫描，这样做不但会消耗更多的 CPU 和内存资源，还可能会影响查询的并发性，但如果是单独运行这样的查询则往往会忽略对并发性的影响。

所以如果在 EXPLAIN中看到有索引合并，应该好好检查一下查询和表的结构，看是不是已经是最优的。也可以通过参数 optimizer switch 来关闭索引合并功能。也可以使用IGNOREINDEX提示让优化器忽略掉某些索引。

那么什么时候需要使用联合索引呢？从本质上来说联合索引也是一个b+树，结构如下（图片来自《Mysql技术内幕InnoDB存储引擎第二版》）：

![image-20230418101800419](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418101800419.png)

对于`select * from table_name where a=xx and b=xx`，显然可以使用`(a,b)`这个联合索引的。而且对于单个a列的查询`select * from table_name where a=xx`也是可以使用这个索引的，但是对于b列的查询是不可以的，因为上图可见，叶子节点的b值显然不是排序的。

联合索引还有一个好处就是对第二个键值已经做好了排序处理。我们举个例子，首先创建表并加入数据，如下图：

![image-20230418103527078](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418103527078.png)

然后加上两个索引：

![image-20230418103549924](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418103549924.png)

然后我们看看当只查询id时，优化器选择的索引是`key_id`，因为这个索引的叶子节点是一个值，所以一页可以放更多数据。

![image-20230418103722210](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418103722210.png)

但当我们执行以下SQL时发现走的是`key_double`这个索引，因为联合索引的第二个值已经排好序了。

![image-20230418103917500](C:\Users\yhr\AppData\Roaming\Typora\typora-user-images\image-20230418103917500.png)

若强制使用`key_id`索引，如下会额外进行一次排序：

![image-20230418104346156](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418104346156.png)

也就是说联合索引 `(a，b)`其实是根据列a、b进行排序，对于联合索引`(a，b，c)`来说，下列语同样可以直接通过联合索引得到结果：

~~~mysql
select * from table_name where a=xx order by b
select * from table_name where a=xx and b=xx  order by c
~~~

但是对于下面的语句，联合索引不能直接得到结果，其还需要执行一次 filesort 排序操作，因为索引` (a，c)`并未排序:

~~~MYSQL
select * from table_name where a=xx order by c
~~~

### 2.5.4 合适的索引列顺序

那么联合索引怎么选择合适的索引列顺序呢？正确的顺序依赖于该索引的查询，并且需要同时考虑如何更好的满足排序和分组的需要。在一个联合索引中，索引列的顺序意味着索引首先按照最左列进行排序，其次是第二列，等等。所以，索引可以按照升序或者降序进行扫描，以满足精确符合列顺序的ORDER BY、GROUP BY和 DISTINCT等子句的查询需求。

所以联合索引的列顺序非常重要，在Lahdenmaki的三星系统中，列顺序也决定了一个索引是否能成为三星索引。

> Lahdenmaki在他的书中介绍了索引的三星系统，即索引将相关的记录放在一起就获得一星；如果索引中的数据顺序与查询中的排序顺序一致则获得二星；如果索引中的列包含了查询中需要的全部列则获得三星。

对于索引的列顺序有一个经验法则：即将选择性最高的列放在索引最前面。当不需要考虑排序和分组时，将选择性最高的列放在前面通常是很好的。这时候索引的作用只是用于优化 WHERE条件的查找。在这种情况下，这样设计的索引确实能够最快地过滤出需要的行，对于在 WHERE子中只使用了索引部分前缀列的查询来说选择性也更高。然而，性能不只是依赖于所有索引列的选择性(整体基数)，也和查询条件的具体值有关，也就是和值的分布有关。这和2.5.2中前缀索引的例子一样，虽然6位和7位差不多，但是值的分布可能会很不均匀。可能需要根据那些运行频率最高的查询来调整索引列的顺序，让这种情况下索引的选择性最高。我们下面举个例子：

~~~mysql
#payment是mysql官方示例库sakila中的表，payment表中staff_id和customer_id都有单列的索引
select * from payment where staff_id =2 and customer_id = 584
~~~

这个查询是应该创建一个`(staff_id,customer_id)`还是应该创建一个`(customer_id,staff_id)`？我们可以允许SQL来确定表中值得分布情况，并确定哪个列的选择性能高。我们先查一下数据基数：

~~~mysql
select sum(staff_id=2),sum(customer_id=584) from payment
~~~

结果：

| sum(staff_id=2) | sum(customer_id=584) |
| --------------- | -------------------- |
| 7990            | 30                   |

​	根据前面的经验法则，将选择性最高的列放在索引最前面，我们应该将customer_id放在前面，因为对应条件值得customer_id更小，这里也可以去计算一下两个列的选择性，如下图：

![image-20230418134808062](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418134808062.png)

得到customer_id列的选择性更高，所以我们可以将customer_id放在第一位,即建立`(customer_id,staff_id)`索引。

当然上面这种经验法则在多数情况是有用的，但是有时特殊情况可能会摧毁整个应用的性能，比如下面这个例子（例子来自《高性能Mysql第三版》）：

![image-20230418141810494](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418141810494.png)

### 2.5.5 覆盖索引

InnoDB支持覆盖索引，也就是说从辅助索引就可以得到查询的记录，而不需要查询聚集索引中的记录。如果一个索引包含或覆盖所有需要查询字段的值，就可以成为覆盖索引。使用覆盖索引的好处就是辅助索引不包含整行记录的所有信息，所以大小远小于聚集索引，因此可以减少大量IO操作。

对于 InnoDB 存储引擎的辅助索引而言，由于其包含了主键信息，因此其叶子节点存放的数据为`(primary key1、primary key2 ···、key1、key2、···)`。例如，下列语句都可仅使用一次辅助联合索引来完成查询：

~~~mysql
select key2 from table where key1 =xxx;
select primarykey2,key2 from table where key1 =xxx;
select primarykey1,key2 from table where key1 =xxx;
~~~

覆盖索引能极大的提升性能，其优点如下：

1. 索引条目通常远小于数据行大小，所以如果只需要读取索引，那 MySQL就会极大地减少数据访问量。这对缓存的负载非常重要，因为这种情况下响应时间大部分花费在数据拷贝上。覆盖索引对于I/O 密集型的应用也有帮助，因为索引比数据更小更容易全部放入内存中。
2. 因为索引是按照列值顺序存储的 (至少在单个页内是如此)，所以对于IO 密集型的范围查询会比随机从磁盘读取每一行数据的I/0 要少得多。
3. 由于InnoDB的聚集索引，覆盖索引对InnoDB表特别有用。 InnoDB的二级索引在叶子节点中保存了行的主键值，所以如果二级主键能够覆盖查询，则可以避免对主键索引的二次查询。

当发起一个索引覆盖的查询，在explain的extra列可以看到`Using index`的信息，比如下面的例子：

![image-20230419100226728](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230419100226728.png)

> sakila是mysql的官方示例数据库

### 2.5.6 使用索引扫描进行排序

MySQL有两种方式可以生成有序的结果：一是通过排序操作；或者按索引顺序扫描。如果explain出来的type列的值是index，则说明MySQL使用了索引扫描来做排序。扫描索引本身是很快的，因为只需要从一条索引记录移动到紧接着的下一条记录，但如果索引不能覆盖查询所需要的全部列，那就不得不每扫描一条索引就回表查询一次，这基本都是随机IO。因此按照索引顺序读取数据的速度通常比顺序地全表扫描慢，尤其是在IO密集型的工作负载时。

MySQL 可以使用同一个索引既满足排序，又用于查找行。因此设计索引时应该尽可能地同时满足这两种。只有当索引的列顺序和ORDER BY子句的顺序完全一致，并且所有列的排序方向（倒序或正序)都一样时，MySQL才能够使用索引来对结果做排序。如查询需要关联多张表，则只有当ORDER BY子句引用的字段全部为第一个表时，才能使用索引做排序，ORDER BY子句和查找型查询的限制是一样的:需要满足索引的最左前缀的要求，否则MySQL都需要执行排序操作，而无法利用索引排序。

有一种情况下ORDER BY子句可以不满足索引的最左前缀的要求，就是前导列为常量的时候。如果 WHERE子句或者 JOIN子句中对这些列指定了常量,就可以“弥补”索引的不足。举个例子，sakila示例数据如下：

![image-20230419104929292](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230419104929292.png)

mysql可以使用`rental_data`索引为下面的查询做排序，我们看一下explain，可以看到并没有文件排序（filesort）操作：

![image-20230419105136048](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230419105136048.png)

这里即使Order BY不满足最左前缀的要求，但是由于索引中的第一个列被指定为常数，而使用第二列进行排序，形成了最左前缀所以也可以用于查询排序。

如果上面的sql写成下面的形式，则不能用索引进行排序：

- 使用了不同的排序方向

  ~~~mysql
  ... where rental_date ="2005-05-25" ORDER BY inventory_id desc,customer_id asc
  ~~~

- Order BY子句中引用了不在索引的列
  ~~~mysql
  ... where rental_date ="2005-05-25" ORDER BY inventory_id ,staff_id 
  ~~~

- Where 和order by 无法组成最左前缀
  ~~~mysql
  ... where rental_date ="2005-05-25" ORDER BY customer_id
  ~~~

- 索引列的第一列是范围条件，所以无法使用索引的其余列
  ~~~mysql
  ... where rental_date > "2005-05-25" ORDER BY inventory_id ,customer_id 
  ~~~

- 在inventory_id列上有多个等于条件，对于排序来说也算是范围查询
  ~~~mysql
  ... where rental_date ="2005-05-25" and inventory_id in (1,2)  ORDER BY customer_id
  ~~~

### 2.5.7 索引和锁

索引可以让查询锁定更少的行，如果你的查询从不访问那些不需要的行，那么就会锁定更少的行。从两个方面来看这对性能都有好处。首先，虽然InnoDB 的行锁效率很高内存使用也很少，但是锁定行的时候仍然会带来额外开销，其次，锁定超过需要的行会增加锁争用并减少并发性。

InnoDB 只有在访问行的时候才会对其加锁，而索引能够减少InnoDB 访问的行数，从而减少锁的数量。但这只有当InnoDB 在存储引擎层能够过滤掉所有不需要的行时才有效。如果索引无法过滤掉无效的行，那么在InnoDB 检索到数据并返回给服务器层以后，MySQL服务器才能应用WHERE子句。这时已经无法避免锁定行了，因为InnoDB已经锁住了这些行。在MySQL 5.1和之后的版本中，InnoDB 可以在服务器端过滤掉行后就释放锁，但是在早期的MySOL版本中InooDB 只有在事务提交后才能释放锁。



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

> **也就是说update和delete操作都会加X锁**

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

一致性非锁定读是指InnoDb通过行多版本控制的方式来读取当前执行时间数据库中的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会去等待行上锁的释放，而是会去读取行的一个快照数据。如下图（图片来自Mysql技术内幕InnoDB存储引擎第二版）：

![image-20230414091443582](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414091443582.png)

上图非常直观的展示了非锁定读，因为读时不需要等待行上X锁的释放。这里快照数据是指该行的之前版本的数据，该实现是通过undo字段来完成的，而undo用来在事务中回滚数据，因此快照数据本身没有额外的开销。此外，读取快照数据不需要上锁，因为没有事务会操作历史数据。如上图，一行记录可能不只有一个快照数据，每行记录可能有多个版本，一般称这种技术为行多版本技术，由此带来的并发控制被称为多版本并发控制（Multi Version Concurrency Control ，MVCC）。

非锁定读极大的提高了数据库的并发性能，在InnoDB中这时默认的读取方式，也就是说读取不会占用和等待表上的锁。但是在不同的事务隔离级别下，读取的方式不同，并不是每个事务隔离级别下都是采用非锁定一致性读。此外，即使都是用非锁定一致性读，但是快照的底层定义也不同。

> **事务的隔离级别：**
>
> Read uncommitted：未提交读，任何读问题都解决不了
>
> Read committed：读已提交，解决脏读，但是不可重复读和虚读有可能发生（Oracle）
>
> Repeatable read：重复读，解决脏读和不可重复读，但是虚读有可能发生（Mysql）
>
> Serializable：解决所有读问题
>
> MySQL的默认隔离级别是：REPEATABLE READ

在事务隔离级别Read committed和Repeatable read下，虽然都使用非锁定一致性读，但是快照数据的定义不同。在 READ COMMITTED 事务隔离级别下，对于快照数据，非锁定一致性读总是读取被锁定行的最新的一份快照数据。而在 REPEATABLE READ 事务隔离级别下，对于快照数据，非锁定一致性读总是读取事务开始时的行数据版本。举例：

我们先在会话1中开启事务，然后查询actor_id =1的数据：

~~~mysql
#会话1
BEGIN;
select * from actor where actor_id =1;
~~~

结果：

| actor_id | first_name | last_name | last_updade         |
| -------- | ---------- | --------- | ------------------- |
| 1        | PENELOPE   | GUINESS   | 2023-04-14 01:47:36 |

然后新开一个会话（这里记为会话2），执行以下SQL：

~~~mysql
#会话2
BEGIN;
UPDATE actor set actor_id = 999 where actor_id =1;
~~~

然后在会话1中再次使用`select * from actor where actor_id =1;`查询，这时不管是Read committed还是Repeatable read两种隔离级别都是刚才的数据：

![image-20230414095651195](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414095651195.png)

然后我们通过`COMMIT;`提交会话2的事务，然后再回到会话1去查询，这时

- 当隔离级别是Read committed时

  > 注意由于mysql默认隔离级别不是Read committed，所以在我们在刚打开这个会话时需要手动设置隔离级别：
  >
  > ~~~mysql
  > #设置当前会话的隔离级别，这种设置的隔离级别只在当前会话生效
  > set session transaction isolation level read committed;
  > ~~~

  ![image-20230414095836550](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414095836550.png)

  查询的数据应该是空：

  ![image-20230414100019716](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414100019716.png)

- 当隔离级别是Repeatable read时

  ![image-20230414100200869](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414100200869.png)

  查询的数据应该还是刚才的数据

  ![image-20230414100225466](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414100225466.png)

为什么两种隔离级别的查询结果不一样呢？这是因为**Read committed（读已提交）的隔离级别下，总是读取行的最新版本，如果行被锁定了就读取该行的最新一个快照**，所以在上面的例子中，在会话2未提交事务时还能读到actor_id=1的数据，但是会话2提交了事务之后，会读到最新版本，也就是actor_id=1的数据已经不存在了，读取了空值。

**而隔离级别是Repeatable read（可重复读）时，总是读取事务开始时的行数据**，所以还是能读到actor_id=1的数据。下图是上面例子的流程图：

![image-20230414101614989](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414101614989.png)

> MYSQL旧版本也就是5.x的变量才是tx_isolation；新版本(8.x)的系统变量改成transaction_isolation

### 3.2.3 一致性锁定读

在默认配置下，隔离级别是Repeatable read（可重复读），且InnoDB的Select操作是一致性非锁定读。但是在某些情况下，用户需要显式的对数据库读取操作进行加锁以保证数据逻辑的一致性。InnoDB对Select语句支持两种一致性锁定读操作：

- SELECT  ...... FOR UPDATE
- SELECT  ...... LOCK IN SHARE MODE

`SELECT  ...... FOR UPDATE`对读取的行记录加一个X锁，其他事务不能对已锁定的行加任何锁。`SELECT  ...... LOCK IN SHARE MODE`会对读取的行记录加一个S锁，其他事务可以向被锁定的行加S锁，但是不能加X锁。

而对于一致性非锁定读，即使该行执行了`SELECT  ...... FOR UPDATE`，也是可以读取的，这跟上一节的例子是一样的，因为update也是加X锁。同时还要注意，`SELECT  ...... FOR UPDATE`和`SELECT  ...... LOCK IN SHARE MODE`必须在一个事务里，当事务提交了，锁也就释放了，因此在使用上述两句 SELECT 锁定语句时，务必加上 BEGIN，START TRANSACTION 或者 SETAUTOCOMMIT=0。

## 3.3 锁的算法

### 3.3.1 行锁的三种算法

InnoDB有三种行锁的算法，分别是：

- Record Lock：单个行记录上的锁
- Gap Lock：间隙锁，锁定一个范围，但不包含记录本身
- Next-Key Lock：Gap Lock+Record Lock，锁定一个范围，且包含锁定记录本身。

> Record Lock总会去锁住索引记录，如果InnoDB存储引擎表在建立的时候没有设置任何一个索引，那么这时InnoDB存储引擎会使用隐式的主键来进行锁定。

Next-Key Lock的锁定技术被称为Next-Key Locking。设计的目的是为了解决Phantom Problem问题（幻读问题）。利用这种锁定技术锁定的不是单个值而是一个范围，next-key lock本质上就是一个拥有左开右闭区间的锁，假如一个索引有10，11，13，20这四个值，那么该索引可能被Next-Key Locking的区间为`(负无穷,10]`,`(10,11]`,`(11,13]`,`(13,20]`,`(20,正无穷)`。假如事务T1已经通过Next-Key Locking锁定了`(10,11]、(11,13]`，当插入新的记录12时，锁定的范围就会变成`(10,11]、(11,12]、(12,13]`。但是当查询的索引含有唯一属性时，InnoDB会对Next-Key Lock进行优化，将其降级为Record Lock，仅锁住索引本身，而不是范围。举个例子：

我们首先先建个表，然后插入几条数据：

![image-20230414110507052](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414110507052.png)

然后我们执行以下SQL

| 时间 | 会话1                                          | 会话2                            |
| ---- | ---------------------------------------------- | -------------------------------- |
| 1    | BEGIN;                                         |                                  |
| 2    | SELECT * FROM test_table WHERE a=5 FOR UPDATE; |                                  |
| 3    |                                                | BEGIN;                           |
| 4    |                                                | INSERT INTO test_table SELECT 4; |
| 5    |                                                | COMMIT;                          |
| 6    | COMMIT;                                        |                                  |

test_table 共有 1、2、5三个值。在上面的例子中，在会话1中首先对 a=5 进行锁定而由于 a 是主键且唯一，因此锁定的仅是 5 这个值，而不是(2，5]这个范围，这样在会话2中插入值 4 就不会阻塞，可以立即插人并返回。即锁定由 Next-Key Lock 算法降级为了 Record Lock，从而提高应用的并发性。

但是如果是辅助索引则情况会完全不同，我们先建个数据表并插入数据：

![image-20230414111115326](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414111115326.png)

| 时间 | 会话1                                           | 会话2                                                 |
| ---- | ----------------------------------------------- | ----------------------------------------------------- |
| 1    | BEGIN;                                          |                                                       |
| 2    | SELECT * FROM test_table2 WHERE b=3 FOR UPDATE; |                                                       |
| 3    |                                                 | BEGIN;                                                |
| 4    |                                                 | #这句会被阻塞<br/>INSERT INTO test_table2 SELECT 4,2; |

这时在第二步，由于SQL语句通过索引列b进行查询，所以使用Next-Key Locking技术继续加锁。且由于这个表有两个索引，所以需要分别锁定。对于聚集索引，仅需要对列a=5（也就是b=3）的索引加上Record Lock。对于辅助索引来说，加的是Next-Key Lock，锁定的范围是`(1,3)`。这里还需要注意，**InnoDB还会对辅助索引的下一个键值加上间隙锁(gap lock)**，所以这里实际上还有一个辅助索引范围为`(3,6)`的锁。这时当执行下面这几句sql时都会被阻塞：

~~~mysql
#由于a=5索引已经加上了X锁，所以执行会被阻塞
SELECT * FROM test_table2 WHERE a=5 LOCK IN SHARE MODE;
#主键插入4没问题，但是辅助索引值插入的2在(1,3)范围中，所以执行会被阻塞
INSERT INTO test_table2 SELECT 4,2;
#主键插入6没问题，但是辅助索引值插入的5也不在(1,3)范围中，但是5在另一个间隙锁的锁定范围(3,6),所以执行会被阻塞
INSERT INTO test_table2 SELECT 6,5;
~~~

虽然上面的SQL会被阻塞，但是下面的SQL就不会被阻塞：

~~~mysql
INSERT INTO test_table2 SELECT 8,6;
INSERT INTO test_table2 SELECT 2,0;
INSERT INTO test_table2 SELECT 6,7;
~~~

从上面的例子中可以看到，Gap Lock的作用是为了阻止多个事务将记录插入到同一范围内，而多个事务将记录插入到同一范围内会引起幻读问题（Phantom Problem）。例如在上面的例子中，会话1中用户已经锁定了b=3的记录，如果此时没有Gap Lock锁定`(3,6)`，那么用户可以插入b=3的记录，这样会导致会话1的用户再次查询同样的条件返回不同的记录，也就是幻读。

> Phantom ：n. 幽灵，鬼魂；幻觉，幻象；adj. 虚幻的，幻觉的；

用户可以通过以下两种方式显示关闭Gap Lock：

- 将事务的隔离级别设置为读已提交
- 将参数`innodb_locks_unsafe_for_binlog`设置为1

同时在InnoDB中，对于INSERT的操作，会检查插入记录的下一条记录是否被锁定，若已经被锁定则不允许查询，就像上面的例子一样。

**对于唯一键值的锁定，Next-Key Lock 降级为 Record Lock仅存在于查询所有的唯一索引列。**若唯一索引由多个列组成，而查询仅是查找多个唯一索引列中的其中一个，那么查询其实是 range 类型查询，而不是 point 类型查询，故InnoDB存储引擎依然使用 Next-Key Lock 进行锁定。

### 3.3.2 mysql加锁总结

下面我们对mysql加行级锁进行总结：

- 普通的select语句不会加行级锁，因为普通的select是通过MVCC并发版本控制读取快照，不需要加锁。
- 要想在select语句中加锁，可以使用以下两种方式：
  - SELECT  ...... FOR UPDATE   **对读取的记录加独占锁(X型锁)**
  - SELECT  ...... LOCK IN SHARE MODE  **对读取的记录加共享锁(S型锁)**
- update 和 delete 操作都会加行级锁，且锁的类型都是独占锁(X型锁)。
- 在读已提交隔离级别（Read committed）下，行级锁的种类只有记录锁，也就是仅仅把一条记录锁上。
- 在可重复读隔离级别（Repeatable read）下，行级锁的种类除了有记录锁，还有间隙锁（目的是为了避免幻读），行级锁主要有三类：
  - Record Lock，记录锁，锁住的是一条记录，而且记录锁是有 S 锁和 X 锁之分的；
  - Gap Lock，间隙锁，锁定一个范围，但是不包含记录本身，它只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象；
  - Next-Key Lock：Record Lock + Gap Lock 的组合，锁定一个范围，并且锁定记录本身。

### 3.3.3 mysql行级锁规则

行级锁加锁规则比较复杂，不同的场景，加锁的形式是不同的：

**加锁的对象是索引，加锁的基本单位是 next-key lock**。它是由记录锁和间隙锁组合而成的，next-key lock 是前开后闭区间，而间隙锁是前开后开区间。但是，next-key lock 在一些场景下会退化成记录锁或间隙锁。

> **也就是说只要是加行级锁，默认都是加next-key lock**

#### 唯一索引等值查询

- 当查询的记录是存在的，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会退化成记录锁，只锁住当前记录。

- 当查询的记录是不存在的，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会退化成间隙锁。

  例如，在下表当中:

  | id   | name   |
  | ---- | ------ |
  | 3    | "小明" |
  | 5    | "小华" |

  此时，如果一条sql语句执行`select * from user where id=4 for update`。这个时候，id为4的这一行记录，因为不存在于表当中，因此会锁住最近的索引范围，也就是会加上一个id为`(3,5)`范围的间隙锁。这个时候，如果有其他的事物想插入id为4的数据，那么就会被阻塞。

> **对于唯一键值的锁定，Next-Key Lock 降级为 Record Lock仅存在于查询所有的唯一索引列。**若唯一索引由多个列组成，而查询仅是查找多个唯一索引列中的其中一个，那么查询其实是 range 类型查询，而不是 point 类型查询，故InnoDB存储引擎依然使用 Next-Key Lock 进行锁定。

#### 唯一索引范围查询

当唯一索引进行范围查询时，会对每一个扫描到的索引加 next-key 锁，然后如果遇到下面这些情况，会退化成记录锁或者间隙锁：

- 情况一：针对大于等于的范围查询，因为存在等值查询的条件，那么如果等值查询的记录是存在于表中，那么该记录的索引中的 next-key 锁会**退化成记录锁**。

  我们用3.3.1中的test_table2实验一下，数据如下：

  | a（主键） | b（辅助索引） |
  | --------- | ------------- |
  | 1         | 1             |
  | 3         | 1             |
  | 4         | 2             |
  | 5         | 3             |
  | 8         | 6             |
  | 20        | 8             |

  我们执行以下语句：

  ~~~mysql
  BEGIN;
  select * from test_table2 where a >= 6 for update;
  ~~~

  然后我们使用`select * from performance_schema.data_locks`查询当前锁，我们可以看到：

  ![image-20230414151853083](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414151853083.png)

  这时由于没有找到主键索引6，所以会对主键索引加next-key锁，即锁住了`(6,8]`，由于是范围查找所以会继续往后找，表中最后一条记录是20所以会加next-key锁，即锁住了`(8,20]`。实际在 Innodb 存储引擎中，会用一个特殊的记录来标识最后一条记录，该特殊的记录的名字叫 supremum pseudo-record（也可以理解为正无穷），所以还会加一个加next-key锁，即锁住了`(20, supremum pseudo-record]`，最后停止扫描。也就是说最后会创建三个X型的next-key锁。

  > 上图中Lock_Type是RECORD表示行锁。LOCK_MODE=X表示X型next-key锁
  >
  > 而且注意上图中还有一个IX意向锁，这是因为mysql需要从下锁到上，见3.2.1。

  当我们执行下面的SQL语句时

  ~~~mysql
  BEGIN;
  select * from test_table2 where a >= 8 for update;
  ~~~

  我们再查看一下当前的锁：

  ![image-20230414163650061](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414163650061.png)

  我们可以看到这次如果条件值的记录存在于表中（即a=8），所以该记录的索引中的 next-key 锁会**退化成记录锁**。后续过程与上面是一样的。

  > 这里X,REC_NOT_GAP表示记录锁

- 情况二：针对小于或者小于等于的范围查询，要看条件值的记录是否存在于表中：

  - 当条件值的记录不在表中，那么不管是「小于」还是「小于等于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。

    我们还是依然用情况1的大于等于的例子来实验，我们执行以下语句：

    ~~~mysql
    BEGIN;
    select * from test_table2 where a <= 6 for update;
    ~~~

    然后查一下锁的情况：

    ![image-20230414164842115](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230414164842115.png)

    最开始查找的是1，所以会加一个next-key锁，即锁住了`(负无穷,1]`，因为是范围查找，所以然后找到第二个加一个next-key锁，即锁住了`(1,3]`，后面过程类似，直到找到了8，因为8不满足条件所以会退化成间隙锁，所以会加一个gap lock 即锁住了`(4,8)`,然后再扫下一个记录不满足条件所以终止扫描。

  - 当条件值的记录在表中，如果是小于条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁；如果「小于等于」条件的范围查询，扫描到终止范围查询的记录时，该记录的索引 next-key 锁不会退化成间隙锁。其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。

    这里可参照上面的例子，过程类似。

#### 非唯一索引等值查询

当我们用非唯一索引进行等值查询的时候，**因为存在两个索引，一个是主键索引，一个是非唯一索引（辅助索引），所以在加锁时，同时会对这两个索引都加锁，但是对主键索引加锁的时候，只有满足查询条件的记录才会对它们的主键索引加锁**。

针对非唯一索引等值查询时，查询的记录存不存在，加锁的规则也会不同：

- 当查询的记录不存在时，**扫描到第一条不符合条件的辅助索引记录，该辅助索引的 next-key 锁会退化成间隙锁。因为不存在满足查询条件的记录，所以不会对主键索引加锁**。

  举个例子，我们使用下面的数据集，数据如下：

  | a（主键） | b（辅助索引） |
  | --------- | ------------- |
  | 1         | 1             |
  | 3         | 2             |
  | 4         | 3             |
  | 5         | 5             |
  | 8         | 7             |
  | 20        | 10            |

  我们执行以下SQL：

  ~~~mysql
  BEGIN;
  select * from test_table2 where b = 6 for update;
  ~~~

  然后我们看一下锁`select * from performance_schema.data_locks`，如下图：

  ![image-20230417151959711](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230417151959711.png)

  我们可以看到当定位到第一条不符合查询条件的二级索引记录，即扫描到 b= 7，于是该辅助索引的 next-key 锁会退化成间隙锁，范围是`(5, 7)`。

- 当查询的记录存在时，由于不是唯一索引，所以肯定存在索引值相同的记录，于是**非唯一索引等值查询的过程是一个扫描的过程，直到扫描到第一个不符合条件的辅助索引记录就停止扫描，然后在扫描的过程中，对扫描到的辅助索引记录加的是 next-key 锁，而对于第一个不符合条件的辅助索引记录，该辅助索引的 next-key 锁会退化成间隙锁。同时，在符合查询条件的记录的主键索引上加记录锁**。

  我们执行以下语句：

  ~~~mysql
  BEGIN;
  select * from test_table2 where b = 5 for update;
  ~~~

  然后我们看一下锁`select * from performance_schema.data_locks`，如下图：

  ![image-20230417133657994](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230417133657994.png)

  由于b不是唯一索引，所以事务对辅助索引字段b，加上了一个`(3,5]`的next-key锁，同时对之后扫描到的第一条不符合的记录加上了一个`(5,7)`的间隙锁。同时还对主键字段a加上了一个`a=5`的记录锁。**注意这些锁锁的都是索引。**

> 注意，我们在上面的例子中的`LOCK_DATA`里会发现当锁是间隙锁或者netx key锁时，`LOCK_DATA`显示的是两个值，那 `LOCK_DATA：7，8` 是什么意思？
>
> - LOCK_DATA 第一个数值，也就是 7， 它代表的是 b 字段的值，LOCK_DATA 第一个数值是 next-key 锁和间隙锁**锁住的范围的右边界值**。
> - LOCK_DATA 第二个数值，也就是 8， 它代表的是 a 字段的值。之所以 LOCK_DATA 要多显示一个数值，是因为针对当某个事务持有非唯一索引的 (5, 7) 间隙锁的时候，其他事务是否可以插入 b= 7 新数据的问题，还需要考虑插入记录的 id 值。而 LOCK_DATA 的第二个数值，就是说明在插入 b= 7 新数据时，哪些范围的 id 值是不可以插入的。
>
> 因此， `LOCK_DATA：7，8` + `LOCK_MODE : X, GAP` 的意思是，一个事务在 b=7 记录的二级索引上（`INDEX_NAME: key_b`），加了 b值范围为` (5, 7) `的 X 型间隙锁，同时针对其他事务插入 b 值为 7 的新记录时，不允许插入的新记录的 id 值小于 20 。如果插入的新记录的 id 值大于 20，则可以插入成功。
>
> 但是我们无法从`select * from performance_schema.data_locks;` 输出的结果分析出在插入 b=5 新记录时，哪些范围的 id 值是可以插入成功的，这时候就得自己画出二级索引的 B+ 树的结构（辅助索引树是按照辅助索引值（b列）顺序存放的，在相同的辅助索引值情况下， 再按主键 id （这里是a列）的顺序存放。），然后确定插入位置后，看下该位置的下一条记录是否存在间隙锁，如果存在间隙锁，则无法插入成功，如果不存在间隙锁，则可以插入成功。

#### 非唯一索引范围查询

非唯一索引和主键索引的范围查询的加锁也有所不同，不同之处在于**非唯一索引范围查询，索引的 next-key lock 不会有退化为间隙锁和记录锁的情况**，也就是非唯一索引进行范围查询时，对二级索引记录加锁都是加 next-key 锁。

举个例子，我们使用下面的数据集，数据如下：

| a（主键） | b（辅助索引） |
| --------- | ------------- |
| 1         | 1             |
| 3         | 2             |
| 4         | 3             |
| 5         | 5             |
| 8         | 7             |
| 20        | 10            |

执行以下SQL：

~~~mysql
BEGIN;
select * from test_table2 where b >= 4 for update;
~~~

然后查看锁情况：

![image-20230417154101290](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230417154101290.png)

我们可以看到对于b列的索引`key_b`这里加的全部都是next key锁，并不会退化成间隙锁和记录锁。当然对于查到的b列对应的主键会加上X型记录锁。在 b >= 4的范围查询中， b = 5 的记录存在并且属于等值查询，为什么不会像唯一索引那样，将 b= 5记录的二级索引上的 next-key 锁退化为记录锁？这是因为 b列字段是非唯一索引，不具有唯一性，所以如果只加记录锁（记录锁无法防止插入，只能防止删除或者修改），就会导致其他事务插入一条b=4 的记录，这样前后两次查询的结果集就不相同了，出现了幻读现象。

#### 没有加索引的查询

如果没有使用索引列作为查询条件，或者查询语句没有走索引查询，导致扫描是全表扫描。那么，每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表，这时如果其他事务对该表进行增、删、改操作的时候，都会被阻塞。

不只是锁定读查询语句不加索引才会导致这种情况，update 和 delete 语句如果查询条件不加索引，那么由于扫描的方式是全表扫描，于是就会对每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表。

因此，在线上在执行 update、delete、select ... for update 等具有加锁性质的语句，一定要检查语句是否走了索引，如果是全表扫描的话，会对每一个索引加 next-key 锁，相当于把整个表锁住了。

我们举个例子，我们使用以下数据集（表名为test_table3）：

| a（主键） | c（没有索引） |
| --------- | ------------- |
| 1         | 1             |
| 3         | 2             |
| 4         | 3             |
| 5         | 5             |
| 8         | 7             |
| 20        | 10            |

执行SQL：

~~~mysql
BEGIN;
select * from t est_table3 where c >= 4 for update;
~~~

查看锁情况：

![image-20230417154824200](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230417154824200.png)

### 3.3.4 next key lock引起的死锁问题

> 此例子参考生产环境发生的问题，模拟高并发的全量更新场景

我们这里举个例子进行模拟，有下面一张表，数据和表结构如下：

~~~mysql
CREATE TABLE `test_deadlock`  (
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `type` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `age` int(11) NULL DEFAULT NULL,
  `version` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  PRIMARY KEY (`name`, `type`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

INSERT INTO `test_deadlock` VALUES ('TEST0', '1', 15, '7.0');
INSERT INTO `test_deadlock` VALUES ('TEST1', '1', 11, '1.0');
INSERT INTO `test_deadlock` VALUES ('TEST1', '2', 12, '2.0');
INSERT INTO `test_deadlock` VALUES ('TEST1', '3', 13, '3.0');
INSERT INTO `test_deadlock` VALUES ('TEST2', '2', 11, '1.0');
INSERT INTO `test_deadlock` VALUES ('TEST2', '3', 12, '2.0');
INSERT INTO `test_deadlock` VALUES ('TEST3', '1', 11, '1.0');
~~~

然后我开一个事务执行以下SQL：

~~~mysql
#事务1
begin
delete from test_deadlock where `name`='TEST1'
~~~

然后再新开一个事务模拟并发：

~~~mysql
#事务2
begin
delete from test_deadlock where `name`='TEST2'
~~~

这是我们可以看一下当前锁情况`SELECT * FROM performance_schema.data_locks`：

![image-20230418152802480](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418152802480.png)

然后事务1执行`insert into test_deadlock values("TEST1","4",NULL,NULL)`，这时会发现sql被阻塞。然后回到事务2执行`insert into test_deadlock values("TEST2","1",NULL,NULL)`，之后就会发生死锁，结果：

~~~
insert into test_deadlock values("TEST2","1",NULL,NULL)
> 1213 - Deadlock found when trying to get lock; try restarting transaction
> 时间: 0.032s
~~~

我们下面来分析一下为什么会死锁，首先事务1执行删除sql，删除name等于TEST1的数据，这时由于表的唯一索引是`(name,type)`，所以这时实际上执行的是非唯一索引的等值加锁规则（具体见3.3.3加锁规则），对于第一个不符合条件的辅助索引记录，该辅助索引的 next-key 锁会退化成间隙锁，所以事务1执行完删除语句后锁情况如下图：

![image-20230418153848283](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418153848283.png)

当事务2执行完删除语句后锁情况如下图（也可以见上面的SQL结果`SELECT * FROM performance_schema.data_locks`）：

![image-20230418155051883](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20230418155051883.png)

这时当我们事务1执行`insert into test_deadlock values("TEST1","4",NULL,NULL)`语句时，由于事务2存在`(TEST1-3,TEST2-2]`的next key lock，所以事务会阻塞，这时事务2再执行sql时，由于事务1存在`(TEST1-3,TEST2-2)`的间隙锁，所以事务2也会阻塞，造成死锁。

解决死锁问题可以有以下几种解决方案：

1. 优化查询：死锁通常是由于查询语句不当或索引不当而引起的。您可以尝试优化查询语句或重新设计索引，以减少死锁的可能性。

2. 减少事务时间：死锁通常是由于事务持有锁定时间过长而引起的。您可以尝试减少事务的持有时间，以减少死锁的可能性。

3. 调整隔离级别：死锁通常是由于并发事务之间的竞争而引起的。您可以尝试调整隔离级别，以减少并发事务之间的竞争，从而减少死锁的可能性。

4. 可以尝试减少事务中的索引扫描和范围查询的数量

由于数据重要性不高最终实际在业务中是通过减少事务时间解决的，即减少长事务，完成即提交事务解决地死锁。当然也可以通过优化表结构和索引解决，只是由于历史原因最终通过减少事务时间解决的。

# 参考资料

- 《Mysql技术内幕InnoDB存储引擎第二版》

- 《高性能Mysql第三版》

- [MYSQL是如何加锁的](https://www.xiaolincoding.com/mysql/lock/how_to_lock.html#%E4%BB%80%E4%B9%88-sql-%E8%AF%AD%E5%8F%A5%E4%BC%9A%E5%8A%A0%E8%A1%8C%E7%BA%A7%E9%94%811)

  > 《Mysql技术内幕InnoDB存储引擎第二版》一书中虽然讲解了加锁规则，但是没有系统性讲解，这里可以结合书中的知识配上这篇博客，可以对知识梳理的更清晰。
