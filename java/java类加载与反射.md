# 类的加载机制与反射

## 类的加载连接和初始化

当我们调用Java命令运行某个Java程序时，该命令将会启动一个Java虚拟机进程，不管该Java程序有多么复杂，该程序启动了多少个线程，它们都处于该Java虚拟机进程里。正如前面介绍的，同一个JVM的所有线程、所有变量都处于同一个进程里，它们都使用该JVM进程的内存区。当系统出现以下几种情况时，JVM进程将被终止。

程序运行到最后正常结束。

程序运行到使用System.exit()或Runtime.getRuntime().exit()代码处结束程序。

程序执行过程中遇到未捕获的异常或错误而结束。

程序所在平台强制结束了JVM进程。

示例：

~~~java
A.java
public class A{
	public static int a=6;
}
Atest.java
public class ATest1{
	main(){
		A a=new A();
		a.a++;
		sout(a.a);
	}
}
public class ATest2{
	main(){
		A b=new A();
		sout(b.a);
	}
}
~~~

ATest1输出7。ATest2时输出6而不是7.因为两次运行处于两个不同的JVM进程中。

### **类的加载**

当程序主动使用某个类时，如果该类还未被加载到内存中，则系统会通过加载、连接、初始化3个步骤来对该类进行初始化。如果没有意外，JVM将会连续完成这3个步骤，所以有时也把这3个步骤统称为类加载或类初始化。

类加载指的是将类的class文件读入内存，并为之创建一个java.lang.Class对象，系统中所有的类实际上也是实例，它们都是java.lang.Class的实例。

类的加载由类加载器完成，类加载器通常由JVM提供，这些类加载器也是所有程序运行的基础，JVM提供的这些类加载器通常被称为系统类加载器。除此之外，开发者可以通过继承ClassLoader基类来创建自己的类加载器。通过使用不同的类加载器，可以从不同来源加载类的二进制数据，通常有如下几种来源。

 从本地文件系统加载class文件，这是绝大部分程序的类加载方式。

从JAR包加载class文件，这种方式也是很常见的，比如JDBC编程时用到的数据库驱动类就放在JAR文件中

通过网络加载class文件。

把一个Java源文件动态编译，并执行加载。

### **类的连接**

当类被加载之后，系统为之生成一个对应的Class对象，接着将会进入连接阶段，连接阶段负责把类的二进制数据合并到JRE中。类连接又可分为如下3个阶段。

（1）验证：验证阶段用于检验被加载的类是否有正确的内部结构，并和其他类协调一致。

（2）准备：类准备阶段则负责为类的静态Field分配内存，并设置默认初始值。

（3）解析：将类的二进制数据中的符号引用替换成直接引用。

### **类的初始化**

在类的初始化阶段，虚拟机负责对类进行初始化，主要就是对静态Field进行初始化。在Java类中对静态Field指定初始值有两种方式：① 声明静态Field时指定初始值；② 使用静态初始化块为静态Field指定初始值

~~~java
public class Test{
    //声明时初始化
    static int a =5;
    static int b;
    static int c;//没有指定初始值，默认采用0
    //使用静态初始化块
    static{
        b=6;
    }
}
~~~

> **正常类的加载顺序：静态变量/静态代码块 -> main方法 -> 非静态变量/代码块 -> 构造方法**
>
> **静态代码块与静态变量的执行顺序同代码定义的顺序；非静态变量与代码块的执行顺序同代码执行顺序**

JVM初始化一个类包含如下几个步骤。

（1）假如这个类还没有被加载和连接，则程序先加载并连接该类。

（2）假如该类的直接父类还没有被初始化，则先初始化其直接父类。

（3）假如类中有初始化语句，则系统依次执行这些初始化语句。

### **初始化类的时机**

当Java程序首次通过下面6种方式来使用某个类或接口时，系统就会初始化该类或接口：

1.创建类的实例。为某个类创建实例的方式包括：使用new操作符来创建实例，通过反射来创建实例，通过反序列化的方式来创建实例。

