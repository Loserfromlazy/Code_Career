# Mysql学习笔记

## 一. mysql简介

**sql分类**：

1. DDL（Data Definition Language）数据定义语言，用来定义数据库对象：数据库，表，列等。create，drop，alter等。
2. DML(Data Manipulation Language)数据操作语言用来对数据库中表的数据进行增删改。insert, delete, update 等。
3. DQL(Data Query Language)数据查询语言用来查询数据库中表的记录(数据)。select, where 等。
4. DCL(Data Control Language)数据控制语言(了解)用来定义数据库的访问权限和安全级别，及创建用户。grant，revoke 等。

**mysql基本操作：**

```sql
启动mysql服务:
net start mysql
连接sql服务
mysql -h 地址 -p 端口 -u 用户名 -p 密码
Enter password:******
退出：
mysql>exit
Bye
```

## 二. Mysql-DDL

### **数据库：**

查看所有数据库

> SHOW DATABASES;

选择数据库

> USE xxx（数据库名）

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_data/mysqlddl1.jpg)

查看当前数据库：

> SELECT DATABASE();

查看当前时间、用户名、数据库版本

> SELECT now(); SELECT  user(); SELECT  version();



创建数据库

> create database 数据库名;
>
> create database if not exists 数据库名称;
>
> create database 数据库名 character set 字符集名;

修改数据库

> 修改数据库编码
>
> alter database 数据库名称 character set 字符集;

删除数据库

> drop database 数据库名称;
>
> drop database if exists 数据库名称;

### **表：**

创建表

> ```sql
> create [TEMPORARY] TABLE [IF NOT EXITS] [库名.]表名 （表的结构定义）[表选项]
> 
> 每个字段必须有数据类型
> 最后一个字段不能有逗号
> TEMPORARY 临时表，会话结束时表自动消失
> 对字段的定义：
> 字段名 数据类型 [NOT NULL | NULL] [DEFAULT default_value] [AUTO_INCREMENT] [UNIQUE [KEY] | [PRIMARY] KEY] [COMMENT '备注']
> ```
>
> 表选项有：
>
> - ENGINE= engine_name (引擎名，常用引擎InnoDB、MyISAM)
> - CHARSET = charset_name(字符集，未设置使用数据库字符集)
> - AUTO_INCREMENT = 行数
> - COMMENT = ‘表注释’
> - DATA DIRECTORY ='数据文件目录'
> - INDEX DIRECTORY ='索引文件目录'
>
> 例子：
>
> ```sql
> CREATE TABLE `tb_ad` (
> `id` int(11) NOT NULL AUTO_INCREMENT COMMENT 'ID',
> `name` varchar(50) DEFAULT NULL COMMENT '广告名称',
> `position` varchar(50) DEFAULT NULL COMMENT '广告位置',
> `start_time` datetime DEFAULT NULL COMMENT '开始时间',
> `end_time` datetime DEFAULT NULL COMMENT '到期时间',
> `status` char(1) DEFAULT NULL COMMENT '状态',
> `image` varchar(100) DEFAULT NULL COMMENT '图片地址',
> `url` varchar(100) DEFAULT NULL COMMENT 'URL',
> `remarks` varchar(1000) DEFAULT NULL COMMENT '备注',
> PRIMARY KEY (`id`)
> ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8
> ```

删除表

> drop table  customer;

修改表

> 重命名：
>
> RENAME TABLE 原表名 TO 新表名
>
> 添加单列：
>
> alter table 表名 add 列名 字段类型 AFTER 字段名/FIRST;（在xx字段后加入或在第一个加入）
>
> 添加多列：
>
> alter table 表名 add 列名1 字段类型1,add 列名2 字段类型2...
>
> 例子：
>
> ```sql
> alter table customer add name varchar(20);
> ```

查看当前数据库中的所有表

> SHOW TABLES;

查看指定表的结构(建表语句)

