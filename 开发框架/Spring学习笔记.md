# Spring学习笔记

Spring是分层的Java SE/EE应用full-stack轻量级开源框架，以IoC（Inverse Of Control：反转控制）和AOP（Aspect Oriented Programming：面向切面编程）为内核，提供了表现层SpringMVC和持久层Spring JDBC以及业务层事务管理等众多的企业级应用技术，还能整合开源世界众多
著名的第三方框架和类库，逐渐成为使用最多的Java EE企业应用开源框架。

本文中大部分主要是Spring Core中的内容的整理总结。

在Core中都包括IOC Container（IOC容器），Events（事件），Resources（资源），i18n（），Validation（验证），DataBinging（数据绑定），Type Conversion（类型转换），SpEL，AOP

# 一、概述

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

# 二、IOC容器

本章介绍了控制反转（IoC）原理的Spring框架实现。IoC也称为依赖注入（DI）。在此过程中，对象仅通过构造函数参数，工厂方法的参数或在构造或从工厂方法返回后在对象实例上设置的属性来定义其依赖项（即，与它们一起使用的其他对象） 。然后，容器在创建bean时注入那些依赖项。此过程从根本上讲是通过使用类的直接构造或诸如服务定位器模式的机制来控制其依赖项的实例化或位置的Bean本身的逆过程（因此称为控件的倒置）。

在`org.springframework.beans`和`org.springframework.context`包是Spring框架的IoC容器的基础。简而言之，BeanFactory提供了配置框架和基本功能，而ApplicationContext添加了更多企业特定的功能。ApplicationContext是BeanFactory的完整超集

在Spring中，构成应用程序主干并由Spring IoC容器管理的对象称为bean。Bean是由Spring IoC容器实例化，组装和以其他方式管理的对象。

## 2.1程序的耦合

​	耦合性(Coupling)，也叫耦合度，是对模块间关联程度的度量。耦合的强弱取决于模块间接口的复杂性、调用模块的方式以及通过界面传送数据的多少。模块间的耦合度是指模块之间的依赖关系，包括控制关系、调用关系、数据传递关系。模块间联系越多，其耦合性越强，同时表明其独立性越差( 降低耦合性，可以提高其独立性)。耦合性存在于各个领域，而非软件设计中独有的，但是我们只讨论软件工程中的耦合。
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
> ~~~java
> public static void main(String[] args) throws Exception {
>     //1.注册驱动
>     //DriverManager.registerDriver(new com.mysql.jdbc.Driver());
>     Class.forName("com.mysql.jdbc.Driver");
>     //2.获取连接
>     //3.获取预处理sql语句对象
>     //4.获取结果集
>     //5.遍历结果集
> }
> ~~~
>
> 我们的类依赖了数据库的具体驱动类（MySQL），如果这时候更换了数据库品牌（比如Oracle），需要
> 修改源码来重新数据库驱动。这显然不是我们想要的。

## 2.2 程序的解耦

思路：

当是我们讲解jdbc时，是通过反射来注册驱动的，代码如下：
`Class.forName("com.mysql.jdbc.Driver");`//此处只是一个字符串
此时的好处是，我们的类中不再依赖具体的驱动类，此时就算删除mysql的驱动jar包，依然可以编译（运行就不要想了，没有驱动不可能运行成功的）。
同时，也产生了一个新的问题，mysql驱动的全限定类名字符串是在java类中写死的，一旦要改还是要修改源码。
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

`org.springframework.context.ApplicationContext`接口代表Spring IoC容器，并负责实例化，配置和组装Bean。容器通过读取配置元数据来获取有关要实例化，配置和组装哪些对象的指令。配置元数据以XML，Java批注或Java代码表示。它能够表达组成应用程序的对象以及这些对象之间的丰富相互依赖关系。

### 2.3.1 配置元数据

Spring IoC容器使用一种形式的配置元数据如下图所示。此配置元数据表示您作为开发人员如何告诉Spring容器实例化，配置和组装应用程序中的对象。

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/spring/container-magic.png)

Spring中主要采用三种方式配置元数据：

