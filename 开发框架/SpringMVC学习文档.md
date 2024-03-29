# SpringMVC学习文档

转载请声明！！！本文如有错误，欢迎指正，感激不尽。

springMVC就是类似与Struts2的mvc框架，始于SpringFrameWork的后续产品

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
<!--以下是springmvc.xml-->
<!-- 配置文件上传解析器-->
<bean id="multipartResolver" <!-- id的值是固定的-->
class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
<!-- 如果是Servlet3.0以上也可以使用Spring提供的StandardServletMultipartResolver -->
<!--<bean id="multipartResolver" class="org.springframework.web.multipart.support.StandardServletMultipartResolver"/>-->
<!-- 设置上传文件的最大尺寸为5MB -->
<property name="maxUploadSize">
<value>5242880</value>
</property>
</bean>
<!--StandardServletMultipartResolver需要在web.xml中配置。以下是web.xml-->
<servlet>
		<servlet-name>springmvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath*:springmvc.xml</param-value>
		</init-param>
		<multipart-config>
			<!--临时文件的目录-->
			<location>d:/</location>
			<!-- 上传文件最大2M -->
			<max-file-size>2097152</max-file-size>
			<!-- 上传文件整个请求不超过4M -->
			<max-request-size>4194304</max-request-size>
		</multipart-config>
	</servlet>

	<servlet-mapping>
		<servlet-name>springmvc</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
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

这里是使用SpringMVC5.2.x版本。

### 9.1 SpringMVC源码测试工程搭建

gradle新建web项目，如下图：

![image-20220402101419155](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402101419155.png)

编写测试Controller,一个是用于普通请求的，一个是用于上传文件的（注意使用文件上传需要自行配置文件解析的Bean）

```java
@Controller
@RequestMapping("/demo")
public class DemoController {

   @RequestMapping("/handle")
   public String handle(String id , Map<String,Object> model){
      System.out.println("id="+id);
      Date date = new Date();
      model.put("date",date);
      return "success";
   }

   @RequestMapping("/upload")
   public String upload(MultipartFile file, Map<String,Object> model){
      System.out.println("file="+file.getName());
      Date date = new Date();
      try {
         InputStream inputStream = file.getInputStream();
         Reader reader = new InputStreamReader(inputStream);
         OutputStream outputStream = new ByteArrayOutputStream();
         byte[] buffer = new byte[1024];
         int len;
         while ((len = inputStream.read(buffer))!=-1){
            outputStream.write(buffer,0,len);
         }
      } catch (IOException e) {
         e.printStackTrace();
      }
      model.put("date",file.getName());
      return "success";
   }
}
```

其余jsp和配置文件略。

### 9.2 DispatcherServlet和请求分析

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
		//5.执行拦截器方法
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

### 9.3 SpringMVC的请求流程分析

我们分析源码一般可以通过debug的方式对源码有更深入的了解，而且debug可以看到整体的调用栈，更方便我们对源码的学习和了解。

首先我们在DispatcherServlet的doService方法上打上断点作为入口，然后发起请求，我们可以看到调用栈，tomcat首先会帮我们找到DispatcherServlet并执行service方法。

