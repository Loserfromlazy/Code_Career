# SpringBoot学习笔记

# 一、概述

## 1.1 Spring优缺点分析

**Spring优点**

Spring是Javaee的轻量级代替品，无需开发重量级的Enterprise JavaBean（EJB）对象，Spring为企业级Java开发提供了一种相对简单的方法，通过依赖注入和面向切面编程，用简单的Java对象（POJO）实现了EJB的功能。

**Spring的缺点**

虽然Spring是轻量级的，但是Spring的配置却是重量级的，一开始，Spring用xml配置，Spring2.5引入了基于注解的组件扫描，这消除了大量针对应用程序自身组件的显式xml配置，Spring3.0引入了基于Java的配置，这是一种类型安全的可重构配置方式，可以代替xml。所有这些配置代表了开发时的损耗。

除此之外，项目的依赖管理也十分耗时耗力，在环境搭建时，需要分析导入那些库的坐标，而且还需要分析导入与所有依赖关系其他哭的坐标，一旦选错了依赖的版本，不兼容问题也十分严重。

## 1.2 SpringBoot的概述

**SpringBoot解决Spring的缺点**

springboot对上述缺点进行改善和优化，**基于约定优于配置的思想**，可以让开发人员不必在配置与逻辑业务之间进行思维的切换，大大提高了开发的效率，一定程度缩短了项目周期。

**SpringBoot的特点**

- 为基于Spring开发提供更快的入门体验
- 开箱即用，无需xml配置，可以通过默认值开满足特定的需求。
- 提供了一些大型项目中常见的非功能性的特性，如嵌入式服务器、安全、指标、健康检测、外部配置等。
- SpringBoot不是对Spring功能上的增强，而是提供了一种快速使用Spring的方式

**SpringBoot的核心功能**

- 起步依赖

  起步依赖本质上是一个Maven项目对象模型（POM），定义了对其他库的传递依赖，这些东西加在一起及支持某项功能。

- 自动配置

  springboot的自动配置是一个运行时（应用程序启动时）的过程

## 1.3 SpringBoot快速入门

1. 创建maven工程

   普通java工程即可

2. 添加SpringBoot的起步依赖

   springboot要求项目要继承springboor的起步以来spring-boot-starter-parent

   ~~~xml
   <parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>2.0.1.RELEASE</version>
   </parent>
   ~~~

   springboot要集成springmvc的controller，所以项目要导入web的启动依赖

   ~~~xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>
   </dependencies>
   ~~~

3. 编写SpringBoot的引导类

   通过springboot提供的引导类起步SpringBoot才可以进行访问

   ~~~java
   package com.yhr.testspringboot;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   
   @SpringBootApplication
   public class MySpringBootApp {
       public static void main(String[] args) {
           SpringApplication.run(MySpringBootApp.class);
       }
   }
   
   ~~~

4. 编写controller

   在引导类MySpringBootApp同级包或者子级包中创建QuickStartController

   ~~~java
   package com.yhr.testspringboot.controller;
   
   import org.springframework.stereotype.Controller;
   import org.springframework.web.bind.annotation.RequestMapping;
   import org.springframework.web.bind.annotation.ResponseBody;
   
   @Controller
   public class QuickStartController {
       
       @RequestMapping("/quick")
       @ResponseBody
       public String quick(){
           return "SpringBoot 访问成功";
       }
   }
   
   ~~~

5. 测试

   启动SpringBoot起步类的主方法，控制台日志打印如下，tomcat已经启动，打开浏览器访问`localhost:8080/quick`

   > 2020-11-26 10:10:50.903  INFO 4060 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
   > 2020-11-26 10:10:50.919  INFO 4060 --- [           main] com.yhr.testspringboot.MySpringBootApp   : Started MySpringBootApp in 3.077 seconds (JVM running for 3.587)

   显示springboot访问成功

## 1.4 SpringBoot快速入门解析

**SpringBoot代码解析**

- @SpringBootApplication：标注SpringBoot的启动类，具备多种功能
- SpringApplication.run(MySpringBootApp.class)代表运行SpringBoot的启动类，参数为SpringBoot启动类的字节码对象

**SpringBoot工程热部署**

在开发中反复修改类、页面等资源，每次修改需要重新启动才生效，这样每次都需要重新启动很麻烦，浪费大量部署的时间，在pom.xml中添加如下配置可以实现在修改代码后不重启就能生效，我们称之为热部署。

~~~xml
<!--热部署配置-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
</dependency>
~~~

> IDEA进行SpringBoot热部署失败的原因：
>
> 根本原因是因为IDEA默认情况下不会自动编译，需要对IDEA进行自动编译的设置。
>
> 1. 点击Settings中Build下的Compiler，在右侧选中Make project automatically
> 2. 退出Settings，按下Shift+Ctrl+Alt+/，选中Registry，然后在弹出框中勾选compiler.automake.allow.when.app.running
> 3. 重启IDEA

