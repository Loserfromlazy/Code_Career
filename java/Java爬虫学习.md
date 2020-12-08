# Java爬虫学习

# 一、概述

## 1.1 介绍

​	网络爬虫也叫网络机器人，可以代替人们自动的进行数据信息的采集与整理。它是一种按照一定的规则，自动地抓取万维网信息的程序或者脚本，可以自动采集所有其能够访问到的页面内容，以获取相关数据。

​	从功能上来讲，爬虫一般分为数据采集，处理，储存三个部分。爬虫从一个或若干初始网页的URL开始，获得初始网页上的URL，在抓取网页的过程中，不断从当前页面上抽取新的URL放入队列,直到满足系统的一定停止条件。

## 1.2 入门

1.  **环境准备**

   jdk1.8、IDEA、maven

2. **导入依赖**

   ~~~xml
   <dependencies>
       <dependency>
        <groupId>org.apache.httpcomponents</groupId>
           <artifactId>httpclient</artifactId>
           <version>4.5.3</version>
       </dependency>
   
       <dependency>
           <groupId>org.slf4j</groupId>
           <artifactId>slf4j-log4j12</artifactId>
           <version>1.7.21</version>
       </dependency>
   </dependencies>
   ~~~
   
3.  log4j.properties

    ~~~properties
    log4j.rootLogger=WARN,A1
    log4j.logger.cn.itcast = DEBUG
    
    log4j.appender.A1=org.apache.log4j.ConsoleAppender
    log4j.appender.A1.layout=org.apache.log4j.PatternLayout
    log4j.appender.A1.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} - %-4r %-5p [%t] %C:%L %x - %m%n
    ~~~

4.  代码

    ~~~java
    package com.loserfromlazy.TestPa;
    
    import org.apache.http.client.methods.CloseableHttpResponse;
    import org.apache.http.client.methods.HttpGet;
    import org.apache.http.impl.client.CloseableHttpClient;
    import org.apache.http.impl.client.HttpClients;
    import org.apache.http.util.EntityUtils;
    
    
    public class Test1 {
        public static void main(String[] args) throws Exception {
            //创建对象
            CloseableHttpClient httpClient = HttpClients.createDefault();
            //get请求
            HttpGet httpGet = new HttpGet("http://www.itcast.cn");
            //获取响应
            CloseableHttpResponse response = httpClient.execute(httpGet);
            //判断是否访问成功
            if (response.getStatusLine().getStatusCode()==200){
                String content = EntityUtils.toString(response.getEntity(), "UTF-8");
                System.out.println(content);
            }
        }
    }
    ~~~

# 二、HttpClient

​	虽然在 JDK 的 java.net 包中已经提供了访问 HTTP 协议的基本功能，但是对于大部分应用程序来说，JDK 库本身提供的功能还不够丰富和灵活。HttpClient 是 Apache Jakarta Common 下的子项目，用来提供高效的、最新的、功能丰富的支持 HTTP 协议的客户端编程工具包，并且它支持 HTTP 协议最新的版本和建议。

**HTTPClient功能介绍**

- 实现了所有HTTP方法
- 支持自动转向
- 支持https协议
- 支持代理服务器

## 2.1 HTTP方法

**http编程的基本步骤：**

1. 创建 HttpClient 的一个实例.
2. 创建某个方法（DeleteMethod，EntityEnclosingMethod，ExpectContinueMethod，GetMethod，HeadMethod，MultipartPostMethod，OptionsMethod，PostMethod，PutMethod，TraceMethod）的一个实例，一般可用要目标URL为参数。
3. 让 HttpClient 执行这个方法.
4. 读取应答信息.
5. 释放连接.
6. 处理应答.

### get无参

