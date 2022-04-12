# Kotlin学习笔记

转载请声明！！！本文如有错误，欢迎各位大佬来指正，感激不尽。

> 本文参考资料：第一行代码：Android版；菜鸟；Kotlin语言中文站

## 一、概述

### 1.1 简介

Kotlin 是一种在 Java 虚拟机上运行的静态类型编程语言，被称之为 Android 世界的Swift，由 JetBrains 设计开发并开源。Kotlin 可以编译成Java字节码，也可以编译成 JavaScript，方便在没有 JVM 的设备上运行。

编程语言大致可以分为两类：编译型语言和解释型语言。编译型语言的特点是编译器会将我们编写的源代码一次性地编译成计算机可识别的二进制文件，然后计算机直接执行，像C和C++都属于编译型语言。解释型语言则完全不一样，它有一个解释器，在程序运行时，解释器会一行行地读取我们编写的源代码，然后实时地将这些源代码解释成计算机可识别的二进制数据后再执行，因此解释型语言通常效率会差一些，像Python和JavaScript都属于解释型语言。

那么Java是什么语言呢？Java代码确实是要先编译再运行的，但是Java代码编译之后生成的并不是计算机可识别的二进制文件，而是一种特殊的class文件，这种class文件只有Java虚拟机（Android中叫ART，一种移动优化版的虚拟机）才能识别，而这个Java虚拟机担当的其实就是解释器的角色，它会在程序运行时将编译后的class文件解释成计算机可识别的二进制数据后再执行，因此，准确来讲，Java属于解释型语言。

到此，可能已经明白Kotlin的工作原理了，就是开发了一门新的编程语言，然后自己做了个编译器，让它将这门新语言的代码编译成同样规格的class文件。

### 1.2 快速入门

我们可以新建一个Kotlin文件，然后创建主函数，打印Hello Kotlin。

> 可以在IDEA中新建Kotlin项目，也可以在IDEA中新建Kotlin的SpringBoot项目，或者使用AndroidStudio创建一个Kotlin的安卓项目。

```kotlin
package com.learn.learnkotlin.learn01

fun main(){
    println("Hello Kotlin")
}
```

## 二、基本语法

本章更多的作用是类似于附录，可以跳过，在后面有相关的问题，可以再回来查阅。

### 2.1 注释

Kotlin与Java类似支持单行和多行注释，实例如下：

```kotlin
// 这是一个单行注释

/* 这是一个多行的
   块注释。 */
```

与 Java 不同, Kotlin 中的块注释允许嵌套。

### 2.2 字符串模板

$ 表示一个变量名或者变量值

$varName 表示变量值

${varName.fun()} 表示变量的方法返回值:

```kotlin
fun main() {
    var a = 1
    val s1 = "a is $a"
    println(s1)
    a = 2
    val s2 = "${s1.replace("is", "was")}, but now is $a"
    println(s2)
}
```

输出结果：

a is 1
a was 1, but now is 2

### 2.3 NULL检查

Kotlin的空安全设计对于声明可为空的参数，在使用时要进行空判断处理，有两种处理方式，字段后加!!像Java一样抛出空异常，另一种字段后加?可不做处理返回值为 **null** 或配合 **?:** 做空判断处理。

举个栗子，下面代码可以自行上机测试体会判空机制。

```kotlin
//类型后面加?表示可为空
//var age: String? = "23"
var age: String? = null
//抛出空指针异常,如果age为空，就会抛出空指针
val ages = age!!.toInt()
//不做处理返回 null,如果age为空，下面会直接输出null
val ages1 = age?.toInt()
//age为空返回-1,如果age为空，下面会直接输出-1
val ages2 = age?.toInt() ?: -1
println(age)
println(ages)
println(ages1)
println(ages2)
```

### 2.4 默认导入

有多个包会默认导入到每个 Kotlin 文件中：

- kotlin.*
- kotlin.annotation.*
- kotlin.collections.*
- kotlin.comparisons.*
- kotlin.io.*
- kotlin.ranges.*
- kotlin.sequences.*
- kotlin.text.*

### 2.5 区间

区间表达式由具有操作符形式 **..** 的 rangeTo 函数辅以 in 和 !in 形成。

~~~kotlin
val range = 0..10
~~~

上面代码表示创建了一个0到10的区间，并且两端都是闭区间，这意味着0到10这两个端点都是包含在区间中的，用数学的方式表达出来就是[0, 10]。

kotlin还可以使用until关键字创建一个左闭右开的区间：

~~~kotlin
val range = 0 until 10
~~~

上面代码表示创建了一个0到10的左闭右开区间用数学的方式表达出来就是[0, 10)。

