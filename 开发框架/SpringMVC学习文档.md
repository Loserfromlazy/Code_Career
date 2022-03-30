# SpringMVC学习文档

springMVC就是类似与Struts2的mvc框架，始于SpringFrameWork的后续产品

为什么要学习SpringMVC？

> | springMVC    |                      与                      | Struts2                                  | 区别                                                         |
> | ------------ | :------------------------------------------: | ---------------------------------------- | ------------------------------------------------------------ |
> | 对比项目     |                  SpringMVC                   | Struts2                                  | 优势                                                         |
> | 国内市场情况 |  有大量用户，一般新项目启动都选择springmvc   | 有部分老用户，一直在使用                 | 国内情况springmvc的shiyonglv已经超过了struts2                |
> | 框架入口     |                 基于servlet                  | 基于filter                               | 只是配置方式不同                                             |
> | 框架设计思想 | 控制器基于方法级别的拦截，处理器设计为单实例 | 控制器基于类级别拦截，处理器设计为多实例 | 由于设计本身的原因，造成了struts2只能设计为多实例模式，相比于springmvc设计为单实例模式，struts2会消耗更多服务器内存 |
> | 参数传递     |               通过方法入参传递               | 通过类的成员变量传递                     | struts2通过成员变量传递参数，导致参数线程不安全，有可能引发并发的问题 |
> | 与spring整合 |    与spring同一家公司可以与spring无缝整合    | 需要整合包                               | springmvc与spring整合更轻松                                  |

## 一.概述

### 1.1SpringMVC入门

#### 1.1.1配置核心控制器web.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://java.sun.com/xml/ns/javaee"
xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
id="WebApp_ID" version="2.5">
<!-- 配置spring mvc的核心控制器-->
<servlet>
<servlet-name>SpringMVCDispatcherServlet</servlet-name>
<servlet-class>
org.springframework.web.servlet.DispatcherServlet
</servlet-class>
<!-- 配置初始化参数，用于读取SpringMVC的配置文件-->
<init-param>
<param-name>contextConfigLocation</param-name>
<param-value>classpath:SpringMVC.xml</param-value>
</init-param>
<!-- 配置servlet的对象的创建时间点：应用加载时创建。
取值只能是非0正整数，表示启动顺序-->
<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
<servlet-name>SpringMVCDispatcherServlet</servlet-name>
<url-pattern>/</url-pattern>
</servlet-mapping>
</web-app>
~~~

#### 1.1.2配置springmvc配置文件springmvc.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:mvc="http://www.springframework.org/schema/mvc"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://www.springframework.org/schema/mvc
http://www.springframework.org/schema/mvc/spring-mvc.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context.xsd">
<!-- 配置创建spring容器要扫描的包-->
<context:component-scan base-package="com.itheima"></context:component-scan>
<!-- 配置视图解析器-->
<bean id="internalResourceViewResolver"
class="org.springframework.web.servlet.view.InternalResourceViewResolver">
<property name="prefix" value="/WEB-INF/pages/"></property>
<property name="suffix" value=".jsp"></property>
</bean>
    <!--开启springmvc框架注解的支持 -->
    <mvc:annotation-friven/>
</beans>
~~~

#### 1.1.3编写控制器并使用注解配置

~~~java
@Controller("helloController")
public class HelloController {
@RequestMapping("/hello")
public String sayHello() {
System.out.println("HelloController的sayHello 方法执行了。。。。");
return "success";
}
}
~~~



### 1.2SpringMVC架构

#### 1.2.1框架默认加载组件

