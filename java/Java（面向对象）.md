# Java面向对象学习文档

## 一、类和对象

面向对象的程序设计过程中有两个重要概念：类（class）和对象（object，也被称为实例，instance），其中类是某一批对象的抽象，可以把类理解成某种概念；对象才是一个具体存在的实体，从这个意义上来看，我们日常所说的人，其实都是人的实例，而不是人类。

### 1.1 类定义

~~~java
[修饰符] class 类名{//修饰符可以是public、final、abstract，或者完全省略这三个修饰符
    零到多个构造器定义
    零到多个Field
    零到多个方法
}
定义Field
[修饰符] Field类型 Field名 [=默认值]；
定义方法
[修饰符] 方法返回值类型 方法名(形参)
{
    //修饰符：修饰符可以省略，也可以是public、protected、private、static、final，其中public、protected、private三个最多只能出现其中之一，可以与static、final组合起来修饰Field。
    方法体
}
定义构造器
[修饰符] 构造器名（就是类名） (形参){
	//修饰符：修饰符可以省略，也可以是public、protected、private其中之一
}
~~~

构造器是一个类创建对象的根本途径，如果一个类没有构造器，这个类通常无法创建实例。因此， Java语言提供了一个功能：如果程序员没有为一个类编写构造器，则系统会为该类提供一个**默认的构造器**。一旦程序员为一个类提供了构造器，系统将不再为该类提供构造器。

Field用于定义该类或该类的实例所包含的状态数据，方法则用于定义该类或该类的实例的行为特征或者功能实现。构造器用于构造该类的实例，Java语言通过new关键字来调用构造器，从而返回该类的实例。

static是一个特殊的关键字，它可用于修饰方法、Field等成员。static修饰的成员表明它属于这个类本身，而不属于该类的单个实例，因为通常把static修饰的Field和方法也称为类Field、类方法。不使用static修饰的普通方法、Field则属于该类的单个实例，而不属于该类。

> 虽然绝大部分资料都喜欢把static称为静态，但实际上这种说法很模糊，完全无法说明static的真正作用。static的真正作用就是用于区分Field、方法、内部类、初始化块这四种成员到底属于类本身还是属于实例。在类中定义的成员，有static修饰的成员属于类本身，没有static修饰的成员属于该类的实例。

### 1.2 对象产生与使用

创建对象的根本途径是构造器，通过new关键字来调用某个类的构造器即可创建这个类的实例。

~~~java
//定义一个Person类的变量
Person p;
//通过new调用Person类的构造器，返回Person实例，将实例赋值给p
p=new Person();
~~~

### 1.3 对象、引用与指针

有这样一行代码：Person p=new Person();，这行代码创建了一个Person实例，也被称为Person对象，这个Person对象被赋给p变量。在这行代码中实际产生了两个东西：一个是p变量，一个是Person对象。

与数组类型类似，类也是一种引用数据类型，因此程序中定义的Person类型的变量实际上是一个引用，它被存放在栈内存里，指向实际的Person对象；而真正的Person对象则存放在堆（heap）内存中。下图显示了将Person对象赋给一个引用变量的示意图：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200709001.png)

栈内存里的引用变量并未真正的存储对象中的Field对象，实际存放在堆内存中，引用变量只是指向堆内存中的对象。从这个角度来看，引用变量与C语言里的指针很像，它们都是存储一个地址值，通过这个地址来引用到实际对象。实际上，Java里的引用就是C里的指针，只是Java语言把这个指针封装起来，避免开发者进行烦琐的指针操作。

如果堆内存里的对象没有任何变量指向该对象，那么程序将无法再访问该对象，这个对象也就变成了垃圾，Java的垃圾回收机制将回收该对象，释放该对象所占的内存区。因此，如果希望通知垃圾回收机制**回收某个对象**，只需切断该对象的所有引用变量和它之间的关系即可，也就是把这些引用变量赋值为null。

### 1.4 this引用

Java提供了一个this关键字，this关键字总是指向调用该方法的对象。根据this出现位置的不同， this作为对象的默认引用有两种情形：

- 构造器中引用该构造器正在初始化的对象
- 在方法中调用改方法的对象。