~~~java
public static void doGet()throws Exception {
    //创建对象
    CloseableHttpClient httpClient = HttpClients.createDefault();

    //创建get请求
    HttpGet httpGet = new HttpGet("http://news.baidu.com/");
    /** 设置超时时间、请求时间、socket时间都为5秒，允许重定向 */
//        RequestConfig requestConfig = RequestConfig.custom().setConnectTimeout(5000)
//                .setConnectionRequestTimeout(5000)
//                .setSocketTimeout(5000)
//                .setRedirectsEnabled(true)
//                .build();
//        httpGet.setConfig(requestConfig);
    //发送请求
    CloseableHttpResponse response = httpClient.execute(httpGet);
    //判断响应码为200
    if (response.getStatusLine().getStatusCode()==200){
        String content = EntityUtils.toString(response.getEntity());
        System.out.println(content);
    }
}
~~~

### get有参

~~~java
public static void doGet(String url) throws Exception{
    //创建对象
    CloseableHttpClient httpClient = HttpClients.createDefault();

    //创建get请求
    HttpGet httpGet = new HttpGet(url);
    //发送请求
    CloseableHttpResponse response = httpClient.execute(httpGet);
    //判断响应码为200
    if (response.getStatusLine().getStatusCode()==200){
        String content = EntityUtils.toString(response.getEntity());
        System.out.println(content);
    }
}
~~~

### post无参

在执行完方法后需要释放连接，以下代码包括了释放连接

~~~java
public static void doPost()throws Exception{
    //创建对象
    CloseableHttpClient httpClient = HttpClients.createDefault();
    //创建post对象
    HttpPost httpPost = new HttpPost("http://news.baidu.com/");
    //发起请求
    CloseableHttpResponse response = null;
    try {
        //使用HttpClient发起请求
        response = httpClient.execute(httpPost);

        //判断响应状态码是否为200
        if (response.getStatusLine().getStatusCode() == 200) {
            //如果为200表示请求成功，获取返回数据
            String content = EntityUtils.toString(response.getEntity(), "UTF-8");
            //打印数据长度
            System.out.println(content);
        }

    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        //释放连接
        if (response == null) {
            try {
                response.close();
            } catch (IOException e) {

                e.printStackTrace();
            }
            httpClient.close();
        }
    }
}
~~~

### post有参

~~~java
public static void doPost(String url)throws Exception{
    //创建对象
    CloseableHttpClient httpClient = HttpClients.createDefault();
    //创建post对象
    HttpPost httpPost = new HttpPost(url);
    //创建存放参数的list集合
    List<NameValuePair> params = new ArrayList<>();
    params.add(new BasicNameValuePair("keys","java"));
    //创建表单数据Entity
    UrlEncodedFormEntity formEntity = new UrlEncodedFormEntity(params,"utf-8");
    //将表单数据设置到HTTPPost中
    httpPost.setEntity(formEntity);
    //发起请求
    CloseableHttpResponse response = httpClient.execute(httpPost);
    //判断状态码
    if (response.getStatusLine().getStatusCode()==200){
        String content = EntityUtils.toString(response.getEntity());
        System.out.println(content);
    }
}
~~~

## 2.2 连接池

如果每次请求都要创建HttpClient，会有频繁创建和销毁的问题，可以使用连接池来解决这个问题。

断点测试发现每个httpClient都不一样

~~~java
public class TestPool {
    public static void main(String[] args) {
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(200);//设置最大连接数
        cm.setDefaultMaxPerRoute(20);//设置每个主机的并发数
        doGet(cm);
        doGet(cm);
    }

