# Redis学习笔记

Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set：有序集合)。

Redis 的配置文件位于 Redis 安装目录下，文件名为 **redis.conf**(Windows 名为 redis.windows.conf)。

**基本命令**

1.启动客户端并检测redis服务

~~~
redis-cli
redis 127.0.0.1:6379>
redis 127.0.0.1:6379>ping
pong
~~~

2.远程服务上执行命令

~~~
redis-cli -h host -p port -a password
redis-cli -h 127.0.0.1 -p 6379 -a "pass"
redis 127.0.0.1:6379>ping
pong
~~~

3.中文乱码

~~~
redis-cli --raw
~~~

# 一、数据结构与应用

## 1.字符串 string

字符串键是redis中最基本的键值对类型，这种类型的键值对会在数据库中把单独的一个键和单独的一个值关联起来。

### SET

**为字符串键设置值**

复杂度O(1)

~~~
set key value
eg：
redis>set number "10086"
OK
~~~

在默认情况下，对一个已经设置了值的字符串键执行SET命令将导致键的旧值被新值覆盖。

从Redis 2.6.12版本开始，用户可以通过向SET命令提供可选的NX选项或者XX选项来指示SET命令是否要覆盖一个已经存在的值：

~~~
set key value[NX|XX]
~~~

如果用户在执行SET命令时给定了NX选项，那么SET命令**只会在键没有值的情况下执行设置操作**，并返回OK表示设置成功；如果键已经存在，那么SET命令将放弃执行设置操作，并返回空值nil表示设置失败。

如果用户在执行SET命令时给定了XX选项，那么SET命令**只会在键已经有值的情况下执行设置操作**，并返回OK表示设置成功；如果给定的键并没有值，那么SET命令将放弃执行设置操作，并返回空值表示设置失败。

### GET

**获取字符串键的值**

复杂度O(1)

~~~
get key
eg：
redis> get number
"10086"
redis>get date
(nil)
~~~

如果用户给定的字符串键在数据库中并没有与之相关联的值，那么GET命令将返回一个空值

### GETSET

**获取旧值并设置新值**

复杂度O(1)

GETSET命令就像GET命令和SET命令的组合版本，GETSET首先获取字符串键目前已有的值，接着为键设置新值，最后把之前获取到的旧值返回给用户

~~~
getset key newvalue
eg：
redis>getset number "11111" --现在为"10086"
"10086"
redis>getset xxx 50 --xxx键不存在
(nil)
~~~

如果被设置的键并不存在于数据库，那么GETSET命令将返回空值作为键的旧值

### MSET

**一次性为多个字符串键设置值**

复杂度O(n),其中n为用户给定的字符串键数量。

~~~
mset key value [key value ...]
eg:
redis>mset message "hello" message2 "world"
OK
~~~

如果给定的字符串键已经有相关联的值，那么MSET命令也会直接使用新值去覆盖已有的旧值。

### MGET

**一次性获取多个字符串的值**

复杂度O(n),其中n为用户给定的字符串键数量。

~~~
mget key [key...]
eg:
redis>mget message message2
1) "hello"
2) "world"
~~~

与GET命令一样，MGET命令在碰到不存在的键时也会返回空值

### MSETNX

**在键不存在的情况下，一次为多个字符串键设置值**

~~~
msetnx key value [key value...]
~~~

MSETNX与MSET的主要区别在于，MSETNX只会在所有给定键都不存在的情况下对键进行设置，而不会像MSET那样直接覆盖键已有的值

如果在给定键当中，即使有一个键已经有值了，那么MSETNX命令也会放弃对所有给定键的设置操作。MSETNX命令在成功执行设置操作时返回1，在放弃执行设置操作时则返回0。

### STRLEN

**获取字符串值的字节长度**

~~~
strlen key
eg：
redis>get number
"10086"
redis>strlen number
(intrger) 5
~~~

对于不存在的键，STRLEN命令将返回0

### 字符串索引

因为每个字符串都是由一系列连续的字节组成的，所以字符串中的每个字节实际上都拥有与之相对应的索引。

字符串值的正数索引以0为开始，从字符串的开头向结尾不断递增。

字符串值的负数索引以-1为开始，从字符串的结尾向开头不断递减。

| h    | e    | l    | l    | o    | ' '  | w    | o    | r    | l    | d    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   |
| -11  | -10  | -9   | -8   | -7   | -6   | -5   | -4   | -3   | -2   | -1   |

