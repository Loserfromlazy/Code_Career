# maven教程

## 1.简介

Maven 是一个项目管理工具，可以对 Java 项目进行构建、依赖管理。

Maven 也可被用于构建和管理各种项目，例如 C#，Ruby，Scala 和其他语言编写的项目。Maven 曾是 Jakarta 项目的子项目，现为由 Apache 软件基金会主持的独立 Apache 项目。

#### Maven 功能

Maven 能够帮助开发者完成以下工作：

- 构建
- 文档生成
- 报告
- 依赖
- SCMs
- 发布
- 分发
- 邮件列表

#### 约定配置

Maven 提倡使用一个共同的标准目录结构，Maven 使用约定优于配置的原则，大家尽可能的遵守这样的目录结构。如下所示：

| 目录                               | 目的                                                         |
| :--------------------------------- | :----------------------------------------------------------- |
| ${basedir}                         | 存放pom.xml和所有的子目录                                    |
| ${basedir}/src/main/java           | 项目的java源代码                                             |
| ${basedir}/src/main/resources      | 项目的资源，比如说property文件，springmvc.xml                |
| ${basedir}/src/test/java           | 项目的测试类，比如说Junit代码                                |
| ${basedir}/src/test/resources      | 测试用的资源                                                 |
| ${basedir}/src/main/webapp/WEB-INF | web应用文件目录，web项目的信息，比如存放web.xml、本地图片、jsp视图页面 |
| ${basedir}/target                  | 打包输出目录                                                 |
| ${basedir}/target/classes          | 编译输出目录                                                 |
| ${basedir}/target/test-classes     | 测试编译输出目录                                             |
| Test.java                          | Maven只会自动运行符合该命名规则的测试类                      |
| ~/.m2/repository                   | Maven默认的本地仓库目录位置                                  |

#### Maven 环境配置

Maven 是一个基于 Java 的工具，所以要做的第一件事情就是安装 JDK。

#### 设置 Maven 环境变量

添加环境变量 MAVEN_HOME：