2.调用某个类的静态方法。

3.访问某个类或接口的静态Field，或为该静态Field赋值。

4.使用反射方式来强制创建某个类或接口对应的java.lang.Class对象。例如代码：Class.forName("Person")，如果系统还未初始化Person类，则这行代码将会导致该Person类被初始化，并返回Person类对应的java.lang.Class对象。

5.初始化某个类的子类。当初始化某个类的子类时，该子类的所有父类都会被初始化。

6.直接使用java.exe命令来运行某个主类。当运行某个主类时，程序会先初始化该主类。

PS：*当某个静态Field使用了final修饰，而且它的值可以在编译时就确定下来，那么程序其他地方使用该静态Field时，实际上并没有使用该静态Field，而是相当于使用常量。*

> static final String str="aaa";

*当使用ClassLoader类的loadClass()方法来加载某个类时，该方法只是加载该类，并不会执行该类的初始化。使用Class的forName()静态方法才会导致强制初始化该类。*

> CLassLoader cl =ClassLoader.getSystemClassLoader();
>
> cl.loadClass("Test");//只加载
>
> Class.forName("Test");//初始化

## 类加载器

类加载器负责将.class文件（可能在磁盘上，也可能在网络上）加载到内存中，并为之生成对应的java.lang.Class对象。一旦一个类被载入JVM中，同一个类就不会被再次载入了。

在Java中，一个类用其全限定类名（包括包名和类名）作为标识；但在JVM中，一个类用其全限定类名和其类加载器作为其唯一标识。

> 例如，如果在pg的包中有一个名为Person的类，被类加载器ClassLoader的实例kl负责加载，则该Person类对应的Class对象在JVM中表示为（Person、pg、kl）。这意味着两个类加载器加载的同名类：（Person、pg、kl）和（Person、pg、kl2）是不同的、它们所加载的类也是完全不同、互不兼容的。

当JVM启动时，会形成由3个类加载器组成的初始类加载器层次结构。

Bootstrap ClassLoader：根类加载器。

> Bootstrap ClassLoader被称为引导（也称为原始或根）类加载器，它负责加载Java的核心类String、System这些核心类库都在E:/Java/jdk1.7.0/jre/lib/rt.jar文件中

Extension ClassLoader：扩展类加载器。

> 它负责加载JRE的扩展目录（%JAVA_HOME%/jre/lib/ext或者由java.ext.dirs系统属性指定的目录）中JAR包的类。通过这种方式，就可以为Java扩展核心类以外的新功能，只要我们把自己开发的类打包成JAR文件，然后放入JAVA_HOME/jre/lib/ext路径即可

System ClassLoader：系统类加载器。

> 它负责在JVM启动时加载来自java命令的-classpath选项、java.class.path系统属性，或CLASSPATH环境变量所指定的JAR包和类路径。程序可以通过ClassLoader的静态方法getSystemClassLoader()来获取系统类加载器。如果没有特别指定，则用户自定义的类加载器都以类加载器作为父加载器。

示例：

```java
public static void main(String[] args) throws Exception {
       ClassLoader systemLoader=ClassLoader.getSystemClassLoader();
        System.out.println("系统类加载器"+systemLoader);
        /*获取系统类加载器的加载路径-*/
        Enumeration<URL> eml = systemLoader.getResources("");
        while (eml.hasMoreElements()){
            System.out.println(eml.nextElement());
        }
        //获取系统类加载器的父类加载器得到拓展类加载器
        ClassLoader extensionLoader =systemLoader.getParent();
        System.out.println("扩展类加载器"+extensionLoader);
        System.out.println("扩展类加载器的加载路径"+System.getProperty("java.ext.dirs"));
        System.out.println("扩展类加载器的parent"+extensionLoader.getParent());

    }
```

运行结果：

> 系统类加载器sun.misc.Launcher$AppClassLoader@b4aac2
> file:/E:/IDEA/test/out/production/test/
> 扩展类加载器sun.misc.Launcher$ExtClassLoader@154617c
> 扩展类加载器的加载路径E:\java_se\jdk1.8.0_101\jre\lib\ext;C:\WINDOWS\Sun\Java\lib\ext
> 扩展类加载器的parentnull

