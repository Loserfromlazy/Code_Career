# Jmeter的使用

## 一、下载与安装

登陆官网

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter1.jpg)

下载并解压

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter2.jpg)

解压后如下图所示：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter3.jpg)

进入bin文件夹，打开jmeter.bat进入软件主界面。语言可在选项中修改。如下图：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter4.jpg)

打开jmeter.bat的同时会也有cmd命令行窗口：

![image-20211108105345236](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211108105345236.png)

上面的意思就是：不要使用GUI运行压力测试，GUI仅用于压力测试的创建和调试；执行压力测试请不要使用GUI。使用下面的命令来执行测试：

```shell
 jmeter -n -t [jmx file] -l [results file] -e -o [Path to web report folder]
```

- [jmx file] 为测试计划文件路径
- [results file] 为测试结果文件路径
- [Path to web report folder] 为web报告保存路径。

例子：`jmeter -n -t testplan/RedisLock.jmx -l testplan/result/result.txt -e -o testplan/webreport`

##   二、快速入门

以`http:localhost:8080/goods/getAll`接口为例，使用Jmeter对其进行压力测试。

1. 新建一个线程组

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter5.jpg)

2. 设置线程组参数

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter6.jpg)

   其中第一个参数为线程数量，第二个参数为所有的线程在多少秒内启动，第三个参数为重复次数

3. 新增http请求默认值

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter7.jpg)

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter8.jpg)

   在线程组上新增http请求默认值，所有的请求都会使用设置的默认值。

4. 添加需要测试的http请求

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter9.jpg)

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter10.jpg)

   红色线会使用上一步的默认值

5. 新增监视器，用于查看结果

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter11.jpg)

6. 点击运行

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter12.jpg)
   
   **运行完后可以保存，在命令行执行命令进行测试并得到测试报告**

## 三、自定义变量&模拟多用户

### 3.1 创建带请求的参数

1. 新建一个http请求

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter9.jpg)

2. 设置请求路径，并添加参数；或者使用json数据

   ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/jmeter/jmeter13.jpg)

   点击运行，开始测试