> 为什么DispatcherServlet的doService方法作为入口？
>
> 因为使用SpringMVC只需要在web.xml中配置DispatcherServlet并拦截所有请求即可，因此tomcat就会找到DispatcherServlet，并执行service方法。（tomcat的源码流程可关注我们另一篇博客[Tomcat学习](https://www.cnblogs.com/yhr520/p/15792651.html)）
>
> ![image-20220330103454888](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220330103454888.png)
>
> 而经过9.1中对Dispatcher的分析，再结合调用栈我们会发现最后service方法最终会进到DispatcherServlet的doService方法中。

我们在doService方法中继续往下跟进入到doDispatch方法，这个方法在9.1中已经给出源码和注释了，现在我们详细分析一下。

在doDispatch中，核心步骤在上面源码的注释中已经给出了，下面再总结一下：

1. 检查是否是上传文件的请求
2. 调用getHandler获取到能够处理当前请求的执行链HandlerExecutionChain
3. 调用getHandlerAdapter获取上一步中Handler的适配器
4. 适配器调用ha.handle()方法，返回一个ModelAndView对象
5. 调用processDispatchResult()方法完成视图渲染跳转

下面我们对每一步都进行详细的分析。

#### 9.3.1 checkMultipart，检查是否是上传文件的请求

在doDispatch中第一步就是检查当前请求是否是文件上传的请求：

这里会调用checkMultipart方法，这里的逻辑很简单，此方法主要是检查是否是上传文件的请求，（通过content-type检查），如果是文件请求，将普通request对象转换成multipartRequest返回，如果不是则返回传入的request对象。流程图如下：

![image-20220402110930999](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402110930999.png)

解析器主要有三种实现，如下图（我们呢调试时使用的是第三个标准解析器，因为不用额外导入jar包）：

常用的一般是下图中标出的两个，其中第三个需要Servlet3.0以上且不仅需要配置Bean，还需要配置参数，配置方式见上面的第五章Spring文件上传

![image-20220402101250797](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402101250797.png)

#### 9.3.2 getHandler 获取Handler

我们跟进getHandler方法，代码如下：

![image-20220402141256861](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402141256861.png)

这个方法根据当前的HandlerMapping集合获取Handler执行链，（关于HandlerMapping集合初始化将在9.4中讲解），我们debug会发现会在RequestMappingHandlerMapping中获得Handler，实际上这个集合就是从@Controller类的类型和方法级的@RequestMapping注释中创建的RequestMappingInfo实例，（RequestMappingHandlerMapping也是最常用的集合）。

我们继续跟进mapping的getHandler方法，此方法会先根据请求路径去注册的mapping中获取HandlerMethod（封装由方法和bean组成的处理程序方法的信息。提供对方法参数、方法返回值、方法注释等的方便访问。），然后将HandlerMethod封装成处理器对象，封装成处理器链对象的主要目的是将拦截器封装进去。

流程如下：

![image-20220402163953843](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402163953843.png)

> 这里的HandlerMethod可以看成是对Controller中方法的描述，而包装后的处理器链可以看成是方法和方法对应的拦截器，如下图在debug的过程中可以有更深的体会，也可以自行查阅HandlerMethod的源码：
>
> ![image-20220402165054514](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402165054514.png)

如果getHandler 最后获取不到Handler对象的话，就会输出404.

#### 9.3.3 getHandlerAdapter 获取HandlerAdapter

在获取到Handler（实际上是一个处理器链对象）之后，我们会调用getHandlerAdapter方法获取处理器适配器，我们跟进此方法，代码如下：

![image-20220402165807941](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220402165807941.png)

这个方法与上面getHandler类似，会根据当前的适配器集合中找到可以支持的适配器，常用的是第三个RequestMappingHandlerAdapter适配器。

RequestMappingHandlerAdapter#support方法就是判断当前的handler是否是HandlerMethod的实例

#### 9.3.4 执行handle方法

源码如下：

**ha.handle⽅法 核心逻辑是解析出请求传输的参数值，传给反射方法对象。通过反射调用去执行其业务。并将返回结果封装成ModelAndView对象返回。**

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

我们跟进handle方法中：

这个方法会执行`handleInternal(request, response, (HandlerMethod) handler);`可以看到其实handler就是HandlerMethod的实例。

我们继续跟进handleInternal，这个方法会检查是否有session需要处理，如果有session需要处理就加锁，没有就直接继续执行

```java
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

   ModelAndView mav;
   checkRequest(request);

    //判断当前是否需要支持在同一个session中只能线性的处理请求
    //synchronizeOnSession默认false
   // Execute invokeHandlerMethod in synchronized block if required.
   if (this.synchronizeOnSession) {
       //获取session
      HttpSession session = request.getSession(false);
      if (session != null) {
          //获取同步锁
         Object mutex = WebUtils.getSessionMutex(session);
         synchronized (mutex) {
             //加锁执行invokeHandlerMethod方法处理参数并调用
            mav = invokeHandlerMethod(request, response, handlerMethod);
         }
      }
      else {
         // No HttpSession available -> no mutex necessary
         mav = invokeHandlerMethod(request, response, handlerMethod);
      }
   }
   else {
      // No synchronization on session demanded at all...
       //不需要对session同步处理，执行invokeHandlerMethod方法处理参数并调用
      mav = invokeHandlerMethod(request, response, handlerMethod);
   }

   if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
      if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
         applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
      }
      else {
         prepareResponse(response);
      }
   }

   return mav;
}
```

我们继续跟进invokeHandlerMethod方法，这个方法中主要是配置一些参数和返回值的处理器，然后调用`invocableMethod.invokeAndHandle(webRequest, mavContainer); `

`getModelAndView(mavContainer, modelFactory, webRequest);`

这两个方法，调用目标的handlerMethod并处理返回的ModelAndView。

```java
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
	//封装request, response
   ServletWebRequest webRequest = new ServletWebRequest(request, response);
   try {
      // 获取容器中全局配置的InitBinder和当前handlerMethod所对应的controller中配置的InitBinder,用于进行参数绑定      
      WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
       // 获取容器中全局配置的ModelAttribute和当前handlerMethod所对应的controller中配置的ModelAttribute,这些配置的方法将会在目标方法调用之前进行执行
      ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
	//封装handlerMethod
      ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
       //设置容器中配置的所有的参数和返回值的处理器
      if (this.argumentResolvers != null) {
         invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      }
      if (this.returnValueHandlers != null) {
         invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      }
      invocableMethod.setDataBinderFactory(binderFactory);
      invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

      ModelAndViewContainer mavContainer = new ModelAndViewContainer();
      mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
      // 调用前面获取到的@ModelAttribute标注的方法从而达到@ModelAttribute标注的方法能够在目标handler调用之前执行，可以自行点进方法中查看Spring的官方注释
      modelFactory.initModel(webRequest, mavContainer, invocableMethod);
      mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

      AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
      asyncWebRequest.setTimeout(this.asyncRequestTimeout);

      WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
      asyncManager.setTaskExecutor(this.taskExecutor);
      asyncManager.setAsyncWebRequest(asyncWebRequest);
      asyncManager.registerCallableInterceptors(this.callableInterceptors);
      asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

      if (asyncManager.hasConcurrentResult()) {
         Object result = asyncManager.getConcurrentResult();
         mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
         asyncManager.clearConcurrentResult();
         LogFormatUtils.traceDebug(logger, traceOn -> {
            String formatted = LogFormatUtils.formatValue(result, !traceOn);
            return "Resume with async result [" + formatted + "]";
         });
         invocableMethod = invocableMethod.wrapConcurrentResult(result);
      }
	  //对请求参数进行处理 调用目标的handlerMethod，并且将返回值封装为一个ModelAndView对象
      invocableMethod.invokeAndHandle(webRequest, mavContainer);
      if (asyncManager.isConcurrentHandlingStarted()) {
         return null;
      }
	  // 对封装的ModelAndView进行处理，主要判断当前请求是否进行重定向
      // 如果进行了重定向还会判断是否需要将flashAttributes封装到新的请求中
      return getModelAndView(mavContainer, modelFactory, webRequest);
   }
   finally {
      webRequest.requestCompleted();
   }
}
```

我们进入invokeAndHandle方法，这个方法中主要执行`invokeForRequest`和`handleReturnValue`。

所以我们先直接进入invokeForRequest方法中，这个方法会先获取当前方法的参数，然后调用反射执行方法。

```
handleReturnValue(
      returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
```

```java
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
	//获取当前方法参数
   Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
   if (logger.isTraceEnabled()) {
      logger.trace("Arguments: " + Arrays.toString(args));
   }
    //调用反射执行方法
   return doInvoke(args);
}
```

我们可以先进入getMethodArgumentValues方法查看，这里主要就是调用我们之前设置的处理器对参数进行处理并返回。

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      Object... providedArgs) throws Exception {
  // 获取当前handler所声明的所有参数，主要包括参数名，参数类型，参数位置，所标注的注解等等属性
   MethodParameter[] parameters = getMethodParameters();
   if (ObjectUtils.isEmpty(parameters)) {
      return EMPTY_ARGS;
   }

   Object[] args = new Object[parameters.length];
   for (int i = 0; i < parameters.length; i++) {
      MethodParameter parameter = parameters[i];
      parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
       //providedArgs 是调用时提供的参数，主要判断这些参数中是否有当前类型，如果有，就直接使用调用方提供的参数，对于请求处理而言，默认情况下，调用方提供的参数都是长度为0的数组，也就是不提供参数，我们在之前invokeAndHandle调用时也没有传入参数
      args[i] = findProvidedArgument(parameter, providedArgs);
      if (args[i] != null) {
         continue;
      }
       //遍历 判断当前的处理器能否解析参数
      if (!this.resolvers.supportsParameter(parameter)) {
         throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
      }
      try {
          //如果能找到对当前参数的进行处理的argumentResolver，就调用其resolveArgument方法从request中获取对应参数并转换
         args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
      }
      catch (Exception ex) {
         // Leave stack trace for later, exception may actually be resolved and handled...
         if (logger.isDebugEnabled()) {
            String exMsg = ex.getMessage();
            if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
               logger.debug(formatArgumentError(parameter, exMsg));
            }
         }
         throw ex;
      }
   }
   return args;
}
```

获取完参数后就会调用`doInvoke(args);`方法。此方法源码略，可以自行进入查看本质上是调用了Java的反射即method.invoke方法。调用完之后会返回方法的返回值。

然后回到invokeAndHandle方法中，之前我们说过这个方法主要执行`invokeForRequest`和`handleReturnValue`，而这个handleReturnValue就是处理返回值的，此方法会将返回值处理返回ModelAndView对象。

我们进入到handleReturnValue，此方法会根据我们的返回值和返回类型获取对应的处理器然后将返回值封装进mavContainer中，这个对象就是ModelAndView的容器对象，就是我们上面介绍invokeHandlerMethod方法。

```java
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	//根据我们的返回值和返回类型获取对应的处理器
   HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
   if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
   }
    //处理返回值
   handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