**使用IDEA快速创建SpringBoot项目**

在新建中选择Spring Initializr而不是maven项目

# 二、SpringBoot原理分析

## 2.1 起步依赖原理分析

### 2.1.1 spring-boot-starter-parent

按住CTRL点击pom中的spring-boot-starter-parent跳转到了spring-boot-starter-parent中的pom.xml，部分配置如下：

~~~xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.0.1.RELEASE</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
~~~

继续跳转到spring-boot-dependencies，部分配置如下：

~~~xml
<properties>
    <activemq.version>5.15.3</activemq.version>
    <antlr2.version>2.7.7</antlr2.version>
    <appengine-sdk.version>1.9.63</appengine-sdk.version>
    <artemis.version>2.4.0</artemis.version>
    <aspectj.version>1.8.13</aspectj.version>
    <assertj.version>3.9.1</assertj.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test-autoconfigure</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator</artifactId>
            <version>2.0.1.RELEASE</version>
        </dependency>
    </dependencies>
</dependencyManagement>
~~~

可见一部分坐标的版本、依赖管理、插件管理已经定义好，所以我们的SpringBoot工程继承spring-boot-starter-parent后已经具备版本锁定等配置了。所以起步依赖的作用就是进行依赖的传递。

### 2.1.2  spring-boot-starter-web

跳转spring-boot-starter-web的pom中部分配置如下

~~~xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.0.1.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-json</artifactId>
      <version>2.0.1.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <version>2.0.1.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.hibernate.validator</groupId>
      <artifactId>hibernate-validator</artifactId>
      <version>6.0.9.Final</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-web</artifactId>
      <version>5.0.5.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.0.5.RELEASE</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
~~~

spring-boot-starter-web就是将web开发要使用的spring-web、spring-webmvc等坐标进行了“打包”，这样我们的工程只要引入spring-boot-starter-web起步依赖的坐标就可以进行web开发了，同样体现了依赖传递的作用。

## 2.2 自动配置原理分析

> 这里用的是2.7.2版本的springboot，不同版本的源码可能会有不同，但大致原理是相同的

查看@SpringBootApplication注解的源码，源码如下：

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //略。。。
}
~~~

其中

@SpringBootConfiguration：等同于@Configuration，既标注该类是一个Spring的配置类
@EnableAutoConfiguration：SpringBoot的自动配置功能开启

查看@EnableAutoConfiguration的源码

~~~java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage//自动配置包：将@SpringBootApplication注解所在的类的包及其子包的组件添加到容器中
@Import({AutoConfigurationImportSelector.class})//上一个注解将所有组件导入到了容器中，这个注解就是将需要自动装配的类以全类名的方式返回
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
~~~

### 2.2.1 @AutoConfigurationPackage

我们先来看`@AutoConfigurationPackage`注解的源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//@Import是spring框架的底层注解，作用是给容器导入某个组件类
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
   String[] basePackages() default {};

   Class<?>[] basePackageClasses() default {};

}
```

我们可以看到此注解的功能是由@Import注解导入了AutoConfigurationPackages.Registrar类实现的，我们跟进去看一下这个类：

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
   @Override
   public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
       //metadata：注解标注的元数据信息，可以debug自行查看其内容
       //获取包名，传入register方法，如下图，获取的是@SpringBootApplication注解所在的启动类的包名
      register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
   }

   @Override
   public Set<Object> determineImports(AnnotationMetadata metadata) {
      return Collections.singleton(new PackageImports(metadata));
   }
}
```

![image-20220816135313303](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816135313303.png)

```java
//将包及子包下的组件扫描到容器中
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
   if (registry.containsBeanDefinition(BEAN)) {
      BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
      ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
      constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
   }
   else {
      GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
      beanDefinition.setBeanClass(BasePackages.class);
      beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
      beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
      registry.registerBeanDefinition(BEAN, beanDefinition);
   }
}
```

也就是说这个注解`@AutoConfigurationPackage`的作用是将主配置类（@SpringBootApplication标注的类）所在包以及子包里面的所有组件扫描并加载到spring的容器中，这也就是为什么我们在利用springboot进行开发的时候，无论是Controller还是Service的路径都是与主配置类同级或者次级的原因

### 2.2.2 @EnableAutoConfiguration

再来看`@EnableAutoConfiguration`的源码中，@Import({AutoConfigurationImportSelector.class})导入了AutoConfigurationImportSelector类。

> 我们需要先了解了解@Import注解，@Import是spring框架的底层注解，作用是给容器导入某个组件类
>
> 而且实现ImportSelector接口的类，需要返回String[]，通过@Import()注解，将数组中的对象名将作为对象生成为bean

