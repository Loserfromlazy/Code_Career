# Web安全与加速学习笔记（未完结）

转载请声明作者和出处！

本文如有错误，欢迎指正，感激不尽。

# 一、HTTP

## 1.1 HTTP报文

如果说HTTP是因特网的信使，那么HTTP报文就是它用来搬东西的包裹了。HTTP报文是在HTTP应用程序之间发送的数据块。这些数据块以一些文本形式的元信息（meta-information）开头，这些信息描述了报文的内容及含义，后面跟着可选的数据部分。这些报文在客户端、服务器和代理之间流动。术语“流入”、“流出”、“上游”及“下游”都是用来描述报文方向的。

> 不管是请求报文还是响应报文，所有报文都会向下游流动。所有报文的发送者都在接收者的上游。

HTTP报文由起始行（start line）、首部（header）块，以及可选的、包含数据的主体（body）三部分组成。

所有的HTTP报文都可以分为两类：请求报文（request message）和响应报文（response message）。

### 1.1.1 报文的语法

所有的HTTP报文都可以分为两类：请求报文和响应报文。请求报文会向Web服务器请求一个动作。响应报文会将请求的结果返回给客户端。请求和响应报文的基本报文结构相同

请求报文格式：

~~~
<method> <request-URL> <version>
<headers>
<entity-body>
~~~

响应报文格式：

~~~
<version> <status> <reason-phrase>
<headers>
<entity-body>
~~~

- method（方法）

  客户端希望服务器对资源的动作，常见的有GET、POST

- 请求URL（request-URL）

  命名了资源的URL路径

- 版本（version）

  报文所使用的HTTP版本，格式：`HTTP/<major>.<minor>`例如：`HTTP /1.1`

- 
