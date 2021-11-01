# Mysql学习笔记

## 一. mysql简介

### 1.1 mysql基础

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

**mysql建表规范**

~~~
/* 建表规范 */ ------------------
    -- Normal Format, NF
        - 每个表保存一个实体信息
        - 每个具有一个ID字段作为主键
        - ID主键 + 原子表
    -- 1NF, 第一范式
        字段不能再分，就满足第一范式。通俗理解即一个字段只存储一项信息。
    -- 2NF, 第二范式
        满足第一范式的前提下，不能出现部分依赖。
        消除复合主键就可以避免部分依赖。增加单列关键字。
        第二范式（2NF）要求数据库表中的每个实例或行必须可以被惟一地区分。为实现区分通常需要我们设计一个主键来实现(这里的主键不包含业务逻辑)。
    -- 3NF, 第三范式
        满足第二范式的前提下，不能出现传递依赖。
        某个字段依赖于主键，而有其他字段依赖于该字段。这就是传递依赖。
        将一个实体信息的数据放在一个表内实现。
        简而言之，第三范式（3NF）要求一个数据库表中不包含已在其它表中已包含的非主键字段。就是说，表的信息，如果能够被推导出来，就不应该单独的设计一个字段来存放(能尽量外键join就用外键join)。很多时候，我们为了满足第三范式往往会把一张表分成多张表。
~~~

### 1.2 mysql架构演变

**单机单库**

小型网站，只需一个mysql就能满足数据读取和写入需求。

瓶颈：

- 数据量不能太大，超出一台服务器承受
- 读写操作量不能太大，超出一台服务器承受
- 可用性差

**主从架构**

主要解决单机单库下的高可用和读扩展问题，主库宕机可以通过主从切换保障高可用。mysql也可以通过主从结构完成读写分离即主库用来写，从库用来读。

瓶颈：

- 数据量不能过大，超出一台服务器承受
- 写操作不能太大，超出一台服务器承受

**分库分表**

对于存储和写瓶颈，可以通过水平拆分来解决，水平和垂直拆分有较大区别，垂直拆分的结果是每一个实例都有全部数据，而水平拆分之后，任何实例都只有全量的1/n。

### 1.3 mysql架构

![image-20211030152239007](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211030152239007.png)

MySQL Server架构自顶向下大致可以分网络连接层、服务层、存储引擎层和系统文件层。

1. 网络连接层

   客户端连接器（Client Connectors）：提供与MySQL服务器建立的支持。目前几乎支持所有主流的服务端编程技术，例如常见的 Java、C、Python、.NET等，它们通过各自API技术与MySQL建立连接

2. 服务层

   服务层是MySQL Server的核心，主要包含系统管理和控制工具、连接池、SQL接口、解析器、查询优化器和缓存六个部分。

   1. 连接池：负责存储和管理客户端与数据库的连接，一个线程负责管理一个连接

   2. 系统管理和控制工具：例如备份恢复、安全管理、集群管理

   3. SQL接口：用于接收客户端发送的各种SQL命令，并且返回用户需要查询的结果。比如DML、DDL、存储过程、视图、触发器等

   4. 解析器：负责将请求的SQL解析生成一个"解析树"。然后根据一些MySQL规则进一步检查解析树是否合法。

   5. 查询优化器:

      当“解析树”通过解析器语法检查后，将交由优化器将其转化成执行计划，然后与存储引擎交互

   6. 缓存:

      缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，权限缓存，引擎缓存等。如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据

3. 存储引擎层

   存储引擎负责MySQL中数据的存储与提取，与底层系统文件进行交互。MySQL存储引擎是插件式的，服务器中的查询执行引擎通过接口与存储引擎进行通信，接口屏蔽了不同存储引擎之间的差异 。现在有很多种存储引擎，各有各的特点，最常见的是MyISAM和InnoDB

4. 系统文件层

   该层负责将数据库的数据和日志存储在文件系统之上，并完成与存储引擎的交互，是文件的物理存储层。主要包含日志文件，数据文件，配置文件，pid 文件，socket 文件等

### 1.4 mysql运行机制

![mysql流程](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/mysql%E6%B5%81%E7%A8%8B.png)

建立连接（Connectors&Connection Pool），通过客户端/服务器通信协议与MySQL建立连接。MySQL 客户端与服务端的通信方式是 “ 半双工 ”。对于每一个 MySQL 的连接，时刻都有一个线程状态来标识这个连接正在做什么

查询缓存（Cache&Buffer）,这是Mysql一个可优化查询的地方，开启了查询缓存且在查询缓存过程中查询到完全相同的sql语句，则直接将查询结果返回给客户端，如果没有开启或没有查询到完全相同的sql语句则由解析器进行语法解析，并生成解析树。

解析器，将客户端发送的SQL进行语法解析，生成"解析树"。预处理器根据一些MySQL规则进一步检查“解析树”是否合法，例如这里将检查数据表和数据列是否存在，还会解析名字和别名，看看它们是否有歧义，最后生成新的“解析树