对于static修饰的方法而言，则可以使用类来直接调用该方法，如果在static修饰的方法中使用this关键字，则这个关键字就无法指向合适的对象。所以，static修饰的方法中不能使用this引用。由于static修饰的方法不能使用this引用，所以static修饰的方法不能访问不使用static修饰的普通成员，因此Java语法规定：静态成员不能直接访问非静态成员。

## 二、方法

### 2.1 方法的所有者

不论是从定义方法的语法来看，还是从方法的功能来看，都不难发现方法和函数之间的相似性。实际上，方法确实是由传统的函数发展而来的，方法与传统的函数有着显著不同：在结构化编程语言里，函数是一等公民，整个软件由一个个的函数组成；在面向对象编程语言里，类才是一等公民，整个系统由一个个的类组成。因此在Java语言里，**方法不能独立存在，方法必须属于类或对象。**

永远不要把方法当成独立存在的实体，正如现实世界由类和对象组成，而方法只能作为类和对象的附属，Java语言里的方法也是一样。Java语言里方法的所属性主要体现在如下几个方面:

- 方法不能独立定义，方法只能在类中定义
- 从逻辑意义上来看，方法要么属于类本身，要么属于类的一个对象
- 永远不能独立执行方法，执行方法必须使用类或者对象作为调用者

使用static修饰的方法属于这个类本身，使用static修饰的方法既可以使用类作为调用者来调用，也可以使用对象作为调用者来调用。但值得指出的是，因为使用static修饰的方法还是属于这个类的，因此使用该类的任何对象来调用这个方法时将会得到相同的执行结果，因为实际上还是使用这些实例所属的类作为调用者。没有static修饰的方法则属于该类的对象，不属于这个类本身。因此没有static修饰的方法只能使用对象作为调用者调用，不能使用类作为调用者调用。使用不同对象作为调用者来调用同一个普通方法，可能得到不同的结果。

### 2.2 方法的参数传递

Java的实参值是如何传入方法的呢？这是由Java方法的参数传递机制来控制的，Java里方法的参数传递方式只有一种：值传递。所谓值传递，就是将实际参数值的副本（复制品）传入方法内，而参数本身不会受到任何影响。

以下面swap函数为例：

~~~java
public static void swap(int a,int b){
    int tmp=a;
    a=b;
    b=tmp;
    System.out.println("swap方法中a"+a+"b"+b);
}
public static void main(String []args){
    int a=6;
    int b=9;
    
    swap(a,b);
    System.out.println("交换结束后a"+a+"b"+b);
}
~~~

> 结果 swap方法a9b6
>
> 交换结束后a6b9

从上面运行结果来看，swap方法里a和b的值是9、6，交换结束后，变量a和b的值依然是6、9。从这个运行结果可以看出，main方法里的变量a和b，并不是swap方法里的a和b。

下面通过示意图来详解上述示例：

首先从Java从main函数开始执行

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712001.png)

执行到swap方法时，将main中的a、b变量作为参数传入swap方法，传入的只是副本而不是其本身，进入swap方法后有四个变量：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712002.png)

在swap中交换后内存中的示意为：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712003.png)

前面看到的是基本类型的参数传递，Java对于引用类型的参数传递，一样采用的是值传递方式：

~~~java
class DataWarp{
    public int a;
    public int b;
}
public class ReferenceTransferTest{
    public static void swap(DataWarp dw){
        int tmp=dw.a;
        dw.a=dw.b;
        dw.b=tmp;
        System.out.pringln("swap方法中，a Field值为"+dw.a+";b的Field值为"+dw.b);
    }
    public static void main(){
        DataWarp dw =new DataWarp();
        dw.a=6;
        dw.b=9;
        swap(dw);
        System.out.pringln("交换结束后，a Field值为"+dw.a+";b的Field值为"+dw.b);
    }
}
~~~

> 运行结果：
>
> swap方法中，a Field值为9;b的Field值为6
>
> 交换结束后，a Field值为9;b的Field值为6

从运行结果来看，在swap方法里，a、b两个Field值被交换成功。不仅如此，main方法里swap方法执行结束后，a、b两个Field值也被交换了。这很容易造成一种错觉：调用swap方法时，传入swap方法的就是dw对象本身，而不是它的复制品。但这只是一种错觉，下面结合示意图来说明程序的执行过程：