> 注意，这里的handleReturnValue会根据我们查询出的HandlerMethodReturnValueHandler不同而执行不同实现类的方法。
>
> 比如如果处理@ResponseBody的处理器的源码如下，这里mavContainer.setRequestHandled(true);的意思是请求是否已经在处理程序中被完全处理，即我在此方法中已经处理完了不需要返回视图了。
>
> ```java
> //这个方法会将返回值以流的形式直接输出，不再返回视图
> @Override
> public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
>       ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
>       throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
> 
>    mavContainer.setRequestHandled(true);
>    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
>    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
> 	//调用输出流的write和flush方法将返回值直接写回
>    // Try even with null return value. ResponseBodyAdvice could get involved.
>    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
> }
> ```
>
> 而处理视图的处理器源码如下， mavContainer.setViewName(viewName);
>
> ```java
> //这个方法会将视图名存储，在后面处理视图的渲染时拿出ModelAndView
> @Override
> public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
>       ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
> 
>    if (returnValue instanceof CharSequence) {
>       String viewName = returnValue.toString();
>        //存储返回的视图名
>       mavContainer.setViewName(viewName);
>       if (isRedirectViewName(viewName)) {
>          mavContainer.setRedirectModelScenario(true);
>       }
>    }
>    else if (returnValue != null) {
>       // should not happen
>       throw new UnsupportedOperationException("Unexpected return type: " +
>             returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
>    }
> }
> ```