    public static void doGet(PoolingHttpClientConnectionManager cm){
        CloseableHttpClient httpClient = HttpClients.custom().setConnectionManager(cm).build();

        HttpGet httpGet = new HttpGet("http://www.itcast.cn/");

        CloseableHttpResponse response = null;

        try {
            response = httpClient.execute(httpGet);

            // 判断状态码是否是200
            if (response.getStatusLine().getStatusCode() == 200) {
            // 解析数据
                String content = EntityUtils.toString(response.getEntity(), "UTF-8");
                System.out.println(content.length());
            }


        } catch (Exception e) {
            e.printStackTrace();
        } finally {
        //释放连接
            if (response == null) {
                try {
                    response.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                //不能关闭HttpClient
                //httpClient.close();
            }
        }
    }

}
~~~

## 2.3 请求参数

有时候因为网络，或者目标服务器的原因，请求需要更长的时间才能完成，我们需要自定义相关时间

~~~java
/** 设置超时时间、请求时间、socket时间都为5秒，允许重定向 */
//        RequestConfig requestConfig = RequestConfig.custom().setConnectTimeout(5000)//创建连接最长时间
//                .setConnectionRequestTimeout(5000)//获取连接的最长时间
//                .setSocketTimeout(5000)//数据传输最长时间
//                .setRedirectsEnabled(true)
//                .build();
//        httpGet.setConfig(requestConfig);
~~~

# 三、 Jsoup

​	jsoup 是一款Java 的HTML解析器，可直接解析某个URL地址、HTML文本内容。它提供了一套非常省力的API，可通过DOM，CSS以及类似于jQuery的操作方法来取出和操作数据。

​	jsoup主要功能如下：

1. 从一个url、文件、或字符串中解析html
2. 使用dom或css选择器来查找、取出数据
3. 可操作html元素属性文本

jsoup依赖：

~~~xml
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.10.3</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.7</version>
</dependency>
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.6</version>
</dependency>
~~~

## 3.1 jsoup解析

### 3.1.1 解析url

​	jsoup可以直接输入url，他会发起并获取数据封装为Document对象，虽然可以代替HTTPClient但是实际开发中需要使用多线程连接池代理等，jsoup对这些支持不好，所以一般将jsoup作为html解析工具。

~~~java
/**
* 测试jsoup解析url
* @throws Exception
*/
@Test
public void testJsoupUrl() throws Exception{
    //解析url地址
    Document document = Jsoup.parse(new URL("http://www.itcast.cn"),1000);
    //获取title
    Element title = document.getElementsByTag("title").first();
    System.out.println(title.text());
}
~~~

### 3.1.2 解析字符串

先准备一份简单的html，之后

~~~java
/**
* 测试jsoup解析字符串
* @throws Exception
*/
@Test
public void testJsoupString() throws Exception{
    //读取文件
    String file = FileUtils.readFileToString(new File("E:\\jsouptext.html"), "UTF-8");
    //解析字符串
    Document document = Jsoup.parse(file);
    //获取title
    Element title = document.getElementsByTag("title").first();
    System.out.println(title.text());
}
~~~

### 3.1.3 解析文件

​	document也可以直接解析文件

~~~java
    /**
     * 测试jsoup解析文件
     * @throws Exception
     */
    @Test
    public void testJsoupFile() throws Exception{
        //解析文件
        Document document = Jsoup.parse(new File("E:\\jsouptext.html"), "UTF-8");
        //获取title
        Element title = document.getElementsByTag("title").first();
        System.out.println(title.text());
    }
~~~

## 3.2 使用DOM遍历文档

**获取元素**

1. 根据id查询元素getElementById
2. 根据标签查询元素getElementByTag
3. 根据class查询元素getElementByClass
4. 根据属性查询元素getElementByAttribute

~~~java
/**
* 测试jsoupDOM获取元素
* @throws Exception
*/
@Test
public void testJsoupDOM() throws Exception{
    //解析文件
    Document document = Jsoup.parse(new File("E:\\jsouptext.html"), "UTF-8");
    //1. 根据id查询元素
    Element element1 = document.getElementById("city_bj");
    //2.根据标签获取元素
    Element element2 = document.getElementsByTag("title").first();
    //3.根据class获取元素
    Element element3 = document.getElementsByClass("s_name").last();
    //4.根据属性获取元素
    Element element4 = document.getElementsByAttribute("abc").first();
    Element element5 = document.getElementsByAttributeValue("class","city_con").first();
    System.out.println(element1.text());
    System.out.println(element2.text());
    System.out.println(element3.text());
    System.out.println(element4.text());
    System.out.println(element5.text());
}
~~~

**元素中获取数据**

1. 从元素中获取id
2. 从元素中获取className
3. 从元素中获取属性值attr
4. 从元素中获取所有属性值attributes
5. 从元素中获取文本内容text

~~~java
/**
* 测试jsoupDOM元素中获取数据
* @throws Exception
*/
@Test
public void testJsoupDOMData() throws Exception{
    //解析文件
    Document document = Jsoup.parse(new File("E:\\jsouptext.html"), "UTF-8");
    //获取element元素
    Element element = document.getElementById("test");
    //1.元素中获取id
    String str1 = element.id();
    //2.从元素中获取className
    String str2 = element.className();
    //3.从元素中获取属性值attr
    String str3 = element.attr("id");
    //4.从元素中获取所有属性值attributes
    String str4 = element.attributes().toString();
    //5.从元素中获取文本内容text
    String str5 = element.text();
    System.out.println(str1);
    System.out.println(str2);
    System.out.println(str3);
    System.out.println(str4);
    System.out.println(str5);

}
~~~

## 3.3 使用选择器语法查找元素

### 3.3.1 Selector选择器

tagname：通过标签查找元素：比如：span

#id：通过id查找元素，比如：#id1

.class：通过class名称查找元素：比如：.class_1

[attribute]：通过属性查找元素

[attr=value]：利用属性值查找元素，比如[class=s_name]

~~~java
/**
* 测试jsoup选择器
* @throws Exception
*/
@Test
public void testJsoupSelector() throws Exception{
    //解析文件
    Document document = Jsoup.parse(new File("E:\\jsouptext.html"), "UTF-8");
    //tagname: 通过标签查找元素，比如：span
    Elements span = document.select("span");
    for (Element element:span) {
        System.out.println(element.text());
    }
    System.out.println("##########");
    //#id: 通过ID查找元素，比如：#city_bjj
    String str = document.select("#city_bj").text();
    System.out.println(str);
    //.class: 通过class名称查找元素，比如：.class_a
    str = document.select(".class_a").text();
    System.out.println(str);
    //[attribute]: 利用属性查找元素，比如：[abc]
    str = document.select("[abc]").text();
    System.out.println(str);
    //[attr=value]: 利用属性值来查找元素，比如：[class=s_name]
    str = document.select("[class=s_name]").text();
    System.out.println(str);

    //组合使用
    //el#id: 元素+ID，比如： h3#city_bj
    str = document.select("h3#city_bj").text();

    //el.class: 元素+class，比如： li.class_a
    str = document.select("li.class_a").text();

    //el[attr]: 元素+属性名，比如： span[abc]
    str = document.select("span[abc]").text();

    //任意组合，比如：span[abc].s_name
    str = document.select("span[abc].s_name").text();

    //ancestor child: 查找某个元素下子元素，比如：.city_con li 查找"city_con"下的所有li
    str = document.select(".city_con li").text();

    //parent > child: 查找某个父元素下的直接子元素，
    //比如：.city_con > ul > li 查找city_con第一级（直接子元素）的ul，再找所有ul下的第一级li
    str = document.select(".city_con > ul > li").text();

    //parent > * 查找某个父元素下所有直接子元素.city_con > *
    str = document.select(".city_con > *").text();

}
~~~

# 四、 WebMagic

​	是一款Java爬虫框架，其底层是HttpClient和Jsoup。WebMagic项目代码分为核心和拓展两部分。核心部分是一个精简的、模块化的爬虫实现，而扩展部分则包括一些便利的、实用性的功能。

## 4.1 架构介绍

​	WebMagic的结构分为Downloader、PageProcessor、Scheduler、Pipeline四大组件，并由Spider将它们彼此组织起来。这四大组件对应爬虫生命周期中的下载、处理、管理和持久化等功能。WebMagic的设计参考了Scapy，但是实现方式更Java化一些。

​	而Spider则将这几个组件组织起来，让它们可以互相交互，流程化的执行，可以认为Spider是一个大的容器，它也是WebMagic逻辑的核心。 

**四个组件**

1. Downloader

