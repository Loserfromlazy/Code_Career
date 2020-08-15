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

## 五. Mysql-用户和权限管理



## 六、备份和还原

## 七、事务

## 八、触发器

## 九、索引