> 基于xml的方式：
>
> - JavaSE应用 
>
>   ~~~java
>   ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
>   或者
>   ApplicationContext applicationContext = new FileSystemXmlApplicationContext("C:/beans.xml");
>   ~~~
>
> - JavaWeb应用
>
>   ~~~xml
>   <!DOCTYPE web-app PUBLIC
>   "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
>   "http://java.sun.com/dtd/web-app_2_3.dtd" >
>   <web-app>
>    <display-name>Archetype Created Web Application</display-name>
>    <!--配置Spring ioc容器的配置⽂件-->
>    <context-param>
>    <param-name>contextConfigLocation</param-name>
>    <param-value>classpath:applicationContext.xml</param-value>
>    </context-param>
>    <!--使⽤监听器启动Spring的IOC容器-->
>    <listener>
>    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
>    </listener>
>   </web-app>
>   ~~~
>
> 基于注解的方式：从Spring2.5后可以基于注解配置。
>
> 在此时有了XML+注解方式实现IOC，此时启动方式与xml方式相同：
>
> - JavaSE应用 
>
>   ~~~java
>   ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
>   或者
>   ApplicationContext applicationContext = new FileSystemXmlApplicationContext("C:/beans.xml");
>   ~~~
>
> - JavaWeb应用
>
>   ~~~xml
>   <!DOCTYPE web-app PUBLIC
>   "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
>   "http://java.sun.com/dtd/web-app_2_3.dtd" >
>   <web-app>
>    <display-name>Archetype Created Web Application</display-name>
>    <!--配置Spring ioc容器的配置⽂件-->
>    <context-param>
>    <param-name>contextConfigLocation</param-name>
>    <param-value>classpath:applicationContext.xml</param-value>
>    </context-param>
>    <!--使⽤监听器启动Spring的IOC容器-->
>    <listener>
>    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
>    </listener>
>   </web-app>
>   ~~~
>
> 基于Java的配置：从Spring3.0开始，Spring JavaConfig项目提供的许多功能成为了Spring核心框架的一部分。因此，您可以使用Java而不是XML文件来定义应用程序类外部的bean。
>
> - JavaSE
>
>   ~~~java
>   ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
>   ~~~
>
> - JavaWeb
>
>   ~~~xml
>   <!DOCTYPE web-app PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
>   "http://java.sun.com/dtd/web-app_2_3.dtd" >
>   <web-app>
>    <display-name>Archetype Created Web Application</display-name>
>    <!--告诉ContextloaderListener知道我们使⽤注解的⽅式启动ioc容器-->
>    <context-param>
>    <param-name>contextClass</param-name>
>    <param-value>org.springframework.web.context.support.AnnotationConfigWebAppli
>   cationContext</param-value>
>    </context-param>
>     <!--配置启动类的全限定类名-->
>    <context-param>
>    <param-name>contextConfigLocation</param-name>
>    <param-value>com.learn.SpringConfig</param-value>
>    </context-param>
>    <!--使⽤监听器启动Spring的IOC容器-->
>    <listener>
>    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
>    </listener>
>   </web-app>
>   ~~~

Spring配置由容器必须管理的至少一个（通常是一个以上）bean定义组成。基于XML的配置元数据将这些bean配置`<bean/>`放在`<beans/>`元素内。Java配置通常为`@Bean`在`@Configuration`类中使用带注释的方法。

这些bean用于组成应用程序的实际对象。比如DAO对象等等。

基于XML配置元数据简单示例：

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

`ApplicationContext`是一个维护bean定义以及相互依赖的注册表的高级工厂的接口。通过使用方法 `T getBean(String name, Class<T> requiredType)`，您可以检索bean的实例。

~~~JAVA
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
// 检索配置的示例
PetStoreService service = context.getBean("petStore", PetStoreService.class);
// 使用配置的示例
List<String> userList = service.getUsernameList();
~~~



## 2.4 Bean概述

spring ioc容器管理一个或多个bean。这些bean是使用提供给容器的配置元数据创建的。（例如XML`<bean>`的形式）。

#### 2.4.1 bean的命名

每个bean具有一个或多个标识符。这些标识符在承载Bean的容器内必须唯一。一个bean通常只有一个标识符。但是，如果需要多个，则可以将多余的别名视为别名。

在基于xml方式配置bean，可以使用id属性指定唯一ID，name属性可以指定别名。如果使用Java配置，则`@Bean`注释可用于提供别名。

bean可以使用驼峰式命名如accountManager。

#### 2.4.2 实例化bean

第一种方式：使用默认无参构造函数

```xml
<!--在默认情况下：
它会根据默认无参构造函数来创建类对象。如果bean中没有默认无参构造函数，将会创建失败。-->
<bean id="accountService" class="com.itheima.service.impl.AccountServiceImpl"/>
```

第二种方式：spring管理静态工厂-使用静态工厂的方法创建对象

```xml
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

```

第三种方式：spring管理实例工厂-使用实例工厂的方法创建对象

```xml
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
```



## 2.5 基于XML的IOC配置

### 2.5.1 IOC入门

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
<bean id="accountService" class="com.learn.service.impl.AccountServiceImpl">
</bean>
<!-- 配置dao -->
<bean id="accountDao" class="com.learn.dao.impl.AccountDaoImpl"></bean>




 <!--Spring ioc 实例化Bean的三种方式-->
    <!--方式一：使用无参构造器（推荐）-->
    <bean id="connectionUtils" class="com.learn.utils.ConnectionUtils"></bean>

    <!--另外两种方式是为了我们自己new的对象加入到SpringIOC容器管理-->
    <!--方式二：静态方法-->
    <!--<bean id="connectionUtils" class="com.learn.factory.CreateBeanFactory" factory-method="getInstanceStatic"/>-->
    <!--方式三：实例化方法-->
    <!--<bean id="createBeanFactory" class="com.learn.factory.CreateBeanFactory"></bean>
    <bean id="connectionUtils" factory-bean="createBeanFactory" factory-method="getInstance"/>-->
~~~

方式一、二和三对应的java方法：

```java
public class CreateBeanFactory {
    //方式一可以将下面两个构造方法去掉，使用默认无参构造器
    
    //方式二
    public static ConnectionUtil getInstanceStatic() {
        return new ConnectionUtil();
    }
    //方式三
    public ConnectionUtil getInstance() {
        return new ConnectionUtil();
    }
}
```

### 2.5.2基于xml的IOC

#### spring中的工厂的类结构图

![img](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/1598493-20200215151523468-1242212011.png)

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

## 2.6 依赖注入

依赖注入：Dependency Injection。它是spring框架核心ioc的具体实现。IOC和DI描述的是同⼀件事情，只不过⻆度不⼀样罢了。
我们的程序在编写时，通过控制反转，把对象的创建交给了spring，但是代码中不可能出现没有依赖的情况。
ioc解耦只是降低他们的依赖关系，但不会消除。例如：我们的业务层仍会调用持久层的方法。那这种业务层和持久层的依赖关系，在使用spring之后，就让spring来维护了。简单的说，就是坐等框架把持久层对象传入业务层，而不用我们自己去获取。