> SHOW CREATE TABLE 指定表;

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_data/mysqlddl2.jpg)

查看指定表的结构

> DESC 表名;

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/java_data/mysqlddl3.jpg)

复制表结构

> CREATE TABLE 表名 LIKE 要复制的表名
>
> 复制表结构和数据
>
> CREATE TABLE 表名 LIKE 要复制的表名 AS SELECT * FROM 要复制的表名

修改列

> 修改类型和列名
>
> ALTER TABLE 表名 CHANGE 原列名 新列名 新类型;
>
> 只修改类型 不修改列名
>
> ALTER TABLE 表名 MODIFY 原列名 新类型;

修改表名称

> alter table 表名 rename 新表名;

删除列

> alter table 表名 drop 列名;

## 三. Mysql-DML

### 增加数据

> insert into 表名（列名1，列名2，。。。列名n）values（值1，值2，。。。值n）
>
> insert into 表名 set 字段名=值
>
> 列名和值要一一对应。如果表名后，不定义列名，则默认给所有列添加值insert into 表名 values(值1,值2,...值n)。除了数字类型，其他类型需要使用引号(单双都可以)引起来。
>
> 例子：INSERT INTO usertable (id,username,sex) VALUES (5,'da','男')
>
> INSERT INTO usertable SET id=99,username='dadda'

### 删除数据

> delete from 表名 条件
>
> 若没有条件则全部删除
>
> 例子：
>
> DELETE FROM usertable WHERE id=99

### 修改数据

> UPDATE 表名 SET 字段名=新值[, 字段名=新值] [更新条件]
>
> 例子：
>
> UPDATE usertable SET username='daxiao',sex='女' WHERE id=1

## 四. Mysql-DQL

### select

SELECT [ALL|DISTINCT] select_expr FROM-> WHERE ->GROUP BY [合计函数] ->HAVING ->ORDER BY ->LIMIT

1. select_expr 
   - 可以使用 * 表示所有字段：select * from tb;
   - 可以使用表达式（计算公式、函数调用、字段也是表达式）：select stu, 25+35, now() from tb
   - 可以为每列使用别名，可以使用或省略as：select stu+10 as add10 from tb
2. form 
   - 可以为表起别名：select * from tb as t,tb1 as b;
   - 多个表横向叠加，数据形成笛卡尔积：select * from tb1,tb2
3. where
   - 表达式由运算符和运算数组成。
   - 运算数：字段、值、函数返回值
   - 运算符：=等于,<>不等于,<=> 安全等于可用于null比较，<=小于等于,>=大于等于,>大于，<小于，（not）between and （不在）在...之间, （not） in （不）在集合中，like 模糊匹配,is (not) null (不)为空。
4. group by
   - group by 字段/别名 [排序方式]
   - asc升序，desc降序
   - 以下合计函数需配合group by使用：
     - count 返回数目:count(*)、count(字段)
     - sum：求和
     - max：求最大值
     - min：求最小值
     - avg：求平均值
   - 例子：`select name,COUNT(*) from tb1 group by name`
5. having 
   - 与where功能用法相同，执行时机不同
   - where是在开始时检测数据，对原数据进行过滤；having对筛选出来的结果进行过滤
   - having必须是查询出来的；where必须是数据表存在的
   - where不可以使用合计函数，having主要就是使用合计函数时使用的。
   - 例子：`select region,sum(population),sum()arer from tb group by region having sun(area)>10000`
6. order by
   - 排序 asc 升序，desc降序
   - 默认升序
7. limit 
   - 对结果进行数量限制，索引从0开始
   - 用法：limit 起始位置，获取条数
   - 或者limit 获取条数 则直接从0开始
8. distinct 去除重复记录

### union

用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中。

select ... union [all|distinct] select ...

默认distinct，建议对每一个select查询加上小括号

**每个select查询的字段列表(数量、类型)应一致，因为结果中的字段名以第一条select语句为准。**