   Downloader负责从互联网上下载页面，以便后续处理。WebMagic默认使用了Apache HttpClient作为下载工具。

2. PageProcessor

   PageProcessor负责解析页面，抽取有用信息，以及发现新的链接。WebMagic使用Jsoup作为HTML解析工具，并基于其开发了解析XPath的工具Xsoup。

3. Scheduler

   Scheduler负责管理待抓取的URL，以及一些去重的工作。WebMagic默认提供了JDK的内存队列来管理URL，并用集合来进行去重。也支持使用Redis进行分布式管理。

4. Pipeline

   Pipeline负责抽取结果的处理，包括计算、持久化到文件、数据库等。WebMagic默认提供了“输出到控制台”和“保存到文件”两种结果处理方案。Pipeline定义了结果保存的方式，如果你要保存到指定数据库，则需要编写对应的Pipeline。对于一类需求一般只需编写一个Pipeline。

**用于数据流转的对象**

1. Request

   Request是对URL地址的一层封装，一个Request对应一个URL地址。它是PageProcessor与Downloader交互的载体，也是PageProcessor控制Downloader唯一方式。除了URL本身外，它还包含一个Key-Value结构的字段extra。你可以在extra中保存一些特殊的属性，然后在其他地方读取，以完成不同的功能。例如附加上一个页面的一些信息等。

2. Page

