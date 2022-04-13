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

### 6.1 类和对象

Kotlin 类可以包含：构造函数和初始化代码块、函数、属性、内部类、对象声明。

在Kotlin中使用class声明类，同时也可以想Java一样声明属性和函数：

```kotlin
class Person {
    var name = ""
    var age = 0;
    fun eat() {
        println(name + "is eating. And he is" + age + "years old!")
    }
}
fun main() {
    var p = Person();
    p.age = 20
    p.name = "张三"
    p.eat()
}
```

以上就是类和对象的基本用法。

> Kotlin中实例化一个类的方式和Java是基本类似的，只是去掉了new关键字而已。之所以这么设计，是因为当你调用了某个类的构造函数时，你的意图只可能是对这个类进行实例化，因此即使没有new关键字，也能清晰表达出你的意图。Kotlin本着最简化的设计原则，将诸如new、行尾分号这种不必要的语法结构都取消了。
>
> kotlin中的类是有修饰符的。这里我们直接用菜鸟的例子：
>
> 类的修饰符包括 classModifier 和_accessModifier_:
>
> - classModifier: 类属性修饰符，标示类本身特性。
>
>   ```
>   abstract    // 抽象类  
>   final       // 类不可继承，默认属性
>   enum        // 枚举类
>   open        // 类可继承，类默认是final的
>   annotation  // 注解类
>   ```
>
> - accessModifier: 访问权限修饰符
>
>   ```
>   private    // 仅在同一个文件中可见
>   protected  // 同一个文件中或子类可见
>   public     // 所有调用的地方都可见
>   internal   // 同一个模块中可见
>   ```
>
> ### 实例
>
> ```kotlin
> // 文件名：example.kt
> package foo
> private fun foo() {} // 在 example.kt 内可见
> public var bar: Int = 5 // 该属性随处可见
> internal val baz = 6    // 相同模块内可见
> ```

### 6.2 属性和getter、setter

当然有人可能会很疑惑，上面的属性是public嘛？为什么没有getset方法也能直接访问属性。实际上不是的。其实kotlin会帮我们自动生成getset方法。如下：

```kotlin
class Person {
    var name = ""
        //以下代码是kotlin默认帮我们生成的，
        //如果我们get、set方法不做改动是不需要增加的
        get() = field//注意get不能私有化，也就是说不能加private
    	//set方法可以私有化，但是私有化后就不能有这种操作了p.age = 20
        set(name){
            field =name
        }
    var age = 0
    /*实际上这句话相当于Java的：
        private Integer age;
        public final Integer getAge(){
            return this.age;
        }
        public final void setAge(Integer age){
            this.age=age;
        }

    */
    fun eat() {
        println(name + "is eating. And he is" + age + "years old!")
    }
}

fun main() {
    var p = Person();
    //表面上是调用属性，实际上是调用get和set方法
    // Remember in kotlin whenever you write foo.bar = value it will be translated into a setter call instead of a PUT FIELD.
    p.age = 20
    p.eat()
}
```

其中field是备用字段。因为Kotlin 中类不能有字段。提供了 Backing Fields(后端变量) 机制,备用字段使用field关键字声明,field 关键词只能用于属性的访问器。

**但是我们要注意val定义的字段没有set函数**。

以上代码我们也可以通过反编译来证明，如下图：

![image-20220413094508464](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220413094508464.png)

非空属性必须在定义的时候初始化,kotlin提供了一种可以延迟初始化的方案,使用 **lateinit** 关键字描述属性：

```kotlin
class MyTest {
    lateinit var subject:String
}
```

### 6.3 继承和构造函数

任何一个面向对象的编程语言都会有构造函数的概念，Kotlin中也有，但是Kotlin将构造函数分成了两种：主构造函数和次构造函数。这也是为什么构造函数要与继承放在一起，因为有继承及必然涉及到构造函数。

我们还是用最经典的Person类举例：

```kotlin
open class Person {
    var name = ""
    var age = 0
    fun eat() {
        println(name + "is eating. And he is" + age + "years old!")
    }
}
```

> 注意：如果想让Person可以被继承需要加上open关键字。
>
> 在Kotlin中任何一个非抽象类默认都是不可以被继承的，相当于Java中给类声明了final关键字。之所以这么设计，其实和val关键字的原因是差不多的，因为类和变量一样，最好都是不可变的，而一个类允许被继承的话，它无法预知子类会如何实现，因此可能就会存在一些未知的风险。EffectiveJava这本书中明确提到，如果一个类不是专门为继承而设计的，那么就应该主动将它加上final声明，禁止它可以被继承

然后我们定义一个Student，使其继承Person类：

~~~kotlin
class Student : Person() {
    var sno = "100000"
}
~~~

> Kotlin 中所有类都继承该 Any 类，它是所有类的超类

但是为什么继承的父类要加括号呢？其实这就涉及到构造函数了。我们来看一下Kotlin的构造函数，先说主构造函数，看以下代码：

