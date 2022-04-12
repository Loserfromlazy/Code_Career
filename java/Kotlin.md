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
    //经过了装箱，创建了两个不同的对象，这里会创建新对象是因为 Int? s可空类型而Int不可空，因此会创建新对象
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



