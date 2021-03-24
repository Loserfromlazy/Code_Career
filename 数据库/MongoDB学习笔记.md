# MongoDB学习笔记

# 一、概述

MongoDB是一个跨平台的，面向文档的数据库，是当前NoSql数据库热门中的一种。他介于关系和非关系数据库之间。它支持的数据结构非常松散，是类似JSON的BSON的格式，因此可以存储比较复杂的数据类型。

**特点**

1. 面向集合存储，已与存储对象类型的数据
2. 模式自由
3. 支持动态查询
4. 支持完全索引，包含内部对象
5. 支持复制和故障修复
6. 使用高效的二进制数据存储，包括大型对象
7. 自动处理碎片，以支持云计算层次的拓展性
8. 支持python、php、ruby、java、c、C#、js、c++、perl
9. 文件存储格式为BSON

**体系结构**

主要有文档、集合、数据库三部分组成。逻辑结构是面向用户的，用户使用mongoDB开发应用程序使用的是逻辑结构。

- MongoDB 的文档（document），相当于关系数据库中的一行记录。
- 多个文档组成一个集合（collection），相当于关系数据库的表。
- 多个集合（collection），逻辑上组织在一起，就是数据库（database）。
- 一个MongoDB 实例支持多个数据库（database）。

下表是mongoDB与MySQL数据库逻辑结构概念的对比

| mongoDB | mysql  |
| ------- | ------ |
| 数据库  | 数据库 |
| 集合    | 表     |
| 文档    | 行     |

**数据类型**

基本数据类型

null：用于表示空值或者不存在的字段{"x":"null"}

布尔型：有true和false{"x":true}

数值：shell默认使用64位浮点型数据{"x":3.14}或{"x":3}.对于整形，可以使用NumberInt（4字节符号整数）或NumberLong（8字节符号整数）。{"x":NumberInt("3")}{"x":NumberLong("3")}

字符串：UTF-8字符串都可以表示为字符串数据类型{"x":"哈哈"}

日期：日期被存储为自新纪元依赖经过的毫秒数，不存储时区，{“x”:new Date()}

正则表达式：查询时，使用正则表达式作为限定条件，语法与JavaScript的正则表达式相同，{“x”:/[abc]/}

数组：数据列表或数据集可以表示为数组，{“x”：[“a“，“b”,”c”]}

内嵌文档：文档可以嵌套其他文档，被嵌套的文档作为值来处理，{“x”:{“y”:3 }}

对象Id：对象id是一个12字节的字符串，是文档的唯一标识，{“x”: objectId() }

代码：查询和文档中可以包括任何JavaScript代码，{“x”:function(){/…/}}

# 二、快速入门

安装略

运行mongodb服务器必须从 MongoDB 目录的 bin 目录中执行 mongod.exe 文件

```
C:\mongodb\bin\mongod --dbpath c:\data\db
```

我们可以在命令窗口中运行 mongo.exe 命令即可连接上 MongoDB

```
C:\mongodb\bin\mongo.exe
```

## **创建和选择数据库**

`use 数据库名称`

如果不存在则创建

eg：创建spit数据库

~~~
use spitdb
~~~

> 查看所有数据库，可以使用 **show dbs** 
>
> 可以使用 db 命令查看当前数据库名

## **删除当前数据库**

~~~
db.dropDatabase()
~~~

## **创建和删除集合**

MongoDB 中使用 **createCollection()** 方法来创建集合。

```
db.createCollection(name, options)
```

参数说明：

- name: 要创建的集合名称
- options: 可选参数, 指定有关内存大小及索引的选项

options 可以是如下参数：

| 字段        | 类型 | 描述                                                         |
| :---------- | :--- | :----------------------------------------------------------- |
| capped      | 布尔 | （可选）如果为 true，则创建固定集合。固定集合是指有着固定大小的集合，当达到最大值时，它会自动覆盖最早的文档。 **当该值为 true 时，必须指定 size 参数。** |
| autoIndexId | 布尔 | 3.2 之后不再支持该参数。（可选）如为 true，自动在 _id 字段创建索引。默认为 false。 |
| size        | 数值 | （可选）为固定集合指定一个最大值，即字节数。 **如果 capped 为 true，也需要指定该字段。** |
| max         | 数值 | （可选）指定固定集合中包含文档的最大数量。                   |

在插入文档时，MongoDB 首先检查固定集合的 size 字段，然后检查 max 字段。

> 如果要查看已有集合，可以使用 **show collections** 或 **show tables** 命令

MongoDB 中使用 drop() 方法来删除集合。

**语法格式：**

```
db.collection.drop()
```

## 文档操作

**插入文档**

~~~
db.集合名称.insert(数据)
~~~

eg：

~~~
db.col.insert({content:"xxxxxxx"},userid:"1001",nickname:"张三",visits:NumberInt(900))
~~~

**查询文档**

~~~
db.col.find()
~~~

这里你会发现每条文档会有一个叫_id的字段，这个相当于我们原来关系数据库中表的主键，当你在插入文档记录时没有指定该字段，MongoDB会自动创建，其类型是ObjectID类型。如果我们在插入文档记录时指定该字段也可以，其类型可以是ObjectID类型，也可以是MongoDB支持的任意类型。

如果想按照条件查询只需要添加参数

~~~
db.col.find({userid:"1001"})
~~~

如果你需要以易读的方式来读取数据，可以使用 pretty() 方法，：

```
db.col.find().pretty()
```

如果你只需要返回符合条件的第一条数据，我们可以使用findOne命令来实现

~~~
db.col.findOne({userid:"1001"})
~~~

如果你想返回指定条数的记录，可以在find方法后调用limit来返回结果

~~~
db.scolpit.find().limit(3)
~~~

**修改文档**

~~~
db.集合名.update(条件,修改后的数据)
~~~

如果我们想修改_id为1的记录，浏览量为1000，输入以下语句：

~~~
db.col.update({_id:"1"},{visits:NumberInt(1000)})
~~~

执行后，我们会发现，这条文档除了visits字段其它字段都不见了，为了解决这个问题，我们需要使用修改器$set来实现，命令如下:

~~~
db.col.update({_id:"2"},{$set:{visits:NumberInt(2000)}})
~~~

**删除文档**

~~~
db.集合名.remove(条件)
~~~

eg : 删除visits=1000的记录`db.col.remove({visits:1000})`

以下语句可以将数据全部删除，请慎用

~~~
db.col.remove({})
~~~

## 统计条数

## 模糊查询

## 大于 小于 不等于

## 包含与不包含

## 条件连接

## 列值增长

# 三、Java操作MongoDB

## 3.1 mongoDB-driver

## 3.2 SpringDataMongoDB





