#### GETRANGE

**获取字符串值制定索引范围上的内容**

通过使用GETRANGE命令，用户可以获取字符串值从start索引开始，直到end索引为止的所有内容

~~~
getrange key start end
eg：
redis>getrange message 0 4 --"hello world"
"hello"
~~~

#### SETRANGE

**对字符串值的指定的索引范围进行设置**

通过使用SETRANGE命令，用户可以将字符串键的值从索引index开始的部分替换为指定的新内容，被替换内容的长度取决于新内容的长度

~~~
setrange key index substitute
eg：
redis>get message
"hello world"
redis> setrange message 6 "redis"
(integer) 11
redis>get message
"hello redis"
redis> setrange message 5 ",hahaha"--新值比原字符长
(integer) 12
redis>get message 
"hello,hahaha"
redis> EXISTS empty_string
(integer) 0
redis> SETRANGE empty_string 5 "Redis!"--填充空字节
(integer) 11
redis> GET empty_string
"\x00\x00\x00\x00\x00Redis!"--一个\x00代表一个空字节
~~~

### APPEND

**追加新内容到值的末尾**

~~~
append key suffix
eg：
redis> get mess
"hello"
redis> append mess " world"
(inetrger) 11
redis> get mess
"hello world"
~~~

如果用户给定的键并不存在，那么APPEND命令会**先将键的值初始化为空字符串""**，然后再执行追加操作

### 字符串键存储数字

每当用户将一个值存储到字符串键里面的时候，Redis都会对这个值进行检测，如果这个值能够被解释为以下两种类型的其中一种，那么Redis就会把这个值当作数字来处理：

第一种类型是能够使用**C语言的long long int类型存储的整数**，在大多数系统中，这种类型存储的都是64位长度的有符号整数，取值范围介于-9223372036854775808和9223372036854775807之间。

第二种类型是能够使用**C语言的long double类型存储的浮点数**，在大多数系统中，这种类型存储的都是128位长度的有符号浮点数，取值范围介于3.36210314311209350626e-4932和1.18973149535723176502e+4932L之间

#### INCRBY、DECRBY

**对整数执行加减法**

当字符串键存储的值能够被Redis解释为整数时，用户就可以通过INCRBY命令和DECRBY命令对被存储的整数值执行加法或减法操作。

~~~
incrby key increment
decrby key increment
~~~

eg：

~~~
redis> set number 10086
redis> decrby number 300
(integer) 9786
redis> incrby number 100
(integer) 9886
~~~

当字符串键的值不能被Redis解释为整数时，对键执行INCRBY命令或是DECRBY命令将返回一个错误

当INCRBY命令或DECRBY命令遇到不存在的键时，命令会先将键的值初始化为0，然后再执行相应的加法操作或减法操作。

#### INCR、DECR

**对整数加一减一操作**

~~~
incr key
decr key
~~~

除了增量和减量被固定为1之外，INCR命令和DECR命令的其他方面与INCRBY命令以及DECRBY命令完全相同。

### INCRBYFLOAT

~~~
incrbyfloat key incrment
eg：
redis>set decimal 3.14
redis> incrbyfloat decimal 2.55
"5.69"
~~~

INCRBYFLOAT命令在遇到不存在的键时，会先将键的值初始化为0，然后再执行相应的加法操作。

用户只能通过给INCRBYFLOAT命令**传入负数增量**来执行浮点数**减法操作。**

在使用INCRBYFLOAT命令处理浮点数的时候，命令最多只会保留计算结果小数点后的17位数字，超过这个范围的小数将被截断.

## 2.散列hash

Redis的散列键会将一个键和一个散列在数据库里关联起来，用户可以在散列中为任意多个字段（f ield）设置值。与字符串键一样，散列的字段和值既可以是文本数据，也可以是二进制数据。通过使用散列键，用户可以把相关联的多项数据存储到同一个散列里面，以便对这些数据进行管理，或者针对它们执行批量操作。

### HSET

**为字段设置值**

~~~
hset hash field value
eg:
redis> hset article title "hello"
(integer) 1
redis> hset article title "hi"
(integer) 0
~~~

如果给定字段并不存在于散列当中，那么这次设置就是一次创建操作，命令将在散列里面关联起给定的字段和值，然后返回1。