#### 9.3.5 渲染视图，跳转页面

在doDispatch方法中，执行完handle之后会执行此Handler的拦截器

```java
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

跟进方法后，发现此方法会遍历所有的拦截器然后进行执行。

```java
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
      throws Exception {

   HandlerInterceptor[] interceptors = getInterceptors();
   if (!ObjectUtils.isEmpty(interceptors)) {
      for (int i = interceptors.length - 1; i >= 0; i--) {
         HandlerInterceptor interceptor = interceptors[i];
         interceptor.postHandle(request, response, this.handler, mv);
      }
   }
}
```

在执行完拦截器后，就会到processDispatchResult方法，此方法主要用于渲染视图并返回前端

```java
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

我们跟进这个处理视图的方法，这个方法的核心代码如下：

```java
// Did the handler return a view to render?
if (mv != null && !mv.wasCleared()) {
    //处理视图
   render(mv, request, response);
   if (errorView) {
      WebUtils.clearErrorRequestAttributes(request);
   }
}
else {
   if (logger.isTraceEnabled()) {
      logger.trace("No view rendering, null ModelAndView returned.");
   }
}
```

这个方法我们主要关注`render(mv, request, response);`这个方法，源码如下：

```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
   // Determine locale for request and apply it to the response.
   Locale locale =
         (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
   response.setLocale(locale);

   View view;
   String viewName = mv.getViewName();
   if (viewName != null) {
      // We need to resolve the view name.
       //根据view名称封装View视图对象
      view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
      if (view == null) {
         throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
               "' in servlet with name '" + getServletName() + "'");
      }
   }
   else {
      // No need to lookup: the ModelAndView object contains the actual View object.
      view = mv.getView();
      if (view == null) {
         throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
               "View object in servlet with name '" + getServletName() + "'");
      }
   }

   // Delegate to the View object for rendering.
   if (logger.isTraceEnabled()) {
      logger.trace("Rendering view [" + view + "] ");
   }
   try {
      if (mv.getStatus() != null) {
         response.setStatus(mv.getStatus().value());
      }
       //渲染数据
      view.render(mv.getModelInternal(), request, response);
   }
   catch (Exception ex) {
      if (logger.isDebugEnabled()) {
         logger.debug("Error rendering view [" + view + "]", ex);
      }
      throw ex;
   }
}
```