系统类加载器的加载路径是程序运行的当前路径，扩展类加载器的加载路径E:\java_se\jdk1.8.0_101\jre\lib\ext，但我们看到扩展类加载器的父加载器是null，并不是根类加载器。这是因为根类加载器并没有继承ClassLoader抽象类，所以扩展类加载器的getParent()方法返回null。但实际上，扩展类加载器的父类加载器是根类加载器，只是根类加载器并不是Java实现的。系统类加载器是AppClassLoader的实例，扩展类加载器是ExtClassLoader的实例。实际上，这两个类都是URLClassLoader类的实例。

> JVM的根类加载器并不是Java实现的，而且由于程序通常无须访问根类加载器，因此访问扩展类加载器的父类加载器时返回null。

### **类加载机制**

JVM的类加载机制主要有如下3种。

全盘负责。所谓全盘负责，就是当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显式使用另外一个类加载器来载入。

父类委托。所谓父类委托，则是先让parent（父）类加载器试图加载该Class，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。

缓存机制。缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区中搜寻该Class，只有当缓存区中不存在该Class对象时，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区中。这就是为什么修改了Class后，必须重新启动JVM，程序所做的修改才会生效的原因。

除此之外，开发者可以实现自己的类加载器，自定义类加载器通过ClassLoader来实现。

JVM中这4种类加载器的层次结构:

![img](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/2937711-a0cfabf31ece41c5.png)

类加载器加载Class大致要经过如下8个步骤。

（1）检测此Class是否载入过（即在缓存区中是否有此Class），如果有则直接进入第8步，否则接着执行第2步。（2）如果父类加载器不存在（如果没有父类加载器，则要么parent一定是根类加载器，要么本身就是根类加载器），则跳到第4步执行；如果父类加载器存在，则接着执行第3步。

（3）请求使用父类加载器去载入目标类，如果成功载入则跳到第8步，否则接着执行第5步。

（4）请求使用根类加载器来载入目标类，如果成功载入则跳到第8步，否则跳到第7步。

（5）当前类加载器尝试寻找Class文件（从与此ClassLoader相关的类路径中寻找），如果找到则执行第6步，如果找不到则跳到第7步。

（6）从文件中载入Class，成功载入后跳到第8步。

（7）抛出ClassNotFoundException异常。

（8）返回对应的java.lang.Class对象。其中，第5、6步允许重写ClassLoader的findClass()方法来实现自己的载入策略，甚至重写loadClass()方法来实现自己的载入过程。

### **创建并使用自定义的类加载器：**

JVM中除根类加载器之外的所有类加载器都是ClassLoader子类的实例，开发者可以通过扩展ClassLoader的子类，并重写该ClassLoader所包含的方法来实现自定义的类加载器。ClassLoader类有如下两个关键方法：

~~~java
loadClass(String name, boolean resolve)//该方法为ClassLoader的入口点，根据指定的二进制名称来加载类，系统就是调用ClassLoader的该方法来获取指定类对应的Class对象。 
findClass(String name)//根据二进制名称来查找类。
~~~

如果需要实现自定义的ClassLoader，则可以通过重写以上两个方法来实现，当然我们推荐重写findClass()方法，而不是重写loadClass()方法。loadClass()方法的执行步骤如下：

（1）用findLoadedClass(String) 来检查是否已经加载类，如果已经加载则直接返回。

（2）在父类加载器上调用loadClass()方法。如果父类加载器为null，则使用根类加载器来加载。

（3）调用findClass(String)方法查找类。

重写findClass()方法可以避免覆盖默认类加载器的父类委托、缓冲机制两种策略；如果重写loadClass()方法，则实现逻辑更为复杂。

**注意**