如果给定的字段原本已经存在于散列里面，那么这次设置就是一次更新操作，命令将使用用户给定的新值去覆盖字段原有的旧值，然后返回0。

### HSETNX

**只在字段不存在的情况设置值**

~~~
hsetnx hash field value
~~~

HSETNX命令在字段不存在并且成功为它设置值时返回1，在字段已经存在并导致设置操作未能成功执行时返回0。

### HGET

**获取字段的值**

~~~
hget hash field
redis> hget article title
"hi"
redis>hget artical xxx
(nil)
~~~

如果用户给定的字段并不存在于散列当中，那么HGET命令将返回一个空值。

### HINCRBY

**对字段存储的整数值执行加法或减法**

~~~
hincrby hash field increment
eg：
redis> hincrby artical count 1 --原来是100
(integer) 101
redis> hincrby artical count -1 --原来是100
(integer) 99
~~~

只能对存储着整数值的字段执行HINCRBY命令，并且用户给定的增量也必须为整数，尝试对非整数值字段执行HINCRBY命令，或者向HINCRBY命令提供非整数增量，都会导致HINCRBY命令拒绝执行并报告错误。

### HINCRBYFLOAT

**对字段存储的数字值执行浮点数加减法**

~~~
hincrbyfloat hash field increment
eg：
redis> hincrbyfloat artical count 2.56 --原来是100
"102.56"
~~~

HINCRBYFLOAT命令在成功执行加法操作之后，将返回给定字段的当前值作为结果

HINCRBYFLOAT命令不仅可以使用浮点数作为增量，还可以使用整数作为增量

与HINCRBY命令一样，只能通过向HINCRBYFLOAT命令传入负值浮点数来实现

### HSTRLEN

**获取字段值的字节长度**

~~~
hstrlen hash field
~~~

如果给定的字段或散列并不存在，那么HSTRLEN命令将返回0作为结果

### HEXISTS

**检查字段是否存在**

~~~
hexists hash field
~~~

如果散列包含了给定的字段，那么命令返回1，否则命令返回0。

### HDEL

**删除字段**

~~~
hdel hash field
~~~

当给定字段存在于散列当中并且被成功删除时，命令返回1；如果给定字段并不存在于散列当中，或者给定的散列并不存在，那么命令将返回0表示删除失败。

### HLEN

**获取散列包含的字段数量**

~~~
hlen hash
eg：
redis> hlen artical
(integer) 4   --包含四个字段
~~~

### HMSET

**一次为多个字段设置值**

~~~
hmset hash field value [...]
~~~

### HMGET

**一次获取多个字段值**

~~~
hmget hash field [field...]
~~~

### HKEYS、HVALS、HGETALL

**获取散列所有字段、值或所有字段和值**

~~~
hkeys hash
hvals hash
hgetall hash
~~~

**字段在散列中的顺序**

Redis散列包含的字段在底层是以无序方式存储的，根据字段插入的顺序不同，包含相同字段的散列在执行HKEYS命令、HVALS命令和HGETALL命令时可能会得到不同的结果，因此用户在使用这3个命令的时候，不应该对它们返回的元素的排列顺序做任何假设。

## 3.列表list

Redis的列表（list）是一种线性的有序结构，可以按照元素被推入列表中的顺序来存储元素，这些元素既可以是文字数据，又可以是二进制数据，并且列表中的元素可以重复出现。

### LPUSH

**将元素推入列表左端**

~~~
lpush list item [item...]
eg:
redis> lpush todo "do1" "do2"
(integer) 2
~~~

在推入操作执行完毕之后，LPUSH命令会返回列表当前包含的元素数量作为返回值

当用户调用LPUSH命令或RPUSH命令尝试将元素推入列表的时候，如果给定的列表并不存在，那么命令将自动创建一个空列表，并将元素推入刚刚创建的列表中。

### RPUSH

**将元素推入列表右端**

~~~
rpush list item [item...]
eg:
redis> rpush todo "do1" "do2"
(integer) 2
~~~

### LPUSHX、RPUSHX

**只对已存在的列表执行推入操作**

LPUSHX命令和RPUSHX命令只会在列表已经存在的情况下，将元素推入列表左端或右端。

与LPUSH命令和RPUSH命令不一样，LPUSHX命令和RPUSHX命令**每次只能推入一个元素**

