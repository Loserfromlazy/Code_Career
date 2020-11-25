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

​	我们通常使用http协议访问网络上的资源，网络爬虫需要编写程序，这里使用Java的HTTP协议客户端HTTPClient这个技术编写程序。

## 2.1 get无参

## 2.2 get有参

## 2.3 post无参

## 2.4 post有参

## 2.5 连接池

## 2.6 请求参数