> 在ClassLoader里还有一个核心方法：Class defineClass(String name, byte[] b, int off, intlen)，该方法负责将指定类的字节码文件（即Class文件，如Hello.class）读入字节数组byte[] b内，并把它转换为Class对象，该字节码文件可以来源于文件、网络等。defineClass()方法管理JVM的许多复杂的实现，它负责将字节码分析成运行时数据结构，并校验有效性等。该方法是final型，不能重写

示例：

注意注释1的代码原来是`Process p=Runtime.getRuntime().exec("javac "+javaFile);`

如果你的编码方式是utf-8而不是默认的GBK就会报错无法编译（大部分人的编译器会调整为utf-8）

~~~java
public class CompileClassLoader extends ClassLoader{
    //读取文件的内容
    private  byte[] getBytes(String filename) throws IOException{
        File file=new File(filename);
        long len=file.length();
        byte[] raw=new byte[(int)len];
        try(FileInputStream fin =new FileInputStream(file);) {
            int r = fin.read(raw);
            if (r != len) throw new IOException("无法读取全部文件" + r + "!=" + len);
            return raw;
        }
    }
    //定义编译指定Java文件的方法
    private boolean compile(String javaFile)throws IOException{
        System.out.println("CompileClassLoader:正在编译"+javaFile+"...");
        //调用javac命令
        Process p=Runtime.getRuntime().exec("javac -encoding utf-8 "+javaFile);//1
        try{
            //其他进程等待这个进程完成
            p.waitFor();
        }catch (InterruptedException ie){
            System.out.println(ie);
        }
        //获取javac线程的退出值
        int ret=p.exitValue();
        return ret==0;
    }
    //重写ClassLoader的findClass方法
    protected Class<?> findClass(String name)throws ClassNotFoundException{
        Class clazz=null;
        //将包路径中的点替换成斜线
        String fileStub=name.replace(".","/");
        String javaFilename=fileStub+".java";
        String classFilename=fileStub+".class";
        File javaFile=new File(javaFilename);
        File classFile=new File(classFilename);
        /* 当指定Java源文件存在，且Class文件不存在；或者Java源文件的修改时间比Class文件的修改时间更晚时，重新编译*/
        if (javaFile.exists()&&(!classFile.exists()||javaFile.lastModified()>classFile.lastModified())){
            try{
                if(!compile(javaFilename)||!classFile.exists()){
                    throw new ClassNotFoundException("ClassNotFoundException"+javaFilename);
                }
            }catch (IOException e){
                e.printStackTrace();
            }
        }
        //如果Class文件存在，系统负责将该文件转换成Class对象
        if(classFile.exists()){
            try{
                //将Class文件的二进制数据读入数组
                byte []raw =getBytes(classFilename);
                //将二进制数据转换Class对象
                clazz=defineClass(name,raw,0,raw.length);
            }catch (IOException e){
                e.printStackTrace();
            }
        }
        //如果clazz为null，辨明加载失败，抛出异常
        if(clazz == null){
            throw new ClassNotFoundException(name);
        }
        return clazz;

    }
    //主方法
    public static void main(String[] args) throws Exception {
        //如果运行时没有参数，即没有目标类
        if (args.length<1){
            System.out.println("缺少目标类，请按如下格式运行Java文件");
            System.out.println("java CompileClassLoader ClassName");
        }
        //第一个参数是需要运行的类
        String progClass=args[0];
        //剩下的参数将作为运行目标类是的参数，将这些参数复制到一个新数组中
        String [] progArgs=new String[args.length-1];
        System.arraycopy(args,1,progArgs,0,progArgs.length);
        CompileClassLoader ccl=new CompileClassLoader();
        //加载需要运行的类
        Class<?> clazz=ccl.loadClass(progClass);
        //获取主方法
        Method main =clazz.getMethod("main",(new String[0]).getClass());
        Object argsArray[] ={progArgs};
        main.invoke(null,argsArray);
    }
}
~~~

~~~java
public class Hello {
    public static void main(String[] args) {
        for (String arg:args){
            System.out.println("运行Hello的参数"+arg);
        }
    }
}
~~~

无需编译Hello类即可直接运行