~~~
lpush list item
rpush list item
~~~

### LPOP

**弹出列表最左端的元素**

~~~
lpop list
eg：
redis> lpop list  --lpush list "list1" "list2"
"list2"
~~~

如果用户给定的列表并不存在，那么LPOP命令将返回一个空值，表示列表为空

### RPOP

**弹出列表最右端的元素**

~~~
rpop list
eg：
redis> rpop list  --rpush list "list1" "list2"
"list2"
~~~

### RPOPLPUSH

**将右端元素推入左端**

~~~
rpoplpush source target
eg:--将list1 右端元素推到list2 左端
redis> rpush list "a" "b" "c"
redis> rpush list1 "a" "b" "c"
redis> rpush list2 "d" "e" "f"
redis> rpoplpush list1 list2
"c"
redis> rpoplpush list list
"c"
~~~

RPOPLPUSH命令会返回被弹出的元素作为结果。

RPOPLPUSH命令允许用户将源列表和目标列表设置为同一个列表，在这种情况下，RPOPLPUSH命令的效果相当于将列表最右端的元素变成列表最左端的元素。

如果用户传给RPOPLPUSH命令的源列表并不存在，那么RPOPLPUSH命令将放弃执行弹出和推入操作，只返回一个空值表示命令执行失败

### LLEN

**获取列表长度**

~~~
llen list
~~~

对于不存在的列表，LLEN命令将返回0作为结果
### LINDEX

**获取指定索引的元素**

~~~
lindex list index
~~~

正数索引从列表的左端开始计算，依次向右端递增：最左端元素的索引为0，左端排行第二的元素索引为1，左端排行第三的元素索引为2，以此类推。**最大的正数索引为列表长度减1，即N-1**。

负数索引从列表的右端开始计算，依次向左端递减：最右端元素的索引为-1，右端排行第二的元素索引为-2，右端排行第三的元素索引为-3，以此类推。**最大的负数索引为列表长度的负数，即-N**。

如果用户给定的索引超出了这一范围，那么LINDEX命令将返回空值，以此来表示给定索引上并不存在任何元素

### LRANGE

**获取指定索引范围的元素**

~~~
lrange list start end
~~~

LRANGE命令接受一个列表、一个开始索引和一个结束索引作为参数，然后依次返回列表从开始索引到结束索引范围内的所有元素，其中开始索引和结束索引**对应的元素也会被包含**在命令返回的元素当中。

一个快捷地获取列表包含的**所有元素**的方法，就是使用0作为起始索引、-1作为结束索引去调用LRANGE命令，这种方法非常适合于查看长度较短的列表

如果用户给定的起始索引和结束索引都超出了范围，那么LRANGE命令将返回空列表作为结果。

如果用户给定的其中一个索引超出了范围，那么LRANGE命令将对超出范围的索引进行**修正**，然后再执行实际的范围获取操作；其中超出范围的起始索引会被修正为0，而超出范围的结束索引则会被修正为-1。

### LSET

**为指定索引设置新元素**

~~~
lset list index newelement
~~~

因为LSET命令只能对列表中已存在的索引进行设置，所以如果用户给定的索引超出了列表的有效索引范围，那么LSET命令将**返回一个错误**

### LINSERT

**将元素插入列表**

~~~
linsert list BERORE|AFTER targetElement newElement
eg:
redis> linsert list after "c" "12345"
(integer) 4
redis>lrange list 0-1
1)"a"
2)"b"
3)"c"
4)"12345"
~~~

LINSERT命令第二个参数的值可以是BEFORE或者AFTER，它们分别用于指示命令将新元素插入目标元素的前面或者后面。命令在完成插入操作之后会返回列表当前的长度。

为了执行插入操作，LINSERT命令要求用户给定的目标元素必须已经存在于列表当中。相反，如果用户给定的目标元素并不存在，那么LINSERT命令将返回-1表示插入失败

### LTRIM

**修剪列表**

~~~
ltrim list start end
eg:
redis> rpush list "a" "b" "c" "d" "e"
redis> ltrim list 0 2 --让列表只保留索引0到索引2范围内的3个元素
OK
redis> rpush list 0 1 2 3 4 5 6 7 8 9 
redis> ltrim list -5 -1 
OK
redis> lrange list 0-1
"5"
"6"
"7"
"8"
"9"
~~~

