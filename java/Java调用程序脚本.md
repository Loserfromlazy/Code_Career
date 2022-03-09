# Java调用程序脚本

在编写Java程序时，有时候我们需要调用其他的诸如exe,shell这样的程序或脚本。在Java中提供了两种方法来启动其他程序：

1. 使用Runtime的exec()方法
2. 使用ProcessBuilder的start()方法

 Runtime和ProcessBulider提供了不同的方式来启动程序，设置启动参数、环境变量和工作目录。但是这两种方法都会返回一个用于管理操作系统进程的Process对象。

## 1. Runtime

Java在执行Runtime.getRuntime().exec(command)之后，Linux会创建一个进程，该进程与JVM进程建立三个管道连接，标准输入流、标准输出流、标准错误流。下面的代码仅使用了标准输入流（这里的输入流是脚本程序输出的内容，java通过输入流读取）。

```java
@Test
public void test2(){
    try {
        String shpath = "XXXX\\test.bat";
        Process ps = Runtime.getRuntime().exec(shpath);
        ps.waitFor();

        BufferedReader br = new BufferedReader(new InputStreamReader(ps.getInputStream()));
        StringBuffer sb = new StringBuffer();
        String line;
        while ((line = br.readLine()) != null) {
            sb.append(line).append("\n");
        }
        String result = sb.toString();
        System.out.println(result);
    } catch (Exception e) {
        e.printStackTrace();
    }
    System.out.println("数据刷新成功");
}
```

### 1.1 waitFor()

上述代码有个很重要的方法就是waitFor()，在官方jdk文档中是这样对它解释的

~~~
Causes the current thread to wait, if necessary, until the process represented by this Process object has terminated. This method returns immediately if the subprocess has already terminated. If the subprocess has not yet terminated, the calling thread will be blocked until the subprocess exits.
如果需要，导致当前线程等待，直到这个process对象表示的进程结束。如果子流程已经终止，则此方法立即返回。如果子进程尚未终止，则调用线程将被阻塞，直到子进程退出。
~~~

也就是说只有子进程结束此方法才能返回。我们看下面的例子，这里我们打开notepad++：

```java
@Test
public void test3(){
    try {
        String shpath = "D:\\notepad++\\notepad++.exe";
        Process ps = Runtime.getRuntime().exec(shpath);
        ps.waitFor();

        BufferedReader br = new BufferedReader(new InputStreamReader(ps.getInputStream()));
        StringBuffer sb = new StringBuffer();
        String line;
        while ((line = br.readLine()) != null) {
            sb.append(line).append("\n");
        }
        String result = sb.toString();
        System.out.println(result);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

![image-20220309105815580](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220309105815580.png)

当我们运行时他会一直阻塞，这时如果我们关闭Runtime打开的notepad++，这个方法就会结束。

### 1.2 输入输出流

当Runtime对象调用exec(cmd)后，JVM会启动一个子进程，该进程会与JVM进程建立三个管道连接：标准输入，标准输出和标准错误流。

我们通过源码也可以看到这三个流：

![image-20220309110206591](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220309110206591.png)

这里有一点需要注意，假设该程序不断在向标准输出流和标准错误流写数据，而JVM不读取的话，当缓冲区满之后将无法继续写入数据，最终造成阻塞在waitfor()这里。

常见的解决方法如下，开两个线程分别读取标准输出流和标准错误流：

```java
@Test
public void test3(){
    try {
        String shpath = "C:\\Users\\yhr\\Desktop\\sign_linux\\test.bat";
        Process ps = Runtime.getRuntime().exec(shpath);
        //获取进程的标准输入流  
        final InputStream is1 = ps.getInputStream();
        //获取进城的错误流  
        final InputStream is2 = ps.getErrorStream();
        //启动两个线程，一个线程负责读标准输出流，另一个负责读标准错误流  
        new Thread() {
            public void run() {
                BufferedReader br1 = new BufferedReader(new InputStreamReader(is1));
                try {
                    String line1 = null;
                    while ((line1 = br1.readLine()) != null) {
                        if (line1 != null){}
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
                finally{
                    try {
                        is1.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

        new Thread() {
            public void  run() {
                BufferedReader br2 = new  BufferedReader(new  InputStreamReader(is2));
                try {
                    String line2 = null ;
                    while ((line2 = br2.readLine()) !=  null ) {
                        if (line2 != null){}
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
                finally{
                    try {
                        is2.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

        ps.waitFor();
        ps.destroy();
        System.out.println("j...");
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```











