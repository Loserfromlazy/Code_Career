# Java教程

# 数组

## 数组概述

Java的数组要求所有的数组元素具有相同的数据类型。因此，在一个数组中，数组元素的类型是唯一的，即一个数组里只能存储一种数据类型的数据，而不能存储多种数据类型的数据。

PS：因为Java语言是面向对象的语言，而类与类之间可以支持继承关系，这样可能产生一个数组里可以存放多种数据类型的假象。例如有一个水果数组，要求每个数组元素都是水果，实际上数组元素既可以是苹果，也可以是香蕉，但这个数组的数组元素的类型还是唯一的，只能是水果类型。

数组也是一种数据类型，它本身是一种引用类型。例如int是一个基本类型，但int[]（这是定义数组的一种方式）就是一种引用类型了。

## 数组定义

> type [] arrayName;
>
> type arraryName[];

数组是一种引用类型的变量，因此使用它定义一个变量时，仅仅表示定义了一个引用变量（也就是定义了一个指针），这个引用变量还未指向任何有效的内存，因此**定义数组时不能指定数组的长度**。而且由于定义数组只是定义了一个引用变量，并未指向任何有效的内存空间，所以还没有内存空间来存储数组元素，因此这个数组也不能使用，只有对数组进行初始化后才可以使用。

## 数组初始化

初始化，就是为数组的数组元素分配内存空间，并为每个数组元素赋初始值。

数组的初始化有如下两种方式。

静态初始化：初始化时由程序员显式指定每个数组元素的初始值，由系统决定数组长度。

> arratName =new type[] {element1,element2...}
>
> 简化写法：arrayName={element1,element2...};
>
> int [] array =new int [] {5,6,8,0}
>
> int [] array={5,6,7,8}

动态初始化：初始化时程序员只指定数组长度，由系统为数组元素分配初始值。

> arrayName =new type[length];
>
> int [] prices =new int[5];

系统按如下规则分配初始值。

数组元素的类型是基本类型中的整数类型（byte、short、int和long），则数组元素的值是0。

 数组元素的类型是基本类型中的浮点类型（float、double），则数组元素的值是0.0.

数组元素的类型是基本类型中的字符类型（char），则数组元素的值是'\u0000'。

数组元素的类型是基本类型中的布尔类型（boolean），则数组元素的值是false。

数组元素的类型是引用类型（类、接口和数组），则数组元素的值是null。

## 数组使用

> System.out.println(arr[1]);
>
> arr[1]="aaaa";

如果访问数组元素时指定的索引值小于0，或者大于等于数组的长度，编译程序不会出现任何错误，但运行时出现异常：java.lang.ArrayIndexOutOfBoundsException:N（数组索引越界异常），异常信息后的N就是程序员试图访问的数组索引。

## ForEach循环

> for( type variableName : attay | collention)
>
> {//variableName  自动迭代访问每个元素
>
> }

eg：

~~~java
for(String book :books ){
    book="aaa";
    System.out.println(book);
}
~~~

## 内存中的数组

数组引用变量只是一个引用，这个引用变量可以指向任何有效的内存，只有当该引用指向有效内存后，才可通过该数组变量来访问数组元素。

与所有引用变量相同的是，引用变量是访问真实对象的根本方式。也就是说，如果我们希望在程序中访问数组对象本身，则只能通过这个数组的引用变量来访问它

实际的数组对象被存储在堆（heap）内存中；如果引用该数组对象的数组引用变量是一个局部变量，那么它被存储在栈（stack）内存中。数组在内存中的存储示意图如下图所示。**PS（图片来源于传播智客）**

![img](https://img2018.cnblogs.com/blog/1095081/201810/1095081-20181031223518828-664145778.png)

![img](https://img2018.cnblogs.com/blog/1095081/201810/1095081-20181031223746305-1801506861.png)

**栈内存与堆内存区别：**

> 当一个方法执行时，每个方法都会建立自己的内存栈，在这个方法内定义的变量将会逐个放入这块栈内存里，随着方法的执行结束，这个方法的内存栈也将自然销毁。因此，所有在方法中定义的局部变量都是放在栈内存中的；当我们在程序中创建一个对象时，这个对象将被保存到运行时数据区中，以便反复利用（因为对象的创建成本通常较大），这个运行时数据区就是堆内存。堆内存中的对象不会随方法的结束而销毁，即使方法结束后，这个对象还可能被另一个引用变量所引用（在方法的参数传递时很常见），则这个对象依然不会被销毁。只有当一个对象没有任何引用变量引用它时，系统的垃圾回收器才会在合适的时候回收它。

当我们看一个数组时，一定要把数组看成两个部分：一部分是数组引用，也就是在代码中定义的数组引用变量；还有一部分是实际的数组对象，这部分是在堆内存里运行的，通常无法直接访问它，只能通过数组引用变量来访问。

## 基本类型数组的初始化

~~~java
int [] arr;
arr=new int [5];
for(int i=0;i<arr.length;i++){
    arr[i]=i+100;
}
~~~

执行第一行内存如下

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/001.jpg)

执行第二行

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/002.jpg)

执行for循环

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/003.jpg)

## 引用类型数组的初始化

~~~java
class person{
    public int age;
    public double height;
    public void info(){
        sout;
    }
}
//main中
Person []stu;
stu=new Person[2];
person zhang=new Person();
zhang.age=18;
zhang.height=120;
person li=new Person();
li.age=14;
li.height=130;
stu[0]=zhang;
stu[1]=li;
li.info();
stu[1].info();
~~~

执行Person[]stu;代码时，这行代码仅仅在栈内存中定义了一个引用变量，也就是一个指针，这个指针并未指向任何有效的内存区.

初始化后内存变化

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/004.jpg)

声明两个Person变量，zhang和li，此时在栈内存中分配两块内存用于存储变量zhang和li，在堆内存中分配两块内存用于存储zhang和li的数据，如下图

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/005.jpg)

给stu数组赋值后堆内存变化如下：

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/006.jpg)

## 多维数组

二维数组定义

~~~
type [][] arrName;
~~~

Java语言采用上面的语法格式来定义二维数组，但它的实质还是一维数组，只是其数组元素也是引用，数组元素里保存的引用指向一维数组。

接着对这个“二维数组”执行初始化，同样可以把这个数组当成一维数组来初始化，把这个“二维数组”当成一个一维数组，其元素的类型是type[]类型，则可以采用如下语法进行初始化：

~~~
arrName=new type [length][]
~~~

## 工具类

Java提供的Arrays类里包含的一些static修饰的方法可以直接操作数组，这个Arrays类里包含了如下几个static修饰的方法（static修饰的方法可以直接通过类名调用）。

int binarySearch(type[]a,type key)：使用二分法查询key元素值在a数组中出现的索引；如果a数组不包含key元素值，则返回负数。调用该方法时要求数组中元素已经按升序排列，这样才能得到正确结果。

 int binarySearch(type[]a,int fromIndex,int toIndex,type key)：这个方法与前一个方法类似，但它只搜索a数组中fromIndex到toIndex索引的元素。

type[]copyOf(type[]original,int newLength)：这个方法将会把original数组复制成一个新数组，其中length是新数组的长度。如果length小于original数组的长度，则新数组就是原数组的前面length个元素；如果length大于original数组的长度，则新数组的前面元素就是原数组的所有元素，后面补充0（数值类型）、false（布尔类型）或者null（引用类型）。

 type[]copyOfRange(type[]original,int from,int to)：这个方法与前面方法相似，但这个方法只复制original数组的from索引到to索引的元素。

boolean equals(type[]a,type[]a2)：如果a数组和a2数组的长度相等，而且a数组和a2数组的数组元素也一 一相同，该方法将返回true。

 void fill(type[]a,type val)：该方法将会把a数组的所有元素都赋值为val。

void fill(type[]a,int fromIndex,int toIndex,type val)：该方法与前一个方法的作用相同，区别只是该方法仅仅将a数组的fromIndex到toIndex索引的数组元素赋值为val。

 void sort(type[]a)：该方法对a数组的数组元素进行排序。

 void sort(type[]a,int fromIndex,int toIndex)：该方法与前一个方法相似，区别是该方法仅仅对fromIndex到toIndex索引的元素进行排序。

String toString(type[] a)：该方法将一个数组转换成一个字符串。该方法按顺序把多个数组元素连缀在一起，多个数组元素使用英文逗号（,）和空格隔开。

# java面向对象

# java集合

Java集合类是一种特别有用的工具类，可以用于存储数量不等的多个对象，并可以实现常用的数据结构，如栈、队列等。除此之外，Java集合还可用于保存具有映射关系的关联数组。Java集合大致可分为Set、List和Map三种体系，其中Set代表无序、不可重复的集合；List代表有序、重复的集合；而Map则代表具有映射关系的集合。从Java 5以后，Java又增加了Queue体系集合，代表一种队列集合实现。

## 概述

集合类和数组不一样，数组元素既可以是基本类型的值，也可以是对象（实际上保存的是对象的引用变量）；而集合里只能保存对象（实际上只是保存对象的引用变量，但通常习惯上认为集合里保存的是对象）。

Java的集合类主要由两个接口派生而出：Collection和Map，Collection和Map是Java集合框架的根接口，这两个接口又包含了一些子接口或实现类。（PS：图片来源于冰湖一角的博客）