   Page代表了从Downloader下载到的一个页面——可能是HTML，也可能是JSON或者其他文本格式的内容。Page是WebMagic抽取过程的核心对象，它提供一些方法可供抽取、结果保存等。

3. ResultItems

   ResultItems相当于一个Map，它保存PageProcessor处理的结果，供Pipeline使用。它的API与Map很类似，值得注意的是它有一个字段skip，若设置为true，则不应被Pipeline处理。

## 4.2 入门案例

 **加入依赖**

**配置文件**

**Java类**                                                                                                                                                                                                                                                        

## 4.3 WebMagic功能

### 4.3.1 实现PageProcessor

### 4.3.2 使用Pipeline保存结果

### 4.3.3 爬虫的配置启动和终止

## 4.4 爬虫分类

网络爬虫按照系统结构和实现技术，大致可以分为以下几种类型：通用网络爬虫、聚焦网络爬虫、增量式网络爬虫、深层网络爬虫。实际的网络爬虫系统通常是几种爬虫技术相结合实现的

### 4.4.1 通用网络爬虫

通用网络爬虫又称全网爬虫（Scalable Web Crawler），爬行对象从一些种子 URL 扩充到整个 Web，主要为门户站点搜索引擎和大型 Web 服务提供商采集数据。

这类网络爬虫的爬行范围和数量巨大，对于爬行速度和存储空间要求较高，对于爬行页面的顺序要求相对较低，同时由于待刷新的页面太多，通常采用并行工作方式，但需要较长时间才能刷新一次页面。

**简单的说就是互联网上抓取所有数据。**

### 4.4.2 聚焦网络爬虫

聚焦网络爬虫（Focused Crawler），又称主题网络爬虫（Topical Crawler），是指选择性地爬行那些与预先定义好的主题相关页面的网络爬虫。

和通用网络爬虫相比，聚焦爬虫只需要爬行与主题相关的页面，极大地节省了硬件和网络资源，保存的页面也由于数量少而更新快，还可以很好地满足一些特定人群对特定领域信息的需求。

**简单的说就是互联网上只抓取某一种数据。**

### 4.4.3 增量式网络爬虫

增量式网络爬虫（Incremental Web Crawler）是指对已下载网页采取增量式更新和只爬行新产生的或者已经发生变化网页的爬虫，它能够在一定程度上保证所爬行的页面是尽可能新的页面。

和周期性爬行和刷新页面的网络爬虫相比，增量式爬虫只会在需要的时候爬行新产生或发生更新的页面，并不重新下载没有发生变化的页面，可有效减少数据下载量，及时更新已爬行的网页，减小时间和空间上的耗费，但是增加了爬行算法的复杂度和实现难度。

**简单的说就是互联网上只抓取刚刚更新的数据。**

### 4.4.4 Deep Web爬虫

Web 页面按存在方式可以分为表层网页（Surface Web）和深层网页（Deep Web，也称 Invisible Web Pages 或 Hidden Web）。

表层网页是指传统搜索引擎可以索引的页面，以超链接可以到达的静态网页为主构成的 Web 页面。

**Deep Web 是那些大部分内容不能通过静态链接获取的、隐藏在搜索表单后的，只有用户提交一些关键词才能获得的 Web 页面。**



# 五、Nutch



