程序从main方法开始执行，main方法开始创建了一个DataWrap对象，并定义了一个dw引用变量来指向DataWrap对象：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712004.png)

main方法中开始调用swap方法，main方法并未结束，系统会分别开辟出main和swap两个栈区，用于存放main和swap方法的局部变量。调用swap方法时，同样采用值传递方式，把main方法里dw变量的值赋给swap方法里的dw形参，从而完成swap方法的dw形参的初始化：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712005.png)

但是main方法中的dw是一个引用（也就是一个指针），它保存了DataWrap对象的地址值，当把dw的值赋给swap方法的dw形参后，即让swap方法的dw形参也保存这个地址值，即也会引用到堆内存中的DataWrap对象。

> 这种参数传递方式是值传递方式，系统一样复制了dw的副本传入swap方法，但关键在于dw只是一个引用变量，所以系统复制了dw变量，但并未复制DataWrap对象。

当程序在swap方法中操作dw形参时，由于dw只是一个引用变量，故实际操作的还是堆内存中的DataWrap对象。此时，不管是操作main方法里的dw变量，还是操作swap方法里的dw参数，其实都是操作它所引用的DataWrap对象，它们操作的是同一个对象。因此，当swap方法中交换dw参数所引用DataWrap对象的a、b两个Field值后，我们看到main方法中dw变量所引用DataWrap对象的a、b两个Field值也被交换了。

为了更好地证明main方法中的dw和swap方法中的dw是两个变量，我们在swap方法的最后一行增加如下代码：

`dw=null;//让他不再指向任何地址`

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/OO/20200712006.png)

main方法调用了swap方法后，再次访问dw变量的a、b两个Field，依然可以输出9、6。可见main方法中的dw变量没有受到任何影响。

### 2.3 形参可变的方法

Java允许定义形参个数可变的参数，从而允许为方法指定数量不确定的形参。如果在定义方法时，在最后一个形参的类型后增加三点（...），则表明该形参可以接受多个参数值，多个参数值被当成数组传入。

~~~java
public class Varargs{
    public static void test(int a,String... books){
        for(String tmp:books){
            System.out.println(tmp);
        }
        System.out.println(a);
    }
    public static void main(int a,String... books){
        test(5,"java","java1")
    }
}
~~~

> 运行结果：
>
> java
>
> java1
>
> 5

从test的方法体代码来看，形参个数可变的参数其实就是一个数组参数，也就是说，下面两个方法签名的效果完全一样：

~~~java
public static void test(int a,String... books);
public static void test(int a,String[] books);
~~~

**长度可变的形参只能处于形参列表的最后。一个方法中最多只能包含一个长度可变的形参。调用包含一个长度可变形参的方法时，这个长度可变的形参既可以传入多个参数，也可以传入一个数组。**

### 2.4 方法重载

Java允许同一个类里定义多个同名方法，只要形参列表不同就行。如果同一个类中包含了两个或两个以上方法的方法名相同，但形参列表不同，则被称为方法重载。

**方法重载的要求就是两同一不同：同一个类中方法名相同，参数列表不同。至于方法的其他部分，如方法返回值类型、修饰符等，与方法重载没有任何关系。**



## 三、成员变量和局部变量

**成员变量**指的是在***类***范围里定义的变量，也就是前面所介绍的Field；**局部变量**指的是在***方法***里定义的变量。

成员变量被分为类Field和实例Field两种，定义Field时没有static修饰的就是*实例Field*，有static修饰的就是*类Field*。

局部变量根据定义形式的不同，又可以被分为如下三种:

- 形参：在定义方法签名时定义的变量，形参的作用域在整个方法内有效。
- 方法局部变量：在方法体内定义的局部变量，它的作用域是从定义该变量的地方生效，到该方法结束时失效。
- 代码块局部变量：在代码块中定义的局部变量，这个局部变量的作用域从定义该变量的地方生效，到该代码块结束时失效。

## 四、隐藏与封装

## 五、构造器

## 六、继承

## 七、多态

## 八、初始化块

## 九、抽象类

## 十、接口

## 十一、final修饰符

## 十二、类成员与对象的处理

## 十三、内部类

## 十四、枚举类

## 十五、对象与垃圾回收