例子：

```sql
select country from websites
union 
select country from apps
order by country;
```

查询websites表和apps表中所有不同的country

### 子查询

#### form形式

把内层的查询结果供外层再次查询

form后面是一个表，必须给子查询取别名

form型需将结果生成临时表格，子查询返回一个表，**表型子查询**

eg：

```sql
select * from (select * from tb where id>0) as t where id>1;
```

#### where形式

把内层查询结果当作外层查询的比较条件

返回的结果集是一个标量集合，一行一列，也就是一个标量值，**标量子查询**

> 从定义上讲，每个标量子查询也是一个行子查询和一个列子查询，反之则不是；每个行子查询和列子查询也是一个表子查询，反之也不是。

不需要起别名

```sql
select * from tb where money =(select max(money) from tb); 
```

子查询返回的结果是一列，**列子查询**

使用in、not in、exists、not exists条件

eg：

```sql
select * from tb1 
where exists(
    select * from tb2 where tb1.id = tb2.id
);
select * from tb1 
where tb1.id in (
    select id from tb2
);
```

查询条件是一个行,**行子查询**

```sql
select * from tb1 
where (id,gender) in (
    select id,gender from tb2
);

```



### 连接查询

将多个表的字段进行连接，可以指定连接条件。三张图片来自菜鸟教程

#### 内连接 inner join

获取两个表中字段匹配关系的记录

默认，可省略inner，连接结果不能出现空行，on表示连接条件，与where类似，using也可用于连接条件

![img](https://www.runoob.com/wp-content/uploads/2014/03/img_innerjoin.gif)

eg：

```sql
select * from tb1 a inner join tb2 b on a.id=b.id
等价于
select * from tb1,tb2 where tb1.id =tb2.id
改为using
select * from tb1 as a join tb2 b using(id)

```

交叉连接，

#### 外连接

如果数据不存在，也会出现在连接结果中。
    -- 左外连接 left join
        如果数据不存在，左表记录会出现，而右表为null填充

![img](https://www.runoob.com/wp-content/uploads/2014/03/img_leftjoin.gif)   

 -- 右外连接 right join
        如果数据不存在，右表记录会出现，而左表为null填充

![img](https://www.runoob.com/wp-content/uploads/2014/03/img_rightjoin.gif)

eg：

```sql
左外连接
select * from tb1 a left join tb2 b on a.id =b.id
右外连接
select * from tb1 a left join tb2 b on a.id =b.id

```



### truncate

用法：truncate [table] tb_name

清空表中数据

区别：

1 truncate 是删除表再创建，delete 是逐条删除
2 truncate 重置auto_increment的值。而delete不会
3 当被用于带分区的表时，truncate 会保留分区

## 五、索引

索引是数据库用来提高性能的最常用工具，是帮助mysql搞笑获取数据的数据结构。

### 概述

**使用索引的原因**

1. 通过创建唯一性索引，可以保证数据库表中的每一行数据的唯一性。
2. 可以大大加快数据的检索速度，这也是创建索引的最主要的原因。
3. 帮助服务器避免排序和临时表
4. 将随机IO变为顺序IO
5. 可以加速表之间的连接，特别是在实现数据的参考完整性方面有意义。

**为什么不对表中的每一个列创建一个索引**

1. 当对表中的数据进行增删改时，索引也需要维护，这样就降低了数据的维护速度。
2. 索引需要占用物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大。
3. 创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加。

**使用索引的注意事项**

1. 在经常需要搜索的列上，可以加快搜索的速度；
2. 在经常使用在WHERE子句中的列上面创建索引，加快条件的判断速度。
3. 在经常需要排序的列上创 建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间；
4. 对于中到大型表索引都是非常有效的，但是特大型表的话维护开销会很大，不适合建索引
5. 在经常用在连接的列上，这 些列主要是一些外键，可以加快连接的速度；
6. 避免 where 子句中对字段施加函数，这会造成无法命中索引。

### 索引提高查询速度的原因

> 本小节图片来自于Java3y，地址 https://juejin.im/post/5b55b842f265da0f9e589e79

首先了解mysql的基本存储结构（记录都存在页里面）：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_cunchu.jpg)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_cunchu2.jpg)

- 各个数据也可以组成双向链表
- 每个数据也中的记录又可以组成单向链表
  - 每个数据页都会为存储在它里面的记录生成一个页目录，在通过主键查找某条记录的时候可以在页目录中使用二分法快速定位到对应的槽，然后在遍历该槽对应分组中的记录即可快速找到指定的记录。
  - 以其他列（非主键）作为搜索条件：只能从最小记录开始一次遍历单链表中的每条记录。

所以，在使用`select * from user where indexname ='xxx'`这样没有进行任何优化的sql渔具，默认会这样做：

1. 定位到记录所在的页：需要遍历双向链表，找到所在的页
2. 从所在的页内中查找相应的记录：由于不是根据主键查询，只能遍历所在页的单链表

这样的时间复杂度为O(n)

**使用索引之后**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_suoyin1.jpg)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_suoyin2.jpg)