#### 2.6.1 构造函数注入

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

#### 2.6.2 set方法注入

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

## 2.7 基于注解的IOC配置

注解配置和xml 配置要实现的功能都是一样的，都是要降低程序间的耦合。只是配置的形式不一样。

### 2.7.1常用注释

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
自动按照类型注入。当使用注解注入属性时，set方法可以省略。它只能注入其他bean类型。当有多个类型匹配时，使用要注入的对象变量名称作为bean的id，在spring容器查找，找到了也可以注入成功。找不到就报错。

~~~java
//在这里@Autowired按照AccountDao类型进行注入，如果有多个AccountDao类型的bean那么则使用@Qualifier注解表明AccountDao类型的bean的id
@Autowired
//@Qualifier("accountDao1")
private AccountDao accountDao;
~~~

**@Qualifier**

作用：
在自动按照类型注入的基础之上，再按照Bean 的id 注入。它在给字段注入时不能独立使用，必须和@Autowire一起使用；但是给方法参数注入时，可以独立使用。
属性：
value：指定bean的id。

**@Resource**

作用：
直接按照Bean的id注入。它也只能注入其他bean类型。
属性：
name：指定bean的id。

> 在JDK11中删除了，如果想使用需要添加依赖
>
> ~~~xml
> <dependency>
>  <groupId>javax.annotation</groupId>
>  <artifactId>javax.annotation-api</artifactId>
>  <version>1.3.2</version>
> </dependency>
> ~~~

**@Value**

作用：
注入基本数据类型和String类型数据的
属性：
value：用于指定值

**@Scope**

用于改变作用范围的 相当于：< bean id="" class="" scope="">

作用：
指定bean的作用范围。
属性：
value：指定范围的值。
取值：singleton prototype request session globalsession



 **@PostConstruct**

和生命周期相关的 相当于：< bean id="" class="" init-method="" destroy-method="" />

作用：
用于指定初始化方法。

 **@PreDestroy**

和生命周期相关的 相当于：< bean id="" class="" init-method="" destroy-method="" />

作用：
用于指定销毁方法。

**@Configuration**

从Spring3.0，@Configuration用于定义配置类，可替换xml配置文件，被注解的类内部包含有一个或多个被@Bean注解的方法，这些方法将会被AnnotationConfigApplicationContext或AnnotationConfigWebApplicationContext类进行扫描，并用于构建bean定义，初始化Spring容器。

**注意**：@Configuration注解的配置类有如下要求：

1. @Configuration不可以是final类型；
2. @Configuration不可以是匿名类；
3. 嵌套的configuration必须是静态类。

**@Bean**

@Bean是一个方法级别上的注解，主要用在@Configuration注解的类里，也可以用在@Component注解的类里。添加的bean的id为方法名

eg:

下面是@Configuration里的一个例子,配置Durid数据源

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource createDataSource() {
        DruidDataSource druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName("xxx");
        druidDataSource.setUrl("xxx");
        druidDataSource.setUserName("xxx");
        druidDataSource.setPassWord("xxx");
        return druidDataSource;
    }
}
```

这个配置就等同于之前在xml里的配置

```xml
<beans>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${db.driverClassName}"></property>
        <property name="url" value="${db.url}"></property>
        <property name="username" value="${db.username}"></property>
        <property name="password" value="${db.password}"></property>
    </bean>