查看AutoConfigurationImportSelector的源码，主要关注selectImports方法（原因见上面），主要源码如下：

~~~java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
        ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

    private static final String[] NO_IMPORTS = {};

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        //检测是否开启了SpringBoot自动装配，默认都是开启的
        if (!isEnabled(annotationMetadata)) {
            //没开启就返回空字符串数组
            return NO_IMPORTS;
        }
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        //获取注解的属性信息
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //获取候选配置信息加载的是，当前项目的classpath目录下的、所有的 spring.factories 文件中的key为 org.springframework.boot.autoconfigure.EnableAutoConfiguration的信息
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        configurations = removeDuplicates(configurations);
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = getConfigurationClassFilter().filter(configurations);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return new AutoConfigurationEntry(configurations, exclusions);
    }

}
~~~

我们让断点停到`getCandidateConfigurations`，如下图：

![image-20220816141820905](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816141820905.png)

我们可以看到getCandidateConfigurations方法加载了很多的配置类全路径，他们是从哪来的呢？我们进入此方法可以看到（见上图）这些配置配来自META-INF/spring.factories文件中。

> 除了上面断言以外，其实一路跟代码也能找到此路径。
>
> ![image-20220816144331323](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816144331323.png)

然后我们继续看此方法：

~~~java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        //获取注解的属性信息
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //获取候选配置信息加载的是，当前项目的classpath目录下的、所有的 spring.factories 文件中的key为 org.springframework.boot.autoconfigure.EnableAutoConfiguration的信息
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        //去除重复的配置这里用了LinkedHashSet进行去重，因为会加载多个spring.factories文件，有可能存在同名的配置
        configurations = removeDuplicates(configurations);
    	//获取配置的exclude信息,比如：
    	//@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class, DruidDataSourceAutoConfigure.class})
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
    	//过滤掉不需要的配置类
        configurations = getConfigurationClassFilter().filter(configurations);
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return new AutoConfigurationEntry(configurations, exclusions);
    }
~~~

此方法中剩下比较重要的就是过滤不需要的配置类，我们可以debug查看，当过滤完后会减少很多的配置类，如下图：

![image-20220816145327649](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816145327649.png)

我们可以跟进filter方法，会发现filter方法中会根据自动配置的元信息进行匹配过滤，这个元信息是在构造函数中创建的，我们一路向下跟最后会发新元信息是从`META-INF/spring-autoconfigure-metadata.properties`文件中拿到的，代码和流程如下图：

![image-20220816150153202](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816150153202.png)

我们看一下此文件，此文件的内容是由配置类全路径、配置类被加载的条件和条件值组成。我们以mongodb的实例来演示：

![image-20220816150724369](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816150724369.png)

因此过滤完后就没有此配置类了：

![image-20220816150816123](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220816150816123.png)

# 三、SpringBoot的配置文件

## 3.1 SpringBoot配置文件类型

springboot是基于约定的，所以很多配置都有默认值，如果想替换默认配置的话，就可以使用application.properties或者application.yml进行设置。

SpringBoot默认会从resources目录下加载application.properties或者application.yml文件。

其中properties是以键值对的方式，下面详细介绍yml配置方式

### 3.1.1 application.yml配置文件

yml是以YMAL(YAML Aint Markup Language)编写的文件格式，是一种直观的能被电脑识别的数据序列化格式，且易被阅读，易与脚本语言交互，yml是以数据为核心的，比xml更简洁。

### 3.1.2 yml配置文件的语法

**配置普通数据**

`key: value`

> name : haha

注意value前有一个空格

**配置对象数据**

`key:`

`key1: value1`

`key2: value2`

或者

`key: {key1: value,key2: value2}`

~~~yml
persion:
	name: haha
	age: 10
	addr: beijing
#或者
persion: {name: haha,age: 10,addr: beijing}
~~~

注意：在yml语法中相同缩进代表同一个级别

**配置map数据**

与上相同

**配置数组（List、Set）数据**

`key:`

​	`- value`

​	`- value`

或者`key: [value1,value2]`

~~~yml
city:
	- beijing
	- nanjing
	- tianjing
#或者
city: [beijing,nanjing,tianjing]
#集合的元素是对象形式
student: 
	-name: haha
	 age: 18
	 score: 80
	-name: lala
	 age: 19
	 score: 60
~~~

注意：value1与之间的- 之间存在一个空格

### 3.1.3 SpringBoot配置信息的查询