![img](https://img-blog.csdnimg.cn/20190408151658886.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Zsb2F0aW5nX2RyZWFtaW5n,size_16,color_FFFFFF,t_70)

处理器映射器与处理器适配器

> 处理器映射器
>
> 从spring3.1开始，废除了DefaultAnnotationHandlerMapping的使用，推荐使用RequestMappingHandlerMapping完成注解式处理器映射
>
> < bean> class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>
>
> 处理器适配器
>
> 从spring3.1开始，废除了AnnotationMethodHandlerMapping的使用，推荐使用RequestMappingHandlerAdapter完成注解式处理器适配
>
> <bean class"org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter "/>
>
> <! --配置注解驱动相当于同时使用最新的处理器映射器和处理器适配器 ，对json数据响应提供支持 -->
>
> <mvc:annotation-driven />  

视图解析器

> mav.setViewName("itemList");

在springmvc.xml中

> <!-- 配置视图解析器 -->
>
> ~~~xml
> 	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
> 		<property name="prefix" value="/WEB-INF/jsp/"></property><!-- 前缀 -->
> 		<property name="suffix" value=".jsp"></property><!-- 后缀 -->
> 	</bean>
> ~~~

### 1.3概述

**Spring web MVC**框架提供了MVC(模型 - 视图 - 控制器)架构和用于开发灵活和松散耦合的Web应用程序的组件。 **MVC**模式导致应用程序的不同方面(输入逻辑，业务逻辑和UI逻辑)分离，同时提供这些元素之间的松散耦合。

- **模型(Model)**封装了应用程序数据，通常它们将由`POJO`类组成。
- **视图(View)**负责渲染模型数据，一般来说它生成客户端浏览器可以解释HTML输出。
- **控制器(Controller)**负责处理用户请求并构建适当的模型，并将其传递给视图进行渲染。



### 1.4优点

1、清晰的角色划分：
前端控制器（DispatcherServlet）
请求到处理器映射（HandlerMapping）
处理器适配器（HandlerAdapter）
视图解析器（ViewResolver）
处理器或页面控制器（Controller）
验证器（Validator）
命令对象（Command 请求参数绑定到的对象就叫命令对象）
表单对象（Form Object 提供给表单展示和提交到的对象就叫表单对象）。
2、分工明确，而且扩展点相当灵活，可以很容易扩展，虽然几乎不需要。
3、由于命令对象就是一个POJO，无需继承框架特定API，可以使用命令对象直接作为业务对象。
4、和Spring 其他框架无缝集成，是其它Web框架所不具备的。
5、可适配，通过HandlerAdapter可以支持任意的类作为处理器。
6、可定制性，HandlerMapping、ViewResolver等能够非常简单的定制。
7、功能强大的数据验证、格式化、绑定机制。
8、利用Spring提供的Mock对象能够非常简单的进行Web层单元测试。
9、本地化、主题的解析的支持，使我们更容易进行国际化和主题的切换。
10、强大的JSP标签库，使JSP编写更容易。
………………还有比如RESTful风格的支持、简单的文件上传、约定大于配置的契约式编程支持、基于注解的零配
置支持等等。

### 1.5.RequestMapping注解

#### 1.5.1作用

用于建立请求URL和处理请求方法之间的对应关系。

出现的位置

类上：请求URL一级目录

方法上：请求URL二级目录

#### 1.5.2属性

value：用于指定请求的URL 与path属性相同（若只有value属性 value可省略）

method：用于指定请求的方式

params：用于指定限制请求参数的条件。它支持简单的表达式。要求请求参数的key和value必须和
配置的一模一样。
	例如：
	params = {"accountName"}，表示请求参数必须有accountName
	params = {"moeny!100"}，表示请求参数中money不能是100。

headers：用于指定限制请求消息头的条件。

## 二.请求参数的绑定

### 2.1绑定机制

> 我们都知道，表单中请求参数都是基于key=value的。
> SpringMVC绑定请求参数的过程是通过把表单提交请求参数，作为控制器中方法参数进行绑定的。
> 例如：
> < a href="account/findAccount?accountId=10">查询账户</a>中请求参数是：
> accountId=10
>
> @RequestMapping("/findAccount")
> public String findAccount(Integer accountId) {
> System.out.println("查询了账户。。。。"+accountId);
> return "success";

### 2.2支持数据类型

基本类型参数：
	**包括基本类型和String类型**
POJO类型参数：
	**包括实体类，以及关联的实体类**
数组和集合类型参数：
	**包括List结构和Map结构的集合（包括数组）**

Model类型：

​	**用于数据的返回详见4.1.1**

SpringMVC绑定请求参数是自动实现的，但是要想使用，必须遵循以下使用要求。

如果是基本类型或者String类型：
	**要求我们的参数名称必须和控制器中方法的形参名称保持一致。(严格区分大小写)**
如果是POJO类型，或者它的关联对象：
	**要求表单中参数名称和POJO类的属性名称保持一致。并且控制器方法的参数类型是POJO类型。若是POJO含有包装对象则通过 . 来传递属性**

如果是集合类型,有两种方式：

​	**第一种：**
要求集合类型的请求参数必须在POJO中。在表单中请求参数名称要和POJO中集合属性名称相同。
给List集合中的元素赋值，使用下标。
给Map集合中的元素赋值，使用键值对。

~~~java
public class User implements Serializable {
private String username;
private String password;
private Integer age;
private List<Account> accounts;
private Map<String,Account> accountMap;

~~~

~~~xml
<form action="account/updateAccount" method="post">
用户名称：<input type="text" name="username" ><br/>
用户密码：<input type="password" name="password" ><br/>
用户年龄：<input type="text" name="age" ><br/>
账户1名称：<input type="text" name="accounts[0].name" ><br/>
账户1金额：<input type="text" name="accounts[0].money" ><br/>
账户2名称：<input type="text" name="accounts[1].name" ><br/>
账户2金额：<input type="text" name="accounts[1].money" ><br/>
账户3名称：<input type="text" name="accountMap['one'].name" ><br/>
账户3金额：<input type="text" name="accountMap['one'].money" ><br/>
账户4名称：<input type="text" name="accountMap['two'].name" ><br/>
账户4金额：<input type="text" name="accountMap['two'].money" ><br/>
<input type="submit" value="保存">
</form>
~~~





​	**第二种：**
接收的请求参数是json格式数据。需要借助一个注解实现。

### 2.3请求参数乱码

在web.xml中配置过滤器

~~~xml
<!-- 配置springMVC编码过滤器-->
<filter>
<filter-name>CharacterEncodingFilter</filter-name>
<filter-class>
org.springframework.web.filter.CharacterEncodingFilter
</filter-class>
    <!-- 设置过滤器中的属性值-->
<init-param>
<param-name>encoding</param-name>
<param-value>UTF-8</param-value>
</init-param>
<!-- 启动过滤器-->
<init-param>
<param-name>forceEncoding</param-name>
<param-value>true</param-value>
</init-param>
</filter>
<!-- 过滤所有请求-->
<filter-mapping>
<filter-name>CharacterEncodingFilter</filter-name>
<url-pattern>/*</url-pattern>
</filter-mapping>

在springmvc的配置文件中可以配置，静态资源不过滤：
<!-- location表示路径，mapping表示文件，**表示该目录下的文件以及子目录的文件-->
<mvc:resources location="/css/" mapping="/css/**"/>
<mvc:resources location="/images/" mapping="/images/**"/>
<mvc:resources location="/scripts/" mapping="/javascript/**"/>

~~~

### 2.4自定义类型转换器

Springmvc在提交请求时所有的参数都是String类型 ，只不过springmvc在封装的时候会对大部分基本数据类型进行数据类型转换。

但使用date时提交2020/01/01 会帮助转换，但提交2020-01-01时springmvc不支持这种格式的日期转换所以需要自定义类型转换器

第一步：定义一个类，实现Converter接口，该接口有两个泛型。

~~~java
public class StringToDateConverter implements Converter<String, Date> {
/**
* 用于把String类型转成日期类型
*/
@Override
public Date convert(String source) {
DateFormat format = null;
try {
if(StringUtils.isEmpty(source)) {
throw new NullPointerException("请输入要转换的日期");
}
format = new SimpleDateFormat("yyyy-MM-dd");
Date date = format.parse(source);
return date;
} catch (Exception e) {
throw new RuntimeException("输入日期有误");
}
}
}

~~~

第二步：在spring配置文件中配置类型转换器。

~~~xml
<!-- 配置类型转换器工厂-->
<bean id="converterService"
class="org.springframework.context.support.ConversionServiceFactoryBean">
<!-- 给工厂注入一个新的类型转换器-->
<property name="converters">
<array>
<!-- 配置自定义类型转换器-->
<bean class="com.itheima.web.converter.StringToDateConverter"></bean>
</array>
</property>
</bean>

~~~

第三步：在annotation-driven标签中引用配置的类型转换服务

~~~xml
<mvc:annotation-driven
conversion-service="converterService"></mvc:annotation-driven>

~~~

## 三.常用注解

### 3.1RequestParam

作用：
把请求中指定名称的参数给控制器中的形参赋值。
属性：
value：请求参数中的名称。
required：请求参数中是否必须提供此参数。默认值：true。表示必须提供，如果不提供将报错。

示例：

~~~java
<!-- requestParams注解的使用-->
<a href="springmvc/useRequestParam?name=test">requestParam注解</a>


@RequestMapping("/useRequestParam")
public String useRequestParam(@RequestParam("name")String username,
@RequestParam(value="age",required=false)Integer age){
System.out.println(username+","+age);
return "success";
}
~~~

### 3.2RequestBody

作用：
用于获取请求体内容。直接使用得到是key=value&key=value...结构的数据。
get请求方式不适用。
属性：
required：是否必须有请求体。默认值是:true。当取值为true时,get请求方式会报错。如果取值
为false，get请求得到是null。

示例

~~~java
post请求jsp代码：
<!-- request body注解-->
<form action="springmvc/useRequestBody" method="post">
用户名称：<input type="text" name="username" ><br/>
用户密码：<input type="password" name="password" ><br/>
用户年龄：<input type="text" name="age" ><br/>
<input type="submit" value="保存">
</form>
get请求jsp代码：
<a href="springmvc/useRequestBody?body=test">requestBody注解get请求</a>
控制器代码：
@RequestMapping("/useRequestBody")
public String useRequestBody(@RequestBody(required=false) String body){
System.out.println(body);
return "success";
}

~~~

**结果：post打印数据get打印null**

### 3.3PathVaribale

作用：
用于绑定url中的占位符。例如：请求url中/delete/{id}，这个{id}就是url占位符。
url支持占位符是spring3.0之后加入的。是springmvc支持rest风格URL的一个重要标志。
属性：
value：用于指定url中占位符名称。
required：是否必须提供占位符。

示例

~~~java
<!-- PathVariable注解-->
<a href="springmvc/usePathVariable/100">pathVariable注解</a>

@RequestMapping("/usePathVariable/{id}")
public String usePathVariable(@PathVariable("id") Integer id){
System.out.println(id);
return "success";
}
~~~

### 3.4CookieValue

作用：
用于把指定cookie名称的值传入控制器方法参数。
属性：
value：指定cookie的名称。
required：是否必须有此cookie。

### 3.5ModelAttribute

作用：
该注解是SpringMVC4.3版本以后新加入的。它可以用于修饰方法和参数。
出现在方法上，表示当前方法会在控制器的方法执行之前，先执行。它可以修饰没有返回值的方法，也可
以修饰有具体返回值的方法。
出现在参数上，获取指定的数据给参数赋值。
属性：
value：用于获取数据的key。key可以是POJO的属性名称，也可以是map结构的key。
应用场景：
当表单提交数据不是完整的实体类数据时，保证没有提交数据的字段使用数据库对象原来的数据。
例如：
我们在编辑一个用户时，用户有一个创建信息字段，该字段的值是不允许被修改的。在提交表单数
据是肯定没有此字段的内容，一旦更新会把该字段内容置为null，此时就可以使用此注解解决问题。

## 四.响应数据和结果试图

### 4.1返回值分类

#### 4.1.1字符串

~~~java
controller方法返回字符串可以指定逻辑视图名，通过视图解析器解析为物理视图地址。
//指定逻辑视图名，经过视图解析器解析为jsp物理路径：/WEB-INF/pages/success.jsp
@RequestMapping("/testReturnString")
public String testReturnString(Model model) {
System.out.println("AccountController的testReturnString 方法执行了。。。。");
User user =new User();
    model.addAttribute("user",user);
return "success";
}

success.jsp
<html>
....
<body>
${user.name}   //如此即可取值
</body>
</html>
~~~

#### 4.2.2void

如果没有return 会自动跳到此路径的jsp 比如 testReturnVoid.jsp

~~~java
@RequestMapping("/testReturnVoid")
public void testReturnVoid(HttpServletRequest request,HttpServletResponse response)
throws Exception {
}
//此时若想跳转
//1：使用request转发
request.getRequestDispatcher("/WEB-INF/pages/success.jsp").forward(request,
response);
//2：使用response页面重定向
response.sendRedirect("testRetrunString");
//3:使用response指定响应结果
response.setCharacterEncoding("utf-8");
response.setContentType("application/json;charset=utf-8");
response.getWriter().write("json串");

~~~

#### 4.2.3ModelAndView

ModelAndView是SpringMVC为我们提供的一个对象，该对象也可以用作控制器方法的返回值。总体上与4.1.1中的类似。

该对象有两个方法：

addObject()	添加模型到对象中 与model类似 在jsp中用el表达式即可获取  ${attributeName}

setViewName()	用于设置逻辑视图名称

示例

~~~java
@RequestMapping("/testReturnModelAndView")
public ModelAndView testReturnModelAndView() {
ModelAndView mv = new ModelAndView();
mv.addObject("username", "张三");
mv.setViewName("success");
return mv;
}
    

~~~

**我们在页面上上获取使用的是requestScope.username取的，所以返回ModelAndView类型时，浏**
**览器跳转只能是请求转发。**

### 4.2转发和重定向

### 4.3ResponseBody 响应json数据

作用：
该注解用于将Controller的方法返回的对象，通过HttpMessageConverter接口转换为指定格式的
数据如：json,xml等，通过Response响应给客户端

~~~jsp
jsp中的代码：
<script type="text/javascript"
src="${pageContext.request.contextPath}/js/jquery.min.js"></script>
<script type="text/javascript">
$(function(){
$("#testJson").click(function(){
$.ajax({
type:"post",
url:"${pageContext.request.contextPath}/testResponseJson",
contentType:"application/json;charset=utf-8",
data:'{"id":1,"name":"test","money":999.0}',
dataType:"json",
success:function(data){
alert(data);
}
});
});
})
</script>
<!-- 测试异步请求-->
<input type="button" value="测试ajax请求json和响应json" id="testJson"/>

~~~

~~~java
控制器中的代码：
/**
* 响应json数据的控制器
* @author 黑马程序员
* @Company http://www.ithiema.com
* @Version 1.0
*/
@Controller("jsonController")
public class JsonController {
/**
* 测试响应json数据
*/
@RequestMapping("/testResponseJson")
@ResponseBody
public  Account testResponseJson(@RequestBody Account account) {
System.out.println("异步请求："+account);
return account;
}
}
~~~

## 五.SpringMVC实现文件上传

### 5.1原理分析

#### 5.1.1前提

A form表单的enctype取值必须是：multipart/form-data
(默认值是:application/x-www-form-urlencoded)
enctype:是表单请求正文的类型
B method属性取值必须是Post
C 提供一个文件选择域< input type="file" />

一定要确保在您的 classpath 中有最新版本的 **commons-fileupload.x.x.jar** 文件。

一定要确保在您的 classpath 中有最新版本的 **commons-io-x.x.jar** 文件。

#### 5.1.2示例

jsp代码

~~~jsp
<form method="post" action="/TomcatTest/UploadServlet" enctype="multipart/form-data">
    选择一个文件:
    <input type="file" name="uploadFile" />
    <br/><br/>
    <input type="submit" value="上传" />
</form>
~~~

servlet代码

~~~java
@WebServlet("/UploadServlet")
public class UploadServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    // 上传文件存储目录
    private static final String UPLOAD_DIRECTORY = "upload";
    // 上传配置
    private static final int MEMORY_THRESHOLD   = 1024 * 1024 * 3;  // 3MB
    private static final int MAX_FILE_SIZE      = 1024 * 1024 * 40; // 40MB
    private static final int MAX_REQUEST_SIZE   = 1024 * 1024 * 50; // 50MB
    /**
     * 上传数据及保存文件
     */
    protected void doPost(HttpServletRequest request,
        HttpServletResponse response) throws ServletException, IOException {
        // 检测是否为多媒体上传
        if (!ServletFileUpload.isMultipartContent(request)) {
            // 如果不是则停止
            PrintWriter writer = response.getWriter();
            writer.println("Error: 表单必须包含 enctype=multipart/form-data");
            writer.flush();
            return;
        }
        // 配置上传参数
        DiskFileItemFactory factory = new DiskFileItemFactory();
        // 设置内存临界值 - 超过后将产生临时文件并存储于临时目录中
        factory.setSizeThreshold(MEMORY_THRESHOLD);
        // 设置临时存储目录
        factory.setRepository(new File(System.getProperty("java.io.tmpdir")));
        ServletFileUpload upload = new ServletFileUpload(factory);
        // 设置最大文件上传值
        upload.setFileSizeMax(MAX_FILE_SIZE);
        // 设置最大请求值 (包含文件和表单数据)
        upload.setSizeMax(MAX_REQUEST_SIZE);
        // 中文处理
        upload.setHeaderEncoding("UTF-8"); 
        // 构造临时路径来存储上传的文件
        // 这个路径相对当前应用的目录
        String uploadPath = getServletContext().getRealPath("/") + File.separator + UPLOAD_DIRECTORY;
        // 如果目录不存在则创建
        File uploadDir = new File(uploadPath);
        if (!uploadDir.exists()) {
            uploadDir.mkdir();
        }
        try {
            // 解析请求的内容提取文件数据
            @SuppressWarnings("unchecked")
            List<FileItem> formItems = upload.parseRequest(request);
 
            if (formItems != null && formItems.size() > 0) {
                // 迭代表单数据
                for (FileItem item : formItems) {
                    // 处理不在表单中的字段
                    if (!item.isFormField()) {
                        String fileName = new File(item.getName()).getName();
                        String filePath = uploadPath + File.separator + fileName;
                        File storeFile = new File(filePath);
                        // 在控制台输出文件的上传路径
                        System.out.println(filePath);
                        // 保存文件到硬盘
                        item.write(storeFile);
                        request.setAttribute("message",
                            "文件上传成功!");
                    }
                }
            }
        } catch (Exception ex) {
            request.setAttribute("message",
                    "错误信息: " + ex.getMessage());
        }
        
    }
}
~~~

### 5.2springmvc传统方式文件上传

传统方式的文件上传，指的是我们上传的文件和访问的应用存在于同一台服务器上。
并且上传完成之后，浏览器可能跳转。

**步骤**

1. 拷贝5.1中的两个jar包

2. jsp页面

   ~~~jsp
   <form action="/fileUpload" method="post" enctype="multipart/form-data">
       <imput type="file" name="uploadFile"/ >
           <input type="submit" value="上传">
   </form>
   ~~~

3. 编写控制器

   ~~~java
   @Controller("fileUploadController")
   public class FileUploadController {
   /**
   * 文件上传
   */
   @RequestMapping("/fileUpload")
   public String testResponseJson(String picname,MultipartFile
   uploadFile,HttpServletRequest request) throws Exception{
           //定义文件名
           String fileName = "";
           //1.获取原始文件名
           String uploadFileName = uploadFile.getOriginalFilename();
           //2.截取文件扩展名
           String extendName uploadFileName.substring(uploadFileName.lastIndexOf(".")+1,
           uploadFileName.length());
           //3.把文件加上随机数，防止文件重复
           String uuid = UUID.randomUUID().toString().replace("-", "").toUpperCase();
           //4.判断是否输入了文件名
           if(!StringUtils.isEmpty(picname)) {
           fileName = uuid+"_"+picname+"."+extendName;
           }else {
           fileName = uuid+"_"+uploadFileName;
           }
           System.out.println(fileName);
           //2.获取文件路径
           ServletContext context = request.getServletContext();
           String basePath = context.getRealPath("/uploads");
           //3.解决同一文件夹中文件过多问题
           String datePath = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
           //4.判断路径是否存在
           File file = new File(basePath+"/"+datePath);
           if(!file.exists()) {
           file.mkdirs();
           }
           //5.使用MulitpartFile接口中方法，把上传的文件写到指定位置
           uploadFile.transferTo(new File(file,fileName));
           return "success";
   	}
   }
   ~~~

4. 配置文件解析器

~~~xml
<!-- 配置文件上传解析器-->
<bean id="multipartResolver" <!-- id的值是固定的-->
class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
<!-- 设置上传文件的最大尺寸为5MB -->
<property name="maxUploadSize">
<value>5242880</value>
</property>
</bean>
注意：
文件上传的解析器id是固定的，不能起别的名称，否则无法实现请求参数的绑定。（不光是文件，其他
字段也将无法绑定）
~~~

### 5.3springmvc跨服务器上传

#### 5.3.1分服务器目的

在实际开发中，我们会有很多处理不同功能的服务器。例如：
应用服务器：负责部署我们的应用
数据库服务器：运行我们的数据库
缓存和消息服务器：负责处理大并发访问的缓存和消息
文件服务器：负责存储用户上传文件的服务器。
(注意：此处说的不是服务器集群）

分服务器目的是为了让服务器各司其职，提高运行效率

#### 5.3.2步骤

准备两个服务器

在文件服务器的tomcat配置中加入允许读写操作

~~~xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
    	<param-name>debug</param-name>
        <param-value>0</param-value>
    </init-param>
    <init-param>							//
    	<param-name>readonly</param-name>	//加入这几行的目的是：
        <param-value>false</param-value>	//
    </init-param>							//接收文件的目标服务器可以支持写入操作
     <init-param>
    	<param-name>listings</param-name>
        <param-value>false</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
~~~

将上面两个jar包，以及jersey-client-1.18.1.jar、jersey-core-1.18.1.jar导入项目

编写控制器

~~~java
@Controller("fileUploadController2")
public class FileUploadController2 {
public static final String FILESERVERURL =
"http://localhost:9090/day06_spring_image/uploads/";
/**
* 文件上传，保存文件到不同服务器
*/
@RequestMapping("/fileUpload2")
public String testResponseJson(String picname,MultipartFile uploadFile) throws Exception{
        //定义文件名
        String fileName = "";
        //1.获取原始文件名
        String uploadFileName = uploadFile.getOriginalFilename();
        //2.截取文件扩展名
        String extendName uploadFileName.substring(uploadFileName.lastIndexOf(".")+1,
        uploadFileName.length());
        //3.把文件加上随机数，防止文件重复
        String uuid = UUID.randomUUID().toString().replace("-", "").toUpperCase();
        //4.判断是否输入了文件名
        if(!StringUtils.isEmpty(picname)) {
        fileName = uuid+"_"+picname+"."+extendName;
        }else {
        fileName = uuid+"_"+uploadFileName;
        }
        System.out.println(fileName);
        //5.创建sun公司提供的jersey包中的Client对象
        Client client = Client.create();
        //6.指定上传文件的地址，该地址是web路径
        WebResource resource = client.resource(FILESERVERURL+fileName);
        //7.实现上传
        String result = resource.put(String.class,uploadFile.getBytes());
        System.out.println(result);
        return "success";
	}
}
~~~

编写jsp

~~~jsp
<form action="fileUpload2" method="post" enctype="multipart/form-data">
名称：<input type="text" name="picname"/><br/>
图片：<input type="file" name="uploadFile"/><br/>
<input type="submit" value="上传"/>
</form>

~~~

配置解析器

~~~xml
<!-- 配置文件上传解析器-->
<bean id="multipartResolver"
class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
<!-- 设置上传文件的最大尺寸为5MB -->
<property name="maxUploadSize">
<value>5242880</value>
</property>
</bean>
~~~

## 六.SpringMVC其他技术

### 6.1 拦截器

Spring MVC 的处理器拦截器类似于Servlet 开发中的过滤器Filter，用于对处理器进行预处理和后处理。用户可以自己定义一些拦截器来实现特定的功能。
**监听器、过滤器和拦截器对比**

- Servlet：处理Request和Response请求

- 过滤器（Filter）：对Request请求起到过滤的效果，作用在Servlet之前，如果配置为`/*`就会对所有的资源方法（包括servlet/js/css等）进行过滤处理

- 监听器（Listener）：实现了javax.servlet.ServletContextListener接口的服务器端组件，它随Web应用的启动而启动，只初始化一次，让背后会一直运行监视，随Web应用停止而销毁。

  作用：做一些初始化工作，web应用中Spring的容器启动ContextLoaderListener。监听Web中的特定事件，比如HttpSession、ServletRequest的创建和销毁，变量的创建、销毁和修改等，可以在某些动作前后增加处理，实现监控，比如统计在线人数，用HttpSessionListener。

- 拦截器（Interceptor）：是SpringMVC、Struts等框架自己的，不会拦截jsp/html/image的访问，只会拦截访问的控制器方法（Handler）

从配置也可以发现servlet、filter、listener是配置在web.xml中的，而拦截器是配置在SpringMVC自己的配置文件中的。

SpringMVC实现拦截器需要实现HandlerInterceptor接口

**拦截器的执行顺序**

单个拦截器的执行顺序：

![springmvc拦截器处理流程20220225](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/springmvc%E6%8B%A6%E6%88%AA%E5%99%A8%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B20220225.png)

如果拦截器配置多个拦截器，那么它们的preHandle()方法会按照配置文件中拦截器的配置顺序执行，而它们的postHandle()方法和afterCompletion()方法会按照配置顺序的反序执行。

示例：

```java
public class MyInterceptor1 implements HandlerInterceptor {
    /**
     * 会在handler方法之前执行
     * @param httpServletRequest request
     * @param httpServletResponse response
     * @param o handler
     * @return boolean 返回的值代表是否放行
     * @author Yuhaoran
     */
    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
        System.out.println("preHandle");
        return true;
    }

    /**
     * 会在handler方法之后，页面尚未跳转执行
     * @param httpServletRequest request
     * @param httpServletResponse response
     * @param o handler
     * @param modelAndView 封装了视图和数据，此时页面尚未跳转，可以在这里对数据和视图进行修改
     */
    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
        System.out.println("postHandle");
    }

    /**
     * 页面已经跳转渲染完毕之后
     * @param httpServletRequest request
     * @param httpServletResponse response
     * @param o handler
     * @param e 可以在这里捕获异常
     */
    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        System.out.println("afterCompletion");
    }
}
```

### 6.2 全局异常处理

~~~java
//可以优雅的捕获所有的Controller对象handler方法抛出的异常
@ControllerAdvice
public class BaseExceptionHandler {

    @ExceptionHandler(Exception.class)
    public Result error(Exception e){
        e.printStackTrace();
        return new Result(1,e.getMessage(),null);
    }
}
~~~

## 七.SSM整合

### 7.1环境准备

创建数据库和表结构

创建maven工程

导入坐标建立依赖

注意mybatis与spring对应关系

| mybatis-spring  | mybatis         | spring          |
| --------------- | --------------- | --------------- |
| 1.0.0 and1.0.1  | 3.0.1 to3.0.5   | 3.0.0 or higher |
| 1.0.2           | 3.0.6           | 3.0.0 or higher |
| 1.1.0 or higher | 3.1.0 or higher | 3.0.0 or higher |
| 1.3.0 or higher | 3.4.0 or higher | 3.0.0 or higher |

编写pojo

便也业务层和持久层接口

### 7.2整合步骤

#### 7.2.1保证Spring在web工程中独立运行

编写spring配置文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd">

	<!-- 配置spring创建时要扫描的包 -->
	<context:component-scan base-package="com.loserfromlazy">
        <!--指定骚包规则，不扫描@Controller注解的java类-->
        <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller">
        </context:exclude-filter>
    </context:component-scan>
</beans>
~~~

使用注解配置业务层和持久层

@Service、@Repository

#### 7.2.2保证SpringMVC在web工程中独立运行

在web.xml中配置核心控制类

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id="WebApp_ID" version="2.5">
  <display-name>boot-crm</display-name>
 
     <!--配置spring mvc的编码过滤器-->
  <filter>
    <filter-name>CharacterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      <!--配置过滤器中的属性值-->
      <init-param>
      	<param-name>encoding</param-name>
         <param-value>UTF-8</param-value>
      </init-param>
      <!--启动过滤器-->
      <init-param>
      	<param-name>forceEncoding</param-name>
         <param-value>true</param-value>
      </init-param>
  </filter>
     <!--过滤所有请求-->
  <filter-mapping>
    <filter-name>encoding</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
    <!--配置spring mvc的核心控制器-->
  <servlet>
    <servlet-name>boot-crm</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
     <!--配置初始化参数，用于读取springmvc的配置文件-->
      <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:spring/springmvc.xml</param-value>
    </init-param>
        <!--配置servlet的对象的创建时间点：应用加载时创建。取值只能是非0正整数，表示启动顺序-->
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>boot-crm</servlet-name>
    <url-pattern>/</url-pattern>
  </servlet-mapping>
</web-app>
~~~

编写springmvc配置文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">
	<!-- 配置Controller扫描 -->
	<context:component-scan base-package="com.yhr.crm.controller"/>
	<!--另一种配置Controller的方式，使用时一种即可-->
	<context:component-scan base-package="com.loserfromlazy">
        <!--指定扫包规则，只扫描@Controller注解的java类-->
        <context:ixclude-filter type="annotation" expression="org.springframework.stereotype.Controller">
        </context:ixclude-filter>
    </context:component-scan>
	<!-- 配置注解驱动 -->
	<mvc:annotation-driven />
<mvc:default-servlet-handler />
	<!-- 配置视图解析器 -->
	<bean	class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<!-- 前缀 -->
		<property name="prefix" value="/WEB-INF/jsp/"/>
		<!-- 后缀 -->
		<property name="suffix" value=".jsp"/>
	</bean>
</beans>
~~~

编写Controller和jsp

#### 7.2.3整合spring和springmvc

配置监听器实现启动服务创建容器

在web.xml中加入

~~~xml
<!--配置spring提供的监听器，用于启动服务时加载容器
该监听器只能加载WEB-INF目录中名称为applicationContext.xml的配置文件-->
  <listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
  </listener>
<!--手动指定配置文件的位置 -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/applicationContext-*.xml</param-value>
  </context-param>
~~~

#### 7.2.4 保证Mybatis在web程序中独立运行

编写dao接口的映射文件

编写SqlMapConfig

~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <!-- 加载属性文件 -->
    <properties resource="log4j.properties">
        <!--properties中还可以配置一些属性名和属性值  -->
        <!-- <property name="jdbc.driver" value=""/> -->
    </properties>
    <!-- 和spring整合后 environments配置将废除-->
    <environments default="development">
        <environment id="development">
        <!-- 使用jdbc事务管理，事务控制由mybatis-->
            <transactionManager type="JDBC" />
        <!-- 数据库连接池，由mybatis管理-->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver" />
                <property name="url" value="jdbc:mysql://localhost:3306/pojo?useSSL=false&amp;serverTimezone=Hongkong&amp;characterEncoding=utf-8&amp;autoReconnect=true" />
                <property name="username" value="root" />
                <property name="password" value="manager" />
            </dataSource>
        </environment>
    </environments>
    <!-- 加载 映射文件 -->
    <mappers>
        <mapper resource="mybatis/user.xml"/>
    </mappers>
</configuration>
~~~

#### 7.2.5 整合Mybatis和Spring

清空SqlMapConfig.xml内容，把其中内容配置到spring中

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
</configuration>
~~~

**PS：由于我们使用的是代理开发dao的方式，所以Dao的具体实现类由Mybatis使用代理方式创建，所以mybatis的配置文件不能删除。**

**当我们整合mybatis和spring时，mybatis创建的mapper.xml和dao接口文件名必须一致**

使用spring接管mybatis的session工程

~~~xml
<!-- 配置 读取properties文件 jdbc.properties -->
	<context:property-placeholder location="classpath:jdbc.properties"/>

	<!-- 配置 数据源 -->
	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
		<property name="driverClassName" value="${jdbc.driver}"/>
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>

	<!-- 配置SqlSessionFactory -->
	<bean class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 设置MyBatis核心配置文件 -->
		<property name="configLocation" value="classpath:mybatis/SqlMapConfig.xml"/>
		<!-- 设置数据源 -->
		<property name="dataSource" ref="dataSource"/>
	</bean>
~~~

配置自动扫描所有mapper接口和文件

~~~xml
<!-- 配置Mapper扫描 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 设置Mapper扫描包 -->
		<property name="basePackage" value="com.loserfromlazy.dao"/>
	</bean>
~~~

配置事务

~~~xml
<!-- 事务管理器 -->
	<bean id="transactionManager"	class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 数据源 -->
		<property name="dataSource" ref="dataSource"/>
	</bean>

	<!-- 通知 -->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<!-- 传播行为 -->
			<tx:method name="save*" propagation="REQUIRED"/>
			<tx:method name="insert*" propagation="REQUIRED"/>
			<tx:method name="add*" propagation="REQUIRED"/>
			<tx:method name="create*" propagation="REQUIRED"/>
			<tx:method name="delete*" propagation="REQUIRED"/>
			<tx:method name="update*" propagation="REQUIRED"/>
			<tx:method name="find*" propagation="SUPPORTS" read-only="true"/>
			<tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
			<tx:method name="get*" propagation="SUPPORTS" read-only="true"/>
			<tx:method name="query*" propagation="SUPPORTS" read-only="true"/>
		</tx:attributes>
	</tx:advice>

	<!-- 切面 -->
	<aop:config>
		<aop:advisor advice-ref="txAdvice"
			pointcut="execution(* com.lofromlazy.service.*.*(..))"/>
	</aop:config>
~~~

applicationContext-dao.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd">

	<!-- 配置 读取properties文件 jdbc.properties -->
	<context:property-placeholder location="classpath:jdbc.properties"/>

	<!-- 配置 数据源 -->
	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
		<property name="driverClassName" value="${jdbc.driver}"/>
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>

	<!-- 配置SqlSessionFactory -->
	<bean class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 设置MyBatis核心配置文件 -->
		<property name="configLocation" value="classpath:mybatis/SqlMapConfig.xml"/>
		<!-- 设置数据源 -->
		<property name="dataSource" ref="dataSource"/>
		<!-- 别名包扫描 -->
		<property name="typeAliasesPackage" value="com.yhr.crm.pojo"/>
	</bean>

	<!-- 配置Mapper扫描 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 设置Mapper扫描包 -->
		<property name="basePackage" value="com.yhr.crm.mapper"/>
	</bean>
</beans>
~~~

## 八、实现简单的SpringMVC

SpringMVC的大致原理如下，我们跟据此流程来手写一个简易的SpringMVC框架。

具体代码见项目地址：[Github](https://github.com/Loserfromlazy/MySpringMVC) [Gitee](https://gitee.com/yhr520/MySpringMVC)

![springmvc流程20220227](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/springmvc%E6%B5%81%E7%A8%8B20220227.png)

## 九、Spring MVC源码分析

众所周知，SpringMVC最重要的一个类就是`DispatcherServlet`，因此我们从这个类入手，分析一下SpringMVC的源码及请求流程。

### 9.1 DispatcherServlet和请求分析

我们首先看一下这个类的继承结构，如下图：

![image-20220330104641700](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220330104641700.png)

我们可以看到，这个类继承了HttpServlet，说明也是遵循了Servlet规范的。在Servlet中处理业务最重要的方法就是doGet、doPost等请求，因此我们主要关注这些请求。

我们顺着继承从上往下看，首先是HttpServletBean，继承了HttpServlet，类的结构如下，并没有重写父类的doGet等请求。

![image-20220330104914926](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220330104914926.png)

然后我们看FrameworkServlet类，这个类重写了继承的doGet等方法，源码如下：

```java
@Override
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   processRequest(request, response);
}

/**
 * Delegate POST requests to {@link #processRequest}.
 * @see #doService
 */
@Override
protected final void doPost(HttpServletRequest request, HttpServletResponse response)
      throws ServletException, IOException {

   processRequest(request, response);
}
```

我们看到这两个方法都访问了processRequest方法，那么我们就跟进这个方法：

![image-20220330105738580](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220330105738580.png)

我们可以看到，这个方法核心逻辑就是调用doService方法，而这个doService方法是一个抽象方法，因此实现是在子类中也就是DispatcherServlet中。**因此一个请求到来后最终会来到DispatcherServlet的doService方法中。**

下面我们就来看看DispatcherServlet类的doService方法：

```java
@Override
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
   logRequest(request);
	//设置request的相关属性，保存快照
		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}
		//把localeResolver、themeResolver以及上下文等放入request的属性中
		// Make framework objects available to handlers and view objects.
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
	//根据flashMapManager获取重定向的原有的请求参数。
		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}
    
   //核心逻辑
   try {
      doDispatch(request, response);
   }
   finally {
      if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
         // Restore the original attribute snapshot, in case of an include.
         if (attributesSnapshot != null) {
            restoreAttributesAfterInclude(request, attributesSnapshot);
         }
      }
   }
}
```

这个方法中主要是执行一下设置request的相关属性，保存快照等操作，然后调用doDispatch(request, response);我们进入这个方法：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request;
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
          //1.检查是否是上传文件的请求
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

          //2.取得当前请求的Controller即Handler处理器
          //这里并不是直接返回Handler而是返回HandlerExecutionChain处理链对象
          //这个对象封装了Handler和Inteceptor
         // Determine handler for the current request.
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
             //返回404
            noHandlerFound(processedRequest, response);
            return;
         }

          //3.确定当前请求的处理程序适配器。
         // Determine handler adapter for the current request.
         HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

         // Process last-modified header, if supported by the handler.
         String method = request.getMethod();
         boolean isGet = "GET".equals(method);
         if (isGet || "HEAD".equals(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
               return;
            }
         }

         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

          //4.处理请求返回视图对象ModelAndView
         // Actually invoke the handler.
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }
		//5.处理结果视图对象
         applyDefaultViewName(processedRequest, mv);
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         // As of 4.3, we're processing Errors thrown from handler methods as well,
         // making them available for @ExceptionHandler methods and other scenarios.
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
       //6.跳转页面渲染视图
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         // Instead of postHandle and afterCompletion
         if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // Clean up any resources used by a multipart request.
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}
```

这个方法流程在注释中已经标注好了，我们在下一节9.2请求流程分析中详细分析每一个方法

### 9.2 SpringMVC的请求流程分析

我们分析源码一般可以通过debug的方式对源码有更深入的了解，而且debug可以看到整体的调用栈，更方便我们对源码的学习和了解。

首先我们在DispatcherServlet的doService方法上打上断点作为入口，然后发起请求，我们可以看到调用栈，tomcat首先会帮我们找到DispatcherServlet并执行service方法。

> 为什么DispatcherServlet的doService方法作为入口？
>
> 因为使用SpringMVC只需要在web.xml中配置DispatcherServlet并拦截所有请求即可，因此tomcat就会找到DispatcherServlet，并执行service方法。（tomcat的源码流程可关注我们另一篇博客[Tomcat学习](https://www.cnblogs.com/yhr520/p/15792651.html)）
>
> ![image-20220330103454888](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220330103454888.png)
>
> 而经过9.1中对Dispatcher的分析，再结合调用栈我们会发现最后service方法最终会进到DispatcherServlet的doService方法中。



> 未完结，上次更新时间2022/3/30







