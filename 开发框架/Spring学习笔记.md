# Spring学习笔记

Spring是分层的Java SE/EE应用full-stack轻量级开源框架，以IoC（Inverse Of Control：
反转控制）和AOP（Aspect Oriented Programming：面向切面编程）为内核，提供了表现层Spring
MVC和持久层Spring JDBC以及业务层事务管理等众多的企业级应用技术，还能整合开源世界众多
著名的第三方框架和类库，逐渐成为使用最多的Java EE企业应用开源框架。

本文中大部分主要是Spring Core中的内容的整理总结。

在Core中都包括IOC Container（IOC容器），Events（事件），Resources（资源），i18n（），Validation（验证），DataBinging（数据绑定），Type Conversion（类型转换），SpEL，AOP

# 一.概述

在Spring框架中最重要的就是控制反转容器和面向切面编程

**发展历程**

1997年IBM提出了EJB的思想
1998年，SUN制定开发标准规范EJB1.0
1999年，EJB1.1发布
2001年，EJB2.0发布
2003年，EJB2.1发布
2006年，EJB3.0发布
Rod Johnson（spring之父）
Expert One-to-One J2EE Design and Development(2002)
阐述了J2EE使用EJB开发设计的优点及解决方案
Expert One-to-One J2EE Development without EJB(2004)
阐述了J2EE开发不使用EJB的解决方式（Spring雏形）
2017年9月份发布了spring的最新版本spring 5.0通用版（GA）

**体系结构**

![img](https://img2018.cnblogs.com/blog/480452/201903/480452-20190318225849216-2097896352.png)

# 二. IOC容器

本章介绍了控制反转（IoC）原理的Spring框架实现。IoC也称为依赖注入（DI）。在此过程中，对象仅通过构造函数参数，工厂方法的参数或在构造或从工厂方法返回后在对象实例上设置的属性来定义其依赖项（即，与它们一起使用的其他对象） 。然后，容器在创建bean时注入那些依赖项。此过程从根本上讲是通过使用类的直接构造或诸如服务定位器模式的机制来控制其依赖项的实例化或位置的Bean本身的逆过程（因此称为控件的倒置）。

在`org.springframework.beans`和`org.springframework.context`包是Spring框架的IoC容器的基础。

在Spring中，构成应用程序主干并由Spring IoC容器管理的对象称为bean。Bean是由Spring IoC容器实例化，组装和以其他方式管理的对象。

## 2.1程序的耦合

​	耦合性(Coupling)，也叫耦合度，是对模块间关联程度的度量。耦合的强弱取决于模块间接口的复杂性、调
用模块的方式以及通过界面传送数据的多少。模块间的耦合度是指模块之间的依赖关系，包括控制关系、调用关
系、数据传递关系。模块间联系越多，其耦合性越强，同时表明其独立性越差( 降低耦合性，可以提高其独立
性)。耦合性存在于各个领域，而非软件设计中独有的，但是我们只讨论软件工程中的耦合。
​	在软件工程中，耦合指的就是就是对象之间的依赖性。对象之间的耦合越高，维护成本越高。因此对象的设计
应使类和构件之间的耦合最小。软件设计中通常用耦合度和内聚度作为衡量模块独立程度的标准。**划分模块的一个**
**准则就是高内聚低耦合。**

**它有如下分类：**
（1）内容耦合。当一个模块直接修改或操作另一个模块的数据时，或一个模块不通过正常入口而转入另
一个模块时，这样的耦合被称为内容耦合。内容耦合是最高程度的耦合，应该避免使用之。
（2）公共耦合。两个或两个以上的模块共同引用一个全局数据项，这种耦合被称为公共耦合。在具有大
量公共耦合的结构中，确定究竟是哪个模块给全局变量赋了一个特定的值是十分困难的。
（3）外部耦合。一组模块都访问同一全局简单变量而不是同一全局数据结构，而且不是通过参数表传
递该全局变量的信息，则称之为外部耦合。
（4）控制耦合。一个模块通过接口向另一个模块传递一个控制信号，接受信号的模块根据信号值而进
行适当的动作，这种耦合被称为控制耦合。
（5）标记耦合。若一个模块A通过接口向两个模块B和C传递一个公共参数，那么称模块B和C之间
存在一个标记耦合。
（6）数据耦合。模块之间通过参数来传递数据，那么被称为数据耦合。数据耦合是最低的一种耦合形
式，系统中一般都存在这种类型的耦合，因为为了完成一些有意义的功能，往往需要将某些模块的输出数据作为另
一些模块的输入数据。
（7）非直接耦合。两个模块之间没有直接关系，它们之间的联系完全是通过主模块的控制和调用来实
现的。

**耦合是影响软件复杂程度和设计质量的一个重要因素，在设计上我们应采用以下原则：如果模块间必须**
**存在耦合，就尽量使用数据耦合，少用控制耦合，限制公共耦合的范围，尽量避免使用内容耦合。**

内聚与耦合
内聚标志一个模块内各个元素彼此结合的紧密程度，它是信息隐蔽和局部化概念的自然扩展。内聚是从功能角度来度量模块内的联系，一个好的内聚模块应当恰好做一件事。它描述的是模块内的功能联系。耦合是软件结构中各模块之间相互连接的一种度量，耦合强弱取决于模块间接口的复杂程度、进入或访问一个模块的点以及通过接口的数据。程序讲究的是低耦合，高内聚。就是同一个模块内的各个元素之间要高度紧密，但是各个模块之间的相互依存度却要不那么紧密。
内聚和耦合是密切相关的，同其他模块存在高耦合的模块意味着低内聚，而高内聚的模块意味着该模块同其他
模块之间是低耦合。在进行软件设计时，应力争做到高内聚，低耦合

> 我们在开发中有些依赖关系是必须的，而有些时可以通过优化代码来解决的。
>
> public class AccountServiceImpl implements IAccountService {
> private IAccountDao accountDao = new AccountDaoImpl();
> }
> 上面的代码表示：
> 业务层调用持久层，并且此时业务层在依赖持久层的接口和实现类。如果此时没有持久层实现类，编译
> 将不能通过。这种编译期依赖关系，应该在我们开发中杜绝。我们需要优化代码解决。再假如
>
> public static void main(String[] args) throws Exception {
> 1.注册驱动
> //DriverManager.registerDriver(new com.mysql.jdbc.Driver());
> Class.forName("com.mysql.jdbc.Driver");
> 2.获取连接
> 3.获取预处理sql语句对象
> 4.获取结果集
> 5.遍历结果集
> }
> }
>
> 我们的类依赖了数据库的具体驱动类（MySQL），如果这时候更换了数据库品牌（比如Oracle），需要
> 修改源码来重新数据库驱动。这显然不是我们想要的。

