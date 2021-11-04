# MySql性能优化

数据库优化维度有四个：

硬件升级、系统配置、表结构设计、sql语句及索引。

![image-20211104183227667](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211104183227667.png)

# 一、系统配置优化

## 1.1 保证从内存中读取数据

MySQL会在内存中保存一定的数据，通过LRU算法将不常访问的数据保存在硬盘中。所以优化方向是尽可能扩大内存中的数据量，将数据保存在内存中，可以提升Mysql性能。通过扩大`innode_buffer_pool_size`,能够最大限度降低磁盘操作。

通过`show global status like 'innode_buffer_pool_pages_%';`查看`innode_buffer_pool_size`大小。以docker的mysql8.0.18为例：

![image-20211104183920345](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211104183920345.png)

如果`Innodb_buffer_pool_pages_free`为0则表示已经用光。

`innode_buffer_pool_size`理论上可以扩大到内存的3/4或4/5

修改my.cnf

`innode_buffer_pool_size = 750M`进行修改

## 1.2 数据预热

默认情况，仅仅有某条数据被读取一次，才会缓存在 innodb_buffer_pool。所以，数据库刚刚启动，须要进行数据预热，将磁盘上的全部数据缓存到内存中。数据预热能够提高读取速度。

mysql数据预热脚本

~~~mysql
SELECT DISTINCT
    CONCAT('SELECT ',rowlist,' FROM ',db,'.',tb,
    ' ORDER BY ',rowlist,';') selectSql
    FROM
    (
        SELECT
            engine,table_schema db,table_name tb,
            index_name,GROUP_CONCAT(column_name ORDER BY seq_in_index) rowlist
        FROM
        (
            SELECT
                B.engine,A.table_schema,A.table_name,
                A.index_name,A.column_name,A.seq_in_index
            FROM
                information_schema.statistics A INNER JOIN
                (
                    SELECT engine,table_schema,table_name
                    FROM information_schema.tables WHERE
                    engine='InnoDB'
                ) B USING (table_schema,table_name)
            WHERE B.table_schema NOT IN ('information_schema','mysql')
            ORDER BY table_schema,table_name,index_name,seq_in_index
        ) A
        GROUP BY table_schema,table_name,index_name
    ) AA
ORDER BY db,tb;
~~~

可以将脚本保存为`xxx.sql`

执行命令

~~~
mysql -urrot -p -AN < /rooot/xxx.sql > /root/xxx.sql
~~~

# 二、表结构设计优化

1. 设计中间表

   设计中间表，一般针对于统计分析功能或者实时性不高的需求

2. 设计冗余字段

   为减少关联查询，创建合理的**冗余字段（需注意数据一致性的问题）**

3. 拆表

   对于字段太多的大表，如有100多个字段

   对于表中经常不被使用的字段或者存储数据比较多的字段

4. 主键优化

   每张表建议一个主键（主键索引），而且最好是int类型，建议自增主键。分布式系统可以雪花算法。

5. 字段的设计

   数据库中的表越小，查询的速度越快，因此在创建表的时候，为了更好的性能，可以将表中的字段的宽度设的尽可能小。尽量把字段设计为NOTNULL，这样在执行查询时，数据库不用去比较Null值。

# 三、sql语句及索引优化

## 3.1 EXPLAIN执行计划