查询优化器：根据“解析树”生成最优的执行计划。MySQL使用很多优化策略生成最优的执行计划，可以分为两类：静态优化（编译时优化）、动态优化（运行时优化）

- 等价变化策略
  - `5=5 and a>5`改成`a>5`
  - `a<b and a=5`改成`b>5 and a=5`
  - 基于联合索引调整条件位置
- 优化count、min、max函数
- 提前终止查询
  - 使用limit，则获取所需的数据，不在遍历后面的数据

查询执行引擎，负责执行 SQL 语句，此时查询执行引擎会根据 SQL 语句中表的存储引擎类型，以及对应的API接口与底层存储引擎缓存或者物理文件的交互，得到查询结果并返回给客户端。若开启用查询缓存，这时会将SQL 语句和结果完整地保存到查询缓存（Cache&Buffffer）中，以后若有相同的 SQL 语句执行则直接返回结果。

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

## 五、mysql-对用户的操作

~~~mysql
-- 刷新权限
FLUSH PRIVILEGES;
-- 增加用户
CREATE USER 用户名 IDENTIFIED BY [PASSWORD] 密码(字符串)
	必须拥有mysql数据库的全局CREATE USER权限，或拥有INSERT权限。
	用户名，注意引号：如 'user_name'@'192.168.1.1'可以用%代替ip地址表示全部ip
-- 重命名用户
RENAME USER old_user TO new_user
-- 分配权限/添加用户
GRANT 权限列表 ON 表名 TO 用户名 [IDENTIFIED BY [PASSWORD] 'password']
     all privileges 表示所有权限
     *.* 表示所有库的所有表
     库名.表名 表示某库下面的某表
    GRANT ALL PRIVILEGES ON `pms`.* TO 'pms'@'%' IDENTIFIED BY 'pms0817';
-- 撤消权限
REVOKE 权限列表 ON 表名 FROM 用户名
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 用户名   -- 撤销所有权限
-- 查看权限
SHOW GRANTS FOR 用户名 
-- 权限列表
ALL [PRIVILEGES]    -- 设置除GRANT OPTION之外的所有简单权限
ALTER   -- 允许使用ALTER TABLE
ALTER ROUTINE   -- 更改或取消已存储的子程序
CREATE  -- 允许使用CREATE TABLE
CREATE ROUTINE  -- 创建已存储的子程序
CREATE TEMPORARY TABLES     -- 允许使用CREATE TEMPORARY TABLE
CREATE USER     -- 允许使用CREATE USER, DROP USER, RENAME USER和REVOKE ALL PRIVILEGES。
CREATE VIEW     -- 允许使用CREATE VIEW
DELETE  -- 允许使用DELETE
DROP    -- 允许使用DROP TABLE
EXECUTE     -- 允许用户运行已存储的子程序
FILE    -- 允许使用SELECT...INTO OUTFILE和LOAD DATA INFILE
INDEX   -- 允许使用CREATE INDEX和DROP INDEX
INSERT  -- 允许使用INSERT
LOCK TABLES     -- 允许对您拥有SELECT权限的表使用LOCK TABLES
PROCESS     -- 允许使用SHOW FULL PROCESSLIST
REFERENCES  -- 未被实施
RELOAD  -- 允许使用FLUSH
REPLICATION CLIENT  -- 允许用户询问从属服务器或主服务器的地址
REPLICATION SLAVE   -- 用于复制型从属服务器（从主服务器中读取二进制日志事件）
SELECT  -- 允许使用SELECT
SHOW DATABASES  -- 显示所有数据库
SHOW VIEW   -- 允许使用SHOW CREATE VIEW
SHUTDOWN    -- 允许使用mysqladmin shutdown
SUPER   -- 允许使用CHANGE MASTER, KILL, PURGE MASTER LOGS和SET GLOBAL语句，mysqladmin debug命令；允许您连接（一次），即使已达到max_connections。
UPDATE  -- 允许使用UPDATE
USAGE   -- “无权限”的同义词
GRANT OPTION    -- 允许授予权限
~~~

## 六、索引

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

### 索引提高查询速度和降低增删改速度的原因

> 本小节**索引提高查询速度和降低增删改速度的原因**图片和内容来自于Java3y大佬，地址 https://juejin.im/post/5b55b842f265da0f9e589e79

首先了解mysql的基本存储结构（记录都存在页里面）：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_cunchu.jpg)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_cunchu2.jpg)

- 各个数据也可以组成双向链表
- 每个数据也中的记录又可以组成单向链表
  - 每个数据页都会为存储在它里面的记录生成一个页目录，在通过主键查找某条记录的时候可以在页目录中使用二分法快速定位到对应的槽，然后在遍历该槽对应分组中的记录即可快速找到指定的记录。
  - 以其他列（非主键）作为搜索条件：只能从最小记录开始一次遍历单链表中的每条记录。