```kotlin
fun main() {
    val teacher =Teacher("111",18)
}
//主构造函数中直接定义属性并初始化属性
class Teacher(val no:String ,val age:Int){
    init {
        println("2222")
    }
}
/*
也可以这样写
class Teacher(_no:String ,_age:Int){
	var no =_no
	var age=_age
    init {
        println("2222")
    }
}

*/
```

主构造器中不能包含任何代码，初始化代码可以放在初始化代码段中，初始化代码段使用 init 关键字作为前缀。

接下来我们在回过来看一下继承时的括号，其实这涉及了Java继承特性中的一个规定，子类中的构造函数必须调用父类中的构造函数，这个规定在Kotlin中也要遵守。我们看以下代码：

```kotlin
class Student(val sno:String) : Person() {
    init {
        println("student")
    }
}
```

Student类声明了一个主构造函数，根据继承特性的规定，子类的构造函数必须调用父类的构造函数，可是主构造函数并没有函数体，我们怎样去调用父类的构造函数呢？Kotlin用了另外一种简单但是可能不太好理解的设计方式：括号。子类的主构造函数调用父类中的哪个构造函数，在继承的时候通过括号来指定。

我们现在将Person改造一下：

![image-20220413113924301](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220413113924301.png)

我们会发现编译器报错了，因为Person中没有无参构造函数，所以要改成这样：

```kotlin
//注意，我们在Student类的主构造函数中增加name和age这两个字段时，不能再将它们声明成val，因为在主构造函数中声明成val或者var的参数将自动成为该类的字段，这就会导致和父类中同名的name和age字段造成冲突
class Student(val sno:String,name: String,age:Int) : Person(name,age) {
    init {
        println("student")
    }
}
```

我们可以在Student类的主构造函数中加上name和age这两个参数，再将这两个参数传给Person类的构造函数。

了解了主构造函数我们接下来在了解一下次构造函数。

Kotlin的任何一个类只能有一个主构造函数，但是可以有多个次构造函数。次构造函数也可以用于实例化一个类，这一点和主构造函数没有什么不同，只不过它是有函数体的。

Kotlin规定，**当一个类既有主构造函数又有次构造函数时，所有的次构造函数都必须调用主构造函数**（包括间接调用）。我们看以下代码：

```kotlin
class Student(val sno:String,name: String,age:Int) : Person(name,age) {
    //次构造函数是通过constructor关键字来定义的
    //第一个次构造函数接收name和age参数，然后它又通过this关键字调用了主构造函数
    constructor(name: String,age: Int):this("",name,age){
    }
    //第二个次构造函数不接收任何参数，它通过this关键字调用了第一个次构造函数
    //由于第二个次构造函数间接调用了主构造函数，因此这仍然是合法的。
    constructor():this("111",0)
}
```

现在我们就拥有了3种方式来对Student类进行实体化：

```kotlin
fun main() {
    val student1 =Student("111","zhangsan",18)
    val student2 =Student("zhangsan",18)
    val student = Student()
}
```

到此我们也了解了次构造函数，但是在Kotlin中还有一个特殊的情况，那就是没有主构造函数，但是有次构造函数，如下：

```kotlin
class Student : Person {
    //由于没有主构造函数，次构造函数只能直接调用父类的构造函数，将this关键字换成super关键字
    constructor(name: String,age: Int):super(name,age){
    }
}
```

注意这里的代码变化，首先Student类的后面没有显式地定义主构造函数，同时又因为定义了次构造函数，所以现在Student类是没有主构造函数的。那么既然没有主构造函数，继承Person类的时候也就不需要再加上括号了。

**如果一个非抽象类没有声明构造函数(主构造函数或次构造函数)，它会产生一个没有参数的构造函数。构造函数是 public 。如果你不想你的类有公共的构造函数，你就得声明一个空的主构造函数：**

```kotlin
class DontCreateMe private constructor () {
}
```

### 6.4 接口

Java中继承使用的关键字是extends，实现接口使用的关键字是implements，而Kotlin中统一使用冒号，中间用逗号进行分隔。

Kotlin中的接口部分和Java几乎是完全一致的所以，不再赘述，这里直接使用菜鸟的例子。

~~~kotlin
interface MyInterface {
    fun bar()    // 未实现
    fun foo() {  //已实现
      // 可选的方法体
      println("foo")
    }
}
class Child : MyInterface {
    override fun bar() {
        // 方法体
    }
}
~~~

### 6.5 Kotlin中的数据类和单例

Kotlin 可以创建一个只包含数据的类，关键字为 **data**：

```kotlin
data class User(val name: String, val age: Int)
```

Kotlin会根据主构造函数中的参数帮你将equals()、hashCode()、toString()等固定且无实际逻辑意义的方法自动生成，从而大大减少了开发的工作量。

编译器会自动的从主构造函数中根据所有声明的属性提取以下函数：

- `equals()` / `hashCode()`
- `toString()` 格式如 `"User(name=John, age=42)"`
- `componentN() functions` 对应于属性，按声明顺序排列
- `copy()` 函数

如果这些函数在类中已经被明确定义了，或者从超类中继承而来，就不再会生成。

为了保证生成代码的一致性以及有意义，数据类需要满足以下条件：

