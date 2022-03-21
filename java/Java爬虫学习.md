# Java爬虫学习

转载请声明！！本文如有错误欢迎指正，感激不尽。

声明：爬虫有风险，学习需谨慎。切勿使用爬虫恶意爬取破坏他人项目或应用。

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

## 2.4 案例HttpClient分段下载

需求：需要下载一些库里的文件，文件以url形式存储，然后通过httpclient将其下载出来使用，但由于服务器很慢，导致一个300M左右的文件需要下载一个小时左右，非常慢，所以采用多线程分段下载对原有httpclient单线程下载进行修改。

原有httpclient单线程下载代码

~~~java
//查询数据库的数据，获取文件地址
List<TbData> tbData = tbDataMapper.selectList(null);
//创建网络请求
CloseableHttpClient httpClient = HttpClients.createDefault();
//创建计数器
AtomicInteger integer = new AtomicInteger(1);
//批量处理
tbData.forEach(data -> {
    //判断文件路径是否为空
    if (StringUtils.isBlank(data.getFileUrl())) {
        System.out.println("路径为空");
    }
    //构建所需的对象
    HttpGet httpGet = new HttpGet(data.getFileUrl());
    CloseableHttpResponse response = null;
    InputStream content = null;
    FileOutputStream fileOutputStream = null;
    try {
        //执行请求进行下载
        System.out.println("正在下载" + data.getName());
        response = httpClient.execute(httpGet);
        //判断响应码为200,即下载成功
        if (response.getStatusLine().getStatusCode() == 200) {
            //将文件保存到本地
            HttpEntity entity = response.getEntity();
            content = entity.getContent();
            File file = new File("D:\\test");
            fileOutputStream = new FileOutputStream(file + "\\" + data.getName() + "_vid" + data.getId() + ".apk");
            byte[] bytes = new byte[1024];
            while (true) {
                int read = content.read(bytes);
                if (read == -1) {
                    break;
                }
                fileOutputStream.write(bytes, 0, read);
            }
            System.out.println("第" + integer + "个文件下载完成");
        } else {
            System.out.println("第" + integer + "个文件获取连接失败,文件名" + adatapp.getName());
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    integer.addAndGet(1);
});
~~~

经了解，在http请求中，可以使用Range进行分段下载，所以可以创建多个线程，来分段下载。

> Range，是在 HTTP/1.1（http://www.w3.org/Protocols/rfc2616/rfc2616.html）里新增的一个 header field，也是现在众多号称多线程下载工具（如 FlashGet、迅雷等）实现多线程下载的核心所在。
>
> **Range**
>
> 用于请求头中，指定第一个字节的位置和最后一个字节的位置，一般格式：
>
> Range:(unit=first byte pos)-[last byte pos]
>
> **Content-Range**
>
> 用于响应头，指定整个实体中的一部分的插入位置，他也指示了整个实体的长度。在服务器向客户返回一个部分响应，它必须描述响应覆盖的范围和整个实体长度。一般格式：
>
> Content-Range: bytes (unit first byte pos) - [last byte pos]/[entity legth]
>
> **请求下载整个文件:**
>
> 1. GET /test.rar HTTP/1.1
> 2. Connection: close
> 3. Host: 116.1.219.219
> 4. Range: bytes=0-801 //一般请求下载整个文件是bytes=0- 或不用这个头
>
> **一般正常回应**
>
> 1. HTTP/1.1 200 OK
> 2. Content-Length: 801
> 3. Content-Type: application/octet-stream
> 4. Content-Range: bytes 0-800/801 //801:文件总大小
>
> 所以如果想多线程现在，可以像下面一样：
>
> 表示头500个字节：Range: bytes=0-499
> 表示第二个500字节：Range: bytes=500-999
>
> .......

代码实现：

首先创建DownloadUtil类

```java
/**
 * <p>
 * DownloadUtil
 * </p>
 *
 * @author Loserfromlazy
 * @since 2021/12/6
 */
public class DownloadUtil {
	//服务器地址
    private String serverUrl;
	//线程池
    private ExecutorService threadPool;
	//下载的文件存储的本地路径
    private String fileName;
    //线程下载成功标志
    private static int flag = 0;
    //线程计数同步辅助
    private CountDownLatch latch;

    public DownloadUtil(String serverPath, String localPath,String fileName) {
        this.serverUrl = serverPath;
        this.localUrl = localPath;
        this.fileName=fileName;
    }

    public DownloadUtil() {
    }
	/**
	* 下载方法
	*/
    public boolean executeDownload() throws IOException {
        try {
            //创建网络请求
            CloseableHttpClient httpClient = HttpClients.createDefault();
            //创建请求配置
            RequestConfig requestConfig = RequestConfig.custom()
                    .setConnectionRequestTimeout(5000).build();
            HttpGet httpGet = new HttpGet(serverUrl);
            httpGet.setHeader("Connection", "Keep-Alive");
            httpGet.setConfig(requestConfig);
            //获取请求状态
            CloseableHttpResponse response = httpClient.execute(httpGet);
            int code = response.getStatusLine().getStatusCode();
            if (code != 200 && code != 206) {
                System.out.println("路径为空");
                return false;
            }
            //获取文件大小
            long contentLength = response.getEntity().getContentLength();
            //创建RandomAccessFile，采用rwd模式
            RandomAccessFile raf = new RandomAccessFile(fileName, "rwd");
            //指定创建的文件的长度
            raf.setLength(contentLength);
            raf.close();
            //分割文件
            //当前线程池中线程数目，这里暂时写死
            int partCount = 10;
            //每一个线程下载的长度
            int partSize = (int) (contentLength / partCount);
            latch = new CountDownLatch(partCount);
            //获取线程池
            threadPool = ThreadPool.getInstance();
            for (int threadId = 1; threadId <= partCount; threadId++) {
                // 每一个线程下载的开始位置
                long startIndex = (threadId - 1) * partSize;
                // 每一个线程下载的开始位置
                long endIndex = startIndex + partSize - 1;
                if (threadId == partCount) {
                    //最后一个线程下载的长度稍微长一点
                    endIndex = contentLength;
                }
                System.out.println("线程" + threadId + "下载:" + startIndex + "字节~" + endIndex + "字节");
                //新建线程，通过线程池执行
                threadPool.execute(new DownLoadThread(threadId, startIndex, endIndex, latch,fileName));
            }
            //等待线程执行完毕
            latch.await();
            if (flag == 0) {
                return true;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 内部线程类
     *
     * @author Loserfromlazy
     * @date 2021/12/6 14:44
     */
    public class DownLoadThread implements Runnable {
        //线程ID
        private int threadId;
        //下载起始位置
        private long startIndex;
        //下载结束位置
        private long endIndex;
		//下载的文件存储的本地路径
        private String fileName;
		//线程计数辅助
        private CountDownLatch latch;

        public DownLoadThread(int threadId, long startIndex, long endIndex, CountDownLatch latch, String fileName) {
            this.threadId = threadId;
            this.startIndex = startIndex;
            this.endIndex = endIndex;
            this.latch = latch;
            this.fileName = fileName;
        }

        @Override
        public void run() {
            try {
                System.out.println("线程" + threadId + "正在下载...");
                //创建网络请求
                CloseableHttpClient httpClient = HttpClients.createDefault();
                //创建请求配置
                RequestConfig requestConfig = RequestConfig.custom()
                        .setConnectionRequestTimeout(5000).build();
                HttpGet httpGet = new HttpGet(serverUrl);
                httpGet.setHeader("Connection", "Keep-Alive");
                //请求服务器下载部分的文件的指定位置
                httpGet.setHeader("Range", "bytes=" + startIndex + "-" + endIndex);
                httpGet.setConfig(requestConfig);
                CloseableHttpResponse response = httpClient.execute(httpGet);
                int code = response.getStatusLine().getStatusCode();
                System.out.println("线程" + threadId + "请求返回code=" + code);
                InputStream is = response.getEntity().getContent();//返回资源
                RandomAccessFile raf = new RandomAccessFile(fileName, "rwd");
                //随机写文件的时候从哪个位置开始写
                raf.seek(startIndex);//定位文件
                int len = 0;
                byte[] buffer = new byte[1024];
                while ((len = is.read(buffer)) != -1) {
                    raf.write(buffer, 0, len);
                }
                is.close();
                raf.close();
                System.out.println("线程" + threadId + "下载完毕");
            } catch (Exception e) {
                //线程下载出错
                DownloadUtil.flag = 1;
                System.out.println(e.getMessage());
            } finally {
                //计数值减一
                latch.countDown();
            }
        }
    }

    /**
     * 下载文件执行器
     */
    public synchronized static boolean downLoad(String serverPath, String localPath,String fileName) {
        ReentrantLock lock = new ReentrantLock();
        lock.lock();
        DownloadUtil m = new DownloadUtil(serverPath, localPath,fileName);
        long startTime = System.currentTimeMillis();
        boolean flag = false;
        try {
            flag = m.executeDownload();
            long endTime = System.currentTimeMillis();
            if (flag) {
                System.out.println("文件下载结束,共耗时" + (endTime - startTime) + "ms");
                return true;
            }
            System.out.println("文件下载失败");
            return false;
        } catch (Exception ex) {
            System.out.println(ex.getMessage());
            return false;
        } finally {
            DownloadUtil.flag = 0; // 重置下载状态
            if (!flag) {
                File file = new File(localPath);
                file.delete();
            }
            lock.unlock();
        }
    }
}
```

获取线程池工具类：

```java
/**
 * <p>
 * ThreadPool
 * </p>
 *
 * @author Loserfromlzy
 * @since 2021/12/6
 */
public class ThreadPool {
	//创建固定大小线程池，生产环境或实际项目禁止使用以下方式创建线程池！！！这里
    private static final ExecutorService executorService = Executors.newFixedThreadPool(10);

    private ThreadPool(){}
	//饿汉式单例
    public static ExecutorService getInstance(){
        return executorService;
    }
}
```

### 拓展CountDownLatch和RandomAccessFile

**`CountDownLatch`**

CountDownLatch是在java1.5被引入，存在于java.util.cucurrent包下。

CountDownLatch这个类使一个线程等待其他线程各自执行完毕后再执行。它是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了。

它其中最主要使用的方法源码如下：

总的来说就是调用await进行等待，直到count降为0，而countDown可以使count减一。

```java
/**
 * Causes the current thread to wait until the latch has counted down to
 * zero, unless the thread is {@linkplain Thread#interrupt interrupted}.
 *
 * <p>If the current count is zero then this method returns immediately.
 *
 * <p>If the current count is greater than zero then the current
 * thread becomes disabled for thread scheduling purposes and lies
 * dormant until one of two things happen:
 * <ul>
 * <li>The count reaches zero due to invocations of the
 * {@link #countDown} method; or
 * <li>Some other thread {@linkplain Thread#interrupt interrupts}
 * the current thread.
 * </ul>
 *
 * <p>If the current thread:
 * <ul>
 * <li>has its interrupted status set on entry to this method; or
 * <li>is {@linkplain Thread#interrupt interrupted} while waiting,
 * </ul>
 * then {@link InterruptedException} is thrown and the current thread's
 * interrupted status is cleared.
 *
 * @throws InterruptedException if the current thread is interrupted
 *         while waiting
 */
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
/**
     * Decrements the count of the latch, releasing all waiting threads if
     * the count reaches zero.
     *
     * <p>If the current count is greater than zero then it is decremented.
     * If the new count is zero then all waiting threads are re-enabled for
     * thread scheduling purposes.
     *
     * <p>If the current count equals zero then nothing happens.
     */
    public void countDown() {
        sync.releaseShared(1);
    }
```

**`RandomAccessFile`**

> RandomAccessFile既可以读取文件内容，也可以向文件输出数据。同时，RandomAccessFile支持“随机访问”的方式，程序快可以直接跳转到文件的任意地方来读写数据。
>
> 由于RandomAccessFile可以自由访问文件的任意位置，**所以如果需要访问文件的部分内容，而不是把文件从头读到尾，使用RandomAccessFile将是更好的选择。**
>
> 与OutputStream、Writer等输出流不同的是，RandomAccessFile允许自由定义文件记录指针，RandomAccessFile可以不从开始的地方开始输出，因此RandomAccessFile可以向已存在的文件后追加内容。**如果程序需要向已存在的文件后追加内容，则应该使用RandomAccessFile。**
>
> RandomAccessFile的方法虽然多，但它有一个最大的局限，就是只能读写文件，不能读写其他IO节点。
>
> **RandomAccessFile的一个重要使用场景就是网络请求中的多线程下载及断点续传。**

RandomAccessFile一共有4种模式。

> **"r" : **   以只读方式打开。调用结果对象的任何 write 方法都将导致抛出 IOException。
>  **"rw":**    打开以便读取和写入。
>  **"rws":**  打开以便读取和写入。相对于 "rw"，"rws" 还要求对“文件的内容”或“元数据”的每个更新都同步写入到基础存储设备。
>  **"rwd" :**  打开以便读取和写入，相对于 "rw"，"rwd" 还要求对“文件的内容”的每个更新都同步写入到基础存储设备。

RandomAccessFile既可以读文件，也可以写文件，所以类似于InputStream的read()方法，以及类似于OutputStream的write()方法，RandomAccessFile都具备。除此之外，RandomAccessFile具备两个特有的方法，来支持其随机访问的特性。

> long getFilePointer( )：返回文件记录指针的当前位置
>
> void  seek(long pos )：将文件指针定位到pos位置

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

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>

<groupId>cn.xxx</groupId>
<artifactId>crawler-webmagic</artifactId>
<version>1.0-SNAPSHOT</version>

<dependencies>
<!--WebMagic-->
<dependency>
<groupId>us.codecraft</groupId>
<artifactId>webmagic-core</artifactId>
<version>0.7.4</version>
</dependency>
<dependency>
<groupId>us.codecraft</groupId>
<artifactId>webmagic-extension</artifactId>
<version>0.7.4</version>
</dependency>
</dependencies>

</project>

~~~

**配置文件**

WebMagic使用slf4j-log4j12作为slf4j的实现。

添加log4j.properties配置文件

```properties
log4j.rootLogger=INFO,A1

log4j.appender.A1=org.apache.log4j.ConsoleAppender
log4j.appender.A1.layout=org.apache.log4j.PatternLayout
log4j.appender.A1.layout.ConversionPattern=%-d{yyyy-MM-dd HH:mm:ss,SSS} [%t] [%c]-[%p] %m%n
```

**Java类**                                                                                                                                                                                                                           

```java
public class JobProcessor implements PageProcessor {

    public void process(Page page) {
        page.putField("author", page.getHtml().css("div.mt>h1").all());
        }
        private Site site = Site.me();
        public Site getSite() {
        return site;
    }

	public static void main(String[] args) {
            Spider.create(new JobProcessor())
    //初始访问url地址
    .addUrl("https://www.jd.com/moreSubject.aspx")
    .run();
    }
}
```

## 4.3 WebMagic功能

### 4.3.1 实现PageProcessor

#### 抽取元素Selectable

WebMagic主要使用了三种抽取技术：XPath、正则表达式和CSS选择器。另外，对于JSON格式的内容，可使用JsonPath进行解析。

1. XPath

   以上是获取属性class=mt的div标签，里面的h1标签的内容

   ```java
   page.getHtml().xpath("//div[@class=mt]/h1/text()")
   ```

2. CSS选择器

   CSS选择器是与XPath类似的语言。在之前已经学习了Jsoup的选择器，它比XPath写起来要简单一些，但是如果写复杂一点的抽取规则，就相对要麻烦一点。

   div.mt>h1表示class为mt的div标签下的直接子元素h1标签

   ```java
   page.getHtml().css("div.mt>h1").toString()
   ```

    

   可是使用:nth-child(n)选择第几个元素，如下选择第一个元素

   ```java
   page.getHtml().css("div#news_div > ul > li:nth-child(1) a").toString()
   ```

   注意：需要使用>，就是直接子元素才可以选择第几个元素

3. 正则表达式则是一种通用的文本抽取语言。在这里一般用于获取url地址。

#### 抽取元素API

Selectable相关的抽取元素链式API是WebMagic的一个核心功能。使用Selectable接口，可以直接完成页面元素的链式抽取，也无需去关心抽取的细节。

 在刚才的例子中可以看到，page.getHtml()返回的是一个Html对象，它实现了Selectable接口。这个接口包含的方法分为两类：抽取部分和获取结果部分。

| 方法                           | 说明                        | 示例                              |
| ------------------------------ | --------------------------- | --------------------------------- |
| xpath(String xpath)            | 使用xpath选择               | html.xpath("div[@class='title']") |
| $(String selector)             | 使用Css选择器选择           | html.$("div.title")               |
| $(String selector,String attr) | 使用Css选择器选择           | html.$("div.title","text")        |
| css(String selector)           | 功能同$(),使用css选择器选择 | html.css("div.title")             |
| links()                        | 选择所有链接                | html.links()                      |
| regex(String regex)            | 使用正则表达式抽取          | html.regex("\\(.\\*?)\\")         |

这部分抽取API返回的都是一个`Selectable`接口，意思是说，是支持链式调用的.

```java
//先获取class为news_div的div
//再获取里面的所有包含文明的元素
List<String> list = page.getHtml()
        .css("div#news_div")
        .regex(".*文明.*").all();
```

#### 抽取结果API

当链式调用结束时，我们一般都想要拿到一个字符串类型的结果。这时候就需要用到获取结果的API了。

我们知道，一条抽取规则，无论是XPath、CSS选择器或者正则表达式，总有可能抽取到多条元素。WebMagic对这些进行了统一，可以通过不同的API获取到一个或者多个元素。

| 方法       | 说明                             | 示例                               |
| ---------- | -------------------------------- | ---------------------------------- |
| get()      | 返回一条String类型的结果         | String link=html.links().get()     |
| toString() | 同get(),返回一条String类型的结果 | String link= html.links.toString() |
| all()      | 返回所有抽取的结果               | List links=html.links().all()      |

当有多条数据的时候，使用get()和toString()都是获取第一个url地址。

```java
String str = page.getHtml()
        .css("div#news_div")
        .links().regex(".*[0-3]$").toString();

String get = page.getHtml()
        .css("div#news_div")
        .links().regex(".*[0-3]$").get();
```

#### 获取链接

有了处理页面的逻辑，我们的爬虫就接近完工了，但是现在还有一个问题：一个站点的页面是很多的，一开始我们不可能全部列举出来，于是如何发现后续的链接，是一个爬虫不可缺少的一部分.

下面的例子就是获取https://www.jd.com/moreSubject.aspx这个页面中所有符合[https://www.jd.com/news.\\w+?.*](https://www.jd.com/news./w+?.*)正则表达式的url地址并将这些链接加入到待抓取的队列中去。

```java
public void process(Page page) {
    page.addTargetRequests(page.getHtml().links()
            .regex("(https://www.jd.com/news.\\w+?.*)").all());
System.out.println(page.getHtml().css("div.mt>h1").all());
}

public static void main(String[] args) {
    Spider.create(new JobProcessor())
            .addUrl("https://www.jd.com/moreSubject.aspx")
            .run();
}
```

### 4.3.2 使用Pipeline保存结果

WebMagic用于保存结果的组件叫做`Pipeline`。我们现在通过“控制台输出结果”这件事也是通过一个内置的Pipeline完成的，它叫做`ConsolePipeline`。

 那么，我现在想要把结果用保存到文件中，怎么做呢？只将Pipeline的实现换成"FilePipeline"就可以了。

```java
public static void main(String[] args) {
    Spider.create(new JobProcessor())
//初始访问url地址
.addUrl("https://www.jd.com/moreSubject.aspx")
  .addPipeline(new FilePipeline("D:/webmagic/"))
            .thread(5)//设置线程数
.run();
}
```

Pipeline的接口定义如下：

```java
public interface Pipeline {

// ResultItems保存了抽取结果，它是一个Map结构，
// 在page.putField(key,value)中保存的数据，
//可以通过ResultItems.get(key)获取
public void process(ResultItems resultItems, Task task);
}
```

可以看到，Pipeline其实就是将PageProcessor抽取的结果，继续进行了处理的，其实在Pipeline中完成的功能，你基本上也可以直接在PageProcessor实现，那么为什么会有Pipeline？有几个原因：

 1.为了模块分离

“页面抽取”和“后处理、持久化”是爬虫的两个阶段，将其分离开来，一个是代码结构比较清晰，另一个是以后也可能将其处理过程分开，分开在独立的线程以至于不同的机器执行。

 2.Pipeline的功能比较固定，更容易做成通用组件

每个页面的抽取方式千变万化，但是后续处理方式则比较固定，例如保存到文件、保存到数据库这种操作，这些对所有页面都是通用的。

 在WebMagic里，一个Spider可以有多个Pipeline，使用Spider.addPipeline()即可增加一个Pipeline。这些Pipeline都会得到处理，例如可以使用

```java
spider.addPipeline(new ConsolePipeline()).addPipeline(new FilePipeline())
```

实现输出结果到控制台，并且保存到文件的目标。

WebMagic中就已经提供了控制台输出、保存到文件、保存为JSON格式的文件几种通用的Pipeline。

| **类**                    | **说明**                         | **备注**                       |
| ------------------------- | -------------------------------- | ------------------------------ |
| ConsolePipeline           | 输出结果到控制台                 | 抽取结果需要实现toString方法   |
| FilePipeline              | 保存结果到文件                   | 抽取结果需要实现toString方法   |
| JsonFilePipeline          | JSON格式保存结果到文件           |                                |
| ConsolePageModelPipeline  | (注解模式)输出结果到控制台       |                                |
| FilePageModelPipeline     | (注解模式)保存结果到文件         |                                |
| JsonFilePageModelPipeline | (注解模式)JSON格式保存结果到文件 | 想持久化的字段需要有getter方法 |

### 4.3.3 爬虫的配置启动和终止

#### Spider

Spider是爬虫启动的入口。在启动爬虫之前，我们需要使用一个PageProcessor创建一个Spider对象，然后使用run()进行启动。

同时Spider的其他组件（Downloader、Scheduler、Pipeline）都可以通过set方法来进行设置。

| **方法**                  | **说明**                                         | **示例**                                                     |
| ------------------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| create(PageProcessor)     | 创建Spider                                       | Spider.create(new  GithubRepoProcessor())                    |
| addUrl(String…)           | 添加初始的URL                                    | spider .addUrl("http://webmagic.io/docs/")                   |
| thread(n)                 | 开启n个线程                                      | spider.thread(5)                                             |
| run()                     | 启动，会阻塞当前线程执行                         | spider.run()                                                 |
| start()/runAsync()        | 异步启动，当前线程继续执行                       | spider.start()                                               |
| stop()                    | 停止爬虫                                         | spider.stop()                                                |
| addPipeline(Pipeline)     | 添加一个Pipeline，一个Spider可以有多个Pipeline   | spider .addPipeline(new  ConsolePipeline())                  |
| setScheduler(Scheduler)   | 设置Scheduler，一个Spider只能有个一个Scheduler   | spider.setScheduler(new  RedisScheduler())                   |
| setDownloader(Downloader) | 设置Downloader，一个Spider只能有个一个Downloader | spider .setDownloader(  new SeleniumDownloader())            |
| get(String)               | 同步调用，并直接取得结果                         | ResultItems result = spider  .get("http://webmagic.io/docs/") |
| getAll(String…)           | 同步调用，并直接取得一堆结果                     | List<ResultItems>  results = spider   .getAll("http://webmagic.io/docs/", "http://webmagic.io/xxx") |

#### 爬虫配置site

Site.me()可以对爬虫进行一些配置配置，包括编码、抓取间隔、超时时间、重试次数等。在这里我们先简单设置一下：重试次数为3次，抓取间隔为一秒。

```java
private Site site = Site.me()
        .setCharset("UTF-8")//编码
.setSleepTime(1)//抓取间隔时间
.setTimeOut(1000*10)//超时时间
.setRetrySleepTime(3000)//重试时间
.setRetryTimes(3);//重试次数
```

站点本身的一些配置信息，例如编码、HTTP头、超时时间、重试策略等、代理等，都可以通过设置Site对象来进行配置。

| **方法**                 | **说明**                                  | **示例**                                                     |
| ------------------------ | ----------------------------------------- | ------------------------------------------------------------ |
| setCharset(String)       | 设置编码                                  | site.setCharset("utf-8")                                     |
| setUserAgent(String)     | 设置UserAgent                             | site.setUserAgent("Spider")                                  |
| setTimeOut(int)          | 设置超时时间，  单位是毫秒                | site.setTimeOut(3000)                                        |
| setRetryTimes(int)       | 设置重试次数                              | site.setRetryTimes(3)                                        |
| setCycleRetryTimes(int)  | 设置循环重试次数                          | site.setCycleRetryTimes(3)                                   |
| addCookie(String,String) | 添加一条cookie                            | site.addCookie("dotcomt_user","code4craft")                  |
| setDomain(String)        | 设置域名，需设置域名后，addCookie才可生效 | site.setDomain("github.com")                                 |
| addHeader(String,String) | 添加一条addHeader                         | site.addHeader("Referer","[https://github.com](https://github.com/)") |
| setHttpProxy(HttpHost)   | 设置Http代理                              | site.setHttpProxy(new  HttpHost("127.0.0.1",8080))           |

### 4.3.4 Scheduler组件

Scheduler是WebMagic中进行URL管理的组件。一般来说，Scheduler包括两个作用：

- 对待抓取的URL队列进行管理。
- 对已抓取的URL进行去重。

WebMagic内置了几个常用的Scheduler。如果你只是在本地执行规模比较小的爬虫，那么基本无需定制Scheduler。

| **类**                    | **说明**                                                     | **备注**                                                     |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| DuplicateRemovedScheduler | 抽象基类，提供一些模板方法                                   | 继承它可以实现自己的功能                                     |
| QueueScheduler            | 使用内存队列保存待抓取URL                                    |                                                              |
| PriorityScheduler         | 使用带有优先级的内存队列保存待抓取URL                        | 耗费内存较QueueScheduler更大，但是当设置了request.priority之后，只能使用PriorityScheduler才可使优先级生效 |
| FileCacheQueueScheduler   | 使用文件保存抓取URL，可以在关闭程序并下次启动时，从之前抓取到的URL继续抓取 | 需指定路径，会建立.urls.txt和.cursor.txt两个文件             |
| RedisScheduler            | 使用Redis保存抓取队列，可进行多台机器同时合作抓取            | 需要安装并启动redis                                          |

去重部分被单独抽象成了一个接口：DuplicateRemover，从而可以为同一个Scheduler选择不同的去重方式，以适应不同的需要，目前提供了两种去重方式。

| **类**                      | **说明**                                                  |
| --------------------------- | --------------------------------------------------------- |
| HashSetDuplicateRemover     | 使用HashSet来进行去重，占用内存较大                       |
| BloomFilterDuplicateRemover | 使用BloomFilter来进行去重，占用内存较小，但是可能漏抓页面 |

RedisScheduler是使用Redis的set进行去重，其他的Scheduler默认都使用HashSetDuplicateRemover来进行去重。

如果要使用BloomFilter，必须要加入以下依赖：

```xml
<!--WebMagic对布隆过滤器的支持-->
<dependency>
<groupId>com.google.guava</groupId>
<artifactId>guava</artifactId>
<version>16.0</version>
</dependency>
```

```java
public static void main(String[] args) {
    Spider.create(new JobProcessor())
//初始访问url地址
.addUrl("https://www.jd.com/moreSubject.aspx")
            .addPipeline(new FilePipeline("D:/webmagic/"))
            .setScheduler(new QueueScheduler()
                    .setDuplicateRemover(new BloomFilterDuplicateRemover(10000000)))//参数设置需要对多少条数据去重
            .thread(1)//设置线程数
.run();
}
```

修改public void process(Page page)方法，添加一下代码

```java
//每次加入相同的url，测试去重
page.addTargetRequest("https://www.jd.com/news.html?id=36480");
```

#### 三种去重方式

- HashSet

使用java中的HashSet不能重复的特点去重。优点是容易理解。使用方便。

缺点：占用内存大，性能较低。

- Redis去重

使用Redis的set进行去重。优点是速度快（Redis本身速度就很快），而且去重不会占用爬虫服务器的资源，可以处理更大数据量的数据爬取。

缺点：需要准备Redis服务器，增加开发和使用成本。

- 布隆过滤器（BloomFilter）

使用布隆过滤器也可以实现去重。优点是占用的内存要比使用HashSet要小的多，也适合大量数据的去重操作。

缺点：有误判的可能。没有重复可能会判定重复，但是重复数据一定会判定重复。

> 布隆过滤器 (Bloom Filter)是由Burton Howard Bloom于1970年提出，它是一种space efficient的概率型数据结构，用于判断一个元素是否在集合中。在垃圾邮件过滤的黑白名单方法、爬虫(Crawler)的网址判重模块中等等经常被用到。
>
> 哈希表也能用于判断元素是否在集合中，但是布隆过滤器只需要哈希表的1/8或1/4的空间复杂度就能完成同样的问题。布隆过滤器可以插入元素，但不可以删除已有元素。其中的元素越多，误报率越大，但是漏报是不可能的。

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

# 五、网页去重

在4.3.4中介绍了对url的去重，避免同样的url下载多次。其实不光url需要去重，我们对下载的内容也需要去重。

## 5.1 去重方案

**指纹码对比**

最常见的去重方案是生成文档的指纹门。例如对一篇文章进行MD5加密生成一个字符串，我们可以认为这是文章的指纹码，再和其他的文章指纹码对比，一致则说明文章重复。但是这种方式是完全一致则是重复的，如果文章只是多了几个标点符号，那仍旧被认为是重复的，这种方式并不合理。

 **BloomFilter**

这种方式就是我们之前对url进行去重的方式，使用在这里的话，也是对文章进行计算得到一个数，再进行对比，缺点和方法1是一样的，如果只有一点点不一样，也会认为不重复，这种方式不合理。

 **KMP算法**

KMP算法是一种改进的字符串匹配算法。KMP算法的关键是利用匹配失败后的信息，尽量减少模式串与主串的匹配次数以达到快速匹配的目的。能够找到两个文章有哪些是一样的，哪些不一样。

这种方式能够解决前面两个方式的“只要一点不一样就是不重复”的问题。但是它的时空复杂度太高了，不适合大数据量的重复比对。

 还有一些其他的去重方式：最长公共子串、后缀数组、字典树、DFA等等，但是这些方式的空复杂度并不适合数据量较大的工业应用场景。我们需要找到一款性能高速度快，能够进行相似度对比的去重方案

Google 的 simhash 算法产生的签名，可以满足要求。

## 5.2 SimHash

这里仅做介绍

> simhash是由 Charikar 在2002年提出来的，为了便于理解尽量不使用数学公式，分为这几步：
>
> **1、分词**，把需要判断文本分词形成这个文章的特征单词。
>
> **2、hash**，通过hash算法把每个词变成hash值，比如“美国”通过hash算法计算为 100101,“51区”通过hash算法计算为 101011。这样我们的字符串就变成了一串串数字。
>
> **3、加权**，通过 2步骤的hash生成结果，需要按照单词的权重形成加权数字串，“美国”的hash值为“100101”，通过加权计算为“4 -4 -4 4 -4 4”
>
> “51区”计算为“ 5 -5 5 -5 5 5”。
>
> **4、合并**，把上面各个单词算出来的序列值累加，变成只有一个序列串。“美国”的“4 -4 -4 4 -4 4”，“51区”的“ 5 -5 5 -5 5 5”把每一位进行累加，“4+5 -4+-5 -4+5 4+-5 -4+5 4+5”  ->  “9 -9 1 -1 1 9”
>
> **5、降维**，把算出来的“9 -9 1 -1 1 9”变成 0 1 串，形成最终的simhash签名。
>
> 我们把库里的文本都转换为simhash签名，并转换为long类型存储，空间大大减少。现在我们虽然解决了空间，但是如何计算两个simhash的相似度呢？
>
>  我们通过海明距离（Hamming distance）就可以计算出两个simhash到底相似不相似。两个simhash对应二进制（01串）取值不同的数量称为这两个simhash的海明距离。举例如下： 10101 和 00110 从第一位开始依次有第一位、第四、第五位不同，则海明距离为3。对于二进制字符串的a和b，海明距离为等于在a XOR b运算结果中1的个数（普遍算法）。

# 六、代理

有些网站不允许爬虫进行数据爬取，因为会加大服务器的压力。其中一种最有效的方式是通过ip+时间进行鉴别，因为正常人不可能短时间开启太多的页面，发起太多的请求。

我们使用的WebMagic可以很方便的设置爬取数据的时间。但是这样会大大降低我们爬取数据的效率，如果不小心ip被禁了，会让我们无法爬去数据，那么我们就有必要使用代理服务器来爬取数据.

## 6.1 代理服务器

代理（英语：Proxy），也称网络代理，是一种特殊的网络服务，允许一个网络终端（一般为客户端）通过这个服务与另一个网络终端（一般为服务器）进行非直接的连接。

提供代理服务的电脑系统或其它类型的网络终端称为代理服务器（英文：Proxy Server）。一个完整的代理请求过程为：客户端首先与代理服务器创建连接，接着根据代理服务器所使用的代理协议，请求对目标服务器创建连接、或者获得目标服务器的指定资源。

## 6.2 使用代理

WebMagic使用的代理APIProxyProvider。因为相对于Site的“配置”，ProxyProvider定位更多是一个“组件”，所以代理不再从Site设置，而是由HttpClientDownloader设置。

| **API**                                                      | **说明** |
| ------------------------------------------------------------ | -------- |
| HttpClientDownloader.setProxyProvider(ProxyProvider  proxyProvider) | 设置代理 |

ProxyProvider有一个默认实现：SimpleProxyProvider。它是一个基于简单Round-Robin的、没有失败检查的ProxyProvider。可以配置任意个候选代理，每次会按顺序挑选一个代理使用。它适合用在自己搭建的比较稳定的代理的场景。

如果需要根据实际使用情况对代理服务器进行管理（例如校验是否可用，定期清理、添加代理服务器等），只需要自己实现APIProxyProvider即可。

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import us.codecraft.webmagic.Page;
import us.codecraft.webmagic.Site;
import us.codecraft.webmagic.Spider;
import us.codecraft.webmagic.downloader.HttpClientDownloader;
import us.codecraft.webmagic.processor.PageProcessor;
import us.codecraft.webmagic.proxy.Proxy;
import us.codecraft.webmagic.proxy.SimpleProxyProvider;

@Component
public class ProxyTest implements PageProcessor {
        
        @Scheduled(fixedDelay = 10000)
        public void testProxy() {
            HttpClientDownloader httpClientDownloader = new HttpClientDownloader();
            httpClientDownloader.setProxyProvider(SimpleProxyProvider.from(new Proxy("39.137.77.68",80)));
            
            Spider.create(new ProxyTest())
                    .addUrl("xxx")//设置请求地址
                    .setDownloader(httpClientDownloader)
                    .run();
        }
        
        @Override
        public void process(Page page) {
                //打印获取到的结果以测试代理服务器是否生效
                System.out.println(page.getHtml());
        }
        
        private Site site = new Site();
        @Override
        public Site getSite() {
        return site;
        }
}


```