再cmd中首先编译自定义类加载器

`javac -encoding utf-8 CompileClassLoader.java `

然后无须编译该Hello.java，可以直接使用如下命令来运行该Hello.java程序。

`java CompileClassLoader Hello aaa bbb`

结果：

> E:\IDEA\test\src>java CompileClassLoader Hello aaa 啊啊啊
> 运行Hello的参数aaa
> 运行Hello的参数啊啊啊

实际上，使用自定义的类加载器，可以实现如下常见功能。

执行代码前自动验证数字签名。

根据用户提供的密码解密代码，从而可以实现代码混淆器来避免反编译Class文件。

根据用户需求来动态地加载类。

根据应用需求把其他数据以字节码的形式加载到应用中。

### URLClassLoader类

Java为ClassLoader提供了一个URLClassLoader实现类，该类也是系统类加载器和扩展类加载器的父类（此处是父类，就是指类与类之间的继承关系）。URLClassLoader功能比较强大，它既可以从本地文件系统获取二进制文件来加载类，也可以从远程主机获取二进制文件来加载类。

在应用程序中可以直接使用URLClassLoader来加载类，URLClassLoader类提供了如下两个构造器。

URLClassLoader(URL[] urls)：使用默认的父类加载器创建一个ClassLoader对象，该对象将从urls所指定的系列路径来查询并加载类。

URLClassLoader(URL[] urls, ClassLoader parent)：使用指定的父类加载器创建一个ClassLoader对象，其他功能与前一个构造器相同。

示例：

~~~java
private static Connection conn;
    public static Connection getConn(String url,String user,String pass) throws Exception{
        if (conn==null) {
            URL[] urls = {new URL("file:mysql-connector-java-x.x.x-bin.jat")};//当前路径
            URLClassLoader urlClassLoader = new URLClassLoader(urls);
            //加载MYSQL的JDBC驱动
            Driver driver = (Driver) urlClassLoader.loadClass("com.mysql.jdbc.Driver").newInstance();
            Properties properties = new Properties();
            properties.setProperty("user", user);
            properties.setProperty("password", pass);
            conn = driver.connect(url, properties);
        }
        return conn;
    }
~~~

## 反射查看类信息

Java程序中的许多对象在运行时都会出现两种类型：编译时类型和运行时类型，例如代码：Person p=new Student();，这行代码将会生成一个p变量，该变量的编译时类型为Person，运行时类型为Student；

为了解决这些问题，程序需要在运行时发现对象和类的真实信息。为了解决这个问题，我们有以下两种做法。

第一种做法是假设在编译时和运行时都完全知道类型的具体信息，在这种情况下，我们可以直接先使用instanceof运算符进行判断前一个对象是否是后一个对象的实例（`a instanceof Math`），再利用强制类型转换将其转换成其运行时类型的变量即可。

第二种做法是编译时根本无法预知该对象和类可能属于哪些类，程序只依靠运行时信息来发现该对象和类的真实信息，这就必须使用反射。

第二种方式有如下两种优势:

 代码更安全。程序在编译阶段就可以检查需要访问的Class对象是否存在。

程序性能更好。因为这种方式无须调用方法，所以性能更好。

**从Class中获取信息**

Class提供的功能非常丰富，它可以获取该类里包含的构造器、方法、内部类、注释等信息，也可以获取该类所包括的属性（Field）信息——通过getFields()或getField(String name)方法即可。

具体方法清查阅API，以下为实例：

~~~java
@SuppressWarnings(value = "unchecked")
@Deprecated
public class ClassTest {
    private ClassTest(){}

    public ClassTest(String name){
        System.out.println("执行有参数的构造器");
    }

    public void info(){
        System.out.println("执行无参数的info方法");
    }
    public  void info(String str){
        System.out.println("执行有参数的info方法"+" 参数值为： "+str);
    }
    //测试用内部类
    class  Inner{}