[SpringBoot配置信息查询](https://docs.spring.io/spring-boot/docs/2.0.1.RELEASE/reference/htmlsingle/#common-application-properties)

可以通过修改application.properties或者application.yml来修改springboot的默认配置，例如：

properties

~~~properties
server.port=8081
server.servlet.context-path=demp
~~~

yml

~~~yml
server:
	port: 8081
	servlet:
		context-path: /demo
~~~

## 3.2 配置文件与配置类的属性映射方式

### 3.2.1 使用注解@Value映射

我们可以通过@Value将配置文件中的值映射到一个Spring管理的Bean上,此时使用的是${}

> ps:
>
> @Value("#{}") 表示SpEl表达式通常用来获取bean的属性，或者调用bean的某个方法。当然还有可以表示常量
>
> 通过@Value("${}") 可以获取对应属性文件中定义的属性值。假如我有一个sys.properties文件 里面规定了一组值： web.view.prefix =/WEB-INF/views/

~~~
#properties
persion:
	name: haha
	age: 18
#yml
persion:
	name: haha
	age: 18
~~~

Bean:

~~~java
@Controller
public class QuickStartController {
    @Value("${person.name}")
    private String name;
    @Value("${person.age}")
    private Integer age;
    
    @RequestMapping("/quick")
    @ResponseBody
    public String quick(){
        return "SpringBoot 访问成功 name="+name +"age="+age;
    }
}
~~~

### 3.2.2 使用注解@ConfigurationProperties映射

通过注解@ConﬁgurationProperties(preﬁx="配置文件中的key的前缀")可以将配置文件中的配置自动与实体进行映射

~~~java
@Controller
@ConﬁgurationProperties(preﬁx="person")
public class QuickStartController {
    
    private String name;

    private Integer age;
    
    public void setName(String name) {
    	this.name = name;
    }
    public void setAge(Integer age) {
    	this.age = age;
    }

    
    @RequestMapping("/quick")
    @ResponseBody
    public String quick(){
        return "SpringBoot 访问成功 name="+name +"age="+age;
    }
}
~~~

使用@ConﬁgurationProperties方式可以进行配置文件与实体字段的自动映射，但需要字段必须提供set方法才可以，而使用@Value注解修饰的字段不需要提供set方法

# 四、SpringBoot整合其他技术

## 4.1 整合Mybatis

mysql8.0以上版本

**添加依赖**

~~~xml
<!--mybatis-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.1</version>
        </dependency>
        <!--MYSQL驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
~~~

**配置文件**

~~~properties
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/taotaostore?useUnicode=true&characterEncoding=UTF8&useSSL=false&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=manager
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
~~~

**数据库建表并创建实体类**

~~~java
//建表略
public class User{
    private Long id;
    private String username;
    private String password;
    private String name;
    //get set方法略
}
~~~

**编写mapper**

接口

~~~java
@Mapper
public interface UserMapeer {
    public List<User> queryUserList();
}

~~~

xml

~~~xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.itheima.mapper.UserMapper">
    <select id="queryUserList" resultType="user">
        select * from user
    </select>
</mapper>

~~~

**Controller**

~~~java
@Controller
public class QuickStartController {
    @Autowired
    private UserMapper userMapper;

    @RequestMapping("/queryUser")
    @ResponseBody
    public List<User> queryUser(){
        List<User> users = userMapper.queryUserList();
        return users;
    }
}
~~~

测试成功

## 4.2 整合Junit

**添加依赖**

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
~~~

**测试类**

~~~java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = MySpringBootApp.class)
public class TestMapper {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void test(){
        List<User> users = userMapper.queryUserList();
        System.out.println(users);
    }
}

~~~

@SpringBootTest的属性指定的是引导类的字节码对象

## 4.3 整合Spring Data JPA

**添加依赖**

~~~xml
<!--springboot JPA依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!--MYSQL驱动-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.11</version>
</dependency>
~~~

**配置**

~~~properties
#JPA
#DB Configuration 同上
#JPA Configuration
spring.jpa.database=mysql
spring.jpa.show-sql=true
spring.jpa.generate-ddl=true
spring.jpa.hibernate.ddl-auto=update

~~~

**实体类**

~~~java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class User_JPA {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private Integer age;

    private String username;

    private String password;

    private String email;

    private String sex;

   //get set方法
}

~~~

**接口**

~~~java
public interface UserRepository extends JpaRepository<User_JPA,Integer>{

    public List<User_JPA> findAll();
}

~~~

**测试**

~~~java
    @Test
    public void test1(){
        List<UserJPA> users = userRepository.findAll();
        System.out.println(users);
    }
~~~

> 如果是jdk9需要以下坐标
>
> ~~~xml
> <dependency>
> 	<groupId>javax.xml.bind</groupId>
>     <artifactId>javax-api</artifactId>
> 	<version>2.3.0</version>
> </dependency>
> ~~~

## 4.4 整合Redis

**添加依赖**

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
~~~

**配置信息**

~~~properties
#redis配置
spring.redis.host=127.0.0.1
spring.redis.port=6379
~~~

**注入RedisTemplate操作测试**









