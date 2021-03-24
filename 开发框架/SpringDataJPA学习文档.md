# SpringData JPA学习文档

# 一、概述

## 1.1 ORM概述

ORM（Object-Relational Mapping） 表示对象关系映射。在面向对象的软件开发中，通过ORM，就可以把对象映射到关系型数据库中。只要有一套程序能够做到建立对象与数据库的关联，操作对象就可以直接操作数据库数据，就可以说这套程序实现了ORM对象关系映射。简单的说：**ORM就是建立实体类和数据库表之间的关系，从而达到操作实体类就相当于操作数据库表的目的。**

为什么使用ORM？

当实现一个应用程序时（不使用O/R Mapping），我们可能会写特别多数据访问层的代码，从数据库保存数据、修改数据、删除数据，而这些代码都是重复的。而使用ORM则会大大减少重复性代码。对象关系映射（Object Relational Mapping，简称ORM），主要实现程序对象到关系数据库数据的映射。

常见的ORM框架？

Mybatis（ibatis）、Hibernate、Jpa

## 1.2 Hibernate与JPA概述

**什么是Hibernate**

Hibernate是一个开放源代码的对象关系映射框架，它对JDBC进行了非常轻量级的对象封装，它将POJO与数据库表建立映射关系，是一个全自动的orm框架，hibernate可以自动生成SQL语句，自动执行，使得Java程序员可以随心所欲的使用对象编程思维来操纵数据库。

**什么是JPA**

JPA的全称是Java Persistence API， 即Java 持久化API，是SUN公司推出的一套基于ORM的规范，内部是由一系列的接口和抽象类构成。

JPA通过JDK 5.0注解描述对象－关系表的映射关系，并将运行期的实体对象持久化到数据库中。

**JPA的优势**

1. 标准化

   JPA 是 JCP 组织发布的 Java EE 标准之一，因此任何声称符合 JPA 标准的框架都遵循同样的架构，提供相同的访问API，这保证了基于JPA开发的企业应用能够经过少量的修改就能够在不同的JPA框架下运行。

2. 容器级特性的支持

   JPA框架中支持大数据集、事务、并发等容器级事务，这使得 JPA 超越了简单持久化框架的局限，在企业应用发挥更大的作用。

3. 简单方便

   JPA的主要目标之一就是提供更加简单的编程模型：在JPA框架下创建实体和创建Java 类一样简单，没有任何的约束和限制，只需要使用 javax.persistence.Entity进行注释，JPA的框架和接口也都非常简单，没有太多特别的规则和设计模式的要求，开发者可以很容易的掌握。JPA基于非侵入式原则设计，因此可以很容易的和其它框架或者容器集成

4. 查询能力

   JPA的查询语言是面向对象而非面向数据库的，它以面向对象的自然语法构造查询语句，可以看成是Hibernate HQL的等价物。JPA定义了独特的JPQL（Java Persistence Query Language），JPQL是EJB QL的一种扩展，它是针对实体的一种查询语言，操作对象是实体，而不是关系数据库的表，而且能够支持批量更新和修改、JOIN、GROUP BY、HAVING 等通常只有 SQL 才能够提供的高级查询特性，甚至还能够支持子查询。

5. 高级

   JPA 中能够支持面向对象的高级特性，如类之间的继承、多态和类之间的复杂关系，这样的支持能够让开发者最大限度的使用面向对象的模型设计企业应用，而不需要自行处理这些特性在关系数据库的持久化。

**Hibernate与JPA关系**

JPA和Hibernate的关系就像JDBC和JDBC驱动的关系，JPA是规范，Hibernate除了作为ORM框架之外，它也是一种JPA实现。JPA怎么取代Hibernate呢？JDBC规范可以驱动底层数据库吗？答案是否定的，也就是说，如果使用JPA规范进行数据库操作，底层需要hibernate作为其实现类完成数据持久化工作。

# 二、快速入门

由于JPA是sun公司制定的API规范，所以我们不需要导入额外的JPA相关的jar包，只需要导入JPA的提供商的jar包。我们选择Hibernate作为JPA的提供商，所以需要导入Hibernate的相关jar包。

## 2.1 环境搭建