| 系统    | 配置                                                         |
| :------ | :----------------------------------------------------------- |
| Windows | 右键 "计算机"，选择 "属性"，之后点击 "高级系统设置"，点击"环境变量"，来设置环境变量，有以下系统变量需要配置：新建系统变量 **MAVEN_HOME**，变量值：**E:\Maven\apache-maven-3.3.9**![img](https://www.runoob.com/wp-content/uploads/2018/09/1536057115-1481-20151218175411912-170761788.png)编辑系统变量 **Path**，添加变量值：**;%MAVEN_HOME%\bin**![img](https://www.runoob.com/wp-content/uploads/2018/09/1536057115-7470-20151218175417006-1644078150.png)**注意：**注意多个值之间需要有分号隔开，然后点击确定。 |

## 2.POM

POM( Project Object Model，项目对象模型 ) 是 Maven 工程的基本工作单元，是一个XML文件，包含了项目的基本信息，用于描述项目如何构建，声明项目依赖，等等。

执行任务或目标时，Maven 会在当前目录中查找 POM。它读取 POM，获取所需的配置信息，然后执行目标。

POM 中可以指定以下配置：

- 项目依赖
- 插件
- 执行目标
- 项目构建 profile
- 项目版本
- 项目开发者列表
- 相关邮件列表信息

所有 POM 文件都需要 project 元素和三个必需字段：groupId，artifactId，version。

| 节点         | 描述                                                         |
| :----------- | :----------------------------------------------------------- |
| project      | 工程的根标签。                                               |
| modelVersion | 模型版本需要设置为 4.0。                                     |
| groupId      | 这是工程组的标识。它在一个组织或者项目中通常是唯一的。例如，一个银行组织 com.companyname.project-group 拥有所有的和银行相关的项目。 |
| artifactId   | 这是工程的标识。它通常是工程的名称。例如，消费者银行。groupId 和 artifactId 一起定义了 artifact 在仓库中的位置。 |
| version      | 这是工程的版本号。在 artifact 的仓库中，它用来区分不同的版本。例如：`com.company.bank:consumer-banking:1.0 com.company.bank:consumer-banking:1.1` |

~~~xml
<project xmlns = "http://maven.apache.org/POM/4.0.0"
    xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation = "http://maven.apache.org/POM/4.0.0
    http://maven.apache.org/xsd/maven-4.0.0.xsd">
 
    <!-- 模型版本 -->
    <modelVersion>4.0.0</modelVersion>
    <!-- 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.companyname.project-group，maven会将该项目打成的jar包放本地路径：/com/companyname/project-group -->
    <groupId>com.companyname.project-group</groupId>
 
    <!-- 项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的 -->
    <artifactId>project</artifactId>
 
    <!-- 版本号 -->
    <version>1.0</version>
</project>
~~~



## 3.构建生命周期及配置文件

#### 构建生命周期

一个典型的 Maven 构建（build）生命周期是由以下几个阶段的序列组成的：

![img](https://www.runoob.com/wp-content/uploads/2018/09/7642256-c967b2c1faeba9ce.png)

| 阶段          | 处理     | 描述                                                     |
| :------------ | :------- | :------------------------------------------------------- |
| 验证 validate | 验证项目 | 验证项目是否正确且所有必须信息是可用的                   |
| 编译 compile  | 执行编译 | 源代码编译在此阶段完成                                   |
| 测试 Test     | 测试     | 使用适当的单元测试框架（例如JUnit）运行测试。            |
| 包装 package  | 打包     | 创建JAR/WAR包如在 pom.xml 中定义提及的包                 |
| 检查 verify   | 检查     | 对集成测试的结果进行检查，以保证质量达标                 |
| 安装 install  | 安装     | 安装打包的项目到本地仓库，以供其他项目使用               |
| 部署 deploy   | 部署     | 拷贝最终的工程包到远程仓库中，以共享给其他开发人员和工程 |

为了完成 default 生命周期，这些阶段（包括其他未在上面罗列的生命周期阶段）将被按顺序地执行。

Maven 有以下三个标准的生命周期：

- **clean**：项目清理的处理
- **default(或 build)**：项目部署的处理
- **site**：项目站点文档创建的处理

#### 构建配置文件

构建配置文件是一系列的配置项的值，可以用来设置或者覆盖 Maven 构建默认值。

使用构建配置文件，你可以为不同的环境，比如说生产环境（Production）和开发（Development）环境，定制构建方式。

配置文件在 pom.xml 文件中使用 activeProfiles 或者 profiles 元素指定，并且可以通过各种方式触发。配置文件在构建时修改 POM，并且用来给参数设定不同的目标环境（比如说，开发（Development）、测试（Testing）和生产环境（Production）中数据库服务器的地址）。

------

##### 构建配置文件的类型

构建配置文件大体上有三种类型:

| 类型                  | 在哪定义                                                     |
| :-------------------- | :----------------------------------------------------------- |
| 项目级（Per Project） | 定义在项目的POM文件pom.xml中                                 |
| 用户级 （Per User）   | 定义在Maven的设置xml文件中 (%USER_HOME%/.m2/settings.xml)    |
| 全局（Global）        | 定义在 Maven 全局的设置 xml 文件中 (%M2_HOME%/conf/settings.xml) |

## 4.maven仓库

在 Maven 的术语中，仓库是一个位置（place）。

Maven 仓库是项目中依赖的第三方库，这个库所在的位置叫做仓库。

在 Maven 中，任何一个依赖、插件或者项目构建的输出，都可以称之为构件。

Maven 仓库能帮助我们管理构件（主要是JAR），它就是放置所有JAR文件（WAR，ZIP，POM等等）的地方。

Maven 仓库有三种类型：

- 本地（local）
- 中央（central）
- 远程（remote）

#### 本地仓库

Maven 的本地仓库，在安装 Maven 后并不会创建，它是在第一次执行 maven 命令的时候才被创建。

运行 Maven 的时候，Maven 所需要的任何构件都是直接从本地仓库获取的。如果本地仓库没有，它会首先尝试从远程仓库下载构件至本地仓库，然后再使用本地仓库的构件。

默认情况下，不管Linux还是 Windows，每个用户在自己的用户目录下都有一个路径名为 .m2/respository/ 的仓库目录。

Maven 本地仓库默认被创建在 %USER_HOME% 目录下。要修改默认位置，在 %M2_HOME%\conf 目录中的 Maven 的 settings.xml 文件中定义另一个路径。

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0     http://maven.apache.org/xsd/settings-1.0.0.xsd">       <localRepository>C:/MyLocalRepository</localRepository> </settings>

当你运行 Maven 命令，Maven 将下载依赖的文件到你指定的路径中。

#### 中央仓库

Maven 中央仓库是由 Maven 社区提供的仓库，其中包含了大量常用的库。

中央仓库包含了绝大多数流行的开源Java构件，以及源码、作者信息、SCM、信息、许可证信息等。一般来说，简单的Java项目依赖的构件都可以在这里下载到。

中央仓库的关键概念：

- 这个仓库由 Maven 社区管理。
- 不需要配置。
- 需要通过网络才能访问。

要浏览中央仓库的内容使用这个仓库，开发人员可以搜索所有可以获取的代码库。

#### 远程仓库

如果 Maven 在中央仓库中也找不到依赖的文件，它会停止构建过程并输出错误信息到控制台。为避免这种情况，Maven 提供了远程仓库的概念，它是开发人员自己定制仓库，包含了所需要的代码库或者其他工程中用到的 jar 文件。

#### Maven 依赖搜索顺序

当我们执行 Maven 构建命令时，Maven 开始按照以下顺序查找依赖的库：

- **步骤 1** － 在本地仓库中搜索，如果找不到，执行步骤 2，如果找到了则执行其他操作。
- **步骤 2** － 在中央仓库中搜索，如果找不到，并且有一个或多个远程仓库已经设置，则执行步骤 4，如果找到了则下载到本地仓库中以备将来引用。
- **步骤 3** － 如果远程仓库没有被设置，Maven 将简单的停滞处理并抛出错误（无法找到依赖的文件）。
- **步骤 4** － 在一个或多个远程仓库中搜索依赖的文件，如果找到则下载到本地仓库以备将来引用，否则 Maven 将停止处理并抛出错误（无法找到依赖的文件）。

#### Maven 阿里云(Aliyun)仓库

Maven 仓库默认在国外， 国内使用难免很慢，我们可以更换为阿里云的仓库。

第一步:修改 maven 根目录下的 conf 文件夹中的 setting.xml 文件，在 mirrors 节点上，添加内容如下：

~~~xml
<mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>        
    </mirror>
</mirrors>
~~~



![img](https://www.runoob.com/wp-content/uploads/2018/09/455FA277-3216-4BA5-94D3-0F650C763D6F.png)

第二步: pom.xml文件里添加：

~~~xml
<repositories>  
        <repository>  
            <id>alimaven</id>  
            <name>aliyun maven</name>  
            <url>http://maven.aliyun.com/nexus/content/groups/public/</url>  
            <releases>  
                <enabled>true</enabled>  
            </releases>  
            <snapshots>  
                <enabled>false</enabled>  
            </snapshots>  
        </repository>  
</repositories>
~~~





## 5.maven插件

## 6.常用maven命令

1. 创建Maven的普通java项目： 
      mvn archetype:create 
      -DgroupId=packageName 
      -DartifactId=projectName  

2. 创建Maven的Web项目：   
       mvn archetype:create 
       -DgroupId=packageName    
       -DartifactId=webappName 
       -DarchetypeArtifactId=maven-archetype-webapp    

3. 编译源代码： mvn compile 

4. 编译测试代码：mvn test-compile    

5. 运行测试：mvn test   

6. 产生site：mvn site   

7. 打包：mvn package   

8. 在本地Repository中安装jar：mvn install 

9. 清除产生的项目：mvn clean   

10. 生成eclipse项目：mvn eclipse:eclipse  

11. 生成idea项目：mvn idea:idea  

12. 组合使用goal命令，如只打包不测试：mvn -Dtest package   

13. 编译测试的内容：mvn test-compile  

14. 只打jar包: mvn jar:jar  

15. 只测试而不编译，也不测试编译：mvn test -skipping compile -skipping test-compile 
          ( -skipping 的灵活运用，当然也可以用于其他组合命令)  

16. 清除eclipse的一些系统设置:mvn eclipse:clean 



## 7.maven依赖管理

#### 依赖的传递

传递性依赖是Maven2.0的新特性。假设你的项目依赖于一个库，而这个库又依赖于其他库。你不必自己去找出所有这些依赖，你只需要加上你直接依赖的库，Maven会隐式的把这些库间接依赖的库也加入到你的项目中。这个特性是靠解析从远程仓库中获取的依赖库的项目文件实现的。一般的，这些项目的所有依赖都会加入到项目中，或者从父项目继承，或者通过传递性依赖。
传递性依赖的嵌套深度没有任何限制，只是在出现循环依赖时会报错。

#### exclude排除jar包冲突

eg:

~~~xml
<dependency>
    <groupId>org.apache.struts</groupId>
    <artifactId>struts2-spring-plugin</artifactId>
    <version>2.3.24</version>
    <!--排除spring-beans -->
    <exclusions>
    	<exclusion>
        	<groupId>org.springfreamwork</groupId>
    <artifactId>spring-beans</artifactId>
        </exclusion>
    </exclusions>
</dependency>
~~~

#### 第一声明者优先（依赖调解原则）

在pom文件中定义，先声明的依赖为准

#### 路径近者优先（依赖调解原则）

在本工程中的pom加入spring-beans-4.2.4的依赖，根据路径近者优先原则，系统将导入spring-beans-4.2.4

#### 版本锁定

eg:

~~~xml
<dependencyManagement>
	<dependencies>
    	<dependency>
    		<groupId>org.springfreamwork</groupId>
            <artifactId>spring-beans</artifactId>
            <vension>4.2.4.RELEASE</vension>
 		</dependency>
    </dependencies>
</dependencyManagement>
~~~

在工程中锁定依赖的版本并不代表在工程中添加了依赖，如果工程需要添加锁定版本的依赖则需单独添加<dependencies>标签

如下：

~~~xml
<dependencies>
	<dependency>
    	<groupId>org.springfreamwork</groupId>
        <artifactId>spring-beans</artifactId>
    </dependency>
</dependencies>
~~~

此处的版本并没有添加依赖，原因是已在<dependencyManagement>中锁定了版本，所以在此处不需要再指定版本。