### 2.6 类型检测

我们可以使用 is 运算符检测一个表达式是否某类型的一个实例(类似于Java中的instanceof关键字)。我们这里直接使用菜鸟教程的例子。

```kotlin
fun getStringLength(obj: Any): Int? {
  if (obj is String) {
    // 做过类型判断以后，obj会被系统自动转换为String类型
    return obj.length 
  }

  //在这里还有一种方法，与Java中instanceof不同，使用!is
  // if (obj !is String){
  //   // XXX
  // }

  // 这里的obj仍然是Any类型的引用
  return null
}
```

## 三、数据类型

### 3.1 变量

在Kotlin中定义变量的方式和Java区别很大，在Java中如果想要定义一个变量，需要在变量前面声明这个变量的类型，比如说int a表示a是一个整型变量，String b表示b是一个字符串变量。而Kotlin中定义一个变量，只允许在变量前声明两种关键字：val和var。

- val（value的简写）用来声明一个不可变的变量，这种变量在初始赋值之后就再也不能重新赋值，对应Java中的final变量。
- var（variable的简写）用来声明一个可变的变量，这种变量在初始赋值之后仍然可以再被重新赋值，对应Java中的非final变量。

举个栗子：

```kotlin
fun main() {
    val a = 10
    println("a = " + a)
}
```

输出结果：a = 10

> 上面代码我把一个整形10赋值给a，a就会自动推到成整形变量

当然如果我们对一个变量延迟赋值的话，Kotlin就无法自动推导它的类型了。这时候就需要显式地声明变量

```kotlin
val a: Int = 10
```

> Kotlin 中没有基础数据类型，只有封装的数字类型，你每定义的一个变量，其实 Kotlin 帮你封装了一个对象，这样可以保证不会出现空指针
>
> | Java基本数据类型 | Kotlin对象数据类型 |
> | ---------------- | ------------------ |
> | int              | Int                |
> | long             | Long               |
> | short            | Short              |
> | float            | Float              |
> | double           | Double             |
> | boolean          | Boolean            |
> | char             | Char               |
> | byte             | Byte               |

### 3.2 数字比较

Kotlin 中没有基础数据类型，只有封装的数字类型，你每定义的一个变量，其实 Kotlin 帮你封装了一个对象，这样可以保证不会出现空指针。数字类型也一样，所以在比较两个数字的时候，就有比较数据大小和比较两个对象是否相同的区别了。

在 Kotlin 中，三个等号 === 表示比较对象地址，两个 == 表示比较两个值大小。

这里直接采用菜鸟教程的例子，简单易懂：

```kotlin
fun main() {
    //这里与Java类似，在范围是 [-128, 127] 之间并不会创建新的对象
    val a: Int = 10000
    // true，值相等，对象地址相等
    println(a === a) 
    //经过了装箱，创建了两个不同的对象，这里会创建新对象是因为 Int? 是可空类型而Int是不可空的类型，因此会创建新对象
    val boxedA: Int? = a
    val anotherBoxedA: Int? = a

    //虽然经过了装箱，但是值是相等的，都是10000
    //  false，值相等，对象地址不一样
    println(boxedA === anotherBoxedA) 
    // true，值相等
    println(boxedA == anotherBoxedA) 
}
```

### 3.3 类型转换

Kotlin的类型转换类似于Java，较小的类型不能隐式转换为较大的类型，可以通过调用toInt()之类的方法进行转换，与Java的方法了类似。

```kotlin
val a: Byte = 1 // OK, 字面值是静态检测的
val b: Int = a // 错误,无法通过编译
```

我们可以用其toInt()方法代替

```kotlin
val a: Byte = 1 // OK, 字面值是静态检测的
val b: Int = a.toInt() // OK
```

## 四、函数

### 4.1 语法规则

函数是用来运行代码的载体，你可以在一个函数里编写很多行代码，当运行这个函数时，函数中的所有代码会全部运行。像我们前面使用过的main()函数就是一个函数，只不过它比较特殊，是程序的入口函数，即程序一旦运行，就是从main()函数开始执行的。

函数的语法规则如下：

~~~kotlin
fun methodName(param1:Int ,param2:Int):Int{
	return 0
}
~~~

fun声明函数，然后是函数名，参数:参数类型，最后是返回值，如果没有可以不写，如果熟悉随便一门编程语言都能看到这个语法规则。

### 4.2 可变参数

如果是可变参数则使用vararg标识，举个栗子：

```kotlin
fun main() {
    vars(1,2,3,4,5)  // 输出12345
}
fun vars(vararg v:Int){
    for(vt in v){
        print(vt)
    }
}
```