</beans>
```

## 2.8 IOC的lazy-Init懒加载

ApplicationContext容器默认是在启动服务时将所有的singleton bean提前实例化，但如果设置为懒加载，那么这个bean只有在用到的时候才会实例化。但是一个bean设置成多例，那么即使设置了lazy-init="false",容器启动也不会实例化bean，⽽是调⽤ getBean ⽅法实例化的。

设置方式

~~~java
<bean id="testBean" calss="com.learn.LazyBean" lazy-init="true" />
~~~



## 2.9 FactoryBean和BeanFactory

BeanFactory接⼝是容器的顶级接⼝，定义了容器的⼀些基础⾏为，负责⽣产和管理Bean的⼀个⼯⼚，具体使⽤它下⾯的⼦接⼝类型，⽐如ApplicationContext。

而FactoryBean是一种工厂类，Spring中的Bean有两种：一种是普通Bean，一种是工厂Bean就是FactoryBean，他可以生成某一个类的实例，也就是说我们可以借助它自定义Bean的创建过程。

Bean创建的三种⽅式中的静态⽅法和实例化⽅法和FactoryBean作⽤类似，FactoryBean使⽤较多，尤其在Spring框架⼀些组件中会使⽤，还有其他框架和Spring框架整合时使⽤。

我们可以通过一个例子了解FactoryBean：

首先创建一个实体类：

```java
public class Person {
    private String name;
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

然后创建一个FactoryBean

```java
@Component("factoryBean")
public class PersonFactoryBean implements FactoryBean<Person> {

    @Value("123,12")
    private String persionInfo;

    public void setPersionInfo(String persionInfo) {
        this.persionInfo = persionInfo;
    }

    @Override
    public Person getObject() throws Exception {
        String[] split = this.persionInfo.split(",");
        //模拟创建复杂对象Person
        Person person = new Person();
        person.setName(split[0]);
        person.setAge(Integer.valueOf(split[1]));

        return person;
    }

    @Override
    public Class<?> getObjectType() {
        return Person.class;
    }
}
```

然后编写测试方法，查看结果：

```java
@Test
public void testIOC() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
    Object factoryBean = applicationContext.getBean("factoryBean");
    System.out.println("bean:"+factoryBean);
}
```

结果：bean:Person{name='123', age=12}

可见，FactoryBean可以直接获取我们需要的bean，而不是获取Factory本身。如果我们获取FactoryBean，需要在id之前添加“&。

```java
@Test
public void testIOC() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
    Object factoryBean = applicationContext.getBean("&factoryBean");
    System.out.println("bean:"+factoryBean);
}
```

结果：bean:com.learn.pojo.PersonFactoryBean@687e99d8

# 三 、AOP

## 3.1概述

AOP：全称是Aspect Oriented Programming即：面向切面编程。

简单的说它就是把我们程序重复的代码抽取出来，在需要执行的时候，使用动态代理的技术，在不修改源码的基础上，对我们的已有方法进行增强。

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
当我们执行时，由于执行有异常，转账失败。但是因为我们是每次执行持久层方法都是独立事务，导致无法实现事务控制（不符合事务的一致性）
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

## 3.2 AOP中的概念

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

## 3.3 基于XML的AOP

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

## 3.4 基于注解的AOP

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

# 四、手写IOC和AOP

在理解了IOC和AOP思想之后，根据银行转账案例我们逐步分析，并一步步手写IOC和AOP。

> 由于要手写IOC和AOP所以就只使用原生的Servlet，以及jdbc。就不引入Spring和SpringMVC以及mybatis了。

## 4.1 银行转账案例

案例代码地址[Github](https://github.com/Loserfromlazy/MY_IOC_AOP/blob/main/TestBank.rar)     [Gitee](https://gitee.com/yhr520/MY_IOC_AOP/blob/main/TestBank.rar)

**案例问题分析**

Servlet部分代码

![image-20211124113927774](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211124113927774.png)

Service部分代码

![image-20211124113947769](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211124113947769.png)

问题分析：

1. 代码耦合，在每一层都要实现下一层的实现类，比如，在service层使用dao层时，直接在impl中通过new获得了dao层对象，然而new关键字将ServiceImpl和DaoImpl耦合在一起，如果技术框架发生变动，譬如dao改成mybatis就需要修改源代码重新编译。

2. service层代码没有事务控制。

   缺少事务控制的原因是两次更新提交了两次事务不属于同一个事务，换句话说事务控制在dao层而不是service层，如果两次更新之间有异常就会造成，一次成功一次未成功。

   ~~~java
   accountDao.updateByCardNo(fromNo);
   //int i = 1/0;
   accountDao.updateByCardNo(toNo);
   ~~~

**问题解决思路**

在上述代码中我们使用new来实例化对象，除了new以外还可以通过反射来实例化对象。如`Class.forName("com.learn.dao.XXXImpl")`,当然可以使用配置文件来存储全限定类名。

关于代码耦合方面，我们可以使用工厂模式去解耦合。

综上，可以使用工厂通过反射技术来生产对象，也就是说，**工厂类解析xml存的全限定类名，然后通过反射技术实例化对象。工厂类同时给外部提供获取对象的接口方法。**

**事务问题解决思路**

让两次update使用同一个connection连接，把事务控制在service层。

## 4.2 案例修改-解耦合

增加beans.xml

~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<!--根标签，里面配置了一个又一个的bean子标签，每一个子标签都代表一个类的配置-->
<beans>
<!--id标识对象，class是类的全限定名称-->
    <bean id="accountDao" class="com.learn.dao.Impl.AccountDaoImpl"></bean>
    <bean id="accountService" class="com.learn.service.Impl.AccountServiceImpl">
        <!-- names属性：用于定位set方法 setAccountDao即set方法名 ref属性：需要传入的值-->
        <property name="AccountDao" ref="accountDao"></property>
    </bean>
</beans>
~~~

增加BeanFactory

~~~java
public class BeanFactory {
    /*
    * BeanFactory首先需要读取解析xml，通过反射技术实例化对象存储map中待用
    * 然后对外提供获取实例对象的接口
    * */
    private static Map<String,Object> map = new HashMap<>();

    static {
        //加载xml
        InputStream resourceAsStream = BeanFactory.class.getClassLoader().getResourceAsStream("beans.xml");
        //解析xml
        SAXReader saxReader = new SAXReader();
        try {
            Document read = saxReader.read(resourceAsStream);
            Element rootElement = read.getRootElement();
            List<Element> beanList = rootElement.selectNodes("//bean");
            for (Element element : beanList) {
                //获取每个元素的id和class
                String id = element.attributeValue("id");
                String clazz = element.attributeValue("class");
                //反射创建对象。
                Class<?> aClass = Class.forName(clazz);
                Object o = aClass.newInstance();
                map.put(id, o);
            }
            //实例化完成后维护对象的依赖关系，检查哪些对象需要传值，根据配置传值
            //有property元素就有传值需求
            List<Element> propertyList = rootElement.selectNodes("//property");
            for (Element element : propertyList) {
                String name = element.attributeValue("name");
                String ref = element.attributeValue("ref");

                //找到需要处理依赖关系的bean
                Element parent = element.getParent();
                //调用父元素反射
                String parentId = parent.attributeValue("id");
                Object parentObject = map.get(parentId);
                //遍历父元素所有方法，找到"set"+name值的方法
                Method[] methods = parentObject.getClass().getMethods();
                for (Method method : methods) {
                    if (method.getName().equalsIgnoreCase("set" + name)) {
                        method.invoke(parentObject, map.get(ref));
                    }
                }
                //把处理之后的parentObject重新放入map
                map.put(parentId,parentObject);
            }


        } catch (DocumentException | ClassNotFoundException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
    //对外提供获取实例对象的接口
    public static Object getBean(String id){
        return map.get(id);
    }
}
~~~

将Servlet和Service中的new实例对象进行修改。

~~~java
在Servlet中进行修改
//    private AccountService accountService = new AccountServiceImpl();
    private AccountService accountService = (AccountService) BeanFactory.getBean("accountService");
在Service中进行修改
//    private AccountDao accountDao = new AccountDaoImpl();
    private AccountDao accountDao;

    public void setAccountDao(AccountDao accountDao) {
        this.accountDao = accountDao;
    }
~~~

## 4.3 案例修改-事务控制

在4.2代码的基础上进行修改

增加工具类获取当前线程

```java
public class ConnectionUtil {

    private ConnectionUtil(){}
    //饿汉式单例
    private static ConnectionUtil connectionUtil = new ConnectionUtil();

    public static ConnectionUtil getInstance(){
        return connectionUtil;
    }
    //使用ThreadLocal存储Connection
    ThreadLocal<Connection> threadLocal = new ThreadLocal<>();

    public Connection getCurrentThreadConn() throws SQLException {
        Connection connection = threadLocal.get();
        if (connection ==null){
            connection = DruidUtils.getInstance().getConnection();
            threadLocal.set(connection);
        }
        return connection;
    }
}
```

增加工具类事务管理

```java
public class TransactionManager {

    private TransactionManager(){}
    //饿汉式单例模式
    private static TransactionManager transactionManager = new TransactionManager();

    public static TransactionManager getInstance(){
        return transactionManager;
    }

    public void begin() throws SQLException {
        ConnectionUtil.getInstance().getCurrentThreadConn().setAutoCommit(false);
    }
    public void commit() throws SQLException {
        ConnectionUtil.getInstance().getCurrentThreadConn().commit();
    }
    public void rollback() throws SQLException {
        ConnectionUtil.getInstance().getCurrentThreadConn().rollback();
    }
}
```

对dao层代码修改

注释掉原来的connection，转而从工具类中获取当前线程的Connection

```java
    @Override
    public BankInfo findByCardNo(String cardNo) throws Exception {
//        Connection connection = DruidUtils.getInstance().getConnection();
        Connection currentThreadConn = ConnectionUtil.getInstance().getCurrentThreadConn();
        String sql = "select * from bank_info where card_no=?";
        PreparedStatement preparedStatement = currentThreadConn.prepareStatement(sql);
        preparedStatement.setString(1,cardNo);
        ResultSet resultSet = preparedStatement.executeQuery();
        BankInfo account = new BankInfo();
        while(resultSet.next()) {
            account.setCardNo(resultSet.getString("card_no"));
            account.setName(resultSet.getString("name"));
            account.setMoney(resultSet.getInt("money"));
        }
        resultSet.close();
        preparedStatement.close();
        //connection.close();
        return account;
    }
```

对service层代码修改

```java
@Override
public void transfer(String fromCardNo, String toCardNo, int money) throws Exception {
    try {
        TransactionManager.getInstance().begin();
        BankInfo fromNo = accountDao.findByCardNo(fromCardNo);
        BankInfo toNo = accountDao.findByCardNo(toCardNo);
        System.out.println(fromNo.toString());
        System.out.println(toNo.toString());
        fromNo.setMoney(fromNo.getMoney() - money);
        toNo.setMoney(toNo.getMoney() + money);
        accountDao.updateByCardNo(fromNo);
        int i = 1/0;
        accountDao.updateByCardNo(toNo);
        TransactionManager.getInstance().commit();
    }catch (Exception e){
        e.printStackTrace();
        TransactionManager.getInstance().rollback();
        //抛出异常便于上层controller层捕获异常
        throw e;
    }
}
```

以上完成了事务控制的功能，但是每个业务代码都加上try-catch控制很麻烦很冗余，所以使用动态代理解决。

下面通过[动态代理技术](https://www.cnblogs.com/yhr520/p/15601620.html)（链接为我的另一篇博客Java代理）以在不改变原有业务代码的逻辑上实现事务控制。

首先增加代理工厂类

```java
public class ProxyFactory {

    private ProxyFactory() {
    }

    //饿汉式单例
    private static ProxyFactory proxyFactory = new ProxyFactory();

    public ProxyFactory getInstance() {
        return proxyFactory;
    }

    /**
     * 获取JDK代理对象
     *
     * @author Yuhaoran
     * @date 2021/11/25 11:15
     */
    public Object getJDKProxy(Object target) {
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Object result;
                try {
                    TransactionManager.getInstance().begin();
                    result = method.invoke(args);
                    TransactionManager.getInstance().commit();
                } catch (Exception e) {
                    e.printStackTrace();
                    TransactionManager.getInstance().rollback();
                    //抛出异常便于上层controller层捕获异常
                    throw e;
                }
                return result;
            }
        });
    }

    /**
     * 获取Cglib代理对象
     *
     * @author Yuhaoran
     * @date 2021/11/25 11:16
     */
    public Object getCglibProxy(Object target) {
        return Enhancer.create(target.getClass(), (net.sf.cglib.proxy.InvocationHandler) (o, method, objects) -> {
            Object result;
            try {
                TransactionManager.getInstance().begin();
                result = method.invoke(objects);
                TransactionManager.getInstance().commit();
            } catch (Exception e) {
                e.printStackTrace();
                TransactionManager.getInstance().rollback();
                //抛出异常便于上层controller层捕获异常
                throw e;
            }
            return result;
        });
    }
}
```

对Servlet中业务对象改造，使用代理对象，这样就可以实现增强事务控制，避免事务代码和业务代码混合。

```java
//private AccountService accountService = (AccountService) BeanFactory.getBean("accountService");
//从工厂中获取代理对象，通过代理对象对事物进行控制
private AccountService accountService = (AccountService) ProxyFactory.getInstance().getCglibProxy(BeanFactory.getBean("accountService"));
```

这样就完成了事务的增强控制，但是由于上面完成了IOC工厂，所以我们可以将这些工具类放入IOC工厂中，替我们创建对象。

最终案例地址，[My_IOC_AOP](https://github.com/Loserfromlazy/MY_IOC_AOP/blob/main/TestBank_IOC_AOP.rar)         [Gitee地址](https://gitee.com/yhr520/MY_IOC_AOP/blob/main/TestBank_IOC_AOP.rar)

## 4.4 使用Spring纯XML模式改造案例

使用Spring替代我们本身的案例，从中深入体会Spring的用法

首先引入Spring和SpringWeb（注意不是SpringMVC）的依赖：

```xml
<!--        Spring-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.12.RELEASE</version>
        </dependency>
<!--        Spring Web-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>5.2.12.RELEASE</version>
        </dependency>
    </dependencies>
```

去除原有beans.xml,使用applicationContext.xml代替，其实两者无区别只是我们惯用applicationContext这个名字罢了。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="accountDao" class="com.learn.dao.Impl.AccountDaoImpl">
        <property name="ConnectionUtil" ref="connectionUtil"> </property>
    </bean>
    <bean id="accountService" class="com.learn.service.Impl.AccountServiceImpl">
        <property name="AccountDao" ref="accountDao"> </property>
    </bean>
    <!--    连接工具类-->
    <bean id="connectionUtil" class="com.learn.util.ConnectionUtil"></bean>
    <!--    事务管理器-->
    <bean id="transactionManager" class="com.learn.util.TransactionManager">
        <property name="ConnectionUtil" ref="connectionUtil"> </property>
    </bean>
    <!--    代理对象工厂-->
    <bean id="proxyFactory" class="com.learn.factory.ProxyFactory">
        <property name="TransactionManager" ref="transactionManager"> </property>
    </bean>
</beans>
```

去除我们自定义的BeanFactory，并在Servlet中配置上Spring的Bean工厂，由于我们是Web应用，所以使用WebApplicationContext，代码如下

```java
//从工厂中获取代理对象，通过代理对象对事物进行控制
private AccountService accountService = null;

@Override
public void init() throws ServletException {
    WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    ProxyFactory proxyFactory = (ProxyFactory)webApplicationContext.getBean("proxyFactory");
    accountService = (AccountService) proxyFactory.getJDKProxy(webApplicationContext.getBean("accountService"));
}
```

最后在web.xml中配置启动器类即可

```xml
<display-name>Archetype Created Web Application</display-name>
<!--配置Spring ioc容器的配置⽂件-->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
<!--使⽤监听器启动Spring的IOC容器-->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

以上我们就算是使用Spring纯XML方式替代掉了我们自己手写的IOC代码

4.4代码地址     [GitHub](https://github.com/Loserfromlazy/MY_IOC_AOP/blob/main/TestBankWithXML.rar)          [Gitee](https://gitee.com/yhr520/MY_IOC_AOP/blob/main/TestBankWithXML.rar)

## 4.5 使用Spring的XML+注解方式改造案例

当然实际上最常用的还是XML+注解方式，所以我们以这种方式将4.4代码继续改造。

使用这种方式的Spring工程，一般我们将三方jar包配置在xml中（如Durid数据连接池），将自己开发的bean（如Service层代码）使用注解。

首先我们可以将xml中只配置三方jar包，删除其余的内容，然后配置包扫描，因为我们要使用注解

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">
    <context:component-scan base-package="com.learn"> </context:component-scan>
    <context:property-placeholder location="classpath:jdbc.properties"/>
    <!--第三方jar中的bean定义在xml中-->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName" value="${jdbc.driver}"/>
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

然后我们将之前的删掉的自己的类用注解表示，并用@Autowire维护依赖关系，以AccountDaoImpl为例：

![image-20211129141332798](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211129141332798.png)

所有的类以此维护即可，但注意ServiceImpl需要标注ID，因为我们需要代理类帮我们处理事务。

![image-20211129141516739](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211129141516739.png)

BankServlet#init()：

```java
@Override
public void init() throws ServletException {
    WebApplicationContext webApplicationContext = WebApplicationContextUtils.getWebApplicationContext(this.getServletContext());
    ProxyFactory proxyFactory = (ProxyFactory)webApplicationContext.getBean("proxyFactory");
    accountService = (AccountService) proxyFactory.getJDKProxy(webApplicationContext.getBean("accountService"));
}
```

项目代码4.5 ： [GitHub](https://github.com/Loserfromlazy/MY_IOC_AOP/blob/main/TestBankWitnXMLANNO.rar)          [Gitee](https://gitee.com/yhr520/MY_IOC_AOP/blob/main/TestBankWitnXMLANNO.rar)

## 4.6 使用Spring的纯注解模式改造案例

使用纯注解方式得启动类，不在是xml，而是一个Config配置类。主要修改是将启动xml修改到启动类中：

![image-20211130101331228](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211130101331228.png)

同时在web.xml中配置为启动java类

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <display-name>Archetype Created Web Application</display-name>
    <!--告诉ContextloaderListener知道我们使用注解的方式启动ioc容器-->
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
    </context-param>
    <!--配置Spring ioc容器的配置⽂件-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>com.learn.SpringConfig</param-value>
    </context-param>
    <!--使⽤监听器启动Spring的IOC容器-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
</web-app>
```

4.6代码：   [Github](https://github.com/Loserfromlazy/MY_IOC_AOP/blob/main/TestBankWithAnno.rar)       [Gitee](https://gitee.com/yhr520/MY_IOC_AOP/blob/main/TestBankWithAnno.rar)

# 五、Spring声明式事务

**编程式事务：**在业务代码中添加事务控制代码，这样的事务控制机制就叫做编程式事务

**声明式事务：**通过xml或者注解配置的⽅式达到事务控制的⽬的，叫做声明式事务

## 5.1 事务

事务指逻辑上的⼀组操作，组成这组操作的各个单元，要么全部成功，要么全部不成功。从⽽确保了数据的准确与安全。

例如A向B转账，那么需要更新A的钱减少，然后更新B的钱增加，这两条语句，要么一起成功，要么一起失败。

### 5.1.1 特性

事务有四大特性：

**原⼦性（Atomicity）**

原⼦性是指事务是⼀个不可分割的⼯作单位，事务中的操作要么都发⽣，要么都不发⽣。从操作的⻆度来描述，事务中的各个操作要么都成功要么都失败。

**⼀致性（Consistency）** 

事务必须使数据库从⼀个⼀致性状态变换到另外⼀个⼀致性状态。例如转账前A有1000，B有1000。转账后A+B也得是2000。⼀致性是从数据的⻆度来说的，（1000，1000） （900，1100），不应该出现（900，1000）。

**隔离性（Isolation）**

事务的隔离性是多个⽤户并发访问数据库时，数据库为每⼀个⽤户开启的事务，每个事务不能被其他事务的操作数据所⼲扰，多个并发事务之间要相互隔离。⽐如：事务1给员⼯涨⼯资2000，但是事务1尚未被提交，员⼯发起事务2查询⼯资，发现⼯资涨了2000块钱，读到了事务1尚未提交的数据（脏读）。

**持久性（Durability）**

持久性是指⼀个事务⼀旦被提交，它对数据库中数据的改变就是永久性的，接下来即使数据库发⽣故障也不应该对其有任何影响。

### 5.1.2 事务隔离级别

如果不考虑隔离性引发的安全性问题

**读问题：**

脏读：一个事务读到另一个事务未提交的数据

不可重复读：一个事务读到另一个事务已经提交的update的数据，导致一个事务中多次查询结果不一致。

虚读、幻读：一个事务读到另一个事务已经提交的insert数据，导致一个事务中多次查询结果不一致。

**写问题：**丢失更新

**设置事务的隔离级别：**

Read uncommitted：未提交读，任何读问题都解决不了

Read committed：已读提交，解决脏读，但是不可重复读和虚读有可能发生（Oracle）

Repeatable read：重复读，解决脏读和不可重复读，但是虚读有可能发生（Mysql）

Serializable：解决所有读问题

MySQL的默认隔离级别是：REPEATABLE READ

查询当前使⽤的隔离级别： select @@tx_isolation;

设置MySQL事务的隔离级别： set session transaction isolation level xxx; （设置的是当前mysql连接会话的，并不是永久改变的）

### 5.1.3 事务的传播行为

**Spring中传播行为用途：**

```
保证多个操作在同一个事务中
PROPAGATION_REQUIRED :默认值，如果A中有事务，使用A中的事务，如果A没有，创建事务将操作包含起来

PRQPAGATION_SUPPORTS：支持事务，如果A中有事务，使用A中的事务，如果A没有事务，不使用事务

PROPAGATION_MANDATORY：如果A中有事务，使用A中的事务，如果A没有事务，抛出异常
```

```
保证多个操作不在同一个事务中：
PROPAGATION_REQUIRES_NEW：如果A中有事务，将A中事务挂起（暂停），创建新事务，只包含自身操作。如果A中没有事务，创建新事务，包含自身操作

PROPAGATION_NOT_SUPPORTED：如果A中有事务，将A的事务挂起，不使用事务管理

PROPAGATION_NEVER：如果A中有事务，抛出异常
```

```
嵌套式事务
PROPAGATION_NESTED：嵌套事务，若果A中有事务，按照A的执行，执行完成后，设置A的保存点，执行B的操作，若果没有异常，执行通过，若果有异常，可以选择回滚到最初始位置，也可以回滚到保存点。
```

## 5.2 Spring中声明式事务配置

### 5.2.1 纯XML

~~~xml
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-context</artifactId>
     <version>5.1.12.RELEASE</version>
</dependency>
<dependency>
     <groupId>org.aspectj</groupId>
     <artifactId>aspectjweaver</artifactId>
     <version>1.9.4</version>
</dependency>
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-jdbc</artifactId>
     <version>5.1.12.RELEASE</version>
</dependency>
<dependency>
     <groupId>org.springframework</groupId>
     <artifactId>spring-tx</artifactId>
     <version>5.1.12.RELEASE</version>
</dependency>
~~~

~~~xml
<tx:advice id="txAdvice" transaction-manager="transactionManager">
     <!--定制事务细节，传播⾏为、隔离级别等-->
     <tx:attributes>
         <!--⼀般性配置-->
         <tx:method name="*" read-only="false"
        propagation="REQUIRED" isolation="DEFAULT" timeout="-1"/>
         <!--针对查询的覆盖性配置-->
         <tx:method name="query*" read-only="true"
        propagation="SUPPORTS"/>
     </tx:attributes>
 </tx:advice>
 <aop:config>
     <!--advice-ref指向增强=横切逻辑+⽅位-->
     <aop:advisor advice-ref="txAdvice" pointcut="execution(*
    com.lagou.edu.service.impl.TransferServiceImpl.*(..))"/>
 </aop:config>
~~~



### 5.2.2 XML+注解

~~~xml
<!--配置事务管理器-->
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManage
r">
     <property name="dataSource" ref="dataSource"></property>
</bean>
<!--开启spring对注解事务的⽀持-->
<tx:annotation-driven transaction-manager="transactionManager"/>
~~~

然后在类或方法上添加@Transactional注解

### 5.2.3 纯注解

在 Spring 的配置类上添加 @EnableTransactionManagement 注解即可

# 六、Spring源码

## 6.1 源码构建

Spring的源码构建只要版本都能对应上，基本上不会有什么问题，构建会很顺畅。我的版本：`jdk1.8.0_301,gradle5.6.4,spring5.2.x`下面分享一下各个版本对应：

| Spring   | Gradle            | JDK  |
| -------- | ----------------- | ---- |
| main分支 | 最新版            | 17   |
| 5.3.x    | 7.4（截至2022.3） | 11   |
| 5.2.x    | 5.6.4             | 8    |

上面表格只是大概的版本参照，有几点需要注意：

- 5.3.x版本在jdk8的环境下能进行构建，但是不能编译，因为缺少jrf包，这个包据说在有的jdk8的版本是有的，我编译时使用的是jdk1.8.0_301也是缺少这个包的，所以最好使用jdk11，如果你非想用jdk8但还失败的话可以尝试更换jdk8的小版本，只要不缺少jrf包即可，但是我个人是没有尝试，有大佬成功也可以告知一下。

- 构建成功，编译也可能不成功，我个人情况就是这样，jdk1.8构建spring5.3.x成功但是编译时报错，因为缺少jar包。

- 构建成功就可以有一个源码阅读环境了，可以在ApplicationContext类下按ctrl+alt+u,有继承关系就代编构建成功，如图：

  ![image-20220310131419205](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310131419205.png)

- PS：编译不成功有90%可能是版本不匹配，不要挣扎用网上的更换IDEA的设置，比如更改java compile，更改project的jdk设置等，我是一个没成功，这种方法可能是版本匹配上还编译失败才能有用。
- 或者想使用jdk8就把spring源码版本降为5.2.x，我全程没有问题，直接构建编译成功。
- gradle的版本可以在gradle/wrapper文件夹下的`gradle-wrapper.properties`文件中看到，你本地的gradle安装什么版本跟它保持一致就行。
- git clone下来的源码默认是main分支，请自行更换分支，比如更换到5.3.x等，我clone下来时就忘了，徒劳挣扎一个小时。

构建流程：

1. 首先git、jdk等环境必须要有，IDE推荐IDEA2019.2.x以上，因为能导入项目后能自动。这里就不过多赘述了。然后去github上克隆源码。这里推荐ssh的方式下载，不用非得fork到自己的仓库。

2. 下载完后切一下分支，当然你想看最新的也可以就留在main分支下。

3. 安装配置Gradle。可以先去gradle/wrapper文件夹下的`gradle-wrapper.properties`文件中看一下gradle的版本，然后去这个地址[Gradle官方](https://services.gradle.org/distributions/)下载安装对应的版本。安装完配置一下环境变量。下载的zip包先不要删除。gradle安装配置这里略。

4. 直接把spring源码导入到IDEA即可（注意IDEA版本大于2019.2），然后配置一下IDEA中的gradle，如下图：

   ![image-20220310131528930](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310131528930.png)

5. 改一下项目中gradle的下载源地址改成阿里的。改一下配置，不让gradle每次都去下载。具体修改如下图：

   ![image-20220310131720408](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310131720408.png)

   ![image-20220310131754241](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310131754241.png)

6. 等待IDEA自动构建完成，按照spring的github的wiki按顺序编译就行。顺序如下：

   ```rust
    spring->spring-oxm -> Tasks->Other->compileJava 
    spring->spring-core -> Tasks->Other->compileJava 
    spring->spring-context -> Tasks->Other->compileJava 
   ```

   ![image-20220310134620351](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310134620351.png)

7. 然后自己新建一个Gradle工程，编写测试代码运行即可，如果能正确运行成功代表编译成功。

   测试代码如下：

   ```java
   public class Test {
      public static void main(String[] args) {
         AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MyConfig.class);
         System.out.println(applicationContext.getBean("User"));
      }
   }
   ```

   ```java
   @Configuration
   public class MyConfig {
   
      @Bean("User")
      public User getUser(){
         return new User();//PS：User实体类随便建一下就行
      }
   }
   ```

8. 如果运行过程中出现CoroutinesUtils.class 找不到的提示，可以在如下位置添加jar包即可解决。

   第一步

   ![image-20220310135103249](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310135103249.png)

   第二步

   ![image-20220310135157835](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310135157835.png)

   第三步

   ![image-20220310135230055](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310135230055.png)

9. 如果出现Kotlin: Language version 1.3is no longer supported，那么就在下图位置更换版本。

10. 如果出现Kotlin: warnings found and -Weeror specified，那就把-Werror删掉即可

> ps：编译的时候是一个模块一个模块来的，哪个模块报这个错就改哪个模块

![image-20220310135400465](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220310135400465.png)

## 6.2 源码