~~~xml
<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.hibernate.version>5.0.7.Final</project.hibernate.version>
	</properties>

	<dependencies>
		<!-- junit -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.12</version>
			<scope>test</scope>
		</dependency>

		<!-- hibernate对jpa的支持包 -->
		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-entitymanager</artifactId>
			<version>${project.hibernate.version}</version>
		</dependency>

		<!-- c3p0 -->
		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-c3p0</artifactId>
			<version>${project.hibernate.version}</version>
		</dependency>

		<!-- log日志 -->
		<dependency>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
			<version>1.2.17</version>
		</dependency>

		<!-- Mysql and MariaDB -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.6</version>
		</dependency>
	</dependencies>


~~~

## 2.2 创建实体类及数据库表

~~~mysql
/*创建客户表*/
    CREATE TABLE cst_customer (
      cust_id bigint(32) NOT NULL AUTO_INCREMENT COMMENT '客户编号(主键)',
      cust_name varchar(32) NOT NULL COMMENT '客户名称(公司名称)',
      cust_source varchar(32) DEFAULT NULL COMMENT '客户信息来源',
      cust_industry varchar(32) DEFAULT NULL COMMENT '客户所属行业',
      cust_level varchar(32) DEFAULT NULL COMMENT '客户级别',
      cust_address varchar(128) DEFAULT NULL COMMENT '客户联系地址',
      cust_phone varchar(64) DEFAULT NULL COMMENT '客户联系电话',
      PRIMARY KEY (`cust_id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

~~~

实体类略

## 2.3 编写映射关系

在实体类上使用jpa注解的形式进行映射配置

~~~java
/**
*		* 所有的注解都是使用JPA的规范提供的注解，
 *		* 所以在导入注解包的时候，一定要导入javax.persistence下的
 */
@Entity //声明实体类
@Table(name="cst_customer") //建立实体类和表的映射关系
public class Customer {
	
	@Id//声明当前私有属性为主键
	@GeneratedValue(strategy=GenerationType.IDENTITY) //配置主键的生成策略
	@Column(name="cust_id") //指定和表中cust_id字段的映射关系
	private Long custId;
	
	@Column(name="cust_name") //指定和表中cust_name字段的映射关系
	private String custName;
	
	@Column(name="cust_source")//指定和表中cust_source字段的映射关系
	private String custSource;
	
	@Column(name="cust_industry")//指定和表中cust_industry字段的映射关系
	private String custIndustry;
	
	@Column(name="cust_level")//指定和表中cust_level字段的映射关系
	private String custLevel;
	
	@Column(name="cust_address")//指定和表中cust_address字段的映射关系
	private String custAddress;
	
	@Column(name="cust_phone")//指定和表中cust_phone字段的映射关系
	private String custPhone;
	
	//getter and setter略
}

~~~

**常用注解**

~~~
@Entity
作用：指定当前类是实体类。
@Table
作用：指定实体类和表之间的对应关系。
属性：
name：指定数据库表的名称
@Id
作用：指定当前字段是主键。
@GeneratedValue
作用：指定主键的生成方式。。
属性：
strategy ：指定主键生成策略。
@Column
作用：指定实体类属性和数据库表之间的对应关系
属性：
name：指定数据库表的列名称。
unique：是否唯一  
nullable：是否可以为空  
inserttable：是否可以插入  
updateable：是否可以更新  
columnDefinition: 定义建表时创建此列的DDL  
secondaryTable: 从表名。如果此列不建在主表上（默认建在主表），该属性定义该列所在从表的名字搭建开发环境[重点]

~~~

## 2.4 配置核心配置文件

persistence.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/persistence  
    http://java.sun.com/xml/ns/persistence/persistence_2_0.xsd"
	version="2.0">
	<!--配置持久化单元 
		name：持久化单元名称 
		transaction-type：事务类型
		 	RESOURCE_LOCAL：本地事务管理 
		 	JTA：分布式事务管理 -->
	<persistence-unit name="myJpa" transaction-type="RESOURCE_LOCAL">
		<!--配置JPA规范的服务提供商 -->
		<provider>org.hibernate.jpa.HibernatePersistenceProvider</provider>
		<properties>
			<!-- 数据库驱动 -->
			<property name="javax.persistence.jdbc.driver" value="com.mysql.jdbc.Driver" />
			<!-- 数据库地址 -->
			<property name="javax.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/ssh" />
			<!-- 数据库用户名 -->
			<property name="javax.persistence.jdbc.user" value="root" />
			<!-- 数据库密码 -->
			<property name="javax.persistence.jdbc.password" value="111111" />

			<!--jpa提供者的可选配置：我们的JPA规范的提供者为hibernate，所以jpa的核心配置中兼容hibernate的配 -->
			<property name="hibernate.show_sql" value="true" />
			<property name="hibernate.format_sql" value="true" />
			<property name="hibernate.hbm2ddl.auto" value="create" />
		</properties>
	</persistence-unit>
</persistence>

~~~

## 2.5 实现保存操作

~~~java
@Test
public void test() {
    /**
	* 创建实体管理类工厂，借助Persistence的静态方法获取
	* 其中传递的参数为持久化单元名称，需要jpa配置文件中指定
	*/
    EntityManagerFactory factory = Persistence.createEntityManagerFactory("myJpa");
    //创建实体管理类
    EntityManager em = factory.createEntityManager();
    //获取事务对象
    EntityTransaction tx = em.getTransaction();
    //开启事务
    tx.begin();
    Customer c = new Customer();
    c.setCustName("aaa");
    //保存操作
    em.persist(c);
    //提交事务
    tx.commit();
    //释放资源
    em.close();
    factory.close();
}
~~~

# 三、JPA中的主键生成策略

通过annotation（注解）来映射hibernate实体的,基于annotation的hibernate主键标识为@Id, 其生成规则由@GeneratedValue设定的.这里的@id和@GeneratedValue都是JPA的标准用法。

JPA提供的四种标准用法TABLE,SEQUENCE,IDENTITY,AUTO。

具体说明如下：

**IDENTITY:**主键由数据库自动生成（主要是自动增长型）

用法：

~~~java
@Id  
@GeneratedValue(strategy = GenerationType.IDENTITY) 
private Long custId;
~~~

**SEQUENCE**：根据底层数据库的序列来生成主键，条件是数据库支持序列。

~~~java
@Id  
@GeneratedValue(strategy = GenerationType.SEQUENCE,generator="payablemoney_seq")  
@SequenceGenerator(name="payablemoney_seq", sequenceName="seq_payment")  
private Long custId;

~~~

**AUTO**：主键由程序控制

~~~java
@Id  
@GeneratedValue(strategy = GenerationType.AUTO)  
private Long custId;
~~~

**TABLE**：使用一个特定的数据库表格来保存主键

~~~java
    @Id  
    @GeneratedValue(strategy = GenerationType.TABLE, generator="payablemoney_gen")  
    @TableGenerator(name = "pk_gen",  
        table="tb_generator",  
        pkColumnName="gen_name",  
        valueColumnName="gen_value",  
        pkColumnValue="PAYABLEMOENY_PK",  
        allocationSize=1  
    ) 
private Long custId;

    //这里应用表tb_generator，定义为 ：
    CREATE TABLE  tb_generator (  
      id NUMBER NOT NULL,  
      gen_name VARCHAR2(255) NOT NULL,  
      gen_value NUMBER NOT NULL,  
      PRIMARY KEY(id)  
    )


~~~

# 四、JPA中的API介绍

## 4.1 Persistence

Persistence对象主要作用是用于获取EntityManagerFactory对象的 。通过调用该类的createEntityManagerFactory静态方法，根据配置文件中持久化单元名称创建EntityManagerFactory。

~~~java
//1. 创建 EntitymanagerFactory
@Test
String unitName = "myJpa";
EntityManagerFactory factory= Persistence.createEntityManagerFactory(unitName);
~~~

## 4.2 EntityManagerFactory

EntityManagerFactory 接口主要用来创建 EntityManager 实例

~~~java
//创建实体管理类
EntityManager em = factory.createEntityManager();
由于EntityManagerFactory 是一个线程安全的对象（即多个线程访问同一个EntityManagerFactory 对象不会有线程安全问题），并且EntityManagerFactory 的创建极其浪费资源，所以在使用JPA编程时，我们可以对EntityManagerFactory 的创建进行优化，只需要做到一个工程只存在一个EntityManagerFactory 即可
~~~

## 4.3 EntityManager

在 JPA 规范中, EntityManager是完成持久化操作的核心对象。实体类作为普通 java对象，只有在调用 EntityManager将其持久化后才会变成持久化对象。EntityManager对象在一组实体类与底层数据源之间进行 O/R 映射的管理。它可以用来管理和更新 Entity Bean, 根椐主键查找 Entity Bean, 还可以通过JPQL语句查询实体。

我们可以通过调用EntityManager的方法完成获取事务，以及持久化数据库的操作。

***getTransaction : 获取事务对象***    

***persist ： 保存操作***    

***merge ： 更新操作***    

***remove ： 删除操作***    

***find/getReference ： 根据id查询***  

## 4.4 EntityTransaction

在 JPA 规范中, EntityTransaction是完成事务操作的核心对象，对于EntityTransaction在我们的java代码中承接的功能比较简单

  begin：开启事务  

commit：提交事务  

rollback：回滚事务  

## 4.5 抽取JPAUtil工具类

~~~java
package loserfromlazy;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.Persistence;

public final class JPAUtil {
	// JPA的实体管理器工厂：相当于Hibernate的SessionFactory
	private static EntityManagerFactory em;
	// 使用静态代码块赋值
	static {
		// 注意：该方法参数必须和persistence.xml中persistence-unit标签name属性取值一致
		em = Persistence.createEntityManagerFactory("myPersistUnit");
	}

	/**
	 * 使用管理器工厂生产一个管理器对象
	 * 
	 * @return
	 */
	public static EntityManager getEntityManager() {
		return em.createEntityManager();
	}
}
~~~

# 五、使用JPA完成增删改查

## 5.1 保存

~~~java
/**
* 保存一个实体
*/
@Test
public void testAdd() {
    // 定义对象
    Customer c = new Customer();
    c.setCustName("name");
    c.setCustLevel("VIP客户");
    c.setCustSource("网络");
    c.setCustIndustry("IT教育");
    c.setCustAddress("北京");
    c.setCustPhone("12345678");
    EntityManager em = null;
    EntityTransaction tx = null;
    try {
        // 获取实体管理对象
        em = JPAUtil.getEntityManager();
        // 获取事务对象
        tx = em.getTransaction();
        // 开启事务
        tx.begin();
        // 执行操作
        em.persist(c);
        // 提交事务
        tx.commit();
    } catch (Exception e) {
        // 回滚事务
        tx.rollback();
        e.printStackTrace();
    } finally {
        // 释放资源
        em.close();
    }
}
~~~

## 5.2 修改

~~~java
    @Test
    public void testMerge(){  
        //定义对象
        EntityManager em=null;  
        EntityTransaction tx=null;  
        try{  
          	//获取实体管理对象
          	em=JPAUtil.getEntityManager();
          	//获取事务对象
          	tx=em.getTransaction();
          	//开启事务
          	tx.begin();
          	//执行操作
          	Customer c1 = em.find(Customer.class, 6L);
          	c1.setCustName("江苏传智学院");
         	em.clear();//把c1对象从缓存中清除出去
          	em.merge(c1);
          	//提交事务
          	tx.commit(); 
        }catch(Exception e){
          	//回滚事务
          	tx.rollback();
          	e.printStackTrace();  
        }finally{  
        	//释放资源
        	em.close();  
        }    
    }

~~~

## 5.3 删除

~~~java
	/**
	 * 删除
	 */
	@Test
	public void testRemove() {
		// 定义对象
		EntityManager em = null;
		EntityTransaction tx = null;
		try {
			// 获取实体管理对象
			em = JPAUtil.getEntityManager();
			// 获取事务对象
			tx = em.getTransaction();
			// 开启事务
			tx.begin();
			// 执行操作
			Customer c1 = em.find(Customer.class, 6L);
			em.remove(c1);
			// 提交事务
			tx.commit();
		} catch (Exception e) {
			// 回滚事务
			tx.rollback();
			e.printStackTrace();
		} finally {
			// 释放资源
			em.close();
		}
	}
~~~

## 5.4 跟据id查询

~~~java
	/**
	 * 查询一个： 使用立即加载的策略
	 */
	@Test
	public void testGetOne() {
		// 定义对象
		EntityManager em = null;
		EntityTransaction tx = null;
		try {
			// 获取实体管理对象
			em = JPAUtil.getEntityManager();
			// 获取事务对象
			tx = em.getTransaction();
			// 开启事务
			tx.begin();
			// 执行操作
			Customer c1 = em.find(Customer.class, 1L);
			// 提交事务
			tx.commit();
			System.out.println(c1); // 输出查询对象
		} catch (Exception e) {
			// 回滚事务
			tx.rollback();
			e.printStackTrace();
		} finally {
			// 释放资源
			em.close();
		}
	}

	// 查询实体的缓存问题
	@Test
	public void testGetOne() {
		// 定义对象
		EntityManager em = null;
		EntityTransaction tx = null;
		try {
			// 获取实体管理对象
			em = JPAUtil.getEntityManager();
			// 获取事务对象
			tx = em.getTransaction();
			// 开启事务
			tx.begin();
			// 执行操作
			Customer c1 = em.find(Customer.class, 1L);
			Customer c2 = em.find(Customer.class, 1L);
			System.out.println(c1 == c2);// 输出结果是true，EntityManager也有缓存
			// 提交事务
			tx.commit();
			System.out.println(c1);
		} catch (Exception e) {
			// 回滚事务
			tx.rollback();
			e.printStackTrace();
		} finally {
			// 释放资源
			em.close();
		}
	}

	// 延迟加载策略的方法：
	/**
	 * 查询一个： 使用延迟加载策略
	 */
	@Test
	public void testLoadOne() {
		// 定义对象
		EntityManager em = null;
		EntityTransaction tx = null;
		try {
			// 获取实体管理对象
			em = JPAUtil.getEntityManager();
			// 获取事务对象
			tx = em.getTransaction();
			// 开启事务
			tx.begin();
			// 执行操作
			Customer c1 = em.getReference(Customer.class, 1L);
			// 提交事务
			tx.commit();
			System.out.println(c1);
		} catch (Exception e) {
			// 回滚事务
			tx.rollback();
			e.printStackTrace();
		} finally {
			// 释放资源
			em.close();
		}
	}

~~~

# 六、Spring Data JPA

Spring Data JPA 是 Spring 基于 ORM 框架、JPA 规范的基础上封装的一套JPA应用框架，可使开发者用极简的代码即可实现对数据库的访问和操作。它提供了包括增删改查等在内的常用功能，且易于扩展！学习并使用 Spring Data JPA 可以极大提高开发效率！

> Spring Data JPA 让我们解脱了DAO层的操作，基本上所有CRUD都可以依赖于它来实现,在实际的工作工程中，推荐使用Spring Data JPA + ORM（如：hibernate）完成操作，这样在切换不同的ORM框架时提供了极大的方便，同时也使数据库层操作更加简单，方便解耦
>
> SpringData Jpa 极大简化了数据库访问层代码。 如何简化的呢？ 使用了SpringDataJpa，我们的dao层中只需要写接口，就自动具有了增删改查、分页查询等方法。

**Spring Data JPA 与JPA以及Hibernate的关系**

JPA是一套规范，内部是有接口和抽象类组成的。hibernate是一套成熟的ORM框架，而且Hibernate实现了JPA规范，所以也可以称hibernate为JPA的一种实现方式，我们使用JPA的API编程，意味着站在更高的角度上看待问题（面向接口编程）

Spring Data JPA是Spring提供的一套对JPA操作更加高级的封装，是在JPA规范下的专门用来进行数据持久化的解决方案

## 6.1 快速入门

### 6.1.1导入坐标

使用Spring Data JPA，需要整合Spring与Spring Data JPA，并且需要提供JPA的服务提供者hibernate，所以需要导入spring相关坐标，hibernate坐标，数据库驱动坐标等

~~~xml
  <properties>
        <spring.version>4.2.4.RELEASE</spring.version>
        <hibernate.version>5.0.7.Final</hibernate.version>
        <slf4j.version>1.6.6</slf4j.version>
        <log4j.version>1.2.12</log4j.version>
        <c3p0.version>0.9.1.2</c3p0.version>
        <mysql.version>5.1.6</mysql.version>
    </properties>

    <dependencies>
        <!-- junit单元测试 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.9</version>
            <scope>test</scope>
        </dependency>
        
        <!-- spring beg -->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>1.6.8</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context-support</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-orm</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>${spring.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>${spring.version}</version>
        </dependency>
        
        <!-- spring end -->

        <!-- hibernate beg -->
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>${hibernate.version}</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-entitymanager</artifactId>
            <version>${hibernate.version}</version>
        </dependency>
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>5.2.1.Final</version>
        </dependency>
        <!-- hibernate end -->

        <!-- c3p0 beg -->
        <dependency>
            <groupId>c3p0</groupId>
            <artifactId>c3p0</artifactId>
            <version>${c3p0.version}</version>
        </dependency>
        <!-- c3p0 end -->

        <!-- log end -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>${log4j.version}</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <!-- log end -->

        
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-jpa</artifactId>
            <version>1.9.0.RELEASE</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.2.4.RELEASE</version>
        </dependency>
        
        <!-- el beg 使用spring data jpa 必须引入 -->
        <dependency>  
            <groupId>javax.el</groupId>  
            <artifactId>javax.el-api</artifactId>  
            <version>2.2.4</version>  
        </dependency>  
          
        <dependency>  
            <groupId>org.glassfish.web</groupId>  
            <artifactId>javax.el</artifactId>  
            <version>2.2.4</version>  
        </dependency> 
        <!-- el end -->
    </dependencies>
~~~

### 6.1.2 整合Spring Data JPA 与Spring

~~~java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:jdbc="http://www.springframework.org/schema/jdbc" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:jpa="http://www.springframework.org/schema/data/jpa" xmlns:task="http://www.springframework.org/schema/task"
	xsi:schemaLocation="
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
		http://www.springframework.org/schema/jdbc http://www.springframework.org/schema/jdbc/spring-jdbc.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
		http://www.springframework.org/schema/data/jpa 
		http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">
	
	<!-- 1.dataSource 配置数据库连接池-->
	<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
		<property name="driverClass" value="com.mysql.jdbc.Driver" />
		<property name="jdbcUrl" value="jdbc:mysql://localhost:3306/jpa" />
		<property name="user" value="root" />
		<property name="password" value="111111" />
	</bean>
	
	<!-- 2.配置entityManagerFactory -->
	<bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="packagesToScan" value="cn.itcast.entity" />
		<property name="persistenceProvider">
			<bean class="org.hibernate.jpa.HibernatePersistenceProvider" />
		</property>
		<!--JPA的供应商适配器-->
		<property name="jpaVendorAdapter">
			<bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
				<property name="generateDdl" value="false" />
				<property name="database" value="MYSQL" />
				<property name="databasePlatform" value="org.hibernate.dialect.MySQLDialect" />
				<property name="showSql" value="true" />
			</bean>
		</property>
		<property name="jpaDialect">
			<bean class="org.springframework.orm.jpa.vendor.HibernateJpaDialect" />
		</property>
	</bean>
    
	
	<!-- 3.事务管理器-->
	<!-- JPA事务管理器  -->
	<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
		<property name="entityManagerFactory" ref="entityManagerFactory" />
	</bean>
	
	<!-- 整合spring data jpa-->
	<jpa:repositories base-package="cn.itcast.dao"
		transaction-manager-ref="transactionManager"
		entity-manager-factory-ref="entityManagerFactory"></jpa:repositories>
		
	<!-- 4.txAdvice-->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="save*" propagation="REQUIRED"/>
			<tx:method name="insert*" propagation="REQUIRED"/>
			<tx:method name="update*" propagation="REQUIRED"/>
			<tx:method name="delete*" propagation="REQUIRED"/>
			<tx:method name="get*" read-only="true"/>
			<tx:method name="find*" read-only="true"/>
			<tx:method name="*" propagation="REQUIRED"/>
		</tx:attributes>
	</tx:advice>
	
	<!-- 5.aop-->
	<aop:config>
		<aop:pointcut id="pointcut" expression="execution(* cn.itcast.service.*.*(..))" />
		<aop:advisor advice-ref="txAdvice" pointcut-ref="pointcut" />
	</aop:config>
	
	<context:component-scan base-package="cn.itcast"></context:component-scan>
		
	<!--组装其它 配置文件-->
	
</beans>
~~~

### 6.1.3 编写映射关系

略 见2.3

### 6.1.4 编写Dao接口

pring Data JPA是spring提供的一款对于数据访问层（Dao层）的框架，使用Spring Data JPA，只需要按照框架的规范提供dao接口，不需要实现类就可以完成数据库的增删改查、分页查询等方法的定义，极大的简化了我们的开发过程。

~~~java
package loserfromlazy;

import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;

import cn.itcast.entity.Customer;

/**
 * JpaRepository<实体类类型，主键类型>：用来完成基本CRUD操作
 * JpaSpecificationExecutor<实体类类型>：用于复杂查询（分页等查询操作）
 */
public interface CustomerDao extends JpaRepository<Customer, Long>, JpaSpecificationExecutor<Customer> {
}

~~~

### 6.1.5 完成CURD

~~~java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:applicationContext.xml")
public class CustomerDaoTest {

    @Autowired
    private CustomerDao customerDao;
    
    /**
     * 保存客户：调用save(obj)方法
     */
    @Test
    public void testSave() {
        Customer c = new Customer();
        c.setCustName("传智播客");
        customerDao.save(c);
    }
    
    /**
     * 修改客户：调用save(obj)方法
     *      对于save方法的解释：如果执行此方法是对象中存在id属性，即为更新操作会先根据id查询，再更新    
     *                      如果执行此方法中对象中不存在id属性，即为保存操作
     *          
     */
    @Test
    public void testUpdate() {
        //根据id查询id为1的客户
        Customer customer = customerDao.findOne(1l);
        //修改客户名称
        customer.setCustName("传智播客顺义校区");
        //更新
        customerDao.save(customer);
    }
    
    /**
     * 根据id删除：调用delete(id)方法
     */
    @Test
    public void testDelete() {
        customerDao.delete(1l);
    }
    
    /**
     * 根据id查询：调用findOne(id)方法
     */
    @Test
    public void testFindById() {
        Customer customer = customerDao.findOne(2l);
        System.out.println(customer);
    }
}
~~~

# 七、Spring Data JPA的查询方式

## 7.1 接口中的方法进行查询

~~~java
JpaRepository的源码：
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {
    List<T> findAll();

    List<T> findAll(Sort var1);

    List<T> findAllById(Iterable<ID> var1);

    <S extends T> List<S> saveAll(Iterable<S> var1);

    void flush();

    <S extends T> S saveAndFlush(S var1);

    void deleteInBatch(Iterable<T> var1);

    void deleteAllInBatch();

    T getOne(ID var1);

    <S extends T> List<S> findAll(Example<S> var1);

    <S extends T> List<S> findAll(Example<S> var1, Sort var2);
}
JpaSpecificationExecutor的源码
public interface JpaSpecificationExecutor<T> {
    Optional<T> findOne(@Nullable Specification<T> var1);

    List<T> findAll(@Nullable Specification<T> var1);

    Page<T> findAll(@Nullable Specification<T> var1, Pageable var2);

    List<T> findAll(@Nullable Specification<T> var1, Sort var2);

    long count(@Nullable Specification<T> var1);
}
~~~

## 7.2 JPQL的方式进行查询

使用Spring Data JPA提供的查询方法已经可以解决大部分的应用场景，但是对于某些业务来说，我们还需要灵活的构造查询条件，这时就可以使用@Query注解，结合JPQL的语句方式完成查询

~~~java
public interface CustomerDao extends JpaRepository<Customer, Long>,JpaSpecificationExecutor<Customer> {    
    //@Query 使用jpql的方式查询。
    @Query(value="from Customer")
    public List<Customer> findAllCustomer();
    
    //@Query 使用jpql的方式查询。?1代表参数的占位符，其中1对应方法中的参数索引
    @Query(value="from Customer where custName = ?1")
    public Customer findCustomer(String custName);
}
~~~

也可以通过使用 @Query 来执行一个更新操作，为此，我们需要在使用 @Query 的同时，用 @Modifying 来将该操作标识为修改查询，这样框架最终会生成一个更新的操作，而非查询

~~~java
@Query(value="update Customer set custName = ?1 where custId = ?2")
@Modifying
public void updateCustomer(String custName,Long custId);
~~~

## 7.3 使用SQL语句进行查询

~~~java
/**
* nativeQuery : 使用本地sql的方式查询
*/
@Query(value="select * from cst_customer",nativeQuery=true)
public void findSql();
~~~

## 7.4 方法命名规则查询

顾名思义，方法命名规则查询就是根据方法的名字，就能创建查询。只需要按照Spring Data JPA提供的方法命名规则定义方法的名称，就可以完成查询工作。Spring Data JPA在程序执行的时候会根据方法名称进行解析，并自动生成查询语句进行查询

按照Spring Data JPA 定义的规则，查询方法以findBy开头，涉及条件查询时，条件的属性用条件关键字连接，要注意的是：条件属性首字母需大写。框架在进行方法名解析时，会先把方法名多余的前缀截取掉，然后对剩下部分进行解析。

~~~java
//方法命名方式查询（根据客户名称查询客户）
public Customer findByCustName(String custName);
~~~

| keyword           | sample                                       | jqpl                                        |
| ----------------- | -------------------------------------------- | ------------------------------------------- |
| And               | findByLastnameAndFirstname                   | ...where x.lastname =?1 and x.firstname =?2 |
| Or                | findByLastnameOrFirstname                    | ...where x.lastname =?1 or x.firstname =?2  |
| Is,Equal          | findByFirstnameIs<br />findByFirstnameEquals | ...where x.firstname =?1                    |
| Between           | findByStartDateBetween                       | ... where x.startDate between ?1 and ?2     |
| LessThan          | findByAgeLessThan                            | … where x.age < ?1                          |
| GreaterThan       | findByAgeGreaterThan                         | … where x.age > ?1                          |
| LessThanEqual     | findByAgeLessThanEqual                       | … where x.age <= ?1                         |
| GreaterThanEqual  | findByAgeGreaterThanEqual                    | … where x.age >= ?1                         |
| IsNull            | findByAgeIsNull                              | …  where x.age is null                      |
| IsNotNull,NotNull | indByAge(Is)NotNull                          | … where x.age not null                      |
| Like              | findByFirstnameLike                          | …  where x.firstname like ?1                |
| OrderBy           | findByAgeOrderByLastnameDesc                 | … where x.age = ?1 order by x.lastname desc |

以上为常用命名规则

# 八、Specifications动态查询

有时我们在查询某个实体的时候，给定的条件是不固定的，这时就需要动态构建相应的查询语句，在Spring Data JPA中可以通过JpaSpecificationExecutor接口查询。相比JPQL,其优势是类型安全,更加的面向对象。

~~~java
import java.util.List;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;

/**
 *	JpaSpecificationExecutor中定义的方法
 **/
 public interface JpaSpecificationExecutor<T> {
   	//根据条件查询一个对象
 	T findOne(Specification<T> spec);	
   	//根据条件查询集合
 	List<T> findAll(Specification<T> spec);
   	//根据条件分页查询
 	Page<T> findAll(Specification<T> spec, Pageable pageable);
   	//排序查询查询
 	List<T> findAll(Specification<T> spec, Sort sort);
   	//统计查询
 	long count(Specification<T> spec);
}
~~~

对于JpaSpecificationExecutor，这个接口基本是围绕着Specification接口来定义的。我们可以简单的理解为，Specification构造的就是查询条件。

## 8.1 使用Specifications完成条件查询

~~~java
	//依赖注入customerDao
	@Autowired
	private CustomerDao customerDao;	
	@Test
	public void testSpecifications() {
      	//使用匿名内部类的方式，创建一个Specification的实现类，并实现toPredicate方法
		Specification <Customer> spec = new Specification<Customer>() {
			public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
				//cb:构建查询，添加查询方式   like：模糊匹配
				//root：从实体Customer对象中按照custName属性进行查询
				return cb.like(root.get("custName").as(String.class), "张%");
			}
		};
		Customer customer = customerDao.findOne(spec);
		System.out.println(customer);
	}
~~~

## 8.2 基于Specifications的分页查询

~~~java
    @Test
	public void testPage() {
		//构造查询条件
		Specification<Customer> spec = new Specification<Customer>() {
			public Predicate toPredicate(Root<Customer> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
				return cb.like(root.get("custName").as(String.class), "传智%");
			}
		};
		
		/**
		 * 构造分页参数
		 * 		Pageable : 接口
		 * 			PageRequest实现了Pageable接口，调用构造方法的形式构造
		 * 				第一个参数：页码（从0开始）
		 * 				第二个参数：每页查询条数
		 */
		Pageable pageable = new PageRequest(0, 5);
		
		/**
		 * 分页查询，封装为Spring Data Jpa 内部的page bean
		 * 		此重载的findAll方法为分页方法需要两个参数
		 * 			第一个参数：查询条件Specification
		 * 			第二个参数：分页参数
		 */
		Page<Customer> page = customerDao.findAll(spec,pageable);
		
	}
~~~

## 8.3 方法对应关系

| 方法名称                 | sql                  |
| ------------------------ | -------------------- |
| equal                    | filed =value         |
| gt(greaterThan)          | filed > value        |
| lt(lessThan)             | filed<value          |
| ge(greaterThanOrEqualTo) | filed >= value       |
| le（ lessThanOrEqualTo） | filed  <= value      |
| notEqule                 | filed  != value      |
| like                     | filed like value     |
| notlike                  | filed not like value |