## 2.2 程序的解耦

思路：

当是我们讲解jdbc时，是通过反射来注册驱动的，代码如下：
Class.forName("com.mysql.jdbc.Driver");//此处只是一个字符串
此时的好处是，我们的类中不再依赖具体的驱动类，此时就算删除mysql的驱动jar包，依然可以编译（运
行就不要想了，没有驱动不可能运行成功的）。
同时，也产生了一个新的问题，mysql驱动的全限定类名字符串是在java类中写死的，一旦要改还是要修改
源码。
解决这个问题也很简单，使用配置文件配置。

**工厂模式解耦合**：

在实际开发中我们可以把三层的对象都使用配置文件配置起来，当启动服务器应用加载的时候，让一个类中的
方法通过读取配置文件，把这些对象创建出来并存起来。在接下来的使用的时候，直接拿过来用就好了。
那么，这个读取配置文件，创建和获取三层对象的类就是工厂。

**控制反转**：

但上述工厂模式解耦合也有问题

1.存哪去？
分析：由于我们是很多对象，肯定要找个集合来存。这时候有Map和List供选择。到底选Map还是List就看我们有没有查找需求。有查找需求，选Map。所以我们的答案就是在应用加载时，创建一个Map，用于存放三层对象。
我们把这个map称之为容器。

2.什么是工厂？

工厂就是负责给我们从容器中获取指定对象的类。这时候我们获取对象的方式发生了改变。
原来：
我们在获取对象时，都是采用new的方式。是主动的。

现在：
我们获取对象时，同时跟工厂要，有工厂为我们查找或者创建对象。是被动的。

这种被动接收的方式获取对象的思想就是**控制反转**，它是spring框架的核心之一。

**明确ioc的作用：**
削减计算机程序的耦合(解除我们代码中的依赖关系)。

## 2.3 IOC概述

`org.springframework.context.ApplicationContext`接口代表Spring IoC容器，并负责实例化，配置和组装Bean。容器通过读取配置元数据来获取有关要实例化，配置和组装哪些对象的指令。配置元数据以XML，Java批注或Java代码表示。它使您能够表达组成应用程序的对象以及这些对象之间的丰富相互依赖关系。

### 2.3.1 配置元数据

Spring IoC容器使用一种形式的配置元数据。此配置元数据表示您作为应用程序开发人员如何告诉Spring容器实例化，配置和组装应用程序中的对象。

Spring中主要采用三种方式配置元数据：

> 基于xml的方式
>
> 基于注释的方式：从Spring2.5后可以基于注解配置
>
> 基于Java的配置：从Spring3.0开始，Spring JavaConfig项目提供的许多功能成为了Spring核心框架的一部分。因此，您可以使用Java而不是XML文件来定义应用程序类外部的bean。

Spring配置由容器必须管理的至少一个（通常是一个以上）bean定义组成。基于XML的配置元数据将这些bean配置为`<bean/>`顶级元素内的`<beans/>`元素。Java配置通常`@Bean`在`@Configuration`类中使用带注释的方法。

简单示例：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
  https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  
        <!-- collaborators and configuration for this bean go here -->
    </bean>
    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>
    <!-- more bean definitions go here -->
</beans>
~~~

id属性是标识单个bean的；class属性是定义Bean的来源

### 2.3.2 实例化容器

提供给`ApplicationContext`构造函数的一个或多个位置路径是资源字符串，可让容器从各种外部资源（例如本地文件系统，Java等）中加载配置元数据`CLASSPATH`。

示例：

~~~java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
~~~

### 2.3.3  使用容器

该`ApplicationContext`是一个维护bean定义以及相互依赖的注册表的高级工厂的接口。通过使用方法 `T getBean(String name, Class<T> requiredType)`，您可以检索bean的实例。

~~~JAVA
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);
// use configured instance
List<String> userList = service.getUsernameList();
~~~



## 2.4 基于XML的IOC配置（IOC解耦合）

### 2.4.1 IOC入门

此入门是解决账户的业务层和持久层的依赖关系。

拷贝jar包到工程

再根目录下创建一个xml文件（配置文件）

导入约束

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
</beans>
~~~

配置文件中配置service和dao

~~~xml
<!-- bean标签：用于配置让spring创建对象，并且存入ioc容器之中
id属性：对象的唯一标识。
class属性：指定要创建对象的全限定类名
-->
<!-- 配置service -->
<bean id="accountService" class="com.itheima.service.impl.AccountServiceImpl">
</bean>
<!-- 配置dao -->
<bean id="accountDao" class="com.itheima.dao.impl.AccountDaoImpl"></bean>
~~~

### 2.4.2基于xml的IOC

#### spring中的工厂的类结构图

