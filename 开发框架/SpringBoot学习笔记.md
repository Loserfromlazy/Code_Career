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

查看@SpringBootApplication注解的源码，源码如下：

~~~java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//一个注解等于三个注解功能
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(//组件扫描
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    Class<?>[] exclude() default {};

    @AliasFor(
        annotation = EnableAutoConfiguration.class
    )
    String[] excludeName() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackages"
    )
    String[] scanBasePackages() default {};

    @AliasFor(
        annotation = ComponentScan.class,
        attribute = "basePackageClasses"
    )
    Class<?>[] scanBasePackageClasses() default {};
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
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
~~~

其中@Import({AutoConfigurationImportSelector.class})导入了AutoConfigurationImportSelector类，点击查看AutoConfigurationImportSelector的源码：部分源码如下：

~~~java

public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!this.isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    } else {
        AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
        AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
        //获取配置
        List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
        configurations = this.removeDuplicates(configurations);
        Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
        this.checkExcludedClasses(configurations, exclusions);
        configurations.removeAll(exclusions);
        configurations = this.filter(configurations, autoConfigurationMetadata);
        this.fireAutoConfigurationImportEvents(configurations, exclusions);
        return StringUtils.toStringArray(configurations);
    }
}
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
~~~

其中SpringFactoriesLoader.loadFactoryNames方法的作用就是从META-INF/spring.factories文件中读取指定类对应的类名称列表。

spring.factories有关源码：

~~~
....
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
.....
~~~

上面配置文件存在大量的以Conﬁguration为结尾的类名称，这些类就是存有自动配置信息的类，而SpringApplication在获取这些类名后再加载。

我们以其中ServletWebServerFactoryAutoConfiguration为例，其部分源码如下：

~~~java
@Configuration
@AutoConfigureOrder(-2147483648)
@ConditionalOnClass({ServletRequest.class})
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@EnableConfigurationProperties({ServerProperties.class})
@Import({ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class, EmbeddedTomcat.class, EmbeddedJetty.class, EmbeddedUndertow.class})
public class ServletWebServerFactoryAutoConfiguration {
~~~

@EnableConﬁgurationProperties(ServerProperties.class) 代表加载ServerProperties服务器配置属性类

ServerProperties源码：

~~~java
@ConfigurationProperties(
    prefix = "server",
    ignoreUnknownFields = true
)
public class ServerProperties {
    private Integer port;
    private InetAddress address;
    @NestedConfigurationProperty
    private final ErrorProperties error = new ErrorProperties();
    private Boolean useForwardHeaders;
    private String serverHeader;
    private int maxHttpHeaderSize = 0;
    。。。。。。
~~~

preﬁx = "server" 表示SpringBoot配置文件中的前缀，SpringBoot会将配置文件中以server开始的属性映射到该类
的字段中。

例如在spring-configuration-metadata.json文件中某段以server前缀加上属性server.port配置了默认值：

~~~json
{
    "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties",
    "defaultValue": 8080,
    "name": "server.port",
    "description": "Server HTTP port.",
    "type": "java.lang.Integer"
},
~~~

以上为默认值，且可以覆盖，如果在yml文件中设置配置即可修改配置。

在spring-boot-starter-parent的pom.xml中如下

~~~xml
<resource>
    <filtering>true</filtering>
    <directory>${basedir}/src/main/resources</directory>
    <includes>
        <include>**/application*.yml</include>
        <include>**/application*.yaml</include>
        <include>**/application*.properties</include>
    </includes>
</resource>
~~~

以上代表在resources中如果设置了配置文件即可覆盖默认配置。

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

[SpringBoot配置信息查询]: https://docs.spring.io/spring-boot/docs/2.0.1.RELEASE/reference/htmlsingle/#common-application-properties

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