LTRIM命令在执行完移除操作之后将返回OK作为结果

## 4.集合set

Redis的集合（set）键允许用户将任意多个各不相同的元素存储到集合中，这些元素既可以是文本数据，也可以是二进制数据。

但集合与列表有以下两个明显的区别：

●列表可以存储重复元素，而集合只会存储非重复元素，尝试将一个已存在的元素添加到集合将被忽略。

●列表以有序方式存储元素，而集合则以无序方式存储元素。



## 5.有序集合zset

Redis的有序集合（sorted set）同时具有“有序”和“集合”两种性质，这种数据结构中的每个元素都由一个成员和一个与成员相关联的分值组成，其中成员以字符串方式存储，而分值则以64位双精度浮点数格式存储。

# 二、Spring DATA Redis

SpringDataredis 是Spring Data 的一员，用于对redis的操作进行封装的框架。

此处引入我另一篇笔记中关于spring data redis的学习笔记。

### 1.快速入门

#### 1.1 准备工作

1. 创建工程，导入依赖。

   ```xml
       <!--缓存-->
       <dependency>
         <groupId>redis.clients</groupId>
         <artifactId>jedis</artifactId>
         <version>2.9.0</version>
       </dependency>
       <dependency>
         <groupId>org.springframework.data</groupId>
         <artifactId>spring-data-redis</artifactId>
         <version>2.0.6.RELEASE</version>
       </dependency>
       <dependency>
         <groupId>junit</groupId>
         <artifactId>junit</artifactId>
         <version>4.12</version>
       </dependency>
       <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-test</artifactId>
         <version>5.0.5.RELEASE</version>
   ```

2. 创建redis-config.properties

   ```
   redis.host=127.0.0.1
   redis.port=6379
   redis.pass
   redis.database=0
   redis.maxIdle=300
   redis.maxWait=3000
   ```

   maxIdle:最大空闲数

   maxWaitMillis:连接时的最大等待毫秒数

3. 创建applicationContext-redis.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?> 
   <beans xmlns="http://www.springframework.org/schema/beans"      
          xmlns:context="http://www.springframework.org/schema/context"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
     xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
      
      
      <context:property-placeholder location="classpath:redis-config.properties" />
      
      <!-- redis 相关配置 --> 
      <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">  
        <property name="maxIdle" value="${redis.maxIdle}" />   
        <property name="maxWaitMillis" value="${redis.maxWait}" />  
      </bean>  
     
      <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="${redis.host}" p:port="${redis.port}" p:password="${redis.pass}" p:pool-config-ref="poolConfig"/>  
      
      <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">  
       	<property name="connectionFactory" ref="jedisConnectionFactory" />
      </bean>
   
   </beans>  
   ```

### 2. 值类型操作

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-redis.xml")
public class TestValue {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 存值
     */
    @Test
    public void setValue(){
        redisTemplate.boundValueOps("name").set("yhr");
    }

    /**
     * 取值
     */
    @Test
    public  void  getValue(){
        String name = (String) redisTemplate.boundValueOps("name").get();
        System.out.println(name);
    }

    /**
     * 删除值
     */
    @Test
    public void deleteValue(){
        redisTemplate.delete("name");
    }
}

```

### 3. Set类型操作

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-redis.xml")
public class TestSet {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 存值
     */
    @Test
    public void setValue(){
        redisTemplate.boundSetOps("nameset").add("刘备");
        redisTemplate.boundSetOps("nameset").add("关羽");
        redisTemplate.boundSetOps("nameset").add("张飞");
    }

    /**
     * 取值
     */
    @Test
    public void getValue(){
        Set nameset = redisTemplate.boundSetOps("nameset").members();
        System.out.println(nameset);
    }

    /**
     * 删除集合某个值
     */
    @Test
    public void deleteValue(){
        redisTemplate.boundSetOps("nameset").remove("张飞");
    }