![img](http://img2018.cnblogs.com/i-beta/1598493/202002/1598493-20200215151523468-1242212011.png)

BeanFactory才是Spring容器中的顶层接口。ApplicationContext是它的子接口。
BeanFactory和ApplicationContext的区别：
创建对象的时间点不一样。
ApplicationContext：只要一读取配置文件，默认情况下就会创建对象。
BeanFactory：什么使用什么时候创建对象。

**ApplicationContext接口的实现类**

ClassPathXmlApplicationContext：
它是从类的根路径下加载配置文件推荐使用这种
FileSystemXmlApplicationContext：
它是从磁盘路径上加载配置文件，配置文件可以在磁盘的任意位置。
AnnotationConfigApplicationContext:
当我们使用注解配置容器对象时，需要使用此类来创建spring容器。它用来读取注解。

#### IOC中的bean标签和管理对象的细节

**bean标签**

作用：
用于配置对象让spring来创建的。
默认情况下它调用的是类中的无参构造函数。如果没有无参构造函数则不能创建成功。
属性：
id：给对象在容器中提供一个唯一标识。用于获取对象。
class：指定类的全限定类名。用于反射创建对象。默认情况下调用无参构造函数。
scope：指定对象的作用范围。

* singleton :默认值，单例的.
* prototype :多例的.
* request :WEB项目中,Spring创建一个Bean的对象,将对象存入到request域中.
* session :WEB项目中,Spring创建一个Bean的对象,将对象存入到session域中.
* global session :WEB项目中,应用在Portlet环境.如果没有Portlet环境那么
globalSession相当于session.
init-method：指定类中的初始化方法名称。
destroy-method：指定类中销毁方法名称。

**bean的作用范围及生命周期**

单例对象：scope="singleton"

> 一个应用只有一个对象的实例。它的作用范围就是整个引用。
> 生命周期：
> 对象出生：当应用加载，创建容器时，对象就被创建了。
> 对象活着：只要容器在，对象一直活着。
> 对象死亡：当应用卸载，销毁容器时，对象就被销毁了。

多例对象：scope="prototype"

> 每次访问对象时，都会重新创建对象实例。
> 生命周期：
> 对象出生：当使用对象时，创建新的对象实例。
> 对象活着：只要对象在使用中，就一直活着。
> 对象死亡：当对象长时间不用时，被java的垃圾回收器回收了。

####  bean实例化的三种方法

第一种方式：使用默认无参构造函数

~~~xml
<!--在默认情况下：
它会根据默认无参构造函数来创建类对象。如果bean中没有默认无参构造函数，将会创建失败。-->
<bean id="accountService" class="com.itheima.service.impl.AccountServiceImpl"/>
~~~

第二种方式：spring管理静态工厂-使用静态工厂的方法创建对象

~~~xml
/**
* 模拟一个静态工厂，创建业务层实现类
*/
public class StaticFactory {
public static IAccountService createAccountService(){
return new AccountServiceImpl();
}
}
<!-- 此种方式是:
使用StaticFactory类中的静态方法createAccountService创建对象，并存入spring容器
id属性：指定bean的id，用于从容器中获取
class属性：指定静态工厂的全限定类名
factory-method属性：指定生产对象的静态方法
-->
<bean id="accountService"
class="com.loserfromlazy.factory.StaticFactory"
factory-method="createAccountService"></bean>

~~~

第三种方式：spring管理实例工厂-使用实例工厂的方法创建对象

~~~xml
/**
* 模拟一个实例工厂，创建业务层实现类
* 此工厂创建对象，必须现有工厂实例对象，再调用方法
*/
public class InstanceFactory {
public IAccountService createAccountService(){
return new AccountServiceImpl();
}
}
<!-- 此种方式是：
先把工厂的创建交给spring来管理。
然后在使用工厂的bean来调用里面的方法
factory-bean属性：用于指定实例工厂bean的id。
factory-method属性：用于指定实例工厂中创建对象的方法。
-->
<bean id="instancFactory" class="com.loserfromlazy.factory.InstanceFactory"></bean>
<bean id="accountService"
factory-bean="instancFactory"
factory-method="createAccountService"></bean>
~~~

## 2.5 依赖注入

依赖注入：Dependency Injection。它是spring框架核心ioc的具体实现。
我们的程序在编写时，通过控制反转，把对象的创建交给了spring，但是代码中不可能出现没有依赖的情况。
ioc解耦只是降低他们的依赖关系，但不会消除。例如：我们的业务层仍会调用持久层的方法。那这种业务层和持久层的依赖关系，在使用spring之后，就让spring来维护了。简单的说，就是坐等框架把持久层对象传入业务层，而不用我们自己去获取。

**构造函数注入**

eg：

~~~java
public class AccountServiceImpl implements IAccountService {
    private String name;
    private Integer age;
    private Date birthday;
    public AccountServiceImpl(String name, Integer age, Date birthday) {
        this.name = name;
        this.age = age;
        this.birthday = birthday;
    }
    @Override
    public void saveAccount() {
        System.out.println(name+","+age+","+birthday);
    }
}
~~~

~~~xml
<!-- 使用构造函数的方式，给service中的属性传值
要求：
类中需要提供一个对应参数列表的构造函数。
涉及的标签：
constructor-arg
属性：
index:指定参数在构造函数参数列表的索引位置
type:指定参数在构造函数中的数据类型
name:指定参数在构造函数中的名称用这个找给谁赋值
=======上面三个都是找给谁赋值，下面两个指的是赋什么值的==============
value:它能赋的值是基本数据类型和String类型
ref:它能赋的值是其他bean类型，也就是说，必须得是在配置文件中配置过的bean
-->
<bean id="accountService" class="com.loserfromlazy.service.impl.AccountServiceImpl">
<constructor-arg name="name" value="张三"></constructor-arg>
<constructor-arg name="age" value="18"></constructor-arg>
<constructor-arg name="birthday" ref="now"></constructor-arg>
</bean>
<bean id="now" class="java.util.Date"></bean>
~~~

**set方法注入**

~~~java
public class AccountServiceImpl implements IAccountService {
    private String name;
    private Integer age;
    private Date birthday;
    public void setName(String name) {
    	this.name = name;
    }
    public void setAge(Integer age) {
   	 this.age = age;
    }
    public void setBirthday(Date birthday) {
   	 this.birthday = birthday;
    }
    @Override
    public void saveAccount() {
   	 System.out.println(name+","+age+","+birthday);
    }
}
~~~

~~~xml
<!-- 通过配置文件给bean中的属性传值：使用set方法的方式
涉及的标签：
property
属性：
name：找的是类中set方法后面的部分
ref：给属性赋值是其他bean类型的
value：给属性赋值是基本数据类型和string类型的
实际开发中，此种方式用的较多。
-->
<bean id="accountService" class="com.loserfromlzay.service.impl.AccountServiceImpl">
<property name="name" value="test"></property>
<property name="age" value="21"></property>
<property name="birthday" ref="now"></property>
</bean>
<bean id="now" class="java.util.Date"></bean>
~~~

**使用p名称控件注入数据（本质还是set方法）**

java类与上面一样

~~~xml
<bean id="accountService"
class="com.loserfromlzay.service.impl.AccountServiceImpl4"
p:name="test" p:age="21" p:birthday-ref="now"/>
</beans>
~~~

**注入集合属性**

~~~java
public class AccountServiceImpl implements IAccountService {
    private String[] myStrs;
    private List<String> myList;
    private Set<String> mySet;
    private Map<String,String> myMap;
    private Properties myProps;
    public void setMyStrs(String[] myStrs) {
   	 this.myStrs = myStrs;
    }
    public void setMyList(List<String> myList) {
   	 this.myList = myList;
    }
    public void setMySet(Set<String> mySet) {
   	 this.mySet = mySet;
    }
    public void setMyMap(Map<String, String> myMap) {
   	 this.myMap = myMap;
    }
    public void setMyProps(Properties myProps) {
   	 this.myProps = myProps;
    }
    @Override
    public void saveAccount() {
        System.out.println(Arrays.toString(myStrs));
        System.out.println(myList);
        System.out.println(mySet);
        System.out.println(myMap);
        System.out.println(myProps);
    }
}
~~~

~~~xml
<!-- 注入集合数据
List结构的：
array,list,set
Map结构的
map,entry,props,prop
-->
<bean id="accountService" class="com.itheima.service.impl.AccountServiceImpl">
<!-- 在注入集合数据时，只要结构相同，标签可以互换-->
<!-- 给数组注入数据-->
	<property name="myStrs">
        <set>
        <value>AAA</value>
        <value>BBB</value>
        <value>CCC</value>
        </set>
	</property>
<!-- 注入list集合数据-->
    <property name="myList">
        <array>
        <value>AAA</value>
        <value>BBB</value>
        <value>CCC</value>
        </array>
    </property>
<!-- 注入set集合数据-->
<property name="mySet">
    <list>
        <value>AAA</value>
        <value>BBB</value>
        <value>CCC</value>
        </list>
    </property>
<!-- 注入Map数据-->
    <property name="myMap">
        <props>
        <prop key="testA">aaa</prop>
        <prop key="testB">bbb</prop>
        </props>
    </property>
<!-- 注入properties数据-->
    <property name="myProps">
        <map>
        <entry key="testA" value="aaa">
        </entry>
        <entry key="testB">
            <value>bbb</value>
        </entry>
        </map>
    </property>
</bean>
~~~

## 2.6 基于注解的IOC配置

注解配置和xml 配置要实现的功能都是一样的，都是要降低程序间的耦合。只是配置的形式不一样。

### 2.6.1常用注释

用于创建对象相当于< bean id="" class="">

**@Component**

作用：
把资源让spring来管理。相当于在xml中配置一个bean。
属性：
value：指定bean的id。如果不指定value属性，默认bean的id是当前类的类名。首字母小写。

**@Controller @Service @Repository**

> 他们三个注解都是针对一个的衍生注解，他们的作用及属性都是一模一样的。
> 他们只不过是提供了更加明确的语义化。
> @Controller：一般用于表现层的注解。
> @Service：一般用于业务层的注解。
> @Repository：一般用于持久层的注解。
> 细节：如果注解中有且只有一个属性要赋值时，且名称是value，value在赋值是可以不写。

用于注入数据的 相当于< property name="" ref="">或者 < property name="" value="">

**@Autowired**

作用：
自动按照类型注入。当使用注解注入属性时，set方法可以省略。它只能注入其他bean类型。当有多个
类型匹配时，使用要注入的对象变量名称作为bean的id，在spring容器查找，找到了也可以注入成功。找不到
就报错。

**@Qualifier**

作用：
在自动按照类型注入的基础之上，再按照Bean 的id 注入。它在给字段注入时不能独立使用，必须和
@Autowire一起使用；但是给方法参数注入时，可以独立使用。
属性：
value：指定bean的id。

**@Resource**

作用：
直接按照Bean的id注入。它也只能注入其他bean类型。
属性：
name：指定bean的id。

**@Value**

作用：
注入基本数据类型和String类型数据的
属性：
value：用于指定值

用于改变作用范围的 相当于：< bean id="" class="" scope="">

**@Scope**

作用：
指定bean的作用范围。
属性：
value：指定范围的值。
取值：singleton prototype request session globalsession

和生命周期相关的 相当于：< bean id="" class="" init-method="" destroy-method="" />

 **@PostConstruct**
作用：
用于指定初始化方法。
 **@PreDestroy**
作用：
用于指定销毁方法。

# 三. SpEL

SpEL（Spring Expression Language），即Spring表达式语言，是比JSP的EL更强大的一种表达式语言。

常见用法：在@Value注解中；在xml配置中；或者是在代码块中使用Expression。

## 3.1 用法

### @Value

~~~java
@Value("#{表达式}")
public String arg;
~~~

### < bean>配置

~~~xml
<bean id="xxx" class="com.xxx.xxx">
    <!--同@Value #{}内是表达式的值，可以放在property或constructor-arg内-->
    <property name="arg" value="#{表达式}"/>
</bean>
~~~

### Expression

~~~java
import org.springframework.expression.Expression;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
 
public class SpELTest {
 
    public static void main(String[] args) {
 
        //创建ExpressionParser解析表达式
        ExpressionParser parser = new SpelExpressionParser();
        //表达式放置
        Expression expression = parser.parseExpression("表达式");
        //执行表达式，默认容器是spring本身的容器：ApplicationContext
        Object value = expression.getValue();
        
        /**如果使用其他的容器，则用下面的方法*/
        //创建一个虚拟的容器EvaluationContext
        StandardEvaluationContext ctx = new StandardEvaluationContext();
        //向容器内添加bean
        BeanA beanA = new BeanA();
        ctx.setVariable("bean_id", beanA);
        
        //setRootObject并非必须；一个EvaluationContext只能有一个RootObject，引用它的属性时，可以不加前缀
        ctx.setRootObject(XXX);
        
        //getValue有参数ctx，从新的容器中根据SpEL表达式获取所需的值
        Object value = expression.getValue(ctx);
    }
}
~~~

## 3.2 语法

# 四. Spring中的配置文件

Spring配置文件是用于指导Spring工厂进行Bean生产、依赖关系注入（装配）及Bean实例分发的"图纸"。

# 五. Spring整合Junit

在测试类中，每个测试方法都有以下两行代码：
	ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
	IAccountService as = ac.getBean("accountService",IAccountService.class);
这两行代码的作用是获取容器，如果不写的话，直接会提示空指针异常。所以又不能轻易删掉。

步骤：

导入jar包

使用@RunWith替换原有运行器

~~~java
@RunWith(SpringJUnit4ClassRunner.class)
public class AccountServiceTest {
}
~~~

使用@ContextConfiguration指定配置文件

~~~java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations= {"classpath:bean.xml"})
public class AccountServiceTest {
}
~~~