    public static void main(String[] args) throws Exception{
        Class<ClassTest> clazz=ClassTest.class;
        //获取Class对象对应的类的全部构造器
        Constructor<?>[] constructors = clazz.getDeclaredConstructors();
        for(Constructor c:constructors){
            System.out.println(c);
        }
        //获取全部public构造器
        Constructor<?>[] publicConstructors = clazz.getConstructors();
        System.out.println("全部public构造器如下：");
        for (Constructor c:publicConstructors){
            System.out.println(c);
        }
        //获取全部public方法
        Method[] methods = clazz.getMethods();
        System.out.println("ClassTest的全部public方法");
        for (Method md:methods){
            System.out.println(md);
        }
        //获取指定方法
        System.out.println("带一个字符串参数的info方法"+clazz.getMethod("info", String.class));
        //获取全部注释
        Annotation[] annotations = clazz.getAnnotations();
        System.out.println("ClassTest的全部Annotation：");
        for (Annotation a:annotations){
            System.out.println(a);
        }
        System.out.println("该类上的@SuppressWarning注释为："+clazz.getAnnotation(SuppressWarnings.class));
        //获取全部内部类
        Class<?>[] inners = clazz.getDeclaredClasses();
        System.out.println("全部内部类：");
        for (Class c:inners){
            System.out.println(c);
        }
        //使用Class.forName()方法加载Inner内部类
        Class inClazz=Class.forName("ClassTest$Inner");
        //通过getDeclaringClass()访问外部类
        System.out.println("inClazz对应的外部类："+inClazz.getDeclaringClass());
        System.out.println("包为"+clazz.getPackage());
        System.out.println("父类为："+clazz.getSuperclass());
    }
}
~~~

定义ClassTest类时使用了@SuppressWarnings注释，但程序运行时无法分析出该类里包含的该注释，这是因为@SuppressWarnings使用了@Retention(value=SOURCE)修饰，这表明@SuppressWarnings只能保存在源代码级别上，而通过ClassTest.class获取该类的运行时Class对象，所以程序无法访问到@SuppressWarnings注释。

## 反射生成并操作对象

### 创建对象

通过反射来生成对象有如下两种方式：

使用Class对象的newInstance()方法来创建该Class对象对应类的实例，这种方式要求该Class对象的对应类有默认构造器，而执行newInstance()方法时实际上是利用默认构造器来创建该类的实例。

先使用Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建该Class对象对应类的实例。通过这种方式可以选择使用指定的构造器来创建实例。

~~~java
public class test {
    private Map<String,Object> objectPool =new HashMap<>();
    //创建对象的方法
    private Object createObject(String clazzName) throws Exception{
        Class<?> clazz =Class.forName(clazzName);
        return clazz.newInstance();
    }
    //根据指定文件来初始化对象池，根据配置文件来创建对象
    public void initPool(String fileName)throws Exception{
        try(FileInputStream fileInputStream=new FileInputStream(fileName)){
            Properties properties=new Properties();
            properties.load(fileInputStream);
            for (String name:properties.stringPropertyNames()){
                //每取出一堆KV对，根据value创建一个对象，调用createObject创建对象，并将对象添加到对象池
                objectPool.put(name,createObject(properties.getProperty(name)));
            }
        }catch (IOException e){
            System.out.println("读取"+fileName+"异常");
        }

    }
    public Object getObject(String name){
        return objectPool.get(name);
    }
    public static void main(String[] args) throws Exception {
        test pf =new test();
        pf.initPool("obj.txt");
        System.out.println(pf.getObject("a"));
        System.out.println(pf.getObject("b"));
    }
}
obj.txt:
a=java.util.Date
b=javax.swing.JFrame
~~~

如果不想利用默认构造器来创建Java对象，而想利用指定的构造器来创建Java对象，则需要利用Constructor对象，每个Constructor对应一个构造器。为了利用指定的构造器来创建Java对象，需要如下3个步骤:

（1）获取该类的Class对象。

（2）利用Class对象的getConstructor()方法来获取指定的构造器。

（3）调用Constructor的newInstance()方法来创建Java对象。