    /**
     * 删除集合所有值
     */
    @Test
    public void deleteAllValue(){
        redisTemplate.delete("nameset");
    }
}
```

### 4. List类型操作

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-redis.xml")
public class TestList {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 右压栈：后添加的对象排在后面
     */
    @Test
    public void setValue1(){
        redisTemplate.boundListOps("namelist1").rightPush("刘备");
        redisTemplate.boundListOps("namelist1").rightPush("关羽");
        redisTemplate.boundListOps("namelist1").rightPush("张飞");
    }

    /**
     * 右压栈取值
     */
    @Test
    public void getValue1(){
        //range(开始索引,查询个数)  -1表示取出全部
        List namelist1 = redisTemplate.boundListOps("namelist1").range(0, 10);
        System.out.println(namelist1);
    }

    /**
     * 左压栈：后添加的对象排在前面
     */
    @Test
    public void setValue2(){
        redisTemplate.boundListOps("namelist2").leftPush("刘备");
        redisTemplate.boundListOps("namelist2").leftPush("关羽");
        redisTemplate.boundListOps("namelist2").leftPush("张飞");
    }

    /**
     * 左压栈取值
     */
    @Test
    public void getValue2(){
        //range(开始索引,查询个数)  -1表示取出全部
        List namelist2 = redisTemplate.boundListOps("namelist2").range(0, 10);
        System.out.println(namelist2);
    }

    /**
     * 查询某个位置元素
     */
    @Test
    public void searchByIndex(){
        String o = (String) redisTemplate.boundListOps("namelist2").index(1);
        System.out.println(o);
    }

    /**
     * 查询某个位置元素
     */
    @Test
    public void delete(){
        redisTemplate.boundListOps("namelist2").remove(1,"张飞");//删除的个数，删除的项目
    }

}
```

### 5. Hash类型操作

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-redis.xml")
public class TestHash {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 存值
     */
    @Test
    public void setValue1(){
        redisTemplate.boundHashOps("namehash").put("a","关羽");
        redisTemplate.boundHashOps("namehash").put("b","张飞");
        redisTemplate.boundHashOps("namehash").put("c","刘备");
    }

    /**
     * 查询所有的key
     */
    @Test
    public void getKeys(){
        Set keys = redisTemplate.boundHashOps("namehash").keys();
        System.out.println(keys);
    }

    /**
     * 查询所有的value
     */
    @Test
    public void getValues(){
        List values = redisTemplate.boundHashOps("namehash").values();
        System.out.println(values);
    }

    /**
     * 根据key查询value
     */
    @Test
    public void getValuesByKey(){
        String s = (String) redisTemplate.boundHashOps("namehash").get("b");
        System.out.println(s);

    }
    /**
     * 删除
     */
    @Test
    public void deleteByKey(){
        redisTemplate.boundHashOps("namehash").delete("b");

    }
}
```

### 6.ZSet类型操作

ZSet是set的升级版本，它在set的基础上增加了一格顺序属性，这一属性在添加元素的同时可以指定，每次指定后，zset会自动重新按照新的值调整顺序。可以理解为右两列的mysql表，一列存储value，一列存储分值。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:applicationContext-redis.xml")
public class testZSet {

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 存值
     */
    @Test
    public  void  setValue(){
        redisTemplate.boundZSetOps("namezset").add("刘备",10000);
        redisTemplate.boundZSetOps("namezset").add("关羽",0);
        redisTemplate.boundZSetOps("namezset").add("张飞",100);

    }

    /**
     * 由低到高查询
     */
    @Test
    public void getValue(){
        Set namezset = redisTemplate.boundZSetOps("namezset").range(0,-1);
        System.out.println(namezset);
    }

    /**
     * 查询，由高到低
     */
    @Test
    public void getValue2(){
        Set namezset = redisTemplate.boundZSetOps("namezset").reverseRange(0,9);
        System.out.println(namezset);
    }

    /**
     * 增加分数
     */
    @Test
    public void addScore(){
        redisTemplate.boundZSetOps("namezset").incrementScore("关羽",2000);
    }

    /**
     * 查询值和分数
     */
    @Test
    public void getValueAndScore(){
        Set<ZSetOperations.TypedTuple> namezset=  redisTemplate.boundZSetOps("namezset").reverseRangeWithScores(0,-1);

        for (ZSetOperations.TypedTuple typedtuple:namezset) {
            System.out.println("姓名"+typedtuple.getValue());
            System.out.println("金币"+typedtuple.getScore());
        }
    }
}
```

### 7.过期时间设置

```java
@Test
    public void setValue(){
        redisTemplate.boundValueOps("name").set("yhr");
        redisTemplate.boundValueOps("name").expire(10, TimeUnit.SECONDS);
    }
```

# 三、附加功能

## 1.数据库

## 2.自动过期