使用@Autowired注入数据

~~~java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations= {"classpath:bean.xml"})
public class AccountServiceTest {
@Autowired
private IAccountService as ;
}
~~~

# 六 . AOP

## 5.1概述

AOP：全称是Aspect Oriented Programming即：面向切面编程。

简单的说它就是把我们程序重复的代码抽取出来，在需要执行的时候，使用动态代理的技术，在不修改源码的
基础上，对我们的已有方法进行增强。

作用：
	在程序运行期间，不修改源码对已有方法进行增强。
优势：
	减少重复代码
	提高开发效率
	维护方便

**AOP使用动态代理技术实现AOP**

~~~java
业务层接口
/**
* 转账
* @param sourceName
* @param targetName
* @param money
*/
void transfer(String sourceName,String targetName,Float money);
业务层实现类：
@Override
public void transfer(String sourceName, String targetName, Float money) {
//根据名称查询两个账户信息
Account source = accountDao.findByName(sourceName);
Account target = accountDao.findByName(targetName);
//转出账户减钱，转入账户加钱
source.setMoney(source.getMoney()-money);
target.setMoney(target.getMoney()+money);
//更新两个账户
accountDao.update(source);
int i=1/0; //模拟转账异常
accountDao.update(target);
}
当我们执行时，由于执行有异常，转账失败。但是因为我们是每次执行持久层方法都是独立事务，导致无法实
事务控制（不符合事务的一致性）
~~~