![img](https://img2018.cnblogs.com/blog/1362965/201901/1362965-20190118094735724-2129767713.png)

![img](https://img2018.cnblogs.com/blog/1362965/201901/1362965-20190118095106326-273814633.png)

## Collection和iterator接口

Collection接口是List、Set和Queue接口的父接口，该接口里定义的方法既可用于操作Set集合，也可用于操作List和Queue集合。Collection接口里定义了如下操作集合元素的方法。以下为快速参考，详见API文档。

~~~java
boolean add(Object o)//该方法用于向集合里添加一个元素。如果集合对象被添加操作改变了，则返回true。
boolean addAll(Collection c)//该方法把集合c里的所有元素添加到指定集合里。如果集合对象被添加操作改变了，则返回true。
void clear()//清除集合里的所有元素，将集合长度变为0。
boolean contains(Object o)//返回集合里是否包含指定元素。
boolean containsAll(Collection c)//返回集合里是否包含集合c里的所有元素。
boolean isEmpty()//返回集合是否为空。当集合长度为0时返回true，否则返回false。
Iterator iterator()//返回一个Iterator对象，用于遍历集合里的元素。
boolean remove(Object o)//删除集合中的指定元素o，当集合中包含了一个或多个元素o时，这些元素将被删除，该方法将返回true。
boolean removeAll(Collection c)//从集合中删除集合c里包含的所有元素（相当于用调用该方法的集合减集合c），如果删除了一个或一个以上的元素，则该方法返回true。
boolean retainAll(Collection c)//从集合中删除集合c里不包含的元素（相当于把调用该方法的集合变成该集合和集合c的交集），如果该操作改变了调用该方法的集合，则该方法返回true。
int size()//该方法返回集合里元素的个数。
Object[] toArray()//该方法把集合转换成一个数组，所有集合元素变成对应的数组元素。
~~~

> 在普通情况下，当我们把一个对象“丢进”集合中后，集合会忘记这个对象的类型——也就是说，系统把所有的集合元素都当成Object类的实例进行处理。从JDK 1.5以后，这种状态得到了改进：可以使用泛型来限制集合里元素的类型，并让集合记住所有集合元素的类型。

### 使用Iterator接口遍历元素

Iterator接口隐藏了各种Collection实现类的底层细节，向应用程序提供了遍历Collection集合元素的统一编程接口。Iterator接口里定义了如下三个方法。

~~~java
boolean hasNext()//如果被迭代的集合元素还没有被遍历，则返回true。
Object next()//返回集合里的下一个元素。
void remove()//删除集合里上一次next方法返回的元素。
~~~

eg:

~~~java
public static void main(){
    Collection books=new HashSet();
    books.add("aaaaa");
    books.add("bbbbb");
    books.add("ccccc");
    Iterator it =books.iterator();
    while(it.hasNext()){
        String book =(String)it.next();//it.next()返回Object需要强转
        System.out.println(book);
        if(book.equals("aaaaa")){
            it.remove();//删除上一次next方法返回的元素
            //下面代码引发异常 iterator迭代过程中不可修改集合元素
            //books.remoce(book);
        }
        book="ceshi";//对book赋值
    }
    System.out.println(books);
}
~~~

当使用Iterator对集合元素进行迭代时，Iterator并不是把集合元素本身传给了迭代变量，而是把集合元素的值传给了迭代变量，所以修改迭代变量的值对集合元素本身没有任何影响。

Iterator迭代器采用的是快速失败（fail-fast）机制，一旦在迭代过程中检测到该集合已经被修改（通常是程序中的其他线程修改），程序立即引发ConcurrentModificationException异常，而不是显示修改后的结果，这样可以避免共享资源而引发的潜在问题。

> Iterator仅用于遍历集合，Iterator本身并不提供盛装对象的能力。如果需要创建Iterator对象，则必须有一个被迭代的集合。没有集合的Iterator仿佛无本之木，没有存在的价值。

### 使用foreach遍历元素

与使用Iterator接口迭代访问集合元素类似的是，foreach循环中的迭代变量也不是集合元素本身，系统只是依次把集合元素的值赋给迭代变量，因此在foreach循环中修改迭代变量的值也没有任何实际意义。

同样，当使用foreach循环迭代访问集合元素时，该集合也不能被改变，否则将引发ConcurrentModificationException异常。

## Set集合

Set集合与Collection基本上完全一样，它没有提供任何额外的方法。实际上Set就是Collection，只是行为略有不同（Set不允许包含重复元素）。

Set集合不允许包含相同的元素，如果试图把两个相同的元素加入同一个Set集合中，则添加操作失败，add方法返回false，且新元素不会被加入。

Set判断两个对象相同不是使用==运算符，而是根据equals方法。也就是说，只要两个对象用equals方法比较返回true，Set就不会接受这两个对象；反之，只要两个对象用equals方法比较返回false，Set就会接受这两个对象（甚至这两个对象是同一个对象，Set也可把它们当成两个对象处理，在后面程序中可以看到这种极端的情况）。

### HashSet类

HashSet是Set接口的典型实现，大多数时候使用Set集合时就是使用这个实现类。HashSet按Hash算法来存储集合中的元素，因此具有很好的存取和查找性能。

特点：

1. 不能保证元素的排列顺序，顺序有可能发生变化
2. HashSet不是同步的，如果多个线程同时访问一个HashSet，假设有两个或者两个以上线程同时修改了HashSet集合时，则必须通过代码来保证其同步。
3. 集合元素值可以是null。

当向HashSet集合中存入一个元素时，HashSet会调用该对象的hashCode()方法来得到该对象的hashCode值，然后根据该HashCode值决定该对象在HashSet中的存储位置。如果有两个元素通过equals()方法比较返回true，但它们的hashCode()方法返回值不相等，HashSet将会把它们存储在不同的位置，依然可以添加成功。

简单地说，HashSet集合判断两个元素相等的标准是两个对象通过equals()方法比较相等，并且两个对象的hashCode()方法返回值也相等。

~~~java
下面程序分别提供了三个类A、B和C，它们分别重写了equals()、hashCode()两个方法的一个或全部
class A{//类A的equals方法总是返回true；没有重写hashCode方法
    public boolean equals(Object obj){return true;}
}
class B{//类B的hashCode方法总是返回1；没有重写equals方法
    public int hashCode(){return 1;}
}
class C{//类C的hashCode方法总是返回2；且重写equals方法
    public int hashCode(){return 2;}
    public boolean equals(Object obj){return true;}
}
//main
HashSet books=new HashSet();
books.add(new A());
books.add(new A());
books.add(new B());
books.add(new B());
books.add(new C());
books.add(new C());
System.out.println(books);
~~~

运行结果：

> [b@1,B@1,C@2,A@5438cd,A@9931f5]‘

从上面程序可以看出，即使两个A对象通过equals()方法比较返回true，但HashSet依然把它们当成两个对象；即使两个B对象的hashCode()返回相同值（都是1），但HashSet依然把它们当成两个对象。这里有一个问题需要注意：当把一个对象放入HashSet中时，如果需要重写该对象对应类的equals()方法，则也应该重写其hashCode()方法。其规则是：**如果两个对象通过equals()方法比较返回true，这两个对象的hashCode值也应该相同。**

如果该对象哈希码与集合已存在对象的哈希码不一致，则该对象没有与其他对象重复，添加到集合中.如果存在于该对象相同的哈希码，那么通过equals方法判断两个哈希码相同的对象是否为同一对象（判断的标准是：属性是否相同）

**注意**

> 当把一个对象放入HashSet中时，如果需要重写该对象对应类的equals()方法，则也应该重写其hashCode()方法。其规则是：如果两个对象通过equals()方法比较返回true，这两个对象的hashCode值也应该相同。
>
> 如果两个对象通过equals()方法比较返回true，但这两个对象的hashCode()方法返回不同的hashCode值时，这将导致HashSet会把这两个对象保存在Hash表的不同位置，从而使两个对象都可以添加成功，这就与Set集合的规则有些出入了。
>
> 如果两个对象的hashCode()方法返回的hashCode值相同，但它们通过equals()方法比较返回false时将更麻烦：因为两个对象的hashCode值相同，HashSet将试图把它们保存在同一个位置，但又不行（否则将只剩下一个对象），所以实际上会在这个位置用链式结构来保存多个对象；而HashSet访问集合元素时也是根据元素的hashCode值来快速定位的，如果HashSet中两个以上的元素具有相同的hashCode值，将会导致性能下降。
>
> 所以如果需要把某个类的对象保存到HashSet集合中，重写这个类的equals()方法和hashCode()方法时，应该尽量保证两个对象通过equals()方法比较返回true时，它们的hashCode()方法返回值也相等。

HashSet中每个能存储元素的“槽位”（slot）通常称为“桶”（bucket），如果有多个元素的hashCode值相同，但它们通过equals()方法比较返回false，就需要在一个“桶”里放多个元素，这样会导致性能下降。

**重写hashCode()方法的基本规则**

1. 在程序运行过程中，同一个对象多次调用hashCode()方法应该返回相同的值。
2. 当两个对象通过equals()方法比较返回true时，这两个对象的hashCode()方法应返回相等的值。
3. 对象中用作equals()方法比较标准的Field，都应该用来计算hashCode值。

**重写hashCode()的一般规则**

1.首先计算一个int类型的哈希值计算规则：

| Boolean                           | hashCode=(f?0:1)                                             |
| --------------------------------- | :----------------------------------------------------------- |
| 整数类型(byte  short   int  char) | hashCode=(int)f                                              |
| long                              | hashCode=(int)(f^(f>>>32))                                   |
| float                             | hashCode=Float.floatToIntBits(f)                             |
| double                            | long l = Double.doubleToLongBits(f);<br/>hashCode=(int)(l^(l>>>32)) |
| 普通引用类型                      | hashCode=f.hashCode()                                        |

  2.计算出来的多个hashCode值组合计算出一个hashCode值返回

~~~java
return f1.hashCode()+(int)f2;//可能相加产生偶然相等
//为了避免直接相加产生偶然相等，可以通过为各个Field乘以任意一个质数后再相加。
return f1.hashCode()*17+(int)f2*13;
~~~

**注意**

> 当向HashSet中添加可变对象时，必须十分小心。如果修改HashSet集合中的对象，有可能导致该对象与集合中的其他对象相等，从而导致HashSet无法准确访问该对象。
>
> 比如改变了Set集合中第一个对象的count实例变量的值，这将导致该该对象与集合中的其他对象相同。这时的HashSet集合会变得十分混乱。

**底层实现**

~~~java
private transient HashMap<E, Object> map;
public HashSet() {
    this.map = new HashMap();
}
~~~

由以上源码可见HashSet底层是由HashMap实现的

### LinkedHashSet类

HashSet还有一个子类LinkedHashSet，LinkedHashSet集合也是根据元素的hashCode值来决定元素的存储位置，但它同时使用链表维护元素的次序，这样使得元素看起来是以插入的顺序保存的。

也就是说，当遍历LinkedHashSet集合里的元素时，LinkedHashSet将会按元素的添加顺序来访问集合里的元素。LinkedHashSet需要维护元素的插入顺序，因此性能略低于HashSet的性能，但在迭代访问Set里的全部元素时将有很好的性能，因为它以链表来维护内部顺序.

~~~java
LinkedHashSet books =new LinkedHashSet();
books.add("aaa");
books.add("bbb");
sout(books);
books.remove("aaa");
books.add("aaa");
sout(books);
~~~

运行结果如下

> ["aaa","bbb"]
>
> ["bbb","aaa"]

### TreeSet

TreeSet是SortedSet接口的实现类，正如SortedSet名字所暗示的，TreeSet可以确保集合元素处于排序状态。与HashSet集合相比，TreeSet还提供了如下几个额外的方法。

~~~java
Comparator comparator()：如果TreeSet采用了定制排序，则该方法返回定制排序所使用的Comparator；如果TreeSet采用了自然排序，则返回null。
Object first()：返回集合中的第一个元素。
Object last()：返回集合中的最后一个元素。
Object lower(Object e)：返回集合中位于指定元素之前的元素（即小于指定元素的最大元素，参考元素不需要是TreeSet集合里的元素）。
Object higher (Object e)：返回集合中位于指定元素之后的元素（即大于指定元素的最小元素，参考元素不需要是TreeSet集合里的元素）。
SortedSet subSet(fromElement, toElement)：返回此Set的子集合，范围从fromElement（包含）到toElement（不包含）。
SortedSet headSet(toElement)：返回此Set的子集，由小于toElement的元素组成。
SortedSet tailSet(fromElement)：返回此Set的子集，由大于或等于fromElement的元素组成。
~~~

表面上看起来这些方法很复杂，其实它们很简单：因为TreeSet中的元素是有序的，所以增加了访问第一个、前一个、后一个、最后一个元素的方法，并提供了三个从TreeSet中截取子TreeSet的方法。

~~~java
TreeSet nums=new TreeSet();
num.add(5);num.add(2);num.add(10);num.add(-9);
System.out.println(nums);
System.out.println(nums.first());
System.out.println(nums.last());
System.out.println(nums.headSet(4));
System.out.println(nums.tailSet(5));
System.out.println(nums.subSet(-3,4));
~~~

结果

> [-9,2,5,10]
>
> -9
>
> 10
>
> [-9,2]
>
> [5,10]
>
> [2]

根据上面程序的运行结果即可看出，TreeSet并不是根据元素的插入顺序进行排序的，而是根据元素实际值的大小来进行排序的。

与HashSet集合采用hash算法来决定元素的存储位置不同，TreeSet采用红黑树的数据结构来存储集合元素。那么TreeSet进行排序的规则是怎样的呢？TreeSet支持两种排序方法：自然排序和定制排序。在默认情况下，TreeSet采用自然排序。

**自然排序**

TreeSet会调用集合元素的compareTo(Object obj)方法来比较元素之间的大小关系，然后将集合元素按升序排列，这种方式就是自然排序。

Java提供了一个Comparable接口，该接口里定义了一个compareTo(Object obj)方法，该方法返回一个整数值，实现该接口的类必须实现该方法，实现了该接口的类的对象就可以比较大小。当一个对象调用该方法与另一个对象进行比较时，例如obj1.compareTo(obj2)，如果该方法返回0，则表明这两个对象相等；如果该方法返回一个正整数，则表明obj1大于obj2；如果该方法返回一个负整数，则表明obj1小于obj2。

如果试图把一个对象添加到TreeSet时，则该对象的类必须实现Comparable接口，否则程序将会抛出异常。如下程序示范了这个错误。

~~~java
class A{}
class Test{
    main(){
        TreeSet ts=new TreeSet();
        ts.add(new A());ts.add(new A());
    }
}
~~~

上面程序试图向TreeSet集合中添加两个A对象，添加第一个对象时，TreeSet里没有任何元素，所以不会出现任何问题；当添加第二个A对象时，TreeSet就会调用该对象的compareTo(Object obj)方法与集合中的其他元素进行比较——如果其对应的类没有实现Comparable接口，则会引发ClassCastException异常

而且向TreeSet中添加的应该是同一个类的对象，否则也会引发ClassCastException异常。

**对于TreeSet集合而言，它判断两个对象是否相等的唯一标准是：两个对象通过compareTo(Object obj)方法比较是否返回0——如果通过compareTo(Object obj)方法比较返回0，TreeSet则会认为它们相等；否则就认为它们不相等。**

**注意**

> 当需要把一个对象放入TreeSet中，重写该对象对应类的equals()方法时，应保证该方法与compareTo(Object obj)方法有一致的结果，其规则是：如果两个对象通过equals()方法比较返回true时，这两个对象通过compareTo(Object obj)方法比较应返回0。
>
> 如果两个对象通过compareTo(Object obj)方法比较返回0时，但它们通过equals()方法比较返回false将很麻烦，因为两个对象通过compareTo(Object obj)方法比较相等，TreeSet不会让第二个元素添加进去，这就会与Set集合的规则产生冲突。

**与HashSet类似的是，如果TreeSet中包含了可变对象，当可变对象的Field被修改时，TreeSet在处理这些对象时将非常复杂，而且容易出错。为了让程序更加健壮，推荐HashSet和TreeSet集合中只放入不可变对象。**

**定制排序**

TreeSet的自然排序是根据集合元素的大小，TreeSet将它们以升序排列。如果需要实现定制排序，例如以降序排列，则可以通过Comparator接口的帮助。该接口里包含一个int compare(T o1, T o2)方法，该方法用于比较o1和o2的大小：如果该方法返回正整数，则表明o1大于o2；如果该方法返回0，则表明o1等于o2；如果该方法返回负整数，则表明o1小于o2。

~~~java
class M{
    int age;
    public M(int age){
        this.age=age;
    }
    public String toString(){.....}
}
class Test{
    public static void main(){
        TreeSet ts =new TreeSet(new Comparator(){
            //根据对象age决定大小
           public int compare(Object o1,Object o2){
               M m1=(M)o1; M m2=(M)o2;
               return m1.age>m2.age?-1:m1.age<m2.age?1:0;
           } 
        });
        ts.add(new M(5)); ts.add(new M(-9)); ts.add(new M(3));
        sout(ts);
    }
}
~~~

结果：

> [M对象(age:5),M对象(age:3),M对象(age:-9)]

### EnumSet类

EnumSet是一个专为枚举类设计的集合类，EnumSet中的所有元素都必须是指定枚举类型的枚举值，该枚举类型在创建EnumSet时显式或隐式地指定。EnumSet的集合元素也是有序的，EnumSet以枚举值在Enum类内的定义顺序来决定集合元素的顺序。

EnumSet集合不允许加入null元素，如果试图插入null元素，EnumSet将抛出NullPointerException异常。如果只是想判断EnumSet是否包含null元素或试图删除null元素都不会抛出异常，只是删除操作将返回false，因为没有任何null元素被删除。

### 各Set的性能分析

HashSet和TreeSet是Set的两个典型实现，到底如何选择HashSet和TreeSet呢？HashSet的性能总是比TreeSet好（特别是最常用的添加、查询元素等操作），因为TreeSet需要额外的红黑树算法来维护集合元素的次序。只有当需要一个保持排序的Set时，才应该使用TreeSet，否则都应该使用HashSet。

HashSet还有一个子类：LinkedHashSet，对于普通的插入、删除操作，LinkedHashSet比HashSet要略微慢一点，这是由维护链表所带来的额外开销造成的；不过，因为有了链表，遍历LinkedHashSet会更快。

## List集合

### List接口和ListIterator接口

List作为Collection接口的子接口，当然可以使用Collection接口里的全部方法。而且由于List是有序集合，因此List集合里增加了一些根据索引来操作集合元素的方法。

~~~java
void add(int index, Object element)：将元素element插入到List集合的index处。
boolean addAll(int index, Collection c)：将集合c所包含的所有元素都插入到List集合的index处。
Object get(int index)：返回集合index索引处的元素。
int indexOf(Object o)：返回对象o在List集合中第一次出现的位置索引。
int lastIndexOf(Object o)：返回对象o在List集合中最后一次出现的位置索引。
Object remove(int index)：删除并返回index索引处的元素。
Object set(int index, Object element)：将index索引处的元素替换成element对象，返回新元素。
List subList(int fromIndex, int toIndex)：返回从索引fromIndex（包含）到索引toIndex（不包含）处所有集合元素组成的子集合。
~~~

所有的List实现类都可以调用这些方法来操作集合元素。与Set集合相比，List增加了根据索引来插入、替换和删除集合元素的方法。

~~~java
List books =new ArrayList();
books.add(new String("aaa"));books.add(new String("bbb"));book.add(new String("ccc"));
System.out.println(books);
//在第二个位置插入
book,add(1,new String("ddd"));
for(int i=0;i<books.size();i++){
    System.out.println(books.get(i));
}
books.remove(2);
System.out.println(books);
//输出1表明在第二个位置
System.out.println(books.indexOf("ddd"));//1
books.set(1,new String("bbb"));
System.out.println(books);
System.out.println(books.subList(1,2));
~~~

结果：

> [aaa,bbb,ccc]
>
> aaa
>
> ddd
>
> bbb
>
> ccc
>
> [aaa,ddd,ccc]
>
> 1
>
> [aaa,bbb,ccc]
>
> [bbb]

从上面运行结果清楚地看出List集合的用法。注意1行代码处，程序试图返回新字符串对象在List集合中的位置，实际上List集合中并未包含该字符串对象。因为List集合添加字符串对象时，添加的是通过new关键字创建的新字符串对象，1行代码处也是通过new关键字创建的新字符串对象，两个字符串显然不是同一个对象，但List的indexOf方法依然可以返回1。List判断两个对象相等的标准是什么呢？List判断两个对象相等只要通过equals()方法比较返回true即可。比如：

~~~java
class A{ public boolean equals(Object obj){return true;}}
List books =new ArrayList();
books.add(new String("aaa"));books.add(new String("bbb"));book.add(new String("ccc"));
books.remove(new A());//1
~~~

执行①行代码时，程序试图删除一个A对象，List将会调用该A对象的equals()方法依次与集合元素进行比较，如果该equals()方法以某个集合元素作为参数时返回true，List将会删除该元素——A类重写了equals()方法，该方法总是返回true。所以我们看到每次从List集合中删除A对象，总是删除List集合中的第一个元素。

**遍历元素**

与Set只提供了一个iterator()方法不同，List还额外提供了一个listIterator()方法，该方法返回一个ListIterator对象，ListIterator接口继承了Iterator接口，提供了专门操作List的方法。ListIterator接口在Iterator接口基础上增加了如下方法。

boolean hasPrevious()：返回该迭代器关联的集合是否还有上一个元素。

Object previous()：返回该迭代器的上一个元素。

void add()：在指定位置插入一个元素。

ListIterator增加了向前迭代的功能（Iterator只能向后迭代），而且ListIterator还可通过add方法向List集合中添加元素（Iterator只能删除元素）。

~~~java
List books =new ArrayList();
books.add(new String("aaa"));books.add(new String("bbb"));
ListIterator lit=books.listIterator();
while(lit.hasNext()){
    sout(lit.next());
    lit.add("xxxxxxxxxx");
}
sout("反向迭代-------------");
while(lit.hasPrevious()){
    sout(lit.previous());
}
~~~

使用ListIterator迭代List集合时，开始也需要采用正向迭代，即先使用next()方法进行迭代，在迭代过程中可以使用add()方法向上一次迭代元素的后面添加一个新元素。结果:

> aaa
>
> bbb
>
> 反向迭代-------------
>
> xxxxxxxxxx
>
> bbb
>
> xxxxxxxxxx
>
> aaa

### ArrayList和Vector实现类

ArrayList和Vector类都是基于数组实现的List类，所以ArrayList和Vector类封装了一个动态的、允许再分配的Object[]数组。从源码（Java8）看，默认大小是10.

~~~java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = new Object[0];
~~~

ArrayList或Vector对象使用initialCapacity参数来设置该数组的长度，当向ArrayList或Vector中添加元素超出了该数组的长度时，它们的initialCapacity会自动增加。ArrayList和Vector还提供了如下两个方法来重新分配Object[]数组。

> void ensureCapacity(int minCapacity)：将ArrayList或Vector集合的Object[]数组长度增加minCapacity。
>
> void trimToSize()：调整ArrayList或Vector集合的Object[]数组长度为当前元素的个数。程序可调用该方法来减少ArrayList或Vector集合对象占用的存储空间。

ArrayList和Vector在用法上几乎完全相同，但由于Vector是一个古老的集合（从JDK 1.0就有了），那时候Java还没有提供系统的集合框架，所以Vector里提供了一些方法名很长的方法，例如addElement(Object obj)，实际上这个方法与add (Object obj)没有任何区别

ArrayList和Vector的显著区别是：ArrayList是线程不安全的，当多个线程访问同一个ArrayList集合时，如果有超过一个线程修改了ArrayList集合，则程序必须手动保证该集合的同步性；但Vector集合则是线程安全的，无须程序保证该集合的同步性。因为Vector是线程安全的，所以Vector的性能比ArrayList的性能要低。

~~~java
public synchronized E firstElement() {
        if (this.elementCount == 0) {
            throw new NoSuchElementException();
        } else {
            return this.elementData(0);
        }
    }
~~~

由上面源码可见Vector是由synchronized实现同步。

实际上，即使需要保证List集合线程安全，也同样不推荐使用Vector实现类。有一个Collections工具类，它可以将一个ArrayList变成线程安全的。

### LinkedList实现类

~~~java
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, Serializable 
~~~

以上为LinkedList的源码（Java8）可见该类实现了List和Deque接口，所以详见下面Deque接口。

### 固定长度的List

操作数组的工具类：Arrays，该工具类里提供了asList(Object... a)方法，该方法可以把一个数组或指定个数的对象转换成一个List集合，这个List集合既不是ArrayList实现类的实例，也不是Vector实现类的实例，而是Arrays的内部类ArrayList的实例。

~~~java
@SafeVarargs
public static <T> List<T> asList(T... var0) {
    return new Arrays.ArrayList(var0);//1
}
上面1位置的这个类是在Arrays类中的内部类见下面
private static class ArrayList<E> extends AbstractList<E> implements RandomAccess, Serializable {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;

        ArrayList(E[] var1) {
            this.a = (Object[])Objects.requireNonNull(var1);
        }

        public int size() {
            return this.a.length;
        }
        //。。。。。。
    }
~~~

Arrays.ArrayList是一个固定长度的List集合，程序只能遍历访问该集合里的元素，不可增加、删除该集合里的元素。试图增加删除元素都会引发UnsupportedOperationException异常。

## Queue

Queue用于模拟队列这种数据结构，队列的头部保存存放时间最长的元素，队列的尾部保存存放时间最短的元素。queue接口中定义了如下几个方法：

~~~java
void add(Object e):将指定元素加入到队列的尾部
Object element():获取队列头部的元素，但是不删除元素
boolean offer(Object e):将指定元素加入到队列的尾部。当时拥有容量限制的队列时，此方法只会返回false，而add会抛出IllegalStateException异常。
Object peek():获取队列头部的元素，但是不删除元素，如果队列为空，则返回null
Object poll():获取队列头部的元素，并删除元素。如果队列为空，则返回null
Object remove():获取队列头部的元素，并删除元素。

~~~

### PriorityQueue实现类

PriorityQueue是一个比较标准的队列实现类。之所以说它是比较标准的队列实现，而不是绝对标准的队列实现，是因为PriorityQueue保存队列元素的顺序并不是按加入队列的顺序，而是按队列元素的大小进行重新排序。因此当调用peek()方法或者poll()方法取出队列中的元素时，并不是取出最先进入队列的元素，而是取出队列中最小的元素。从这个意义上来看，PriorityQueue已经违反了队列的最基本规则：先进先出（FIFO）。

~~~java
    public static void main(String []args){
        Queue queue = new PriorityQueue();
        queue.add(30);
        queue.offer(15);
        queue.add(6);
        queue.offer(52);
        queue.add(-9);
        System.out.println(queue);
    }
运行结果：
[-9, 6, 15, 52, 30]
~~~

**PriorityQueue不允许插入null元素**，它还需要对队列元素进行排序，PriorityQueue的元素有两种排序方式。自然排序、定制排序。PriorityQueue队列对元素的要求与TreeSet对元素的要求基本一致。

### Deque接口

Deque接口是Queue接口的子接口，它代表一个双端队列，Deque接口里定义了一些双端队列的方法，这些方法允许从两端来操作队列的元素。

~~~java
void addFirst(Object e)：将指定元素插入该双端队列的开头。
void addLast(Object e)：将指定元素插入该双端队列的末尾。
Iterator descendingIterator()：返回该双端队列对应的迭代器，该迭代器将以逆向顺序来迭代队列中的元素。
Object getFirst()：获取但不删除双端队列的第一个元素。
Object getLast()：获取但不删除双端队列的最后一个元素。
boolean offerFirst(Object e)：将指定元素插入该双端队列的开头。
boolean offerLast(Object e)：将指定元素插入该双端队列的末尾。
Object peekFirst()：获取但不删除该双端队列的第一个元素；如果此双端队列为空，则返回null。
Object peekLast()：获取但不删除该双端队列的最后一个元素；如果此双端队列为空，则返回null。
Object pollFirst()：获取并删除该双端队列的第一个元素；如果此双端队列为空，则返回null。
Object pollLast()：获取并删除该双端队列的最后一个元素；如果此双端队列为空，则返回null。
Object pop()（栈方法）：pop出该双端队列所表示的栈的栈顶元素。相当于removeFirst()。
void push(Object e)（栈方法）：将一个元素push进该双端队列所表示的栈的栈顶。相当于addFirst(e)。
Object removeFirst()：获取并删除该双端队列的第一个元素。
Object removeFirstOccurrence(Object o)：删除该双端队列的第一次出现的元素o。
removeLast()：获取并删除该双端队列的最后一个元素。
removeLastOccurrence(Object o)：删除该双端队列的最后一次出现的元素o。
~~~

从上面方法中可以看出，Deque不仅可以当成双端队列使用，而且可以被当成栈来使用，因为该类里还包含了pop（出栈）、push（入栈）两个方法。

**Deque方法与Queue方法对照**

| Queue的方法      | Deque的方法               |
| ---------------- | ------------------------- |
| add(e)/offer(e)  | addLast(e)/offerLast(e)   |
| remove()/pool()  | removeFirst()/poolFirst() |
| element()/peek() | getFirst()/peekFirst()    |

**Deque方法与Stack方法对照**

| Stack的方法 | Deque的方法               |
| ----------- | ------------------------- |
| push(e)     | addFirst(e)/offerFirst(e) |
| pop()       | removeFirst()/poolFirst() |
| peek()      | getFirst()/peekFirst()    |

### Deque接口与ArrayDeque实现类

Deque接口提供了一个典型的实现类：ArrayDeque，从该名称就可以看出，它是一个基于数组实现的双端队列，创建Deque时同样可指定一个numElements参数，该参数用于指定Object[]数组的长度；如果不指定numElements参数，Deque底层数组的长度为16。以下为ArrayDeque的构造函数源码（Java8）

~~~java
    public ArrayDeque() {
        this.elements = new Object[16];
    }
~~~

以下示例将ArrayDeque用作栈来使用：

~~~java
public static void main(String[] args) {
        ArrayDeque stack = new ArrayDeque();
        stack.push("aaa");
        stack.push("bbb");
        stack.push("ccc");
        //输出 ccc,bbb,aaa
        System.out.println(stack);
        System.out.println(stack.peek());
        System.out.println(stack);
        System.out.println(stack.pop());
        System.out.println(stack);
    }
~~~

使用ArrayDeque的性能会更加出色，因此现在的程序中需要使用“栈”这种数据结构时，推荐使用ArrayDeque或LinkedList，而不是Stack。

### Deque接口与LinkedList实现类

LinkedList类是List接口的实现类——这意味着它是一个List集合，可以根据索引来随机访问集合中的元素。除此之外，LinkedList还实现了Deque接口，因此它可以被当成双端队列来使用，自然也可以被当成“栈”来使用了。

简单示例：

~~~java
public static void main(String[] args) {
        LinkedList linkedList = new LinkedList();
        linkedList.offer("aaa");//加入队列尾部
        linkedList.push("bbb");//加入栈的顶部
        linkedList.offerFirst("ccc");//加入到队列的头部（相当于栈的顶部）
        for (int i = 0; i <linkedList.size() ; i++) {
            System.out.println(linkedList.get(i));
        }
        System.out.println(linkedList.peekFirst());
        System.out.println(linkedList.peekLast());
        System.out.println(linkedList.pop());
        System.out.println(linkedList);
        System.out.println(linkedList.pollLast());
        System.out.println(linkedList);
    }
~~~

运行结果：

> ccc
> bbb
> aaa
> ccc
> aaa
> ccc
> [bbb, aaa]
> aaa
> [bbb]

> LinkedList与ArrayList、ArrayDeque的实现机制完全不同，ArrayList、ArrayDeque内部以数组的形式来保存集合中的元素，因此随机访问集合元素时有较好的性能；而LinkedList内部以链表的形式来保存集合中的元素，因此随机访问集合元素时性能较差，但在插入、删除元素时性能非常出色（只需改变指针所指的地址即可）。需要指出的是，虽然Vector也是以数组的形式来存储集合元素的，但因为它实现了线程同步功能，所以各方面性能都有所下降。

## Map

Map用于保存具有映射关系的数据，因此Map集合里保存着两组值，一组值用于保存Map里的key，另外一组值用于保存Map里的value，key和value都可以是任何引用类型的数据。Map的key不允许重复，即同一个Map对象的任何两个key通过equals方法比较总是返回false。

**Map与Set关系**

> Set与Map之间的关系非常密切。虽然Map中放的元素是key-value对，Set集合中放的元素是单个对象，但如果我们把key-value对中的value当成key的附庸：key在哪里，value就跟在哪里。这样就可以像对待Set一样来对待Map了。事实上，Map提供了一个Entry内部类来封装key-value对，而计算Entry存储时则只考虑Entry封装的key。从Java源码来看， Java是先实现了Map，然后通过包装一个所有value都为null的Map就实现了Set集合。

如果把Map里的所有value放在一起来看，它们又非常类似于一个List：元素与元素之间可以重复，每个元素可以根据索引来查找，只是Map中的索引不再使用整数值，而是以另一个对象作为索引。如果需要从List集合中取出元素，则需要提供该元素的数字索引；如果需要从Map中取出元素，则需要提供该元素的key索引。因此，Map有时也被称为字典，或关联数组。Map接口中定义了如下常用的方法。

~~~java
void clear()：删除该Map对象中的所有key-value对。
boolean containsKey(Object key)：查询Map中是否包含指定的key，如果包含则返回true。
boolean containsValue(Object value)：查询Map中是否包含一个或多个value，如果包含则返回true。
Set entrySet()：返回Map中包含的key-value对所组成的Set集合，每个集合元素都是Map.Entry （Entry是Map的内部类）对象。
Object get(Object key)：返回指定key所对应的value；如果此Map中不包含该key，则返回null。
boolean isEmpty()：查询该Map是否为空（即不包含任何key-value对），如果为空则返回true。
Set keySet()：返回该Map中所有key组成的Set集合。
Object put(Object key, Object value)：添加一个key-value对，如果当前Map中已有一个与该key相等的key-value对，则新的key-value对会覆盖原来的key-value对。
void putAll(Map m)：将指定Map中的key-value对复制到本Map中。
Object remove(Object key)：删除指定key所对应的key-value对，返回被删除key所关联的value，如果该key不存在，则返回null。
int size()：返回该Map里的key-value对的个数。
Collection values()：返回该Map里所有value组成的Collection。
~~~

Map中包括一个内部类Entry，该类封装了一个key-value对。Entry包含如下三个方法。

Object getKey()：返回该Entry里包含的key值。

Object getValue()：返回该Entry里包含的value值。

Object setValue(V value)：设置该Entry里包含的value值，并返回新设置的value值。

*所有的Map实现类都重写了toString()方法，调用Map对象的toString()方法总是返回如下格式的字符串：{key1=value1,key2=value2...}。*

### HashMap和Hashtable实现类

HashMap和Hashtable都是Map接口的典型实现类，它们之间的关系完全类似于ArrayList和Vector的关系：Hashtable是一个古老的Map实现类，它从JDK 1.0起就已经出现了，当它出现时，Java还没有提供Map接口，所以它包含了两个烦琐的方法，即elements()（类似于Map接口定义的values()方法）和keys()（类似于Map接口定义的keySet()方法），现在很少使用这两个方法

区别：

> Hashtable是一个线程安全的Map实现，但HashMap是线程不安全的实现，所以HashMap比Hashtable的性能高一点；但如果有多个线程访问同一个Map对象时，使用Hashtable实现类会更好。
>
>  Hashtable不允许使用null作为key和value，如果试图把null值放进Hashtable中，将会引发NullPointerException异常；但HashMap可以使用null作为key或value。

由于HashMap里的key不能重复，所以HashMap里最多只有一个key-value对的key为null，但可以有无数多个key-value对的value为null。例如：

~~~java
main(){
    HashMap hm=new HashMap();
    hm.put(null,null);
    //1处无法将key-value对放入，因为Map中已经有一个key-value对的key为null值
    hm.put(null,null);//1
    //2代码处可以放入该key-value对，因为一个HashMap中可以有多个value为null值
    hm.put("a",null);//2
    sout(hm);
}
~~~

为了成功地在HashMap、Hashtable中存储、获取对象，用作key的对象必须实现hashCode()方法和equals()方法。与HashSet集合不能保证元素的顺序一样，HashMap、Hashtable也不能保证其中key-value对的顺序。

**类似于HashSet，HashMap、Hashtable判断两个key相等的标准也是：两个key通过equals()方法比较返回true，两个key的hashCode值也相等。**

与HashSet类似的是，如果使用可变对象作为HashMap、Hashtable的key，并且程序修改了作为key的可变对象，则也可能出现与HashSet类似的情形：程序再也无法准确访问到Map中被修改过的key。

遍历

遍历Map中的全部key-value对：调用Map对象的keySet()方法返回全部key组成的Set集合，通过遍历该Set集合的所有元素（就是Map的全部key）就可以遍历Map中的所有key-value对。

~~~java
for(Object key:hm.keySet()){
    sout(key);
    sout(ht.get(key));
}
~~~

### LinkedHashMap实现类

HashSet有一个子类是LinkedHashSet，HashMap也有一个LinkedHashMap子类；LinkedHashMap也使用双向链表来维护key-value对的次序（其实只需要考虑key的次序），该链表负责维护Map的迭代顺序，迭代顺序与key-value对的插入顺序保持一致。

LinkedHashMap可以避免对HashMap、Hashtable里的key-value对进行排序（只要插入key-value对时保持顺序即可），同时又可避免使用TreeMap所增加的成本。

~~~java
LinkedHashMap scores=new LinkedHashMap();
scores.put("语文",80);
scores.put("数学",70);
scores.put("英语",90);
for(Object key :scores.keySet()){
    sout(key+"--->"+scores.get(key));
}
~~~

### 使用Properties读写属性文件

Properties类是Hashtable类的子类，正如它的名字所暗示的，该对象在处理属性文件时特别方便（Windows操作平台上的ini文件就是一种属性文件）。Properties类可以把Map对象和属性文件关联起来，从而可以把Map对象中的key-value对写入属性文件中，也可以把属性文件中的“属性名=属性值”加载到Map对象中。由于属性文件里的属性名、属性值只能是字符串类型，所以Properties里的key、value都是字符串类型。该类提供了如下三个方法来修改Properties里的key、value值。

~~~java
String getProperty(String key)//获取Properties中指定属性名对应的属性值，类似于Map的get(Object key)方法。
String getProperty(String key, String defaultValue)//该方法与前一个方法基本相似。该方法多一个功能，如果Properties中不存在指定的key时，则该方法指定默认值。
Object setProperty(String key, String value)//设置属性值，类似于Hashtable的put()方法。
~~~

除此之外，它还提供了两个读写Field文件的方法。

~~~java
void load(InputStream inStream)//从属性文件（以输入流表示）中加载key-value对，把加载到的key-value对追加到Properties里（Properties是Hashtable的子类，它不保证key-value对之间的次序）。
void store(OutputStream out, String comments)//将Properties中的key-value对输出到指定的属性文件（以输出流表示）中。
~~~

示例：

~~~java
Properties props =new Properties();
props.setProperty("username","haha");
props.setProperty("password","123456");
props.store(new FileOutPutStream("a.ini"),"comment line");//1
Properties props2 =new Properties();
props2.setProperty("gender","male");
props2.load(new FileInputStream("a.ini"))//2
sout(props2)
~~~

其中①代码处将Properties对象中的key-value对写入a.ini文件中；②代码处则从a.ini文件中读取key-value对，并添加到props2对象中.

Properties可以把key-value对以XML文件的形式保存起来，也可以从XML文件中加载key-value对，用法与此类似，此处不再赘述。

### SortedMap接口和TreeMap实现类

正如Set接口派生出SortedSet子接口，SortedSet接口有一个TreeSet实现类一样，Map接口也派生出一个SortedMap子接口，SortedMap接口也有一个TreeMap实现类。

TreeMap就是一个红黑树数据结构，每个key-value对即作为红黑树的一个节点。TreeMap存储key-value对（节点）时，需要根据key对节点进行排序。TreeMap可以保证所有的key-value对处于有序状态。TreeMap也有两种排序方式。

自然排序：TreeMap的所有key必须实现Comparable接口，而且所有的key应该是同一个类的对象，否则将会抛出ClassCastException异常。

 定制排序：创建TreeMap时，传入一个Comparator对象，该对象负责对TreeMap中的所有key进行排序。采用定制排序时不要求Map的key实现Comparable接口。

TreeSet类似的是，TreeMap中也提供了一系列根据key顺序访问key-value对的方法。

~~~java
Map.Entry firstEntry()//返回该Map中最小key所对应的key-value对，如果该Map为空，则返回null。
Object firstKey()//返回该Map中的最小key值，如果该Map为空，则返回null。
Map.Entry lastEntry()//返回该Map中最大key所对应的key-value对，如果该Map为空或不存在这样的key-value对，则都返回null。
Object lastKey()//返回该Map中的最大key值，如果该Map为空或不存在这样的key，则都返回null。
Map.Entry higherEntry(Object key)//返回该Map中位于key后一位的key-value对（即大于指定key的最小key所对应的key-value对）。如果该Map为空，则返回null。
Object higherKey(Object key)//返回该Map中位于key后一位的key值（即大于指定key的最小key值）。如果该Map为空或不存在这样的key-value对，则都返回null。
Map.Entry lowerEntry(Object key)//返回该Map中位于key前一位的key-value对（即小于指定key的最大key所对应的key-value对）。如果该Map为空或不存在这样的key-value对，则都返回null。
Object lowerKey(Object key)//返回该Map中位于key前一位的key值（即小于指定key的最大key值）。如果该Map为空或不存在这样的key，则都返回null。
NavigableMap subMap(Object fromKey, boolean fromInclusive, Object toKey,boolean toInclusive)//返回该Map的子Map，其key的范围是从fromKey（是否包括取决于第二个参数）到toKey（是否包括取决于第四个参数）。
SortedMap subMap(Object fromKey, Object toKey)//返回该Map的子Map，其key的范围是从fromKey（包括）到toKey（不包括）。
SortedMap tailMap(Object fromKey)//返回子Map，其key的范围是大于fromKey（包括）的所有key
NavigableMap tailMap(Object fromKey, boolean inclusive)//返回该Map的子Map，其key的范围是大于fromKey（是否包括取决于第二个参数）的所有key。
SortedMap headMap(Object toKey)//返回该Map的子Map，其key的范围是小于toKey（不包括）的所有key。
NavigableMap headMap(Object toKey, boolean inclusive)//返回该Map的子Map，其key的范围是小于toKey（是否包括取决于第二个参数）的所有key。
~~~

因为TreeMap中的key-value对是有序的，所以增加了访问第一个、前一个、后一个、最后一个key-value对的方法，并提供了几个从TreeMap中截取子TreeMap的方法。

**类似于TreeSet中判断两个元素相等的标准，TreeMap中判断两个key相等的标准是：两个key通过compareTo()方法返回0，TreeMap即认为这两个key是相等的。**

**注意**

> 再次强调：Set和Map的关系十分密切，Java源码就是先实现了HashMap、TreeMap等集合，然后通过包装一个所有的value都为null的Map集合实现了Set集合类。

### 各种实现类的分析

对于Map的常用实现类而言，HashMap和Hashtable的效率大致相同，因为它们的实现机制几乎完全一样；但HashMap通常比Hashtable要快一点，因为Hashtable需要额外的线程同步控制。

TreeMap通常比HashMap、Hashtable要慢（尤其在插入、删除key-value对时更慢），因为TreeMap底层采用红黑树来管理key-value对（红黑树的每个节点就是一个key-value对）。

使用TreeMap有一个好处：TreeMap中的key-value对总是处于有序状态，无须专门进行排序操作。当TreeMap被填充之后，就可以调用keySet()，取得由key组成的Set，然后使用toArray()方法生成key的数组，接下来使用Arrays的binarySearch()方法在已排序的数组中快速地查询对象。

对于一般的应用场景，程序应该多考虑使用HashMap，因为HashMap正是为快速查询设计的（HashMap底层其实也是采用数组来存储key-value对）。但如果程序需要一个总是排好序的Map时，则可以考虑使用TreeMap。LinkedHashMap比HashMap慢一点，因为它需要维护链表来保持Map中key-value时的添加顺序。IdentityHashMap性能没有特别出色之处，因为它采用与HashMap基本相似的实现，只是它使用==而不是equals()方法来判断元素相等。

## 操作集合的工具类：Collections

Java提供了一个操作Set、List和Map等集合的工具类：Collections，该工具类里提供了大量方法对集合元素进行排序、查询和修改等操作，还提供了将集合对象设置为不可变、对集合对象实现同步控制等方法。

### 排序操作

static void reverse(List list)：反转指定List集合中元素的顺序。

static void shuffle(List list)：对List集合元素进行随机排序（shuffle方法模拟了“洗牌”动作）。

static void sort(List list)：根据元素的自然顺序对指定List集合的元素按升序进行排序。

static void sort(List list, Comparator c)：根据指定Comparator产生的顺序对List集合元素进行排序。

static void swap(List list, int i, int j)：将指定List集合中的i处元素和j处元素进行交换。

static void rotate(List list , int distance)：当distance为正数时，将list集合的后distance个元素“整体”移到前面；当distance为负数时，将list集合的前distance个元素“整体”移到后面。该方法不会改变集合的长度。

~~~java
ArrayList nums=new ArrayList();
nums.add(2);nums.add(-5);nums.add(3);nums.add(0);
sout(nums);
Collections.reverse(nums);//反转
sout(nums);
Collections.sort(nums);//排序
sout(nums);
Collections.shuffle(nums);//随机排序
sout(nums);
~~~

### 查找替换操作

Collections还提供了如下用于查找、替换集合元素的常用方法。

static int binarySearch(List list, Object key)：使用二分搜索法搜索指定的List集合，以获得指定对象在List集合中的索引。如果要使该方法可以正常工作，则必须保证List中的元素已经处于有序状态。

static Object max(Collection coll)：根据元素的自然顺序，返回给定集合中的最大元素。

static Object max(Collection coll, Comparator comp)：根据Comparator指定的顺序，返回给定集合中的最大元素。

static Object min(Collection coll)：根据元素的自然顺序，返回给定集合中的最小元素。

static Object min(Collection coll, Comparator comp)：根据Comparator指定的顺序，返回给定集合中的最小元素。

static void fill(List list, Object obj)：使用指定元素obj替换指定List集合中的所有元素。

static int frequency(Collection c, Object o)：返回指定集合中指定元素的出现次数。

static int indexOfSubList(List source, List target)：返回子List对象在父List对象中第一次出现的位置索引；如果父List中没有出现这样的子List，则返回-1。

static int lastIndexOfSubList(List source, List target)：返回子List对象在父List对象中最后一次出现的位置索引；如果父List中没有出现这样的子List，则返回-1。

static boolean replaceAll(List list, Object oldVal, Object newVal)：使用一个新值newVal替换List对象的所有旧值oldVal。

### 同步控制

Collections类中提供了多个synchronizedXxx()方法，该方法可以将指定集合包装成线程同步的集合，从而可以解决多线程并发访问集合时的线程安全问题。

Java中常用的集合框架中的实现类HashSet、TreeSet、ArrayList、ArrayDeque、LinkedList、HashMap和TreeMap都是线程不安全的。如果有多个线程访问它们，而且有超过一个的线程试图修改它们，则可能出现错误。Collections提供了多个静态方法可以把它们包装成线程同步的集合。

~~~java
public static void main(){
    Collection c =Collections.synchronizedCollection(new ArrayList());
    List list =Collections.synchronizedList(new ArrayList());
    Set s=Collections.synchronizedSet(new HashSet());
    Map m=Collections.synchronizedLMap(new HashMap();
}
~~~

将新创建的集合对象传给了Collections的synchronizedXxx方法，这样就可以直接获取List、Set和Map的线程安全实现版本。

### 设置不可变集合

Collections提供了如下三类方法来返回一个不可变的集合。

emptyXxx()：返回一个空的、不可变的集合对象，此处的集合既可以是List，也可以是Set，还可以是Map。

singletonXxx()：返回一个只包含指定对象（只有一个或一项元素）的、不可变的集合对象，此处的集合既可以是List，也可以是Set，还可以是Map。

unmodifiableXxx：返回指定集合对象的不可变视图，此处的集合既可以是List，也可以是Set，还可以是Map。

上面三类方法的参数是原有的集合对象，返回值是该集合的“只读”版本。通过Collections提供的三类方法，可以生成“只读”的Collection或Map。

~~~java
List unmodifiableList=Collections.emptyList();//创建一个空的、不可改变的List对象
Set unmodifiableSet=Collections.singleton("aaa");//创建一个只有一个元素，且不可改变的Set对象
Map scores = new HashMap();
spours.put("yuwen",90);spours.put("yingyu",80);
Map unmodifiableMap=Collections.unmodifiableMap(scores);
unmodifiableList.add("xxx");
unmodifiableSet.add("xxx");
unmodifiableMap.put("yuwen",90);
~~~

上面程序的3行粗体字代码分别定义了一个空的、不可变的List对象，一个只包含一个元素的、不可变的Set对象和一个不可变的Map对象。不可变的集合对象只能访问集合元素，不可修改集合元素。所以上面程序中①②③处的代码都将引发UnsupportedOperationException异常。

# Java泛型

此篇博客用sout代替System.out.pringln();

## 概述

Java集合有个缺点——当我们把一个对象“丢进”集合里后，集合就会“忘记”这个对象的数据类型，当再次取出该对象时，该对象的编译类型就变成了Object类型（其运行时类型没变）。

Java集合之所以被设计成这样，是因为设计集合的程序员不会知道我们用它来保存什么类型的对象，所以他们把集合设计成能保存任何类型的对象，只要求具有很好的通用性。但这样做带来如下两个问题：

集合对元素类型没有任何限制，这样可能引发一些问题。例如，想创建一个只能保存Dog对象的集合，但程序也可以轻易地将Cat对象“丢”进去，所以可能引发异常。

由于把对象“丢进”集合时，集合丢失了对象的状态信息，集合只知道它盛装的是Object，因此取出集合元素后通常还需要进行强制类型转换。这种强制类型转换既增加了编程的复杂度，也可能引发ClassCastException异常。

**编译时不检查类型可能引发的异常**：

~~~java
main(){
    List strList=new ArrayList();
    strList.add("aaa");strList.add("aaa");strList.add("aaa");
    strList.add(5);//1
    for(int i=0;i<strList.size();i++){
        String str =(String)strList.get(i);//2
    }
}
~~~

上面程序创建了一个List集合，而且只希望该List集合保存字符串对象——但我们没有办法进行任何限制，如果程序在①处“不小心”把一个Integer对象“丢进”了List集合中，这将导致程序在②处引发ClassCastException异常，因为程序试图把一个Integer对象转换为String类型。

**手动实现编译时检查类型**：

~~~java
class StrList{
    private List strList=new ArrayList();
    public boolean add(String ele){
        return  strList.add(ele);
    }
    public String get(int index){
        return (String)strList.get(index);
    }
    public int size(){
        return strList.size();
    }
}
public static void main(String[] args) {
        StrList strList=new StrList();
        strList.add("aaa");
        strList.add("bbb");
        strList.add("ccc");
        strList.add(5);
        System.out.println(strList);
        for (int i = 0; i <strList.size() ; i++) {
            String str=strList.get(i);
        }
    }
~~~

这种做法虽然有效，但局限性非常明显——程序员需要定义大量的List子类，这是一件让人沮丧的事情。从Java 5以后，Java引入了“参数化类型（parameterized type）”的概念，允许我们在创建集合时指定集合元素的类型，正如List< String>，这表明该List只能保存字符串类型的对象。Java的参数化类型被称为泛型（Generic）。

## 使用泛型

~~~java
List<String> stringList =new ArrayList<String>();//1
~~~

上面程序成功创建了一个特殊的List集合：strList，这个List集合只能保存字符串对象，不能保存其他类型的对象。创建这种特殊集合的方法是：在集合接口、类后增加尖括号，尖括号里放一个数据类型，即表明这个集合接口、集合类只能保存特定类型的对象。注意①处的类型声明，它指定strList不是一个任意的List，而是一个String类型的List，写作：List<String>。

## 泛型的菱形语法

在Java 7以前，如果使用带泛型的接口、类定义变量，那么调用构造器创建对象时构造器的后面也必须带泛型，这显得有些多余了。例如如下两条语句：

~~~java
List<String> strList=new ArrayList<String>();
Map<String,Integer> scores=new HashMap<String,Integer>();
~~~

上面两条语句中后面的方括号里的字完全是多余的，在Java 7以前这是必需的，不能省略。从Java7开始，Java允许在构造器后不需要带完整的泛型信息，只要给出一对尖括号（<>）即可，Java可以推断尖括号里应该是什么泛型信息。即上面两条语句可以改写为如下形式：

List<String> strList=new ArrayList<>();

Map<String , Integer> scores=new HashMap<>();

## 深入泛型

所谓泛型，就是允许在定义类、接口、方法时使用类型形参，这个类型形参将在声明变量、创建对象、调用方法时动态地指定（即传入实际的类型参数，也可称为类型实参）。Java 5改写了集合框架中的全部接口和类，为这些接口、类增加了泛型支持，从而可以在声明集合变量、创建集合对象时传入类型实参，这就是在前面程序中看到的List<String>和ArrayList<String>两种类型。

### 定义泛型接口、类

Java 5改写后List接口、Iterator接口、Map的代码片段:

~~~java
public interface List<E>{
    void add(E,x);
    Iterator<E> iterator();//1
}
public interface Iterator<E>{
    E next();
    boolean hasNest();
}
public interface Map<K,V>{
    Set<K> keySet();//2
    V put(K key,V value);
}
~~~

允许在定义接口、类时声明类型形参，类型形参在整个接口、类体内可当成类型使用，几乎所有可使用普通类型的地方都可以使用这种类型形参。除此之外，我们发现①②处方法声明返回值类型是Iterator< E>、Set< K>，这表明Set<K>形式是一种特殊的数据类型，是一种与Set不同的数据类型——可以认为是Set类型的子类

通过上面介绍可以发现，我们可以为任何类、接口增加泛型声明（并不是只有集合类才可以使用泛型声明，虽然集合类是泛型的重要使用场所）。下面自定义一个Apple类，这个Apple类就可以包含一个泛型声明。

~~~java
class Apple<T>{
    private T info;
    public Apple(){}
    public Apple(T info){
        this.info=info;
    }
    public void setInfo(T info){
        this.info=info;
    }
    public T getInfo(){
        return this.info;
    }
}
public static void main(String[] args) {
       Apple<String> apple=new Apple<>("苹果");
        System.out.println(apple.getInfo());
        Apple<Double> apple1=new Apple<>(5.67);
        System.out.println(apple.getInfo());

    }
~~~

上面程序定义了一个带泛型声明的Apple< T>类（不要理会这个类型形参是否具有实际意义），使用Apple<T>类时就可为T类型形参传入实际类型，这样就可以生成如Apple< String>、Apple< Double>…形式的多个逻辑子类（物理上并不存在）。

当创建带泛型声明的自定义类，为该类定义构造器时，构造器名还是原来的类名，不要增加泛型声明。例如，为Apple< T>类定义构造器，其构造器名依然是Apple，而不是Apple< T>！调用该构造器时却可以使用Apple< T>的形式，当然应该为T形参传入实际的类型参数。Java 7提供了菱形语法，允许省略<>中的类型实参。

### 从泛型类派生子类

当创建了带泛型声明的接口、父类之后，可以为该接口创建实现类，或从该父类派生子类，但需要指出的是，当使用这些接口、父类时不能再包含类型形参。例如，下面代码就是错误的。

~~~java
public class A extends Apple<T>{}
~~~

如果想从Apple类派生一个子类，则可以改为如下代码：

~~~java
public class A extends Apple<String>
~~~

### 并不存在泛型类

可以把ArrayList< String>类当成ArrayList的子类，事实上，ArrayList< String>类也确实像一种特殊的ArrayList类，这个ArrayList< String>对象只能添加String对象作为集合元素。但实际上，系统并没有为ArrayList< String>生成新的class文件，而且也不会把ArrayList< String>当成新类来处理。

~~~java
List<String> l1 =new ArrayList<>();
List<Integer l2=nwq ArrayList<>();
sout(l1.getClass()==l2.getClass());
~~~

结果为true。因为不管泛型的实际类型参数是什么，他在运行时总有同样的类。

不管为泛型的类型形参传入哪一种类型实参，对于Java来说，它们依然被当成同一个类处理，在内存中也只占用一块内存空间，因此在静态方法、静态初始化块或者静态变量的声明和初始化中不允许使用类型形参。例如：

~~~java
public class R<T>{
    //下列代码错误，不能再静态Field（成员变量）生命中使用类型形参。
    static T info;
    T age;
    public void foo(T msg){}
    //下面代码错误，不能再静态方法声明中使用类型形参
    public static void bar(T msg){}
}
~~~

由于系统中并不会真正生成泛型类，所以instanceof运算符后不能使用泛型类。（instanceof主要用来判断一个类是否实现了某个接口，或者判断一个实例对象是否属于一个类。）

## 类型通配符

当使用一个泛型类时（包括声明变量和创建对象两种情况），都应该为这个泛型类传入一个类型实参。如果没有传入类型实际参数，编译器就会提出泛型警告。例如下列代码

~~~java
public void test(List c){//泛型警告
    for(int i=0;i<c.size();i++){
        System.out.println(c.get(i));
    }
}
~~~

此处使用List接口时没有传入实际类型参数，这将引起泛型警告。修改后：

~~~java
public void test(List<Object> c){
    for(int i=0;i<c.size();i++){
        System.out.println(c.get(i));
    }
}
List<String> str=new ArrayList<>();
test(str);//编译错误，List<String>对象不能被当成List<Object>对象使用，也就是说， List<String>类并不是List<Object>类的子类。
~~~

**如果Foo是Bar的一个子类型（子类或者子接口），而G是具有泛型声明的类或接口，G< Foo>并不是G< Bar>的子类型！它与我们的习惯看法不同。**

为了表示各种泛型List的父类，我们需要使用类型通配符，类型通配符是一个问号（?），将一个问号作为类型实参传给List集合，写作：List<?>（意思是未知类型元素的List）。这个问号（?）被称为通配符，它的元素类型可以匹配任何类型。所以改写后的代码如下：

~~~java
public void test(List<?> c){
    for(int i=0;i<c.size();i++){
        System.out.println(c.get(i));
    }
}
~~~

**List< ?>，这种写法可以适应于任何支持泛型声明的接口和类，比如写成Set< ?>、Collection< ?>、Map< ? , ?>等。**

但这种带通配符的List仅表示它是各种泛型List的父类，并不能把元素加入到其中。

~~~java
List<?> c=new ArrayList<String>():
c.add(new Object());//编译错误
~~~

因为我们不知道上面程序中c集合里元素的类型，所以不能向其中添加对象。根据前面的List<E>接口定义的代码可以发现：add方法有类型参数E作为集合的元素类型，所以传给add的参数必须是E类的对象或者其子类的对象。但因为在该例中不知道E是什么类型，所以程序无法将任何对象“丢进”该集合。唯一的例外是null，它是所有引用类型的实例。

**通配符上限**

实际上，我们需要一种泛型表示方法，它可以表示所有Shape泛型List的父类。为了满足这种需求， Java泛型提供了被限制的泛型通配符。被限制的泛型通配符表示如下：

~~~java
List<? extends Shape>
~~~

List<? extends Shape>是受限制通配符的例子，此处的问号（?）代表一个未知的类型，就像前面看到的通配符一样。但是此处的这个未知类型一定是Shape的子类型（也可以是Shape本身），因此我们把Shape称为这个通配符的上限（upper bound）。

**形参上限**

~~~java
public class Apple<T extends Number>{
	T col;
}
~~~

上面程序定义了一个Apple泛型类，该Apple类的类型形参的上限是Number类，这表明使用Apple类时为T形参传入的实际类型参数只能是Number或Number类的子类。

## 泛型方法

格式：

> 修饰符 <T,S> 返回值类型 方法名 （形参列表）{
>
> ​	//方法体。。。
>
> }

示例：

~~~java
public class GenericMethodTest{
    static <T> void fromArrayToCollection(T[] a,Collection<T> c){
        for(T t:a){
            c.add(t);
        }
    }
    main(){
        Object [] object=new Object[100];
        Collection<Object> collection =new ArrayList<>();
        fromArrayToCollection(object,collection);
    }
}
~~~

上面程序定义了一个泛型方法，该泛型方法中定义了一个T类型形参，这个T类型形参就可以在该方法内当成普通类型使用。与接口、类声明中定义的类型形参不同的是，方法声明中定义的形参只能在该方法里使用，而接口、类声明中定义的类型形参则可以在整个接口、类中使用。

# Java输入输出

## File类

File类是java.io包下代表与平台无关的文件和目录，也就是说，如果希望在程序中操作文件和目录，都可以通过File类来完成。值得指出的是，不管是文件还是目录都是使用File来操作的，File能新建、删除、重命名文件和目录，File不能访问文件内容本身。如果需要访问文件内容本身，则需要使用输入/输出流。

### 访问文件和目录

File类可以使用文件路径字符串来创建File实例，该文件路径字符串既可以是绝对路径，也可以是相对路径。在默认情况下，系统总是依据**用户的工作路径来解释相对路径**，这个路径由系统属性“user.dir”指定，**通常也就是运行Java虚拟机时所在的路径。**

1. 访问文件名相关方法

   ~~~java
   String getName()：返回此File对象所表示的文件名或路径名（如果是路径，则返回最后一级子路径名）。
   String getPath()：返回此File对象所对应的路径名。
   File getAbsoluteFile()：返回此File对象所对应的绝对路径所对应的File对象。
   String getAbsolutePath()：返回此File对象所对应的绝对路径名。
   String getParent()：返回此File对象所对应目录（最后一级子目录）的父目录名。
   boolean renameTo(File newName)：重命名此File对象所对应的文件或目录，如果重命名成功，则返回true；否则返回false。
   ~~~

2. 文件检测相关方法

   ~~~java
   boolean exists()：判断File对象所对应的文件或目录是否存在。
   boolean canWrite()：判断File对象所对应的文件和目录是否可写。
   boolean canRead()：判断File对象所对应的文件和目录是否可读。
   boolean isFile()：判断File对象所对应的是否是文件，而不是目录。
   boolean isDirectory()：判断File对象所对应的是否是目录，而不是文件。
   boolean isAbsolute()：判断File对象所对应的文件或目录是否是绝对路径。该方法消除了不同平台的差异，可以直接判断File对象是否为绝对路径。在UNIX/Linux/BSD等系统上，如果路径名开头是一条斜线（/），则表明该File对象对应一个绝对路径；在Windows等系统上，如果路径开头是盘符，则说明它是一个绝对路径。
   ~~~

3. 获取文件信息

   ~~~java
   long lastModified()：返回文件的最后修改时间。
   long length()：返回文件内容的长度。
   ~~~

4. 文件操作相关方法

   ~~~java
   boolean createNewFile()：当此File对象所对应的文件不存在时，该方法将新建一个该File对象所指定的新文件，如果创建成功则返回true；否则返回false。
   boolean delete()：删除File对象所对应的文件或路径。
   static File createTempFile(String prefix, String suffix)：在默认的临时文件目录中创建一个临时的空文件，使用给定前缀、系统生成的随机数和给定后缀作为文件名。这是一个静态方法，可以直接通过File类来调用。prefix参数必须至少是3个字节长。建议前缀使用一个短的、有意义的字符串，比如"hjb"或"mail"。suffix参数可以为null，在这种情况下，将使用默认的后缀“.tmp”。
   static File createTempFile(String prefix, String suffix, File directory) ：在directory所指定的目录中创建一个临时的空文件，使用给定前缀、系统生成的随机数和给定后缀作为文件名。这是一个静态方法，可以直接通过File类来调用。
   void deleteOnExit()：注册一个删除钩子，指定当Java虚拟机退出时，删除File对象所对应的文件和目录。
   ~~~

5. 目录操作相关方法

   ~~~java
   boolean mkdir()：试图创建一个File对象所对应的目录，如果创建成功，则返回true；否则返回false。调用该方法时File对象必须对应一个路径，而不是一个文件。
   String[] list()：列出File对象的所有子文件名和路径名，返回String数组。
   File[] listFiles()：列出File对象的所有子文件和路径，返回File数组。
   static File[] listRoots()：列出系统所有的根路径。这是一个静态方法，可以直接通过File类来调用。上面详细列出了File类的常用方法，下面程序以几个简单方法来测试一下File类的功能。
   ~~~

示例：

~~~java
 public static void main(String[] args) throws IOException {
        File file=new File(".");//以当前路径创建File对象
        System.out.println(file.getName());//获取文件名，输出一点
        System.out.println(file.getParent());//获取相对路径父路径可能出错,输出null
        System.out.println(file.getAbsoluteFile());//获取绝对路径
        //在当前路径下创建临时文件tmpFile.
        File tmpFile=File.createTempFile("aaa",".txt",file);
        tmpFile.deleteOnExit();//当JVM退出时删除该文件
        File newFile=new File(System.currentTimeMillis()+"");//以系统当前时间为文件名创建新文件
        System.out.println("newFile是否存在"+newFile.exists());
        newFile.createNewFile();//以newFile对象来创建一个文件
        newFile.mkdir();//以newFile对象来创建一个目录，因为newFile已经存在，所以返回false
        String [] fileList=file.list();//列出当前路径下所有文件和路径
        System.out.println("--------------------当前路径下的文件和路径--------------");
        for(String fileName:fileList){
            System.out.println(fileName);
        }
        //listRoot()静态方法列出所有的磁盘根路径
        System.out.println("---------------------系统所有根路径---------------");
        File [] roots=File.listRoots();
        for (File root: roots
             ) {
            System.out.println(root);
        }
    }
~~~

运行上面程序，可以看到程序列出当前路径的所有文件和路径时，列出了程序创建的临时文件，但程序运行结束后，aaa.txt临时文件并不存在，因为程序指定虚拟机退出时自动删除该文件。

当使用相对路径的File对象来获取父路径时可能引起错误，因为该方法返回将File对象所对应的目录名、文件名里最后一个子目录名、子文件名删除后的结果

**注意**

> Windows的路径分隔符使用反斜线（\），而Java程序中的反斜线表示转义字符，所以如果需要在Windows的路径下包括反斜线，则应该使用两条反斜线，如`F:\\abc\\test.txt`，或者直接使用斜线（/）也可以，Java程序支持将斜线当成平台无关的路径分隔符。

### 文件过滤器

在File类的list()方法中可以接收一个FilenameFilter参数，通过该参数可以只列出符合条件的文件。这里的FilenameFilter接口和javax.swing.filechooser包下的FileFilter抽象类的功能非常相似，可以把FileFilter当成FilenameFilter的实现类，但可能Sun在设计它们时产生了一些小小遗漏，所以没有让FileFilter实现FilenameFilter接口。

FilenameFilter接口里包含了一个**accept(File dir, String name)方法**，该方法将依次对指定File的所有子目录或者文件进行迭代，如果该方法返回true，则list()方法会列出该子目录或者文件。

~~~java
public static void main(String[] args)  {
    File file =new File(".");
    String nameList=file.list(new MyFilenameFilter());
    for(String name:nameList){
         System.out.println(name);
    }
}
class MyFilenameFilter implements FilenameFilter{
    public boolean accept(File dir,String name){
        return name.endWith(".java")||new File(name).isDirectory();
    }
}
~~~

当前路径下所有的*.java文件以及文件夹被列出.

**注意**

> 这种用法是一个典型的Command设计模式，因为File的list()方法需要一个判断规则：判断哪些文件应该被列出——这段判断规则需要一个代码块，但目前的JDK版本不支持直接向方法传入代码块，所以Java使用了FilenameFilter的accept()方法来封装该代码块。也就是说，所传入的MyFilenameFilter对象的作用就是传入accept()方法的方法体，该方法体指定哪些文件应该被列出。

## 理解IO流

Java的IO流是实现输入/输出的基础，它可以方便地实现数据的输入/输出操作，在Java中把不同的输入/输出源（键盘、文件、网络连接等）抽象表述为“流”（stream），通过流的方式允许Java程序使用相同的方式来访问不同的输入/输出源。stream是从起源（source）到接收（sink）的有序数据。

### 流的分类

1. 输入和输出流

   输入流：只能从中读取数据，而不能向其写入数据。

   输出流：只能向其写入数据，而不能从中读取数据。

2. 字节和字符流

   字节流和字符流的用法几乎完全一样，区别在于字节流和字符流所操作的数据单元不同——字节流操作的数据单元是8位的字节，而字符流操作的数据单元是16位的字符。字节流主要由InputStream和OutputStream作为基类，而字符流则主要由Reader和Writer作为基类。

3. 节点流和处理流

   可以从/向一个特定的IO设备（如磁盘、网络）读/写数据的流，称为节点流，节点流也被称为低级流（Low Level Stream）

   处理流则用于对一个已存在的流进行连接或封装，通过封装后的流来实现数据读/写功能。处理流也被称为高级流。当使用处理流进行输入/输出时，程序并不会直接连接到实际的数据源，没有和实际的输入/输出节点连接。使用处理流的一个明显好处是，只要使用相同的处理流，程序就可以采用完全相同的输入/输出代码来访问不同的数据源，随着处理流所包装节点流的变化，程序实际所访问的数据源也相应地发生变化。

**注意**

> 实际上，Java使用处理流来包装节点流是一种典型的装饰器设计模式，通过使用处理流来包装不同的节点流，既可以消除不同节点流的实现差异，也可以提供更方便的方法来完成输入/输出功能。因此处理流也被称为包装流。

Java的IO流共涉及40多个类，这些类看上去芜杂而凌乱，但实际上非常规则，而且彼此之间存在非常紧密的联系。Java的IO流的40多个类都是从如下4个抽象基类派生的。

 InputStream/Reader：所有输入流的基类，前者是字节输入流，后者是字符输入流。

OutputStream/Writer：所有输出流的基类，前者是字节输出流，后者是字符输出流。

## 字节流和字符流

### InputStream和Reader

InputStream和Reader是所有输入流的抽象基类，本身并不能创建实例来执行输入，但它们将成为所有输入流的模板，所以它们的方法是所有输入流都可使用的方法。

在InputStream里包含如下3个方法：

~~~java
int read()：从输入流中读取单个字节，返回所读取的字节数据（字节数据可直接转换为int类型）。
int read(byte[] b)：从输入流中最多读取b.length个字节的数据，并将其存储在字节数组b中，返回实际读取的字节数。
int read(byte[] b, int off, int len)：从输入流中最多读取len个字节的数据，并将其存储在数组b中，放入数组b中时，并不是从数组起点开始，而是从off位置开始，返回实际读取的字节数。
~~~

在Reader里包含如下3个方法：

~~~java
int read()：从输入流中读取单个字符（相当于从图15.5所示的水管中取出一滴水），返回所读取的字符数据（字符数据可直接转换为int类型）。
int read(char[] cbuf)：从输入流中最多读取cbuf.length个字符的数据，并将其存储在字符数组cbuf中，返回实际读取的字符数。
int read(char[] cbuf, int off, int len)：从输入流中最多读取len个字符的数据，并将其存储在字符数组cbuf中，放入数组cbuf中时，并不是从数组起点开始，而是从off位置开始，返回实际读取的字符数。
~~~

InputStream和Reader都是抽象类，本身不能创建实例，但它们分别有一个用于读取文件的输入流：FileInputStream和FileReader，它们都是节点流——会直接和指定文件关联。

程序如何判断到了最后呢？直到read(char[] cbuf)或read(byte[] b)方法**返回-1**，即表明到了输入流的结束点。

使用FileInputStream来读取：

~~~java
public static void main(String[] args) throws IOException {
   FileInputStream fis =new FileInputStream("E:\\temp\\hello.txt");
   //创建1024字节的数组
   byte [] buff=new byte[1024];
   int hasRead=0;//保存实际读取的字节数
   while((hasRode=fis.read(buff))>0){
       //取出字节，将字节数组转换成字符串输入
       System.out.println(new String (buff,0,hasRead));
   }
   //关闭文件输入流
   fis.close();
}
~~~

使用FileReader来读取文件：

~~~java
public static void main(String[] args)  {
    FileReader fileReader=new FileReader("E:\\temp\\hello.txt");
    char [] cbuff =new char[32];
    int hasRead=0;//保存实际读取的字节数
    while((hasRead=fileReader.read(cbuff))>0){
        //取出字节，将字节数组转换成字符串输入
        System.out.println(new String (cbuff,0,hasRead));
    }
    //关闭文件输入流
    fis.close();
}
~~~

这两个程序并没有太大的不同，程序只是将字符数组的长度改为32，这意味着程序需要多次调用read()方法才可以完全读取输入流的全部数据。

**InputStream和Reader还支持如下几个方法来移动记录指针：**

~~~java
 void mark(int readAheadLimit)：在记录指针当前位置记录一个标记（mark）。
 boolean markSupported()：判断此输入流是否支持mark()操作，即是否支持记录标记。
 void reset()：将此流的记录指针重新定位到上一次记录标记（mark）的位置。
 long skip(long n)：记录指针向前移动n个字节/字符。
~~~

### OutputStream和Writer

OutputStream和Writer也非常相似，它们采用如图15.6所示的模型来执行输出，两个流都提供了如下3个方法：

~~~java
void write(int c)：将指定的字节/字符输出到输出流中，其中c既可以代表字节，也可以代表字符。
void write(byte[]/char[] buf)：将字节数组/字符数组中的数据输出到指定输出流中。
void write(byte[]/char[] buf, int off, int len)：将字节数组/字符数组中从off位置开始，长度为len的字节/字符输出到输出流中。
~~~

因为字符流直接以字符作为操作单位，所以Writer可以用字符串来代替字符数组，即以String对象作为参数。Writer里还包含如下两个方法：

~~~java
void write(String str)：将str字符串里包含的字符输出到指定输出流中。
void write(String str, int off, int len)：将str字符串里从off位置开始，长度为len的字符输出到指定输出流中。
~~~

下面程序使用FileInputStream来执行输入，并使用FileOutputStream来执行输出，用以实现复制文件的功能：

~~~java
public static void main(String[] args) throws IOException {
       FileInputStream fileInputStream=new FileInputStream("E:\\temp\\hello.txt");
       FileOutputStream fileOutputStream=new FileOutputStream("E:\\temp\\newFile.txt");
       byte [] bytes=new byte[32];
       int hasRead=0;
       while ((hasRead=fileInputStream.read(bytes))>0){
           fileOutputStream.write(bytes,0,hasRead);
       }
        fileInputStream.close();
       fileOutputStream.close();
    }
~~~

运行上面程序，将看到系统路径下多了一个文件：newFile.txt，该文件的内容和hello.txt文件的内容完全相同。

如果希望直接输出字符串内容，则使用Writer会有更好的效果，如下程序所示：

~~~java
FileWriter fileWriter=new FileWriter("E:\\temp\\poem.txt");
      fileWriter.write("锦瑟 - 李商隐\r\n");
      fileWriter.write("锦瑟无端五十弦，一弦一柱思华年\n\n");
      fileWriter.write("庄生晓梦迷蝴蝶，望帝春心托杜鹃\n\n");
      fileWriter.write("沧海明月珠有泪，蓝天日暖玉生烟\n\n");
      fileWriter.write("此情可待成追忆，只是当时已惘然\n\n");
      fileWriter.close();
~~~

将会在当前目录下输出一个poem.txt文件，文件内容就是程序中输出的内容。

## 输入输出流

### 处理流

处理流可以隐藏底层设备上节点流的差异，并对外提供更加方便的输入/输出方法，让程序员只需关心高级流的操作。因此，我们使用处理流时的典型思路是，使用处理流来包装节点流，程序通过处理流来执行输入/输出功能，让节点流与底层的I/O设备、文件交互。

关于使用处理流的优势，归纳起来就是2点：①对开发人员来说，使用处理流进行输入/输出操作更简单；②使用处理流的执行效率更高。

示例：使用PrintStream处理流来包装OutputStream，使用处理流后的输出流在输出时将更加方便：

~~~java
public static void main(String[] args) throws IOException {
     FileOutputStream fileOutputStream=new FileOutputStream("E:\\temp\\newFile.txt");
     PrintStream printStream=new PrintStream(fileOutputStream);
     //使用PrintStream执行输出
        printStream.println("字符串");
        printStream.println(new test());
        printStream.close();
    }
~~~

从前面的代码可以看出，程序使用处理流非常简单，通常只需要在创建处理流时传入一个节点流作为构造器参数即可，这样创建的处理流就是包装了该节点流的处理流。

在使用处理流包装了底层节点流之后，关闭输入/输出流资源时，只要关闭最上层的处理流即可。关闭最上层的处理流时，系统会自动关闭被该处理流包装的节点流。

**输入输出流体系**

通常来说，我们认为字节流的功能比字符流的功能强大，因为计算机里所有的数据都是二进制的，而字节流可以处理所有的二进制文件——但问题是，如果使用字节流来处理文本文件，则需要使用合适的方式把这些字节转换成字符，这就增加了编程的复杂度。所以通常有一个规则：如果进行输入/输出的内容是文本内容，则应该考虑使用字符流；如果进行输入/输出的内容是二进制内容，则应该考虑使用字节流。

**注意**

> 我们常常会把计算机的文件分为文本文件和二进制文件两大类——所有能用记事本打开并看到其中字符内容的文件称为文本文件，反之则称为二进制文件。但实质是，计算机里的所有文件都是二进制文件，文本文件只是二进制文件的一种特例，当二进制文件里的内容恰好能被正常解析成字符时，则该二进制文件就变成了文本文件。更甚至于，即使是正常的文本文件，如果打开该文件时强制使用了“错误”的字符集，例如使用notepad++打开刚刚生成的poem.txt文件时指定使用UTF-8字符集，如图15.9所示，则将看到打开的poem.txt文件内容变成了乱码。因此，如果希望看到正常的文本文件内容，则必须在打开文件时与保存文件时使用相同的字符集（Windows下简体中文默认使用GBK字符集，而Linux下简体中文默认使用UTF-8字符集）。

### 转换流

输入/输出流体系中还提供了两个转换流，这两个转换流用于实现将字节流转换成字符流，其中InputStreamReader将字节输入流转换成字符输入流，OutputStreamWriter将字节输出流转换成字符输出流。

下面以获取键盘输入为例来介绍转换流的用法。Java使用System.in代表标准输入，即键盘输入，但这个标准输入流是InputStream类的实例，使用不太方便，而且键盘输入内容都是文本内容，所以可以使用InputStreamReader将其转换成字符输入流，普通的Reader读取输入内容时依然不太方便，我们可以将普通的Reader再次包装成BufferedReader，利用BufferedReader的readLine()方法可以一次读取一行内容。如下程序所示:

~~~java
public static void main(String[] args) throws IOException {
        InputStreamReader reader=new InputStreamReader(System.in);
        BufferedReader bufferedReader=new BufferedReader(reader);
        String buffer=null;
        while ((buffer=bufferedReader.readLine())!=null){
            if(buffer.equals("exit")){
                System.exit(1);
            }
            System.out.println("输入内容为："+buffer);
        }
    }
~~~

上面程序中的粗体字代码负责将System.in包装成BufferedReader，BufferedReader流具有缓冲功能，它可以一次读取一行文本——以换行符为标志，如果它没有读到换行符，则程序阻塞，等到读到换行符为止。运行上面程序可以发现这个特征，当我们在控制台执行输入时，只有按下回车键，程序才会打印出刚刚输入的内容。

## 重定向标准输入输出

Java的标准输入/输出分别通过System.in和System.out来代表，在默认情况下它们分别代表键盘和显示器，当程序通过System.in来获取输入时，实际上是从键盘读取输入；当程序试图通过System.out执行输出时，程序总是输出到屏幕。

在System类里提供了如下3个重定向标准输入/输出的方法：

~~~java
static void setErr(PrintStream err)：重定向 “标准”错误输出流。
static void setIn(InputStream in)：重定向“标准”输入流。
static void setOut(PrintStream out)：重定向 “标准”输出流。
~~~

示例：

~~~java
PrintStream ps=new PrintStream(new FileOutputStream("temp.txt"));
System.setOut(ps);
System.out.println("普通字符串");
System.out.println(new test());
~~~

上面程序中的粗体字代码创建了一个PrintStream输出流，并将系统的标准输出重定向到该PrintStream输出流。运行上面程序时将看不到任何输出——这意味着标准输出不再输出到屏幕，而是输出到out.txt文件，运行结束后，打开系统当前路径下的out.txt文件，即可看到文件里的内容，正好与程序中的输出一致。

## 对象序列化

对象序列化的目标是将对象保存到磁盘中，或允许在网络中直接传输对象。对象序列化机制允许把内存中的Java对象转换成平台无关的二进制流，从而允许把这种二进制流持久地保存在磁盘上，通过网络将这种二进制流传输到另一个网络节点。其他程序一旦获得了这种二进制流（无论是从磁盘中获取的，还是通过网络获取的），都可以将这种二进制流恢复成原来的Java对象。

**序列化的意义**

对象的序列化（Serialize）指将一个Java对象写入IO流中，与此对应的是，对象的反序列化（Deserialize）则指从IO流中恢复该Java对象。如果需要让某个对象支持序列化机制，则必须让它的类是可序列化的（serializable）。为了让某个类是可序列化的，该类必须实现如下两个接口之一：

 Serializable

Externalizable

Java的很多类已经实现了Serializable，该接口是一个标记接口，实现该接口无须实现任何方法，它只是表明该类的实例是可序列化的。

所有可能在网络上传输的对象的类都应该是可序列化的，否则程序将会出现异常，比如RMI（Remote Method Invoke，即远程方法调用，是JavaEE的基础）过程中的参数和返回值；所有需要保存到磁盘里的对象的类都必须可序列化，比如Web应用中需要保存到HttpSession或ServletContext属性的Java对象。

因为序列化是RMI过程的参数和返回值都必须实现的机制，而RMI又是Java EE技术的基础——所有的分布式应用常常需要跨平台、跨网络，所以要求所有传递的参数、返回值必须实现序列化。因此序列化机制是Java EE平台的基础。通常建议：程序创建的每个JavaBean类都实现

## NIO

新IO和传统的IO有相同的目的，都是用于进行输入/输出，但新IO使用了不同的方式来处理输入/输出，新IO采用内存映射文件的方式来处理输入/输出，新IO将文件或文件的一段区域映射到内存中，这样就可以像访问内存一样来访问文件了（这种方式模拟了操作系统上的虚拟内存的概念），通过这种方式来进行输入/输出比传统的输入/输出要快得多。

Channel（通道）和Buffer（缓冲）是新IO中的两个核心对象，Channel是对传统的输入/输出系统的模拟，在新IO系统中所有的数据都需要通过通道传输；Channel与传统的InputStream、OutputStream最大的区别在于它提供了一个map()方法，通过该map()方法可以直接将“一块数据”映射到内存中。如果说传统的输入/输出系统是面向流的处理，则新IO则是面向块的处理。

## NIO.2

Java 7对原有的NIO进行了重大改进，改进主要包括如下两方面的内容。

提供了全面的文件IO和文件系统访问支持。

基于异步Channel的IO。

第一个改进表现为Java7新增的java.nio.file包及各个子包；第二个改进表现为Java7在java.nio.channels包下增加了多个以Asynchronous开头的Channel接口和类。Java7把这种改进称为NIO.2

# 类的加载机制与反射

## 类的加载连接和初始化

当我们调用Java命令运行某个Java程序时，该命令将会启动一个Java虚拟机进程，不管该Java程序有多么复杂，该程序启动了多少个线程，它们都处于该Java虚拟机进程里。正如前面介绍的，同一个JVM的所有线程、所有变量都处于同一个进程里，它们都使用该JVM进程的内存区。当系统出现以下几种情况时，JVM进程将被终止。

程序运行到最后正常结束。

程序运行到使用System.exit()或Runtime.getRuntime().exit()代码处结束程序。

程序执行过程中遇到未捕获的异常或错误而结束。

程序所在平台强制结束了JVM进程。

示例：

~~~java
A.java
public class A{
	public static int a=6;
}
Atest.java
public class ATest1{
	main(){
		A a=new A();
		a.a++;
		sout(a.a);
	}
}
public class ATest2{
	main(){
		A b=new A();
		sout(b.a);
	}
}
~~~

ATest1输出7。ATest2时输出6而不是7.因为两次运行处于两个不同的JVM进程中。

### **类的加载**

当程序主动使用某个类时，如果该类还未被加载到内存中，则系统会通过加载、连接、初始化3个步骤来对该类进行初始化。如果没有意外，JVM将会连续完成这3个步骤，所以有时也把这3个步骤统称为类加载或类初始化。

类加载指的是将类的class文件读入内存，并为之创建一个java.lang.Class对象，系统中所有的类实际上也是实例，它们都是java.lang.Class的实例。

类的加载由类加载器完成，类加载器通常由JVM提供，这些类加载器也是所有程序运行的基础，JVM提供的这些类加载器通常被称为系统类加载器。除此之外，开发者可以通过继承ClassLoader基类来创建自己的类加载器。通过使用不同的类加载器，可以从不同来源加载类的二进制数据，通常有如下几种来源。

 从本地文件系统加载class文件，这是绝大部分程序的类加载方式。

从JAR包加载class文件，这种方式也是很常见的，比如JDBC编程时用到的数据库驱动类就放在JAR文件中

通过网络加载class文件。

把一个Java源文件动态编译，并执行加载。

### **类的连接**

当类被加载之后，系统为之生成一个对应的Class对象，接着将会进入连接阶段，连接阶段负责把类的二进制数据合并到JRE中。类连接又可分为如下3个阶段。

（1）验证：验证阶段用于检验被加载的类是否有正确的内部结构，并和其他类协调一致。

（2）准备：类准备阶段则负责为类的静态Field分配内存，并设置默认初始值。

（3）解析：将类的二进制数据中的符号引用替换成直接引用。

### **类的初始化**

在类的初始化阶段，虚拟机负责对类进行初始化，主要就是对静态Field进行初始化。在Java类中对静态Field指定初始值有两种方式：① 声明静态Field时指定初始值；② 使用静态初始化块为静态Field指定初始值

~~~java
public class Test{
    //声明时初始化
    static int a =5;
    static int b;
    static int c;//没有指定初始值，默认采用0
    //使用静态初始化块
    static{
        b=6;
    }
}
~~~

> **正常类的加载顺序：静态变量/静态代码块 -> main方法 -> 非静态变量/代码块 -> 构造方法**
>
> **静态代码块与静态变量的执行顺序同代码定义的顺序；非静态变量与代码块的执行顺序同代码执行顺序**

JVM初始化一个类包含如下几个步骤。

（1）假如这个类还没有被加载和连接，则程序先加载并连接该类。

（2）假如该类的直接父类还没有被初始化，则先初始化其直接父类。

（3）假如类中有初始化语句，则系统依次执行这些初始化语句。

### **初始化类的时机**

当Java程序首次通过下面6种方式来使用某个类或接口时，系统就会初始化该类或接口：

1.创建类的实例。为某个类创建实例的方式包括：使用new操作符来创建实例，通过反射来创建实例，通过反序列化的方式来创建实例。

2.调用某个类的静态方法。

3.访问某个类或接口的静态Field，或为该静态Field赋值。

4.使用反射方式来强制创建某个类或接口对应的java.lang.Class对象。例如代码：Class.forName("Person")，如果系统还未初始化Person类，则这行代码将会导致该Person类被初始化，并返回Person类对应的java.lang.Class对象。

5.初始化某个类的子类。当初始化某个类的子类时，该子类的所有父类都会被初始化。

6.直接使用java.exe命令来运行某个主类。当运行某个主类时，程序会先初始化该主类。

PS：*当某个静态Field使用了final修饰，而且它的值可以在编译时就确定下来，那么程序其他地方使用该静态Field时，实际上并没有使用该静态Field，而是相当于使用常量。*

> static final String str="aaa";

*当使用ClassLoader类的loadClass()方法来加载某个类时，该方法只是加载该类，并不会执行该类的初始化。使用Class的forName()静态方法才会导致强制初始化该类。*

> CLassLoader cl =ClassLoader.getSystemClassLoader();
>
> cl.loadClass("Test");//只加载
>
> Class.forName("Test");//初始化

## 类加载器

类加载器负责将.class文件（可能在磁盘上，也可能在网络上）加载到内存中，并为之生成对应的java.lang.Class对象。一旦一个类被载入JVM中，同一个类就不会被再次载入了。

在Java中，一个类用其全限定类名（包括包名和类名）作为标识；但在JVM中，一个类用其全限定类名和其类加载器作为其唯一标识。

> 例如，如果在pg的包中有一个名为Person的类，被类加载器ClassLoader的实例kl负责加载，则该Person类对应的Class对象在JVM中表示为（Person、pg、kl）。这意味着两个类加载器加载的同名类：（Person、pg、kl）和（Person、pg、kl2）是不同的、它们所加载的类也是完全不同、互不兼容的。

当JVM启动时，会形成由3个类加载器组成的初始类加载器层次结构。

Bootstrap ClassLoader：根类加载器。

> Bootstrap ClassLoader被称为引导（也称为原始或根）类加载器，它负责加载Java的核心类String、System这些核心类库都在E:/Java/jdk1.7.0/jre/lib/rt.jar文件中

Extension ClassLoader：扩展类加载器。

> 它负责加载JRE的扩展目录（%JAVA_HOME%/jre/lib/ext或者由java.ext.dirs系统属性指定的目录）中JAR包的类。通过这种方式，就可以为Java扩展核心类以外的新功能，只要我们把自己开发的类打包成JAR文件，然后放入JAVA_HOME/jre/lib/ext路径即可

System ClassLoader：系统类加载器。

> 它负责在JVM启动时加载来自java命令的-classpath选项、java.class.path系统属性，或CLASSPATH环境变量所指定的JAR包和类路径。程序可以通过ClassLoader的静态方法getSystemClassLoader()来获取系统类加载器。如果没有特别指定，则用户自定义的类加载器都以类加载器作为父加载器。

示例：

```java
public static void main(String[] args) throws Exception {
       ClassLoader systemLoader=ClassLoader.getSystemClassLoader();
        System.out.println("系统类加载器"+systemLoader);
        /*获取系统类加载器的加载路径-*/
        Enumeration<URL> eml = systemLoader.getResources("");
        while (eml.hasMoreElements()){
            System.out.println(eml.nextElement());
        }
        //获取系统类加载器的父类加载器得到拓展类加载器
        ClassLoader extensionLoader =systemLoader.getParent();
        System.out.println("扩展类加载器"+extensionLoader);
        System.out.println("扩展类加载器的加载路径"+System.getProperty("java.ext.dirs"));
        System.out.println("扩展类加载器的parent"+extensionLoader.getParent());

    }
```

运行结果：

> 系统类加载器sun.misc.Launcher$AppClassLoader@b4aac2
> file:/E:/IDEA/test/out/production/test/
> 扩展类加载器sun.misc.Launcher$ExtClassLoader@154617c
> 扩展类加载器的加载路径E:\java_se\jdk1.8.0_101\jre\lib\ext;C:\WINDOWS\Sun\Java\lib\ext
> 扩展类加载器的parentnull

系统类加载器的加载路径是程序运行的当前路径，扩展类加载器的加载路径E:\java_se\jdk1.8.0_101\jre\lib\ext，但我们看到扩展类加载器的父加载器是null，并不是根类加载器。这是因为根类加载器并没有继承ClassLoader抽象类，所以扩展类加载器的getParent()方法返回null。但实际上，扩展类加载器的父类加载器是根类加载器，只是根类加载器并不是Java实现的。系统类加载器是AppClassLoader的实例，扩展类加载器是ExtClassLoader的实例。实际上，这两个类都是URLClassLoader类的实例。

> JVM的根类加载器并不是Java实现的，而且由于程序通常无须访问根类加载器，因此访问扩展类加载器的父类加载器时返回null。

### **类加载机制**

JVM的类加载机制主要有如下3种。

全盘负责。所谓全盘负责，就是当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显式使用另外一个类加载器来载入。

父类委托。所谓父类委托，则是先让parent（父）类加载器试图加载该Class，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。

缓存机制。缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区中搜寻该Class，只有当缓存区中不存在该Class对象时，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区中。这就是为什么修改了Class后，必须重新启动JVM，程序所做的修改才会生效的原因。

除此之外，开发者可以实现自己的类加载器，自定义类加载器通过ClassLoader来实现。

JVM中这4种类加载器的层次结构:

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/001.png?Expires=1587000985&OSSAccessKeyId=TMP.3Kk1591ybAo5yAmodqB3V4KtRNw5fsU3HXT8mQcFDf3FbSuShR7FZe75XtjoiykAiEWAk28VTJzyjpqcmVqCtWnyr8Tm2g&Signature=1dfv7rqAG4keiQie6xV7dGfexg4%3D)

类加载器加载Class大致要经过如下8个步骤。

（1）检测此Class是否载入过（即在缓存区中是否有此Class），如果有则直接进入第8步，否则接着执行第2步。（2）如果父类加载器不存在（如果没有父类加载器，则要么parent一定是根类加载器，要么本身就是根类加载器），则跳到第4步执行；如果父类加载器存在，则接着执行第3步。

（3）请求使用父类加载器去载入目标类，如果成功载入则跳到第8步，否则接着执行第5步。

（4）请求使用根类加载器来载入目标类，如果成功载入则跳到第8步，否则跳到第7步。

（5）当前类加载器尝试寻找Class文件（从与此ClassLoader相关的类路径中寻找），如果找到则执行第6步，如果找不到则跳到第7步。

（6）从文件中载入Class，成功载入后跳到第8步。

（7）抛出ClassNotFoundException异常。

（8）返回对应的java.lang.Class对象。其中，第5、6步允许重写ClassLoader的findClass()方法来实现自己的载入策略，甚至重写loadClass()方法来实现自己的载入过程。

### **创建并使用自定义的类加载器：**

JVM中除根类加载器之外的所有类加载器都是ClassLoader子类的实例，开发者可以通过扩展ClassLoader的子类，并重写该ClassLoader所包含的方法来实现自定义的类加载器。ClassLoader类有如下两个关键方法：

~~~java
loadClass(String name, boolean resolve)//该方法为ClassLoader的入口点，根据指定的二进制名称来加载类，系统就是调用ClassLoader的该方法来获取指定类对应的Class对象。 
findClass(String name)//根据二进制名称来查找类。
~~~

如果需要实现自定义的ClassLoader，则可以通过重写以上两个方法来实现，当然我们推荐重写findClass()方法，而不是重写loadClass()方法。loadClass()方法的执行步骤如下：

（1）用findLoadedClass(String) 来检查是否已经加载类，如果已经加载则直接返回。

（2）在父类加载器上调用loadClass()方法。如果父类加载器为null，则使用根类加载器来加载。

（3）调用findClass(String)方法查找类。

重写findClass()方法可以避免覆盖默认类加载器的父类委托、缓冲机制两种策略；如果重写loadClass()方法，则实现逻辑更为复杂。

**注意**

> 在ClassLoader里还有一个核心方法：Class defineClass(String name, byte[] b, int off, intlen)，该方法负责将指定类的字节码文件（即Class文件，如Hello.class）读入字节数组byte[] b内，并把它转换为Class对象，该字节码文件可以来源于文件、网络等。defineClass()方法管理JVM的许多复杂的实现，它负责将字节码分析成运行时数据结构，并校验有效性等。该方法是final型，不能重写

示例：

注意注释1的代码原来是`Process p=Runtime.getRuntime().exec("javac "+javaFile);`

如果你的编码方式是utf-8而不是默认的GBK就会报错无法编译（大部分人的编译器会调整为utf-8）

~~~java
public class CompileClassLoader extends ClassLoader{
    //读取文件的内容
    private  byte[] getBytes(String filename) throws IOException{
        File file=new File(filename);
        long len=file.length();
        byte[] raw=new byte[(int)len];
        try(FileInputStream fin =new FileInputStream(file);) {
            int r = fin.read(raw);
            if (r != len) throw new IOException("无法读取全部文件" + r + "!=" + len);
            return raw;
        }
    }
    //定义编译指定Java文件的方法
    private boolean compile(String javaFile)throws IOException{
        System.out.println("CompileClassLoader:正在编译"+javaFile+"...");
        //调用javac命令
        Process p=Runtime.getRuntime().exec("javac -encoding utf-8 "+javaFile);//1
        try{
            //其他进程等待这个进程完成
            p.waitFor();
        }catch (InterruptedException ie){
            System.out.println(ie);
        }
        //获取javac线程的退出值
        int ret=p.exitValue();
        return ret==0;
    }
    //重写ClassLoader的findClass方法
    protected Class<?> findClass(String name)throws ClassNotFoundException{
        Class clazz=null;
        //将包路径中的点替换成斜线
        String fileStub=name.replace(".","/");
        String javaFilename=fileStub+".java";
        String classFilename=fileStub+".class";
        File javaFile=new File(javaFilename);
        File classFile=new File(classFilename);
        /* 当指定Java源文件存在，且Class文件不存在；或者Java源文件的修改时间比Class文件的修改时间更晚时，重新编译*/
        if (javaFile.exists()&&(!classFile.exists()||javaFile.lastModified()>classFile.lastModified())){
            try{
                if(!compile(javaFilename)||!classFile.exists()){
                    throw new ClassNotFoundException("ClassNotFoundException"+javaFilename);
                }
            }catch (IOException e){
                e.printStackTrace();
            }
        }
        //如果Class文件存在，系统负责将该文件转换成Class对象
        if(classFile.exists()){
            try{
                //将Class文件的二进制数据读入数组
                byte []raw =getBytes(classFilename);
                //将二进制数据转换Class对象
                clazz=defineClass(name,raw,0,raw.length);
            }catch (IOException e){
                e.printStackTrace();
            }
        }
        //如果clazz为null，辨明加载失败，抛出异常
        if(clazz == null){
            throw new ClassNotFoundException(name);
        }
        return clazz;

    }
    //主方法
    public static void main(String[] args) throws Exception {
        //如果运行时没有参数，即没有目标类
        if (args.length<1){
            System.out.println("缺少目标类，请按如下格式运行Java文件");
            System.out.println("java CompileClassLoader ClassName");
        }
        //第一个参数是需要运行的类
        String progClass=args[0];
        //剩下的参数将作为运行目标类是的参数，将这些参数复制到一个新数组中
        String [] progArgs=new String[args.length-1];
        System.arraycopy(args,1,progArgs,0,progArgs.length);
        CompileClassLoader ccl=new CompileClassLoader();
        //加载需要运行的类
        Class<?> clazz=ccl.loadClass(progClass);
        //获取主方法
        Method main =clazz.getMethod("main",(new String[0]).getClass());
        Object argsArray[] ={progArgs};
        main.invoke(null,argsArray);
    }
}
~~~

~~~java
public class Hello {
    public static void main(String[] args) {
        for (String arg:args){
            System.out.println("运行Hello的参数"+arg);
        }
    }
}
~~~

无需编译Hello类即可直接运行

再cmd中首先编译自定义类加载器

`javac -encoding utf-8 CompileClassLoader.java `

然后无须编译该Hello.java，可以直接使用如下命令来运行该Hello.java程序。

`java CompileClassLoader Hello aaa bbb`

结果：

> E:\IDEA\test\src>java CompileClassLoader Hello aaa 啊啊啊
> 运行Hello的参数aaa
> 运行Hello的参数啊啊啊

实际上，使用自定义的类加载器，可以实现如下常见功能。

执行代码前自动验证数字签名。

根据用户提供的密码解密代码，从而可以实现代码混淆器来避免反编译Class文件。

根据用户需求来动态地加载类。

根据应用需求把其他数据以字节码的形式加载到应用中。

### URLClassLoader类

Java为ClassLoader提供了一个URLClassLoader实现类，该类也是系统类加载器和扩展类加载器的父类（此处是父类，就是指类与类之间的继承关系）。URLClassLoader功能比较强大，它既可以从本地文件系统获取二进制文件来加载类，也可以从远程主机获取二进制文件来加载类。

在应用程序中可以直接使用URLClassLoader来加载类，URLClassLoader类提供了如下两个构造器。

URLClassLoader(URL[] urls)：使用默认的父类加载器创建一个ClassLoader对象，该对象将从urls所指定的系列路径来查询并加载类。

URLClassLoader(URL[] urls, ClassLoader parent)：使用指定的父类加载器创建一个ClassLoader对象，其他功能与前一个构造器相同。

示例：

~~~java
private static Connection conn;
    public static Connection getConn(String url,String user,String pass) throws Exception{
        if (conn==null) {
            URL[] urls = {new URL("file:mysql-connector-java-x.x.x-bin.jat")};//当前路径
            URLClassLoader urlClassLoader = new URLClassLoader(urls);
            //加载MYSQL的JDBC驱动
            Driver driver = (Driver) urlClassLoader.loadClass("com.mysql.jdbc.Driver").newInstance();
            Properties properties = new Properties();
            properties.setProperty("user", user);
            properties.setProperty("password", pass);
            conn = driver.connect(url, properties);
        }
        return conn;
    }
~~~

## 反射查看类信息

Java程序中的许多对象在运行时都会出现两种类型：编译时类型和运行时类型，例如代码：Person p=new Student();，这行代码将会生成一个p变量，该变量的编译时类型为Person，运行时类型为Student；

为了解决这些问题，程序需要在运行时发现对象和类的真实信息。为了解决这个问题，我们有以下两种做法。

第一种做法是假设在编译时和运行时都完全知道类型的具体信息，在这种情况下，我们可以直接先使用instanceof运算符进行判断前一个对象是否是后一个对象的实例（`a instanceof Math`），再利用强制类型转换将其转换成其运行时类型的变量即可。

第二种做法是编译时根本无法预知该对象和类可能属于哪些类，程序只依靠运行时信息来发现该对象和类的真实信息，这就必须使用反射。

第二种方式有如下两种优势:

 代码更安全。程序在编译阶段就可以检查需要访问的Class对象是否存在。

程序性能更好。因为这种方式无须调用方法，所以性能更好。

**从Class中获取信息**

Class提供的功能非常丰富，它可以获取该类里包含的构造器、方法、内部类、注释等信息，也可以获取该类所包括的属性（Field）信息——通过getFields()或getField(String name)方法即可。

具体方法清查阅API，以下为实例：

~~~java
@SuppressWarnings(value = "unchecked")
@Deprecated
public class ClassTest {
    private ClassTest(){}

    public ClassTest(String name){
        System.out.println("执行有参数的构造器");
    }

    public void info(){
        System.out.println("执行无参数的info方法");
    }
    public  void info(String str){
        System.out.println("执行有参数的info方法"+" 参数值为： "+str);
    }
    //测试用内部类
    class  Inner{}

    public static void main(String[] args) throws Exception{
        Class<ClassTest> clazz=ClassTest.class;
        //获取Class对象对应的类的全部构造器
        Constructor<?>[] constructors = clazz.getDeclaredConstructors();
        for(Constructor c:constructors){
            System.out.println(c);
        }
        //获取全部public构造器
        Constructor<?>[] publicConstructors = clazz.getConstructors();
        System.out.println("全部public构造器如下：");
        for (Constructor c:publicConstructors){
            System.out.println(c);
        }
        //获取全部public方法
        Method[] methods = clazz.getMethods();
        System.out.println("ClassTest的全部public方法");
        for (Method md:methods){
            System.out.println(md);
        }
        //获取指定方法
        System.out.println("带一个字符串参数的info方法"+clazz.getMethod("info", String.class));
        //获取全部注释
        Annotation[] annotations = clazz.getAnnotations();
        System.out.println("ClassTest的全部Annotation：");
        for (Annotation a:annotations){
            System.out.println(a);
        }
        System.out.println("该类上的@SuppressWarning注释为："+clazz.getAnnotation(SuppressWarnings.class));
        //获取全部内部类
        Class<?>[] inners = clazz.getDeclaredClasses();
        System.out.println("全部内部类：");
        for (Class c:inners){
            System.out.println(c);
        }
        //使用Class.forName()方法加载Inner内部类
        Class inClazz=Class.forName("ClassTest$Inner");
        //通过getDeclaringClass()访问外部类
        System.out.println("inClazz对应的外部类："+inClazz.getDeclaringClass());
        System.out.println("包为"+clazz.getPackage());
        System.out.println("父类为："+clazz.getSuperclass());
    }
}
~~~

定义ClassTest类时使用了@SuppressWarnings注释，但程序运行时无法分析出该类里包含的该注释，这是因为@SuppressWarnings使用了@Retention(value=SOURCE)修饰，这表明@SuppressWarnings只能保存在源代码级别上，而通过ClassTest.class获取该类的运行时Class对象，所以程序无法访问到@SuppressWarnings注释。

## 反射生成并操作对象

### 创建对象

通过反射来生成对象有如下两种方式：

使用Class对象的newInstance()方法来创建该Class对象对应类的实例，这种方式要求该Class对象的对应类有默认构造器，而执行newInstance()方法时实际上是利用默认构造器来创建该类的实例。

先使用Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建该Class对象对应类的实例。通过这种方式可以选择使用指定的构造器来创建实例。

~~~java
public class test {
    private Map<String,Object> objectPool =new HashMap<>();
    //创建对象的方法
    private Object createObject(String clazzName) throws Exception{
        Class<?> clazz =Class.forName(clazzName);
        return clazz.newInstance();
    }
    //根据指定文件来初始化对象池，根据配置文件来创建对象
    public void initPool(String fileName)throws Exception{
        try(FileInputStream fileInputStream=new FileInputStream(fileName)){
            Properties properties=new Properties();
            properties.load(fileInputStream);
            for (String name:properties.stringPropertyNames()){
                //每取出一堆KV对，根据value创建一个对象，调用createObject创建对象，并将对象添加到对象池
                objectPool.put(name,createObject(properties.getProperty(name)));
            }
        }catch (IOException e){
            System.out.println("读取"+fileName+"异常");
        }

    }
    public Object getObject(String name){
        return objectPool.get(name);
    }
    public static void main(String[] args) throws Exception {
        test pf =new test();
        pf.initPool("obj.txt");
        System.out.println(pf.getObject("a"));
        System.out.println(pf.getObject("b"));
    }
}
obj.txt:
a=java.util.Date
b=javax.swing.JFrame
~~~

如果不想利用默认构造器来创建Java对象，而想利用指定的构造器来创建Java对象，则需要利用Constructor对象，每个Constructor对应一个构造器。为了利用指定的构造器来创建Java对象，需要如下3个步骤:

（1）获取该类的Class对象。

（2）利用Class对象的getConstructor()方法来获取指定的构造器。

（3）调用Constructor的newInstance()方法来创建Java对象。

~~~java
public static void main(String[] args) throws Exception {
        //获取JFrame对应的Class对象
       Class<?> jframeClazz=Class.forName("javax.swing.JFrame");
       //获取一个带字符串参数的构造器
        Constructor constructor=jframeClazz.getConstructor(String.class);
        Object obj=constructor.newInstance("测试");
        System.out.println(obj);
    }
~~~

### 调用方法

当获得某个类对应的Class对象后，就可以通过该Class对象的getMethods()方法或者getMethod()方法来获取全部方法或指定方法——这两个方法的返回值是Method数组，或者Method对象。

每个Method对象对应一个方法，获得Method对象后，程序就可通过该Method来调用它对应的方法。在Method里包含一个invoke()方法，该方法的签名如下。

Object invoke(Object obj, Object... args)：该方法中的obj是执行该方法的主调，后面的args是执行该方法时传入该方法的实参。

~~~java
//获取实现类的Class对象
Class<?> targetClass =target.getClass();
//获取对应的方法
Method method =targetClass.getMethod(methodName,String.class);
//通过invoke方法执行
method.invoke(target,config.getProperty(name))
~~~

当通过Method的invoke()方法来调用对应的方法时，Java会要求程序必须有调用该方法的权限。如果程序确实需要调用某个对象的private方法，则可以先调用Method对象的如下方法。

> setAccessible(boolean flag)：将Method对象的accessible设置为指定的布尔值。值为true，指示该Method在使用时应该取消Java语言的访问权限检查；值为false，则指示该Method在使用时要实施Java语言的访问权限检查。

### 访问属性值

通过Class对象的getFields()或getField()方法可以获取该类所包括的全部Field或指定Field。Field提供了如下两组方法来读取或设置Field值。

getXxx(Object obj)：获取obj对象该Field的属性值。此处的Xxx对应8个基本类型，如果该属性的类型是引用类型，则取消get后面的Xxx。

setXxx(Object obj , Xxx val)：将obj对象的该Field设置成val值。此处的Xxx对应8个基本类型，如果该属性的类型是引用类型，则取消set后面的Xxx。

使用这两个方法可以随意地访问指定对象的所有属性，包括private访问控制的属性。

~~~java
public class test {
    public static void main(String[] args) throws Exception{
        Person p=new Person();
        Class<Person> personClass=Person.class;
        Field name = personClass.getDeclaredField("name");
        //设置通过反射访问该Field时取消访问权限检查
        name.setAccessible(true);
        name.set(p,"HaHa");
        Field age = personClass.getDeclaredField("age");
        age.setAccessible(true);
        age.set(p,30);
        System.out.println(p);
    }
}
class Person{
    private  String name;
    private int age;

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
~~~

### 操作数组

在java.lang.reflect包下还提供了一个Array类，Array对象可以代表所有的数组。程序可以通过使用Array来动态地创建数组，操作数组元素等。

Array提供了如下几类方法。

static Object newInstance(Class<?> componentType, int... length)：创建一个具有指定的元素类型、指定维度的新数组。

static xxx getXxx(Object array, int index)：返回array数组中第index个元素。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变为get(Object array, intindex)。

static void setXxx(Object array, int index, xxx val)：将array数组中第index个元素的值设为val。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变成set(Object array, intindex, Object val)。

~~~java
//创建一个元素类型为String，长度为10的数组
        Object arr= Array.newInstance(String.class,10);
        //为arr数组中index为5，6的元素赋值
        Array.set(arr,5,"aaa");
        Array.set(arr,6,"bbb");
        //依次取出index为5，6的元素的值
        Object b1=Array.get(arr,5);
        Object b2=Array.get(arr,6);
        System.out.println(b1);
        System.out.println(b2);
~~~



# 

# 单元测试

### 1.概述

java单元测试是最小的功能单元测试代码, 单元测试就是针对单个java方法的测试。java程序的最小功能单元是方法。

- main方法进行测试的缺点:
  - 只能有一个main()方法, 不能把测试代码分离出来
  - 无法打印出测试结果和期望结果.例如: expected: 3628800, but actual: 123456
- 单元测试的优点:
  - 确保单个方法正常运行
  - 如果修改了方法代码, 只需要保其对应的单元测试通过就可以了
  - 测试代码本省就可以作为示例代码
  - 可以自动化运行所有测试并获得报告

**前期准备**

导入maven依赖（或导入jar包）：

```java
<dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
     <!-- junit的版本有3.x, 4.x, 5.x 5.x还没有发布, 现在都用是4.x -->
</dependency>
```

### 2.单元测试

java中常用的注解介绍：

@BeforeClass：针对所有测试，只执行一次，且必须为static void 

@Before：初始化方法 

@Test：测试方法，在这里可以测试期望异常和超时时间 

@Test(timeout=1000)可以设置超时时间，单位毫秒

@Test(expected=Exception.class), 对可能发生的每种类型的异常进行测试

> ```java
> // 运行如下代码, 正常运行, 确实发生了ArithmeticException异常, 代码通过
> @Test(expected = ArithmeticException.class)
>  public void testException() {
>     int i = 1 / 0;
>  }
> //运行如下代码: 有报错信息
> @Test(expected = ArithmeticException.class)
>  public void testException() {
>     int i = 1 / 1;
>  }
> ```

@After：释放资源 

@AfterClass：针对所有测试，只执行一次，且必须为static void 

@Ignore：忽略的测试方法  

编写java类

```java
public class Calculate{
    public int calculate(int a,int b){
        return a+b;
    }
}
```

编写测试类

```java
public class TestCaculate{
    @Test
    public void calculate(){
        assertEquals(3,new Calculate().calculate(1,2));
        assertEquals(6,new Calculate().calculate(1,2));
    }
}
```

> 测试结果：java.lang.AssertionError: expected:<6> but was:<3>

第一个方法是正确的没有报错

第二个方法清楚的显示了结果的对比情况

### 3.断言

包名：import static org.junit.Assert.*;

![img](https://images2015.cnblogs.com/blog/1160323/201706/1160323-20170615090821321-70871338.png)

Assert.assertEquals方法

**函数原型1：assertEquals([String message],expected,actual)**

参数说明：

message是个可选的消息，假如提供，将会在发生错误时报告这个消息。

expected是期望值，通常都是用户指定的内容。

actual是被测试的代码返回的实际值。

**函数原型2：assertEquals([String message],expected,actual,tolerance)**

参数说明：

message是个可选的消息，假如提供，将会在发生错误时报告这个消息。

expected是期望值，通常都是用户指定的内容。

actual是被测试的代码返回的实际值。

tolerance是误差参数，参加比较的两个浮点数在这个误差之内则会被认为是相等的。

### 4.Spring整合Junit

在测试类中，每个测试方法都有以下两行代码：
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
IAccountService as = ac.getBean("accountService",IAccountService.class);
这两行代码的作用是获取容器，如果不写的话，直接会提示空指针异常。所以又不能轻易删掉。

步骤：

导入jar包

使用@RunWith替换原有运行器

```java
@RunWith(SpringJUnit4ClassRunner.class)
public class AccountServiceTest {
}
```

使用@ContextConfiguration指定配置文件

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations= {"classpath:bean.xml"})
public class AccountServiceTest {
}
```

使用@Autowired注入数据

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations= {"classpath:bean.xml"})
public class AccountServiceTest {
@Autowired
private IAccountService as ;
}
```

# jmeter测试

## 安装jdk

**jmeter运行所需环境**

这里简略介绍安装过程

1.解压jdk安装包

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/001.jpg)

2.配置环境变量

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/002.jpg)

3.查看java –version，安装成功

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/003.jpg)

## 安装jmeter

官网地址http://jmeter.apache.org/download_jmeter.cgi     jmeter自带中文

下载后解压，点击 jmeter.bat运行

## 新手入门

### 设置为中文

![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/004.jpg)

### 设置为默认中文

首先打开jmeter.properties(可以用文本工具如notepad++打开) 在第37行设置language=zh_CN



![文件无法预览。](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/005.jpg)

### 性能测试之并发测试



## jmeter性能测试--接口测试