没有用索引是需要遍历双向链表来定位对应的页，现在通过"目录"就可以很快的定位到对应的页上。(二分查找，时间复杂度近似O(logn))

它的底层结构就是B+树。

### mysql索引数据结构

**哈希索引**

对于哈希索引来说，底层的数据结构就是哈希表，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；其余大部分场景，建议选择BTree索引。哈希索引只有Memory引擎支持。

**BTree索引**

最常见的索引类型，大部分都支持B树索引。

**R-tree 索引**（空间索引）：

空间索引是MyISAM引擎的一个特殊索引类型，主要用于地理空间数据类型，通常使用较少，不做特别介绍。

**Full-text** （全文索引） ：

全文索引也是MyISAM的一个特殊索引类型，主要用于全文索引，InnoDB从Mysql5.6版本开始支持全文索引。

*平常所说的索引，如果没有特别指明，都是指B+树（多路搜索树，并不一定是二叉的）结构组织的索引。其中聚集索引、复合索引、前缀索引、唯一索引默认都是使用 B+tree 索引，统称为 索引。*

### 索引分类

**普通索引**

这是一种最基本的索引类型，而且他没有唯一性之类的限制。普通索引可以通过以下方式创建：

创建索引：

~~~sql
create index indexName ON tableName (columnName)
~~~

修改表：

~~~sql
alter table tablename ADD INDEX indexName(columnName) 
~~~

创建表时指定：

~~~sql
create table tablename(
id int not null,
    name verchar(10) not null,
    index [indexname](name(length))
);
~~~

**唯一性索引**

和普通索引基本相同，但唯一性索引列的所有值都只能出现一次，即必须唯一。唯一性索引可以通过以下方式创建：

创建索引：

```mysql
create unique index indexName ON tableName (columnName)
```

修改表：

```mysql
alter table tablename ADD unique INDEX indexName(columnName) 
```

创建表时指定：

```mysql
create table tablename(
id int not null,
    name verchar(10) not null,
    unique [indexname](name(length))
);
```

**主键**

主键是一种唯一性索引，但它必须指定为“PRIMARY KEY”。如果你曾经用过AUTO_INCREMENT类型的列，你可能已经熟悉主键之类的概念了。

**全文索引**

全文索引的索引类型为FULLTEXT。全文索引可以在VARCHAR或者TEXT类型的列上创建。

~~~mysql
alter table tablename add fulltext indexname(columnname)
~~~

### 索引语法

**创建索引**

见上

**删除索引**

~~~mysql
drop index indexname on mytable
alter table tablename drop index indexname
~~~

删除主键时只需指定PRIMARY KEY，但在删除索引时，你必须知道索引名。

**显示索引信息**

~~~mysql
show index from tablename;
~~~