解决办法：
让业务层来控制事务的提交和回滚。

~~~java
public class AccountServiceImpl implements IAccountService {
private IAccountDao accountDao = new AccountDaoImpl();
@Override
public void saveAccount(Account account) {
    try {
        TransactionManager.beginTransaction();
        accountDao.save(account);
        TransactionManager.commit();
    } catch (Exception e) {
        TransactionManager.rollback();
        e.printStackTrace();
    }finally {
    	TransactionManager.release();
    }
}
@Override
public void updateAccount(Account account) {
	try {
            TransactionManager.beginTransaction();
            accountDao.update(account);
            TransactionManager.commit();
        } catch (Exception e) {
            TransactionManager.rollback();
            e.printStackTrace();
        }finally {
        	TransactionManager.release();
        }
	}

}
TransactionManager类的代码：
public class TransactionManager {
//定义一个DBAssit
private static DBAssit dbAssit = new DBAssit(C3P0Utils.getDataSource(),true);
//开启事务
public static void beginTransaction() {
    try {
    	dbAssit.getCurrentConnection().setAutoCommit(false);
    } catch (SQLException e) {
   	 	e.printStackTrace();
    }
}
//提交事务
public static void commit() {
    try {
    	dbAssit.getCurrentConnection().commit();
    } catch (SQLException e) {
    	e.printStackTrace();
    }
}
//回滚事务
public static void rollback() {
    try {
    	dbAssit.getCurrentConnection().rollback();
    } catch (SQLException e) {
    	e.printStackTrace();
    }
}
//释放资源
public static void release() {
    try {
    	dbAssit.releaseConnection();
    } catch (Exception e) {
    	e.printStackTrace();
    }
}
}
~~~

因为添加了事务控制所以业务层变得臃肿，业务层方法变得臃肿了，里面充斥着很多重复代码。并且业务层方法和事务控制方法耦合了。

使用动态代理来解决此问题，动态代理由两种方式

基于接口的动态代理

> 提供者：JDK官方的Proxy类。
> 要求：被代理类最少实现一个接口。

基于子类的动态代理

> 提供者：第三方的CGLib，如果报asmxxxx异常，需要导入asm.jar。
> 要求：被代理类不能用final修饰的类（最终类）。

## 5.2 AOP中的概念

Joinpoint(连接点):

> 所谓连接点是指那些被拦截到的点。在spring中,这些点指的是方法,因为spring只支持方法类型的
> 连接点。

Pointcut(切入点):

> 所谓切入点是指我们要对哪些Joinpoint进行拦截的定义。

Advice(通知/增强):

> 所谓通知是指拦截到Joinpoint之后所要做的事情就是通知。
> 通知的类型：前置通知,后置通知,异常通知,最终通知,环绕通知。

Introduction(引介):

> 引介是一种特殊的通知在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方
> 法或Field。

Target(目标对象):

> 代理的目标对象。

Weaving(织入):

> 是指把增强应用到目标对象来创建新的代理对象的过程。
> spring采用动态代理织入，而AspectJ采用编译期织入和类装载期织入。

Proxy（代理）:

> 一个类被AOP织入增强后，就产生一个结果代理类。

Aspect(切面):

> 是切入点和通知（引介）的结合。

## 5.3 基于XML的AOP

步骤

**把通知类用bean配置**

~~~xml
<!-- 配置通知-->
<bean id="txManager" class="com.itheima.utils.TransactionManager">
<property name="dbAssit" ref="dbAssit"></property>
</bean>
~~~

**使用aop:config声明aop配置**

~~~xml
aop:config:
作用：用于声明开始aop的配置
<aop:config>
<!-- 配置的代码都写在此处-->
</aop:config>
~~~

**使用aop:aspect配置切面**

~~~xml
aop:aspect
作用：
用于配置切面。
属性：
id：给切面提供一个唯一标识。
ref：引用配置好的通知类bean的id。
<aop:aspect id="txAdvice" ref="txManager">
<!--配置通知的类型要写在此处-->
</aop:aspect>
~~~

**使用aop:pointcut 配置切入点表达式**

~~~xml
aop:pointcut：
作用：
用于配置切入点表达式。就是指定对哪些类的哪些方法进行增强。
属性：
expression：用于定义切入点表达式。
id：用于给切入点表达式提供一个唯一标识
<aop:pointcut expression="execution(
public void com.itheima.service.impl.AccountServiceImpl.transfer(
java.lang.String, java.lang.String, java.lang.Float)
)" id="pt1"/>
~~~

**使用aop:xxx 配置对应的通知类型**

~~~xml
aop:before
作用：
用于配置前置通知。指定增强的方法在切入点方法之前执行
属性：
method:用于指定通知类中的增强方法名称
ponitcut-ref：用于指定切入点的表达式的引用
poinitcut：用于指定切入点表达式
执行时间点：
切入点方法执行之前执行
<aop:before method="beginTransaction" pointcut-ref="pt1"/>
aop:after-returning
作用：
用于配置后置通知
属性：
method：指定通知中方法的名称。
pointct：定义切入点表达式
pointcut-ref：指定切入点表达式的引用
执行时间点：
切入点方法正常执行之后。它和异常通知只能有一个执行
<aop:after-returning method="commit" pointcut-ref="pt1"/>
aop:after-throwing
作用：
用于配置异常通知
属性：
method：指定通知中方法的名称。
pointct：定义切入点表达式
pointcut-ref：指定切入点表达式的引用
执行时间点：
切入点方法执行产生异常后执行。它和后置通知只能执行一个
<aop:after-throwing method="rollback" pointcut-ref="pt1"/>
aop:after
作用：
用于配置最终通知
属性：
method：指定通知中方法的名称。
pointct：定义切入点表达式
pointcut-ref：指定切入点表达式的引用
执行时间点：
无论切入点方法执行时是否有异常，它都会在其后面执行。
<aop:after method="release" pointcut-ref="pt1"/>
~~~