- 主构造函数至少包含一个参数。
- 所有的主构造函数的参数必须标识为`val` 或者 `var` ;
- 数据类不可以声明为 `abstract`, `open`, `sealed` 或者 `inner`;
- 数据类不能继承其他类 (但是可以实现接口)。

接下来我们再来看另外一个Kotlin中特有的功能——单例类。

在Java中有很多种实现的方式，但是在Kotlin中只需要只需要将class关键字改成object关键字即可。

~~~kotlin
object Singleton{
}
~~~

可以看到，在Kotlin中我们不需要私有化构造函数，也不需要提供getInstance()这样的静态方法，只需要把class关键字改成object关键字，一个单例类就创建完成了。

### 6.6 Lambda表达式

Kotlin从第一个版本开始就支持了Lambda编程，并且Kotlin中的Lambda功能极为强大。

语法格式：

~~~
{参数名1:参数类型，参数名2:参数类型->函数体}
~~~

这里给个示例：

```kotlin
// 测试
fun main() {
    val sumLambda: (Int, Int) -> Int = {x,y -> x+y}
    println(sumLambda(1,2))  // 输出 3
}
```

当然，如果你使用Java8以上版本，肯定对lambda表达式比较熟悉，在业务中一大使用场景就是使用lambda操纵集合。这里我们也用kotlin中的lambda操纵集合来了解lambda表达式。

我们先来了解以下Kotlin中的集合的创建，这里主要介绍几个创建集合的Kotlin的API，你也可以使用Java的方式，但是Kotlin的方式也需要掌握。见下面的代码：

![image-20220413132548261](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220413132548261.png)

同理Set集合也是一样，只是将创建集合的方式换成了setOf()和mutableSetOf()函数而已。对于Map，Kotlin中对map可以像数组一样操作，如下：

```kotlin
fun testKotlinMap(){
    val map = HashMap<Int,String>()
    map.put(1,"111")
    map.put(2,"222")
    map[3] ="333"
    println(map.get(1))
    println(map[3])
}
```

当然mapOf()和mutableMapOf()函数也是有的，这里就不在演示。

下面我们通过一个需求来了解kotlin的lambda表达式：

如果有一个`List<String>`集合，我想取出其中最长的String，在Java中可以这么实现：

```java
List<String> list =new ArrayList<>();
list.add("apple");
list.add("banana");
list.add("strawberry");
list.add("pear");
String s = list.stream().max(Comparator.comparingInt(String::length)).get();
System.out.println(s);
```

下面我们通过kotlin来实现：

```kotlin
fun main() {
    val listKT = listOf("apple","banana","strawberry","pear")
    val lambda = {fruit:String ->fruit.length}
    val maxByOrNull = listKT.maxByOrNull(lambda)
    println(maxByOrNull)
}
```

我们下面来简化这段代码：

```kotlin
val maxByOrNull = listKT.maxByOrNull({fruit:String ->fruit.length})
//然后Kotlin规定，当Lambda参数是函数的最后一个参数时，可以将Lambda表达式移到函数括号的外面
val maxByOrNull = listKT.maxByOrNull(){fruit:String ->fruit.length}
//如果Lambda参数是函数的唯一一个参数的话，还可以将函数的括号省略
val maxByOrNull = listKT.maxByOrNull{fruit:String ->fruit.length}
//由于Kotlin拥有出色的类型推导机制，Lambda表达式中的参数列表其实在大多数情况下不必声明参数类型
val maxByOrNull = listKT.maxByOrNull{fruit ->fruit.length}
//当Lambda表达式的参数列表中只有一个参数时，也不必声明参数名，而是可以使用it关键字来代替，那么代码就变成了：
val maxByOrNull = listKT.maxByOrNull{it.length}
```

我们再看kotlin中的filter和map，其实与Java的写法类似，我们还是以需求为例，这回我们将list中的数据长度小于5全变成大写：

```kotlin
val listKT = listOf("apple","banana","strawberry","pear")
val list = listKT.filter { it.length <= 5 }.map { it.toUpperCase() }
list.forEach { println(it) }
```

这里仅介绍了常用的kotlin中的lambda表达式的api，但只要掌握了基本的语法规则，其他函数式API的用法只要看一看文档就能掌握了。

**Java函数式API的调用**

接下来我们看一下在Kotlin中调用Java方法时使用函数式API

我们还是通过例子了解：

在Java中启动线程可以这么写：

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("111");
    }
}).start();
//或是lambda形式
new Thread(() -> System.out.println("111")).start();
```

我们给改成Kotlin的版本：

```kotlin
Thread(object :Runnable{
    override fun run() {
        println("111")
    }
}).start()
```

> Kotlin创建匿名类实例的时候就不能再使用new了，而是改用了object关键字

下面我们改成Lambda的形式：

```kotlin
//Thread类的构造方法是符合Java函数式API的使用条件的，因此简化成这样
Thread(Runnable{ println("111") }).start()
//如果一个Java方法的参数列表中有且仅有一个Java单抽象方法接口参数，我们还可以将接口名进行省略
Thread({ println("111") }).start()
//根据上面的例子对这个lambda进行简化
Thread(){ println("111") }.start()
//再简化
Thread({ println("111") }.start()
```