~~~java
public static void main(String[] args) throws Exception {
        //获取JFrame对应的Class对象
       Class<?> jframeClazz=Class.forName("javax.swing.JFrame");
       //获取一个带字符串参数的构造器
        Constructor constructor=jframeClazz.getConstructor(String.class);
        Object obj=constructor.newInstance("测试");
        System.out.println(obj);
    }
~~~

### 调用方法

当获得某个类对应的Class对象后，就可以通过该Class对象的getMethods()方法或者getMethod()方法来获取全部方法或指定方法——这两个方法的返回值是Method数组，或者Method对象。

每个Method对象对应一个方法，获得Method对象后，程序就可通过该Method来调用它对应的方法。在Method里包含一个invoke()方法，该方法的签名如下。

Object invoke(Object obj, Object... args)：该方法中的obj是执行该方法的主调，后面的args是执行该方法时传入该方法的实参。

~~~java
//获取实现类的Class对象
Class<?> targetClass =target.getClass();
//获取对应的方法
Method method =targetClass.getMethod(methodName,String.class);
//通过invoke方法执行
method.invoke(target,config.getProperty(name))
~~~

当通过Method的invoke()方法来调用对应的方法时，Java会要求程序必须有调用该方法的权限。如果程序确实需要调用某个对象的private方法，则可以先调用Method对象的如下方法。

> setAccessible(boolean flag)：将Method对象的accessible设置为指定的布尔值。值为true，指示该Method在使用时应该取消Java语言的访问权限检查；值为false，则指示该Method在使用时要实施Java语言的访问权限检查。

### 访问属性值

通过Class对象的getFields()或getField()方法可以获取该类所包括的全部Field或指定Field。Field提供了如下两组方法来读取或设置Field值。

getXxx(Object obj)：获取obj对象该Field的属性值。此处的Xxx对应8个基本类型，如果该属性的类型是引用类型，则取消get后面的Xxx。

setXxx(Object obj , Xxx val)：将obj对象的该Field设置成val值。此处的Xxx对应8个基本类型，如果该属性的类型是引用类型，则取消set后面的Xxx。

使用这两个方法可以随意地访问指定对象的所有属性，包括private访问控制的属性。

~~~
Field [] getFields() 返回所有公共成员变量对象的数组
Field [] getDeclaredFields() 返回所有成员变量对象的数组
Field getField(String name) 返回单个公共成员变量对象
Field getDeclaredField(String name)返回单个成员变量对象
~~~

~~~java
public class test {
    public static void main(String[] args) throws Exception{
        Person p=new Person();
        Class<Person> personClass=Person.class;
        Field name = personClass.getDeclaredField("name");
        //设置通过反射访问该Field时取消访问权限检查
        name.setAccessible(true);
        name.set(p,"HaHa");
        Field age = personClass.getDeclaredField("age");
        age.setAccessible(true);
        age.set(p,30);
        System.out.println(p);
    }
}
class Person{
    private  String name;
    private int age;

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
~~~

### 操作数组

在java.lang.reflect包下还提供了一个Array类，Array对象可以代表所有的数组。程序可以通过使用Array来动态地创建数组，操作数组元素等。

Array提供了如下几类方法。

static Object newInstance(Class<?> componentType, int... length)：创建一个具有指定的元素类型、指定维度的新数组。

static xxx getXxx(Object array, int index)：返回array数组中第index个元素。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变为get(Object array, intindex)。

static void setXxx(Object array, int index, xxx val)：将array数组中第index个元素的值设为val。其中xxx是各种基本数据类型，如果数组元素是引用类型，则该方法变成set(Object array, intindex, Object val)。

~~~java
//创建一个元素类型为String，长度为10的数组
        Object arr= Array.newInstance(String.class,10);
        //为arr数组中index为5，6的元素赋值
        Array.set(arr,5,"aaa");
        Array.set(arr,6,"bbb");
        //依次取出index为5，6的元素的值
        Object b1=Array.get(arr,5);
        Object b2=Array.get(arr,6);
        System.out.println(b1);
        System.out.println(b2);
~~~

# 