**切入点表达式说明**

~~~
execution:匹配方法的执行(常用)
execution(表达式)
表达式语法：execution([修饰符] 返回值类型包名.类名.方法名(参数))
写法说明：
全匹配方式：
public void
com.itheima.service.impl.AccountServiceImpl.saveAccount(com.itheima.domain.Account)
访问修饰符可以省略
void
com.itheima.service.impl.AccountServiceImpl.saveAccount(com.itheima.domain.Account)
返回值可以使用*号，表示任意返回值
*
com.itheima.service.impl.AccountServiceImpl.saveAccount(com.itheima.domain.Account)
包名可以使用*号，表示任意包，但是有几级包，需要写几个*
* *.*.*.*.AccountServiceImpl.saveAccount(com.itheima.domain.Account)
使用..来表示当前包，及其子包
* com..AccountServiceImpl.saveAccount(com.itheima.domain.Account)
类名可以使用*号，表示任意类
* com..*.saveAccount(com.itheima.domain.Account)
方法名可以使用*号，表示任意方法
* com..*.*( com.itheima.domain.Account)
参数列表可以使用*，表示参数可以是任意数据类型，但是必须有参数
* com..*.*(*)
参数列表可以使用..表示有无参数均可，有参数可以是任意类型
* com..*.*(..)
全通配方式：
* *..*.*(..)
注：
通常情况下，我们都是对业务层的方法进行增强，所以切入点表达式都是切到业务层实现类。
execution(* com.itheima.service.impl.*.*(..))
~~~

**环绕通知**

~~~xml
配置方式:
<aop:config>
<aop:pointcut expression="execution(* com.itheima.service.impl.*.*(..))"
id="pt1"/>
<aop:aspect id="txAdvice" ref="txManager">
<!-- 配置环绕通知-->
<aop:around method="transactionAround" pointcut-ref="pt1"/>
</aop:aspect>
</aop:config>
aop:around：
作用：
用于配置环绕通知
属性：
method：指定通知中方法的名称。
pointct：定义切入点表达式
pointcut-ref：指定切入点表达式的引用
说明：
它是spring框架为我们提供的一种可以在代码中手动控制增强代码什么时候执行的方式。
注意：
通常情况下，环绕通知都是独立使用的
~~~

## 5.4 基于注解的AOP

步骤

把通知类使用注解配置

~~~java
@Component("txManager")
public class TransactionManager {
//定义一个DBAssit
@Autowired
private DBAssit dbAssit ;
}
~~~

再通知类上使用@Aspect注解声明切面

~~~java
作用：
把当前类声明为切面类。
@Component("txManager")
@Aspect//表明当前类是一个切面类
public class TransactionManager {
//定义一个DBAssit
@Autowired
private DBAssit dbAssit ;
}
~~~

再增强的方法上使用注解配置

~~~java
@Before
作用：
把当前方法看成是前置通知。
属性：
value：用于指定切入点表达式，还可以指定切入点表达式的引用。
//开启事务
@Before("execution(* com.itheima.service.impl.*.*(..))")
public void beginTransaction() {
try {
dbAssit.getCurrentConnection().setAutoCommit(false);
} catch (SQLException e) {
e.printStackTrace();
}
}
@AfterReturning
作用：
把当前方法看成是后置通知。
属性：
value：用于指定切入点表达式，还可以指定切入点表达式的引用
//提交事务
@AfterReturning("execution(* com.itheima.service.impl.*.*(..))")
public void commit() {
try {
dbAssit.getCurrentConnection().commit();
} catch (SQLException e) {
e.printStackTrace();
}
}
@AfterThrowing
作用：
把当前方法看成是异常通知。
属性：
value：用于指定切入点表达式，还可以指定切入点表达式的引用
//回滚事务
@AfterThrowing("execution(* com.itheima.service.impl.*.*(..))")
public void rollback() {
try {
dbAssit.getCurrentConnection().rollback();
} catch (SQLException e) {
e.printStackTrace();
}
}
@After
作用：
把当前方法看成是最终通知。
属性：
value：用于指定切入点表达式，还可以指定切入点表达式的引用
//释放资源
@After("execution(* com.itheima.service.impl.*.*(..))")
public void release() {
try {
dbAssit.releaseConnection();
} catch (Exception e) {
e.printStackTrace();
}
}
~~~

在Spring配置文件中开启spring对注解AOP的支持

< aop:aspectj-autoproxy/>

**环绕通知注解配置**

~~~java
@Around
作用：
把当前方法看成是环绕通知。
属性：
value：用于指定切入点表达式，还可以指定切入点表达式的引用。
/**
* 环绕通知
* @param pjp
* @return
*/
@Around("execution(* com.itheima.service.impl.*.*(..))")
public Object transactionAround(ProceedingJoinPoint pjp) {
//定义返回值
Object rtValue = null;
try {
//获取方法执行所需的参数
Object[] args = pjp.getArgs();
//前置通知：开启事务
beginTransaction();
//执行方法
rtValue = pjp.proceed(args);
//后置通知：提交事务
commit();
}catch(Throwable e) {
//异常通知：回滚事务
rollback();
e.printStackTrace();
}finally {
//最终通知：释放资源
release();
}
return rtValue;
}
~~~

切入点表达式注解

~~~java
@Pointcut
作用：
指定切入点表达式
属性：
value：指定表达式的内容
@Pointcut("execution(* com.itheima.service.impl.*.*(..))")
private void pt1() {}
引用方式：
/**
* 环绕通知
* @param pjp
* @return
*/
@Around("pt1()")//注意：千万别忘了写括号
public Object transactionAround(ProceedingJoinPoint pjp) {
//定义返回值
Object rtValue = null;
try {
//获取方法执行所需的参数
Object[] args = pjp.getArgs();
//前置通知：开启事务
beginTransaction();
//执行方法
rtValue = pjp.proceed(args);
//后置通知：提交事务
commit();
}catch(Throwable e) {
//异常通知：回滚事务
rollback();
e.printStackTrace();
}finally {
//最终通知：释放资源
release();
}
return rtValue;
}
~~~

不使用XML的配置方式

~~~java
@Configuration
@ComponentScan(basePackages="com.itheima")
@EnableAspectJAutoProxy
public class SpringConfiguration {
}
~~~

# 六. Spring中的JdbcTemplate