### 4.3 简化

如果一个函数中只有一行代码则可以简化，举个栗子：

~~~kotlin
fun main() {
    println(maxNumbers(1,2))
    println(maxNumbersSimple(1,2))
    println(maxNumbersSimpleMore(1,2))
}
fun maxNumbers(num1:Int,num2:Int):Int{
    return max(num1,num2)
}
fun maxNumbersSimple(num1:Int,num2:Int):Int = max(num1,num2)
fun maxNumbersSimpleMore(num1:Int,num2:Int) = max(num1,num2)
~~~

如果只有一行代码可以省略成maxNumbersSimple函数的样子，而用等号连接了max函数，因此kotlin可以推断maxNumbers函数的返回值，因此又可以省略返回值，最终简化为maxNumbersSimpleMore的形式。

## 五、程序的逻辑控制

### 5.1 if条件语句

kotlin的if语句与Java几乎没有区别，举个栗子：

```kotlin
fun main() {
    println(maxNumbers(1,2))
}
fun maxNumbers(num1:Int,num2:Int):Int{
    var value = 0;
    if (num1>num2){
        value = num1
    }else{
        value = num2
    }
    return value
}
```

但是这两个语言唯一的区别就是，kotlin的if可以有返回值，返回值就是if语句每一个条件中最后一行代码的返回值。举例：

```kotlin
fun main() {
    println(maxNumbers(1,2))
}
fun maxNumbers(num1:Int,num2:Int):Int{
    val value = if (num1>num2){
        num1
    }else{
        num2
    }
    return value
}
```

上面代码可以简化：

~~~kotlin
fun maxNumbers(num1:Int,num2:Int):Int{
    return if (num1>num2){
        num1
    }else{
        num2
    }
}
//再简化
fun maxNumbers(num1:Int,num2:Int) =if (num1>num2){
    num1
}else{
    num2
}
//再简化
fun maxNumbers(num1:Int,num2:Int) = if (num1>num2) num1 else num2
~~~

通过kotlin的简单if语句可以发现kotlin的语法特性还是很有趣的，总体来说就是简单。

kotlin的if语句还可以使用区间，举个栗子：

```kotlin
fun main() {
    val x = 5
    val y = 9
    if (x in 1..8) {
        println("x 在区间内")
    }
}
```

### 5.2 when条件语句

Kotlin中的when语句有点类似于Java中的switch语句，但它又远比switch语句强大得多。

when语句格式：

~~~
匹配值 -> { 执行逻辑 }
~~~

我们直接使用菜鸟教程的例子来体会when的用法：

```kotlin
fun main() {
    var x = 0
    when (x) {
        //匹配值多个分支可以放在一起
        0, 1 -> println("x == 0 or x == 1")
        else -> println("otherwise")
    }

    when (x) {
        1 -> println("x == 1")
        2 -> println("x == 2")
        //相当于Java中的default块
        else -> { // 等于default
            println("x 不是 1 ，也不是 2")
        }
    }

    when (x) {
        //可以使用区间作为匹配值
        in 0..10 -> println("x 在该区间范围内")
        else -> println("x 不在该区间范围内")
    }
}
```

when语句中还可以使用is关键字，如下

```kotlin
fun checkNumber (num:Number){
    when(num){
        is Int -> println("Int")
        is Double -> println("Double")
        else -> println("not support")
    }
}
```

### 5.3 for循环

Java中主要有两种循环语句：while循环和for循环。而Kotlin也提供了while循环和for循环，其中while循环不管是在语法还是使用技巧上都和Java中的while循环没有任何区别，这里我们直接看kotlin的for循环。

Kotlin在for循环方面做了很大幅度的修改，Java中最常用的for-i循环在Kotlin中直接被舍弃了，而Java中另一种for-each循环则被Kotlin进行了大幅度的加强，变成了for-in循环，所以我们只需要学习for-in循环的用法就可以了。

for 循环可以对任何提供迭代器（iterator）的对象进行遍历，语法如下:

```kotlin
for (item in collection) print(item)
```

kotlin的for-in循环不光能遍历集合等对象，而且能遍历kotlin自己的区间：

~~~kotlin
for (i in 0..10) print(i)
//输出：012345678910
~~~

而如果你想跳过其中的一些元素，可以使用step关键字,如下：

~~~kotlin
for (i in 0..10 step 2) print(i)
//输出：0246810
~~~

以上都是升序，如果你想创建一个降序的区间，可以使用downTo关键字，用法如下：

```kotlin
for (i in 10 downTo 1) print(i)
//输出：10987654321
```

## 六、面向对象编程

> 未完结