这个方法中我们主要关注两个方法`resolveViewName`和`view.render`,一个是用于封装视图对象，另一个是渲染数据的方法。

我们下面跟进这个两个方法中：

1. resolveViewName

   ```java
   @Nullable
   protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
         Locale locale, HttpServletRequest request) throws Exception {
   
      if (this.viewResolvers != null) {
         for (ViewResolver viewResolver : this.viewResolvers) {
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
               return view;
            }
         }
      }
      return null;
   }
   ```

   在这个方法中会遍历全部ViewResolver然后调用能处理我们视图名的处理器去处理，实际上一般只有一个就是我们配置在springmvc.xml中的视图解析器，如下面代码：

   ```xml
   <!--配置springmvc的视图解析器-->
   <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
      <property name="prefix" value="/WEB-INF/"/>
      <property name="suffix" value=".jsp"/>
   </bean>
   ```

   我们跟进viewResolver.resolveViewName方法中，源码如下图：

   ![image-20220407152545803](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220407152545803.png)

   我们跟进createView中，源码如下：

   ![image-20220407152746542](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220407152746542.png)

   一般我们在springmvc.xml中的视图解析器中都会配置前缀和后缀，所以会走到父类的createView中，我们在往下跟源码，流程如下图，最后会根据我们的配置拼装View对象，然后实例化View并返回。

   ![image-20220407154302739](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220407154302739.png)

2. view.render

   这个方法中主要是调用上一步我们创建的view对象的render方法，将Model中的数据存储到request中，然后转发到对应的视图，整理流程如下：

   ![image-20220407160635565](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220407160635565.png)

以上就是渲染视图并跳转页面的流程和代码分析

### 9.4 SpringMVC组件初始化分析

SpringMVC的组件DispatcherServlet#onRefresh方法中初始化的，SpringMVC一共有九大组件，在这个方法中也能看到，源码如下：

```java
@Override
protected void onRefresh(ApplicationContext context) {
   initStrategies(context);
}

/**
 * Initialize the strategy objects that this servlet uses.
 * <p>May be overridden in subclasses in order to initialize further strategy objects.
 */
protected void initStrategies(ApplicationContext context) {
   initMultipartResolver(context);
   initLocaleResolver(context);
   initThemeResolver(context);
   initHandlerMappings(context);
   initHandlerAdapters(context);
   initHandlerExceptionResolvers(context);
   initRequestToViewNameTranslator(context);
   initViewResolvers(context);
   initFlashMapManager(context);
}
```