它是spring框架中提供的一个对象，是对原始Jdbc API对象的简单封装。spring框架为我们提供了很多
的操作模板类。
操作关系型数据的：
	JdbcTemplate
	HibernateTemplate
操作nosql数据库的：
	RedisTemplate
操作消息队列的：
	JmsTemplate

**使用前需配置数据源**（以DBCP为例，不同数据源大同小异）

~~~xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
<property name="url" value="jdbc:mysql:// /spring_day02"></property>
<property name="username" value="root"></property>
<property name="password" value="1234"></property>
</bean>
~~~

## 6.1 JdbcTemplate的增删改查操作

配置文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
<!-- 配置一个数据库的操作模板：JdbcTemplate -->
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
<property name="dataSource" ref="dataSource"></property>
</bean>
<!-- 配置数据源-->
<bean id="dataSource"
class="org.springframework.jdbc.datasource.DriverManagerDataSource">
<property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
<property name="url" value="jdbc:mysql:///spring_day02"></property>
<property name="username" value="root"></property>
<property name="password" value="1234"></property>
</bean>
</beans>
~~~

**基本使用**

~~~java
public class JdbcTemplateDemo{
public static void main(String[] args) {
//1.获取Spring容器
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
//2.根据id获取bean对象
JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//3.执行操作
jt.execute("insert into account(name,money)values('eee',500)");
}
}
~~~

**保存操作**

~~~java
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
//1.获取Spring容器
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
//2.根据id获取bean对象
JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//3.执行操作
//保存
jt.update("insert into account(name,money)values(?,?)","fff",5000);
}
}
~~~

**更新操作**

~~~java
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
//1.获取Spring容器
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
//2.根据id获取bean对象
JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//3.执行操作
//修改
jt.update("update account set money = money-? where id = ?",300,6);
}
}
~~~

**删除操作**

~~~java
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
//1.获取Spring容器
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
//2.根据id获取bean对象
JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//3.执行操作
//删除
jt.update("delete from account where id = ?",6);
}
}
~~~

**查询所有**

~~~java
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
    //1.获取Spring容器
    ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
    //2.根据id获取bean对象
    JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
    //3.执行操作
    //查询所有
    List<Account> accounts = jt.query("select * from account where money > ? ",
    new AccountRowMapper(), 500);
    for(Account o : accounts){
    System.out.println(o);
	}
}
}
public class AccountRowMapper implements RowMapper<Account>{
@Override
public Account mapRow(ResultSet rs, int rowNum) throws SQLException {
    Account account = new Account();
    account.setId(rs.getInt("id"));
    account.setName(rs.getString("name"));
    account.setMoney(rs.getFloat("money"));
    return account;
}
~~~

**查询一个**

~~~java
使用RowMapper的方式：常用的方式
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
    //1.获取Spring容器
    ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
    //2.根据id获取bean对象
    JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
    //3.执行操作
    //查询一个
    List<Account> as = jt.query("select * from account where id = ? ",
    new AccountRowMapper(), 55);
    System.out.println(as.isEmpty()?"没有结果":as.get(0));
}
}
使用ResultSetExtractor的方式:不常用的方式
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
    //1.获取Spring容器
    ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
    //2.根据id获取bean对象
    JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
    //3.执行操作
    //查询一个
    Account account = jt.query("select * from account where id = ?",
    new AccountResultSetExtractor(),3);
    System.out.println(account);
}
}

~~~

**查询返回一行一列**

~~~java
public class JdbcTemplateDemo3 {
public static void main(String[] args) {
//1.获取Spring容器
ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
//2.根据id获取bean对象
JdbcTemplate jt = (JdbcTemplate) ac.getBean("jdbcTemplate");
//3.执行操作
//查询返回一行一列：使用聚合函数，在不使用group by字句时，都是返回一行一列。最长用的
就是分页中获取总记录条数
Integer total = jt.queryForObject("select count(*) from account where money > ?
",Integer.class,500);
System.out.println(total);
}
}
~~~

# 七. Spring中的事务控制

第一：JavaEE体系进行分层开发，事务处理位于业务层，Spring提供了分层设计业务层的事务处理解决方
案。
第二：spring 框架为我们提供了一组事务控制的接口。具体在后面介绍。这组接口是在
spring-tx-5.0.2.RELEASE.jar中。
第三：spring的事务控制都是基于AOP的，它既可以使用编程的方式实现，也可以使用配置的方式实现。我
们学习的重点是使用配置的方式实现。

## 7.1事务

什么是事务：逻辑上的一组操作，组成这组操作的各个单元，要么全部成功，要么全部失败

事物的特性：

> 原子性：事务不可分割
>
> 一致性：事务执行前后数据完整性必须保持一致
>
> 隔离性：一个事务执行不应该受到其他的事物的干扰
>
> 持久性：一旦事务执行，数据就持久化到数据库

如果不考虑隔离性引发的安全性问题

读问题：

脏读：一个事务读到另一个事务未提交的数据

不可重复读：一个事务读到另一个事务已经提交的update的数据，导致一个事务中多次查询结果不一致。

虚读、幻读：一个事务读到另一个事务已经提交的insert数据，导致一个事务中多次查询结果不一致。

写问题：

 丢失更新

解决读问题：

设置事务的隔离级别：

Read uncommitted：未提交读，任何读问题都解决不了

Read committed：已读提交，解决脏读，但是不可重复读和虚读有可能发生（Oracle）

Repeatable read：重复读，解决脏读和不可重复读，但是虚读有

可能发生（Mysql）

Serializable：解决所有读问题

## 7.2 Spring中事物的API

**PlatformTransactionManager**

此接口是spring的事务管理器，它里面提供了我们常用的操作事务的方法：

> TransactionStatus   getTransaction(TransactionDefinitiond  definition)//获取事务的状态信息
>
> void  commit(TransactionStatus   status)//提交事务
>
> void   rollback(TransactionStatus   status)//回滚事务

真正管理事务的对象

> org.springframework.jdbc.datasource.DataSourceTransactionManager 使用Spring
> JDBC或iBatis 进行持久化数据时使用
> org.springframework.orm.hibernate5.HibernateTransactionManager 使用
> Hibernate版本进行持久化数据时使用

**TransactionDefinition**

它是事务的定义信息对象

> String  getName()//获取事务对象名称
>
> int getIsolationLevel()//获取事务隔离级
>
> int getPropagationBehavior()//获取事务传播行为
>
> int getTimeout()//获取事务超过时间
>
> boolean isReadOnly()//获取事务是否只读

**Spring中传播行为用途：**

`保证多个操作在同一个事务中`

PROPAGATION_REQUIRED :默认值，如果A中有事务，使用A中的事务，如果A没有，创建事务将操作包含起来

PRQPAGATION_SUPPORTS：支持事务，如果A中有事务，使用A中的事务，如果A没有事务，不使用事务

PROPAGATION_MANDATORY：如果A中有事务，使用A中的事务，如果A没有事务，抛出异常

`保证多个操作不在同一个事务中：`