所以，在使用`select * from user where indexname ='xxx'`这样没有进行任何优化的sql语句，默认会这样做：

1. 定位到记录所在的页：需要遍历双向链表，找到所在的页
2. 从所在的页内中查找相应的记录：由于不是根据主键查询，只能遍历所在页的单链表

这样的时间复杂度为O(n)

**使用索引之后**

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_suoyin1.jpg)

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/mysql/mysql_suoyin2.jpg)

没有用索引是需要遍历双向链表来定位对应的页，现在通过"目录"就可以很快的定位到对应的页上。(二分查找，时间复杂度近似O(logn))

它的底层结构就是B+树。

但是如果一棵普通的树在**极端**的情况下，是能**退化成链表**的(树的优点就不复存在了)

B+树是平衡树的一种，是不会退化成链表的，树的高度都是相对比较低的(基本符合**矮矮胖胖(均衡)的结构**)【这样一来我们检索的时间复杂度就是O(logn)】！从上一节的图我们也可以看见，建立索引实际上就是建立一颗B+树。

- B+树是一颗平衡树，如果我们对这颗树增删改的话，那肯定会**破坏它的原有结构**。
- **要维持平衡树，就必须做额外的工作**。正因为这些额外的工作**开销**，导致索引会降低增删改的速度

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

### 索引原理

MySQL官方对索引定义：是存储引擎用于快速查找记录的一种数据结构。需要额外开辟空间和数据维护工作。

索引是物理数据页存储，在数据文件中（InnoDB，ibd文件）利用数据页存储。索引可以加快检索速度，但是同时也会降低增删改速度，索引维护需要代价。

索引设计的理论知识：二分查找、Hash和B+Tree

二分查找和Hash可以看我的另一篇博客

[Java数据结构](https://www.cnblogs.com/yhr520/p/12593578.html)

如果博客园最近有问题所以查看不了则可以看我的GitHub[Java数据结构](https://github.com/Loserfromlazy/Code_Career/blob/master/java/Java%E7%AE%97%E6%B3%95%E4%B8%8E%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.md)

**B+Tree**:

MySQL数据库索引采用的是B+Tree结构，在B-Tree结构上做了优化改造。

B-Tree结构：

- 索引值和data数据分布在整颗树的结构中
- 每个节点可以存放多个索引值以及对应的data数据
- 树节点中的多个索引值从左到右升序排列

![BTREE20211101](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/BTREE20211101.png)

B树的搜索：从根节点开始，对节点内的索引值序列采用二分法查找，如果命中就结束查找。没有命中会进入子节点重复查找过程，直到所对应的的节点指针为空，或已经是叶子节点了才结束。

B+Tree结构：

- 非叶子节点不存储data数据，只存储索引值，这样便于存储更多的索引值
- 叶子节点包含了所有的索引值和data数据
- 叶子节点用指针连接，提高区间的访问性能

![BPLUSTREE20211101](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/BPLUSTREE20211101.png)

相比B树，B+树进行范围查找时，只需要查找定位两个节点的索引值，然后利用叶子节点的指针进行遍历即可。而B树需要遍历范围内所有的节点和数据，显然B+Tree效率高

**聚簇索引和辅助索引**

聚簇索引和非聚簇索引：B+Tree的叶子节点存放主键索引值和行记录就属于聚簇索引；如果索引值和行记录分开存放就属于非聚簇索引。

主键索引和辅助索引：B+Tree的叶子节点存放的是主键字段值就属于主键索引；如果存放的是非主键值就属于辅助索引（二级索引）。

在InnoDB引擎中，主键索引采用的就是聚簇索引结构存储。

- 聚簇索引（聚集索引）

  聚簇索引是一种数据存储方式，InnoDB的聚簇索引就是按照主键顺序构建 B+Tree结构。B+Tree的叶子节点就是行记录，行记录和主键值紧凑地存储在一起。 这也意味着 InnoDB 的主键索引就是数据表本身，它按主键顺序存放了整张表的数据，占用的空间就是整个表数据量的大小。通常说的**主键索引**就是聚集索引。

  InnoDB的表要求必须要有聚簇索引：

  > 如果表定义了主键，则主键索引就是聚簇索引
  >
  > 如果表没有定义主键，则第一个非空unique列作为聚簇索引
  >
  > 否则InnoDB会从建一个隐藏的row-id作为聚簇索引

- 辅助索引

  InnoDB辅助索引，也叫作二级索引，是根据索引列构建 B+Tree结构。但在 B+Tree 的叶子节点中只存了索引列和主键的信息。二级索引占用的空间会比聚簇索引小很多， 通常创建辅助索引就是为了提升查询效率。一个表InnoDB只能创建一个聚簇索引，但可以创建多个辅助索引

![image-20211101193110408](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211101193110408.png)

非聚簇索引

与InnoDB表存储不同，MyISAM数据表的索引文件和数据文件是分开的，被称为非聚簇索引结构。