那么SpringMVC是在什么时候进行初始化的呢？其实我们可以在onRefresh方法上打上断点，然后debug模式启动项目。

启动项目之后，发送一个请求，我们会发现只有第一次请求的时候会在onRefresh方法停下。我们再观察调用栈，截图如下：

![image-20220407170558129](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220407170558129.png)

我们会发现他在refresh中的finishRefresh方法中进行的初始化，这个方法是Spring容器初始化的最后一步。我们再继续往前看，发现初始化容器是在HttpServletBean中调用的，也就是说是在tomcat获取请求后，会找到DispatcherServlet（tomcat的部分可以查看tomcat的请求流程源码），在HttpServletBean执行init的时候初始化了SpringMVC的容器，同时初始化了SpringMVC的九大组件。

这九个组件的初始化流程略有差异，但大体上都是初始化SpringMVC默认指定的Bean，这里我们以initHandlerMappings为例，看一下这个组件的初始化流程。

首先跟进initHandlerMappings方法，源码如下：

```java
private void initHandlerMappings(ApplicationContext context) {
   this.handlerMappings = null;
//先在ApplicationContext中找HandlerMappings
   if (this.detectAllHandlerMappings) {
      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
      Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
         this.handlerMappings = new ArrayList<>(matchingBeans.values());
         // We keep HandlerMappings in sorted order.
         AnnotationAwareOrderComparator.sort(this.handlerMappings);
      }
   }
    
   else {
      try {
         HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
         this.handlerMappings = Collections.singletonList(hm);
      }
      catch (NoSuchBeanDefinitionException ex) {
         // Ignore, we'll add a default HandlerMapping later.
      }
   }

    //如果没有则通过策略模式生成默认的HandlerMappings
   // Ensure we have at least one HandlerMapping, by registering
   // a default HandlerMapping if no other mappings are found.
   if (this.handlerMappings == null) {
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isTraceEnabled()) {
         logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
               "': using default strategies from DispatcherServlet.properties");
      }
   }
}
```

这个方法主要是先在ApplicationContext中找HandlerMappings如果没有则通过策略模式生成默认的HandlerMappings。

我们在debug时一般在ApplicationContext中是找不到的，这时会进入到else的代码块，我们继续往下跟进入到`getDefaultStrategies(context, HandlerMapping.class);`方法中，源码如下：

```java
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
   String key = strategyInterface.getName();
    //获取默认策略
   String value = defaultStrategies.getProperty(key);
   if (value != null) {
      String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
      List<T> strategies = new ArrayList<>(classNames.length);
      for (String className : classNames) {
         try {
            Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
             //创建策略类
            Object strategy = createDefaultStrategy(context, clazz);
            strategies.add((T) strategy);
         }
         catch (ClassNotFoundException ex) {
            throw new BeanInitializationException(
                  "Could not find DispatcherServlet's default strategy class [" + className +
                  "] for interface [" + key + "]", ex);
         }
         catch (LinkageError err) {
            throw new BeanInitializationException(
                  "Unresolvable class definition for DispatcherServlet's default strategy class [" +
                  className + "] for interface [" + key + "]", err);
         }
      }
      return strategies;
   }
   else {
      return new LinkedList<>();
   }
}
```

在这个方法中主要关注两个地方:

一个是获取默认策略（` String value = defaultStrategies.getProperty(key);`），

一个是创建策略类（`Object strategy = createDefaultStrategy(context, clazz);`）。

那么默认策略是哪来的呢？我们其实在静态代码块中就能看到数据的来源，如下图

![image-20220408144826447](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220408144826447.png)

实际上所有组件的默认类都是在DispatcherServlet.properties中规定的。

然后我们再看创建策略类的，这里我们跟进去代码会发现实际上是调用了getBean方法，这个方法在Spring源码分析中仔细的讲过了，这里就不再赘述。

综上，就是SpringMVC的组件初始化分析。