 PROPAGATION_REQUIRES_NEW：如果A中有事务，将A中事务挂起（暂停），创建新事务，只包含自身操作。如果A中没有事务，创建新事务，包含自身操作

PROPAGATION_NOT_SUPPORTED：如果A中有事务，将A的事务挂起，不使用事务管理

 PROPAGATION_NEVER：如果A中有事务，抛出异常

`嵌套式事务`

PROPAGATION_NESTED：嵌套事务，若果A中有事务，按照A的执行，执行完成后，设置A的保存点，执行B的操作，若果没有异常，执行通过，若果有异常，可以选择回滚到最初始位置，也可以回滚到保存点。

**事务的隔离级别**

ISOLATION_DEFAULT	默认级别，属于下列某一种

ISOLATION_READ_UNCOMMITED	可以读取未提交数据

ISOLATION_READ_COMMITED	只能读取已提交数据，解决脏读（Oracle默认级别）

ISOLATION_REPEATABLE_READ	是否读取替他十五提交后的数据，解决不可重复读（MYSQL默认级别）

ISOLATION_SERIALIZABLE	是否读取其他事务提交添加后的数据，解决幻影读

## 7.3 基于XML的声明式事务控制（配置方式）

步骤

**1.拷贝jar包**

**2.创建spring配置文件**

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:aop="http://www.springframework.org/schema/aop"
xmlns:tx="http://www.springframework.org/schema/tx"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://www.springframework.org/schema/tx
http://www.springframework.org/schema/tx/spring-tx.xsd
http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop.xsd">
</beans>
~~~

**3.创建数据库和表、编写业务层持久层的接口和实现类，编写业务层和持久层的配置文件**

**4.（开始事务配置）配置事务管理器**

~~~xml
<!-- 配置一个事务管理器-->
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<!-- 注入DataSource -->
<property name="dataSource" ref="dataSource"></property>
</bean>
~~~

**5.配置事物的通知引用事务管理器**

~~~xml
<!-- 事务的配置-->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
</tx:advice>
~~~

**6.配置事务的属性**

~~~xml
<!--在tx:advice标签内部配置事务的属性-->
<tx:attributes>
<!-- 指定方法名称：是业务核心方法
read-only：是否是只读事务。默认false，不只读。
isolation：指定事务的隔离级别。默认值是使用数据库的默认隔离级别。
propagation：指定事务的传播行为。
timeout：指定超时时间。默认值为：-1。永不超时。
rollback-for：用于指定一个异常，当执行产生该异常时，事务回滚。产生其他异常，事务不回滚。
没有默认值，任何异常都回滚。
no-rollback-for：用于指定一个异常，当产生该异常时，事务不回滚，产生其他异常时，事务回
滚。没有默认值，任何异常都回滚。
-->
<tx:method name="*" read-only="false" propagation="REQUIRED"/>
<tx:method name="find*" read-only="true" propagation="SUPPORTS"/>
</tx:attributes>

~~~

**7.配置事务的AOP切入点表达式**

~~~xml
<!-- 配置aop -->
<aop:config>
<!-- 配置切入点表达式-->
<aop:pointcut expression="execution(* com.itheima.service.impl.*.*(..))"
id="pt1"/>
</aop:config>
~~~

**8.配置切入点表达式和事务通知的关系**

~~~xml
<!-- 在aop:config标签内部：建立事务的通知和切入点表达式的关系-->
<aop:advisor advice-ref="txAdvice" pointcut-ref="pt1"/>
~~~

## 7.4 基于注解的配置方式

步骤

123步与上述相同

**4.配置事务管理器并注入数据源**

~~~xml
<!-- 配置事务管理器-->
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource"></property>
</bean>
~~~

**5.在业务层使用@Transactional**

~~~java
@Service("accountService")
@Transactional(readOnly=true,propagation=Propagation.SUPPORTS)
public class AccountServiceImpl implements IAccountService {
    @Autowired
    private IAccountDao accountDao;
    @Override
    public Account findAccountById(Integer id) {
    	return accountDao.findAccountById(id);
    }
    @Override
    @Transactional(readOnly=false,propagation=Propagation.REQUIRED)
    public void transfer(String sourceName, String targeName, Float money) {
        //1.根据名称查询两个账户
        Account source = accountDao.findAccountByName(sourceName);
        Account target = accountDao.findAccountByName(targeName);
        //2.修改两个账户的金额
        source.setMoney(source.getMoney()-money);//转出账户减钱
        target.setMoney(target.getMoney()+money);//转入账户加钱
        //3.更新两个账户
        accountDao.updateAccount(source);
        accountDao.updateAccount(target);
    }
}
该注解的属性和xml中的属性含义一致。该注解可以出现在接口上，类上和方法上。
出现接口上，表示该接口的所有实现类都有事务支持。
出现在类上，表示类中所有方法有事务支持
出现在方法上，表示方法有事务支持。
以上三个位置的优先级：方法>类>接口
~~~

**6.在配置文件中开启事务注解的支持**

~~~xml
<!-- 开启spring对注解事务的支持-->
<tx:annotation-driven transaction-manager="transactionManager"/>
~~~

# 八. Spring5中的新内容

**Junit5的支持**

完全支持JUnit 5 Jupiter，所以可以使用JUnit 5 来编写测试以及扩展。此外还提供了一个编程以及
扩展模型，Jupiter 子项目提供了一个测试引擎来在Spring 上运行基于Jupiter 的测试。
另外，Spring Framework 5 还提供了在Spring TestContext Framework 中进行并行测试的扩展。
针对响应式编程模型，spring-test 现在还引入了支持Spring WebFlux 的WebTestClient 集成测
试的支持，类似于MockMvc，并不需要一个运行着的服务端。使用一个模拟的请求或者响应，WebTestClient
就可以直接绑定到WebFlux 服务端设施。
你可以在这里找到这个激动人心的TestContext 框架所带来的增强功能的完整列表。
当然，Spring Framework 5.0 仍然支持我们的老朋友JUnit! 在我写这篇文章的时候，JUnit 5 还
只是发展到了GA 版本。对于JUnit4，Spring Framework 在未来还是要支持一段时间的。















