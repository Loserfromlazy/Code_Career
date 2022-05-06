# Java高并发编程篇七NIO与Netty

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷1尼恩编著、以及个人博客等互联网资源
>
> 本文主要是对尼恩大佬的Java高并发核心编程卷1一书中的知识的学习记录以及对不懂的地方进行补充学习记录。
>
> **本文并不是对原书的照搬，而是对原书学习理解后重新编写代码并记录整理笔记和自己的理解，本文不会发博客，只会存在本人的Github的[Code_Career项目](https://github.com/Loserfromlazy/Code_Career)中**
>
> 由于原书很长所以分篇章来进行学习整理，本文是第七部分NIO和Netty篇。
>
> 此笔记中的例子全部是本人上机编写运行后的代码非原书中的代码例子。
>
> 此笔记中的图片非特殊标注全部是自己根据理解手画的或者是截图后二次编写的，请勿盗图。
>
> 部分源码不会全部展示，请自行去查阅源代码

由于之前学习过NIO和Netty，所以本文将结合自己的博文笔记（[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)）和Java高并发核心编程卷1进行查缺补漏和整理。

**本文会慢慢进行整理，因为之前整理过自己的笔记，所以二次整理会慢慢来，也有可能会跳章节进行整理，但是最终会完成整理的。在整理完之前可以看我的个人博文[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)**

> 未完结后续目录如下：
>
> 十三、Netty
>
> 十四、Decoder和Encoder
>
> 十五、序列化和反序列化

# 十、IO底层原理

## 10.1 IO读写的基本原理

### 10.1.1 IO读写原理

> 为了避免用户进程直接操作内核，保证内核安全，操作系统将内存（虚拟内存）划分为两部分：一部分是内核空间（Kernel-Space），另一部分是用户空间（User-Space）。
>
> 在Linux系统中，内核模块运行在内核空间，对应的进程处于内核态；用户程序运行在用户空间，对应的进程处于用户态。操作系统的核心是内核程序，它独立于普通的应用程序，既有权限访问受保护的内核空间，也有权限访问硬件设备，而普通的应用程序并没有这样的权限。每个应用程序进程都有一个单独的用户空间，对应的进程处于用户态，用户态进程不能访问内核空间中的数据，也不能直接调用内核函数，因此需要将进程切换到内核态才能进行系统调用。内核态进程可以执行任意命令，调用系统的一切资源，而用户态进程只能执行简单的运算，不能直接调用系统资源。
>
> **用户态进程必须通过系统调用（System Call）向内核发出指令，完成调用系统资源之类的操作。**
>
> 用户程序进行IO的读写依赖于底层的IO读写，基本上会用到底层的read和write两大系统调用。虽然在不同的操作系统中read和write两大系统调用的名称和形式可能不完全一样，但是它们的基本功能是一样的。
>
> 操作系统层面的read系统调用并不是直接从物理设备把数据读取到应用的内存中，write系统调用也不是直接把数据写入物理设备。上层应用无论是调用操作系统的read还是调用操作系统的write，都会涉及缓冲区。具体来说，**上层应用通过操作系统的read系统调用把数据从内核缓冲区复制到应用程序的进程缓冲区，通过操作系统的write系统调用把数据从应用程序的进程缓冲区复制到操作系统的内核缓冲区**。简单来说，应用程序的IO操作实际上不是物理设备级别的读写，而是缓存的复制。read和write两大系统调用都不负责数据在内核缓冲区和物理设备（如磁盘、网卡等）之间的交换。这个底层的读写交换操作是由操作系统内核（Kernel）来完成的。所以，在应用程序中，无论是对socket的IO操作还是对文件的IO操作，都属于上层应用的开发，它们在输入（Input）和输出（Output）维度上的执行流程是类似的，都是在内核缓冲区和进程缓冲区之间进行数据交换。



本小节总结

1. **用户态进程必须通过系统调用（System Call）向内核发出指令，完成调用系统资源之类的操作。**
2. **上层应用通过操作系统的read系统调用把数据从内核缓冲区复制到应用程序的进程缓冲区，通过操作系统的write系统调用把数据从应用程序的进程缓冲区复制到操作系统的内核缓冲区**

### 10.1.2 内核缓冲区与进程缓冲区

> **缓冲区的目的是减少与设备之间的频繁物理交换。**计算机的外部物理设备与内存和CPU相比，有着非常大的差距，外部设备的直接读写涉及操作系统的中断。发生系统中断时，需要保存之前的进程数据和状态等信息，结束中断之后，还需要恢复之前的进程数据和状态等信息。
>
> 为了减少底层系统的频繁中断所导致的时间损耗、性能损耗，出现了内核缓冲区。操作系统会对内核缓冲区进行监控，等待缓冲区达到一定数量的时候，再进行IO设备的中断处理，集中执行物理设备的实际IO操作，通过这种机制来提升系统的性能。至于具体什么时候执行系统中断（包括读中断、写中断）则由操作系统的内核来决定，应用程序不需要关心。
>
> 上层应用使用read系统调用时，仅仅把数据从内核缓冲区复制到应用的缓冲区（进程缓冲区）；上层应用使用write系统调用时，仅仅把数据从应用的缓冲区复制到内核缓冲区。内核缓冲区与应用缓冲区在数量上也不同。
>
> 在Linux系统中，操作系统内核只有一个内核缓冲区。每个用户程序（进程）都有自己独立的缓冲区，叫作用户缓冲区或者进程缓冲区。在大多数情况下，Linux系统中用户程序的IO读写程序并没有进行实际的IO操作，而是在用户缓冲区和内核缓冲区之间直接进行数据的交换。

本小节总结：

1. **操作系统内核只有一个内核缓冲区。每个用户程序（进程）都有自己独立的缓冲区，叫作用户缓冲区或者进程缓冲区**
2. **上层应用使用read系统调用时，仅仅把数据从内核缓冲区复制到应用的缓冲区（进程缓冲区）；上层应用使用write系统调用时，仅仅把数据从应用的缓冲区复制到内核缓冲区**

### 10.1.3 系统调用流程

Java客户端和服务端之间完成一次socket请求和响应（包括read和write）的数据交换，其完整的流程如下：

- 客户端发送请求：Java客户端程序通过write系统调用将数据复制到内核缓冲区，Linux将内核缓冲区的请求数据通过客户端机器的网卡发送出去。在服务端，这份请求数据会从接收网卡中读取到服务端机器的内核缓冲区。服务端获取请求：Java服务端程序通过read系统调用从Linux内核缓冲区读取数据，再送入Java进程缓冲区。
- 服务端业务处理：Java服务器在自己的用户空间中完成客户端的请求所对应的业务处理。服务端返回数据：Java服务器完成处理后，构建好的响应数据将从用户缓冲区写入内核缓冲区，这里用到的是write系统调用，操作系统会负责将内核缓冲区的数据发送出去。发送给客户端：服务端Linux系统将内核缓冲区中的数据写入网卡，网卡通过底层的通信协议将数据发送给目标客户端。

## 10.2 四种IO模型

### 10.2.1 阻塞非阻塞与同步异步

- 阻塞与非阻塞：阻塞IO指的是需要内核IO操作彻底完成后才返回到用户空间执行用户程序的操作指令。“阻塞”指的是用户程序（发起IO请求的进程或者线程）的执行状态。可以说传统的IO模型都是阻塞IO模型，并且在Java中默认创建的socket都属于阻塞IO模型。图片来自于我自己的博客

  ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty1.png)

- 同步与异步：简单来说，可以将同步与异步看成发起IO请求的两种方式。同步IO是指用户空间（进程或者线程）是主动发起IO请求的一方，系统内核是被动接收方。异步IO则反过来，系统内核是主动发起IO请求的一方，用户空间是被动接收方。图片来自于我自己的博客

  ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/netty/netty2.png)

### 10.2.2 四种模型

1. 同步阻塞

   同步阻塞IO（Blocking IO）指的是用户空间（或者线程）主动发起，需要等待内核IO操作彻底完成后才返回到用户空间的IO操作。在IO操作过程中，发起IO请求的用户进程（或者线程）处于阻塞状态。

2. 同步非阻塞

   同步非阻塞IO指的是用户进程主动发起，不需要等待内核IO操作彻底完成就能立即返回用户空间的IO操作。在IO操作过程中，发起IO请求的用户进程（或者线程）处于非阻塞状态。图片来自于我自己的博客

3. 异步阻塞——IO多路复用

   为了提高性能，操作系统引入了一种新的系统调用，专门用于查询IO文件描述符（含socket连接）的就绪状态。在Linux系统中，新的系统调用为select/epoll系统调用。通过该系统调用，一个用户进程（或者线程）可以监视多个文件描述符，一旦某个描述符就绪（一般是内核缓冲区可读/可写），内核就能够将文件描述符的就绪状态返回给用户进程（或者线程），用户空间可以根据文件描述符的就绪状态进行相应的IO系统调用。

4. 异步非阻塞

   异步IO（Asynchronous IO，AIO）指的是用户空间的线程变成被动接收者，而内核空间成为主动调用者。在异步IO模型中，当用户线程收到通知时，数据已经被内核读取完毕并放在了用户缓冲区内，内核在IO完成后通知用户线程直接使用即可。异步IO类似于Java中典型的回调模式，用户进程（或者线程）向内核空间注册了各种IO事件的回调函数，由内核去主动调用。

### 10.2.3 通过合理配置来支持并发连接

在生产环境Linux系统中，基本上都需要解除文件句柄数的限制。原因是Linux系统的默认值为1024，也就是说，一个进程最多可以接受1024个socket连接，这是远远不够的。

文件句柄也叫文件描述符。在Linux系统中，文件可分为普通文件、目录文件、链接文件和设备文件。文件描述符（File Descriptor）是内核为了高效管理已被打开的文件所创建的索引，是一个非负整数（通常是小整数），用于指代被打开的文件。所有的IO系统调用（包括socket的读写调用）都是通过文件描述符完成的。

在Linux下，通过调用ulimit命令可以看到一个进程能够打开的最大文件句柄数量。这个命令的具体使用方法是：

~~~bash
ulimit -n
~~~

![image-20220505153614383](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220505153614383.png)

上图是阿里云服务器执行命令后的结果。

ulimit命令是用来显示和修改当前用户进程的基础限制命令，-n选项用于引用或设置当前的文件句柄数量的限制值，Linux系统的默认值为1024。

理论上，1024个文件描述符对绝大多数应用（例如Apache、桌面应用程序）来说已经足够，对于一些用户基数很大的高并发应用则是远远不够的。一个高并发的应用面临的并发连接数往往是十万级、百万级、千万级，甚至像腾讯QQ一样的上亿级。文件句柄数不够，会导致什么后果呢？当单个进程打开的文件句柄数量超过了系统配置的上限值时会发出“Socket/File:Can't open so many files”的错误提示。所以，对于高并发、高负载的应用，必须调整这个系统参数，以适应并发处理大量连接的应用场景。可以通过ulimit来设置这两个参数，方法如下：

~~~
ulimit -n 1000000
~~~

在上面的命令中，n的值设置越大，可以打开的文件句柄数量越大。建议以root用户来执行此命令。

ulimit命令只能用于临时修改，如果想永久地把最大文件描述符数量值保存下来，可以编辑/etc/rc.local开机启动文件，在文件中添加如下内容：

~~~
ulimit -SHn 1000000
~~~

选项-S表示软性极限值，-H表示硬性极限值。硬性极限值是实际的限制，就是最大可以是100万，不能再多了。软性极限值则是系统发出警告（Warning）的极限值，超过这个极限值，内核会发出警告。

可使用`cat /proc/sys/fs/file-max`来查看自己机器支持的最大限制，下图是我个人的配置一般的阿里云服务器的最大支持数

![image-20220505154038290](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220505154038290.png)

要彻底解除Linux系统的最大文件打开数量的限制，可以通过编辑Linux的极限配置文件/etc/security/limits.conf来做到。修改此文件，加入如下内容：

~~~
soft nofile 1000000
hard nofile 1000000
~~~

下图是我自己的服务器的配置文件截图：

![image-20220505154341994](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220505154341994.png)

# 十一、Java NIO

> NIO的实例暂时略，可以先看我的博客中的简单的示例，后续会进行补充

## 11.1 简介

Java NIO类库包含以下三个核心组件：Channel（通道）Buffer（缓冲区）Selector（选择器）。JavaNIO属于异步阻塞模型（IO多路复用模型）。

### 11.1.1 通道

在OIO中，同一个网络连接会关联到两个流：一个是输入流（Input Stream），另一个是输出流（Output Stream）。Java应用程序通过这两个流不断地进行输入和输出的操作。

在NIO中，一个网络连接使用一个通道表示，所有NIO的IO操作都是通过连接通道完成的。一个通道类似于OIO中两个流的结合体，既可以从通道读取数据，也可以向通道写入数据。

### 11.1.2 选择器

IO多路复用指的是一个进程/线程可以同时监视多个文件描述符（含socket连接），一旦其中的一个或者多个文件描述符可读或者可写，该监听进程/线程就能够进行IO就绪事件的查询。

在Java应用层面，存在一个非常重要的Java NIO组件——选择器，选择器可以理解为一个IO事件的监听与查询器。通过选择器，一个线程可以查询多个通道的IO事件的就绪状态。

从编程实现维度来说，IO多路复用编程的第一步是把通道注册到选择器中，第二步是通过选择器所提供的事件查询（select）方法来查询这些注册的通道是否有已经就绪的IO事件（例如可读、可写、网络连接完成等）。由于一个选择器只需要一个线程进行监控，因此我们可以很简单地使用一个线程，通过选择器去管理多个连接通道。

### 11.1.3 缓冲区

应用程序与通道的交互主要是进行数据的读取和写入。为了完成NIO的非阻塞读写操作，NIO为大家准备了第三个重要的组件——Buffer。所谓通道的读取，就是将数据从通道读取到缓冲区中；所谓通道的写入，就是将数据从缓冲区写入通道中。

## 11.2 缓冲区组件 Buffer

Buffer类是一个抽象类，对应于Java的主要数据类型。在NIO中，有8种缓冲区类，分别是ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer、LongBuffer、ShortBuffer、MappedByteBuffer。前7种Buffer类型覆盖了能在IO中传输的所有Java基本数据类型，第8种类型是一种专门用于内存映射的ByteBuffer类型。不同的Buffer子类可以操作的数据类型能够通过名称进行判断，比如IntBuffer只能操作Integer类型的对象。

### 11.2.1 Buffer重要属性

如下图：

![image-20220505161952732](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220505161952732.png)

1. position

   在写模式下，position值的变化规则如下：

   - 在刚进入写模式时，position值为0，表示当前的写入位置为从头开始。
   - 每当一个数据写到缓冲区之后，position会向后移动到下一个可写的位置。
   - 初始的position值为0，最大可写值为limit-1。当position值达到limit时，缓冲区就已经无空间可写了。

   在读模式下，position值的变化规则如下：

   - 当缓冲区刚开始进入读模式时，position会被重置为0。
   - 当从缓冲区读取时，也是从position位置开始读。读取数据后，position向前移动到下一个可读的位置。
   - 在读模式下，limit表示可读数据的上限。position的最大值为最大可读上限limit，当position达到limit时表明缓冲区已经无数据可读。

2. limit

   Buffer类的limit属性表示可以写入或者读取的数据最大上限，其属性值的具体含义也与缓冲区的读写模式有关。不同模式limit含义不同：

   - 在写模式下，limit属性值的含义为可以写入的数据最大上限。在刚进入写模式时，limit的值会被设置成缓冲区的capacity值，表示可以一直将缓冲区的容量写满。
   - 在读模式下，limit值的含义为最多能从缓冲区读取多少数据。

   > 在从写模式到读模式的翻转过程中，position和limit属性值会进行调整，具体的规则是：（1）limit属性被设置成写模式时的position值，表示可以读取的最大数据位置。（2）position由原来的写入位置变成新的可读位置，也就是0，表示可以从头开始读。

3. capacity

   Buffer类的capacity属性表示内部容量的大小。一旦写入的对象数量超过了capacity，缓冲区就满了，不能再写入了。

4. mark

   在缓冲区操作过程当中，可以将当前的position值临时存入mark属性中；需要的时候，再从mark中取出暂存的标记值，恢复到position属性中，重新从position位置开始处理。

### 11.2.2 Buffer重要方法

1. allocate

   在使用Buffer实例之前，我们首先需要获取Buffer子类的实例对象，并且分配内存空间。需要获取一个Buffer实例对象时，并不是使用子类的构造器来创建，而是**调用子类的allocate()**方法。比如：

   ```java
   final ByteBuffer buffer = ByteBuffer.allocate(1024);
   ```

2. put

   put()方法很简单，只有一个参数，即需要写入的对象，只不过要求写入的数据类型与缓冲区的类型保持一致。

   ```java
   public class TestBuffer {
       public static void main(String[] args) {
           final IntBuffer intBuffer = IntBuffer.allocate(10);
           for (int i = 0; i < 5; i++) {
               intBuffer.put(i);
           }
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
       }
   }
   ```

   运行结果：

   5
   10
   10

   写入了5个元素之后，缓冲区的position属性值变成了5，所以指向了第6个（从0开始的）可以进行写入的元素位置。limit、capacity两个属性的值都没有发生变化。

3. flip

   如果需要读取数据，要将缓冲区转换成读模式。flip()翻转方法是Buffer类提供的一个模式转变的重要方法，作用是将写模式翻转成读模式。

   ```java
   public class TestBuffer {
       public static void main(String[] args) {
           final IntBuffer intBuffer = IntBuffer.allocate(10);
           for (int i = 0; i < 5; i++) {
               intBuffer.put(i);
           }
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
           intBuffer.flip();
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
       }
   }
   ```

   运行结果：

   5
   10
   10
   0
   5
   10

   从运行结果可知，flip反转后直接从0开始读，limit变成了读的上限，也就是例子中的5

   > 读取完成后可以调用Buffer.clear()清空或者Buffer.compact()压缩方法，它们可以将缓冲区转换为写模式

4. get

   读取数据的方法很简单，可以调用get()方法每次从position的位置读取一个数据，并且进行相应的缓冲区属性的调整。

   ```java
   public class TestBuffer {
       public static void main(String[] args) {
           final IntBuffer intBuffer = IntBuffer.allocate(10);
           for (int i = 0; i < 5; i++) {
               intBuffer.put(i);
           }
           intBuffer.flip();
           intBuffer.get();
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
       }
   }
   ```

   运行结果：

   1
   5
   10

   从结果可知，读取操作会改变可读位置position的属性值，而可读上限limit值并不会改变。

   > 缓冲区可以重复读的，既可以通过倒带方法rewind()去完成，也可以通过mark()和reset()两个方法组合实现。

5. rewind

   已经读完的数据，如果需要再读一遍，可以调用rewind()方法。

   ```java
   public class TestBuffer {
       public static void main(String[] args) {
           final IntBuffer intBuffer = IntBuffer.allocate(10);
           for (int i = 0; i < 5; i++) {
               intBuffer.put(i);
           }
           intBuffer.flip();
           intBuffer.get();
           intBuffer.get();
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
           intBuffer.rewind();
           System.out.println(intBuffer.position());
           System.out.println(intBuffer.limit());
           System.out.println(intBuffer.capacity());
       }
   }
   ```

   运行结果：

   2
   5
   10
   0
   5
   10

   rewind ()方法主要是调整了缓冲区的position属性与mark属性，具体的调整规则如下：

   - position重置为0，所以可以重读缓冲区中的所有数据。
   - limit保持不变，数据量还是一样的，仍然表示能从缓冲区中读取的元素数量。
   - mark被清理，表示之前的临时位置不能再用了

6. mark和reset

   mark()和reset()两个方法是配套使用的：Buffer.mark()方法将当前position的值保存起来放在mark属性中，让mark属性记住这个临时位置；然后可以调用Buffer.reset()方法将mark的值恢复到position中。

   ```java
   public class TestBuffer {
       public static void main(String[] args) {
           final IntBuffer intBuffer = IntBuffer.allocate(10);
           for (int i = 0; i < 5; i++) {
               intBuffer.put(i);
           }
           intBuffer.flip();
           System.out.print(intBuffer.get()+"  ");
           intBuffer.mark();
           for (int i = 0; i < 4; i++) {
               System.out.print(intBuffer.get());
           }
           intBuffer.reset();
           System.out.println();
           for (int i = 0; i < 4; i++) {
               System.out.print(intBuffer.get());
           }
       }
   }
   ```

   运行结果：

   0  1234
   1234

7. clear

   在读模式下，调用clear()方法将缓冲区切换为写模式。此方法的作用是：（1）将position清零。（2）limit设置为capacity最大容量值，可以一直写入，直到缓冲区写满。

## 11.3 通道组件 Channel

Java NIO中一个socket连接使用一个Channel来表示。从更广泛的层面来说，一个通道可以表示一个底层的文件描述符，例如硬件设备、文件、网络连接等。Java NIO的通道还可以更加细化。例如，不同的网络传输协议类型，在Java中都有不同的NIO Channel实现。

这里不对Java NIO的全部通道类型进行过多的描述，仅着重介绍其中最为重要的四种Channel实现：

1. FileChannel：文件通道，用于文件的数据读写。FileChannel为阻塞模式，不能设置为非阻塞模式。
2. SocketChannel：套接字通道，用于套接字TCP连接的数据读写。
3. ServerSocketChannel：服务器套接字通道（或服务器监听通道），允许我们监听TCP连接请求，为每个监听到的请求创建一个SocketChannel通道。
4. DatagramChannel：数据报通道，用于UDP的数据读写。

### 11.3.1 FileChannel

FileChannel暂时可以看[我的博客](https://www.cnblogs.com/yhr520/p/15384520.html)的2.1和2.2

### 11.3.2 SocketChannel

在NIO中，涉及网络连接的通道有两个：一个是SocketChannel，负责连接的数据传输；另一个是ServerSocketChannel，负责连接的监听。其中，NIO中的SocketChannel传输通道与OIO中的Socket类对应，NIO中的ServerSocketChannel监听通道对应于OIO中的ServerSocket类。

无论是ServerSocketChannel还是SocketChannel，都支持阻塞和非阻塞两种模式。调用configureBlocking()方法：

- socketChannel.configureBlocking(false)设置为非阻塞模式。
- socketChannel.configureBlocking(true)设置为阻塞模式。

SocketChannel主要有以下几种操作：

**获取通道**

```java
//客户端操作
//获取通道
SocketChannel socketChannel = SocketChannel.open();
//设置非阻塞模式
socketChannel.configureBlocking(false);
//发起连接
socketChannel.connect(new InetSocketAddress("127.0.0.1",80));
//服务端操作
//获取通道
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
//绑定一个端口号
serverSocketChannel.bind(new InetSocketAddress(80));
//设置非阻塞方式
serverSocketChannel.configureBlocking(false);
```

**读取数据**

```java
ByteBuffer buffer = ByteBuffer.allocate(1024);
//read方法返回读取的字节数，如果是-1则表示对方已经结束输出。
final int read = socketChannel.read(buffer);
```

**写入数据**

```java
buffer.flip();
socketChannel.write(buffer);
```

**关闭通道**

```java
//调用此方法向对方输出-1（-1是输出的结束标志）
socketChannel.shutdownOutput();
socketChannel.close();
```

## 11.4 选择器组件 Selector

简单地说，选择器的使命是完成IO的多路复用，其主要工作是通道的注册、监听、事件查询。一个通道代表一条连接通路，通过选择器可以同时监控多个通道的IO（输入输出）状况。选择器和通道的关系是监控和被监控的关系。

通道和选择器之间的关联通过register（注册）的方式完成。调用通道的`Channel.register(Selectorsel，int ops)`方法，可以将通道实例注册到一个选择器中。register方法有两个参数：第一个参数指定通道注册到的选择器实例；第二个参数指定选择器要监控的IO事件类型。可供选择器监控的通道IO事件类型包括以下四种：

1. 可读：SelectionKey.OP_READ。
2. 可写：SelectionKey.OP_WRITE。
3. 连接：SelectionKey.OP_CONNECT。
4. 接收：SelectionKey.OP_ACCEPT。

以上事件类型常量定义在SelectionKey类中。如果选择器要监控通道的多种事件，可以用“按位或”运算符来实现。例如，同时监控可读和可写IO事件：

~~~java
int key = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
~~~

> IO事件不是对通道的IO操作，而是通道处于某个IO操作的就绪状态，表示通道具备执行某个IO操作的条件。例如，某个SocketChannel传输通道如果完成了和对端的三次握手过程，就会发生“连接就绪”（OP_CONNECT）事件；某个ServerSocketChannel服务器连接监听通道，在监听到一个新连接到来时，则会发生“接收就绪”（OP_ACCEPT）事件；一个SocketChannel通道有数据可读，就会发生“读就绪”（OP_READ）事件；一个SocketChannel通道等待数据写入，就会发生“写就绪”（OP_WRITE）事件。

### 11.4.1 SelectableChannel

并不是所有的通道都是可以被选择器监控或选择的。例如，FileChannel就不能被选择器复用。判断一个通道能否被选择器监控或选择有一个前提：判断它是否继承了抽象类SelectableChannel（可选择通道），如果是，就可以被选择，否则不能被选择。**一个通道若能被选择，则必须继承SelectableChannel类。**

Java NIO中所有网络连接socket通道都继承了SelectableChannel类，都是可选择的。FileChannel并没有继承SelectableChannel，因此不是可选择通道。类的继承关系如下：

![image-20220506085325517](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220506085325517.png)

### 11.4.2 SelectionKey

通道和选择器的监控关系注册成功后就可以选择就绪事件，具体的选择工作可调用Selector的select()方法来完成。通过select()方法，选择器可以不断地选择通道中所发生操作的就绪状态，返回注册过的那些感兴趣的IO事件。结构如下：

![image-20220506085709386](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220506085709386.png)

> `<<`是左移运算符，用来将一个数的各二进制位全部左移若干位。简单介绍一种方便计算的方法：
>
> 8 << 1的值为8*2=16；
>
> 8 << 2的值为8*(2^2)=32；
>
> 8 << n的值为8*（2^n）。

这个类里面还有很重要的两个方法：

1. void attach(Object o)：将对象附加到选择键。此方法可以将任何Java POJO对象作为附件添加到SelectionKey实例。此方法非常重要，因为在单线程版本的Reactor模式实现中可以将Handler实例作为附件添加到SelectionKey实例。
2. Object attachment()：从选择键获取附加对象。此方法与attach(Object o)是配套使用的，其作用是取出之前通过attach(Object o)方法添加到SelectionKey实例的附加对象。这个方法同样非常重要，当IO事件发生时，选择键将被select方法查询出来，可以直接将选择键的附件对象取出。

### 11.4.3 使用选择器的流程

主要有三步：

1. 获取选择器实例。选择器实例是通过调用静态工厂方法open()来获取的

   ```java
   Selector selector = Selector.open();
   ```

2. 将通道注册到选择器实例,注册到选择器的通道必须处于非阻塞模式下

   ```java
   //获取通道
   ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
   //设置非阻塞方式
   serverSocketChannel.configureBlocking(false);
   //绑定一个端口号
   serverSocketChannel.bind(new InetSocketAddress(9999));
   //获取选择器
   Selector selector = Selector.open();
   //将通道注册到选择器上，并指定监听事件为“接收连接”
   serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);
   ```

3. 选出感兴趣的IO就绪事件（选择键集合）。通过Selector的select()方法，选出已经注册的、已经就绪的IO事件，并且保存到SelectionKey集合中。

   ```java
   //将通道注册到选择器上，并指定监听事件为“接收连接”
   serverSocketChannel.register(selector,SelectionKey.OP_ACCEPT);
   //轮询，选择IO就绪事件
   while (selector.select()>0){
       final Set<SelectionKey> selectionKeys = selector.selectedKeys();
       final Iterator<SelectionKey> iterator = selectionKeys.iterator();
       while (iterator.hasNext()){
           final SelectionKey selectionKey = iterator.next();
           if (selectionKey.isAcceptable()){
               //IO事件，监听的通道有新连接
           }else if (selectionKey.isConnectable()){
               //IO事件：传输通道建立成功
           }else if (selectionKey.isReadable()){
               //IO事件：传输通道可读
           }else if (selectionKey.isWritable()){
               //UI事件：传输通道可写
           }
       }
   }
   ```

select方法源码如下：

```java
//非阻塞，立即返回，不管有没有IO事件都立即返回
public abstract int selectNow() throws IOException;
//限时阻塞
public abstract int select(long timeout)
    throws IOException;
//阻塞调用，直到一个通道发生了注册IO事件
public abstract int select() throws IOException;
```

### 11.4.4 NIO实例

暂略

# 十二、Reactor模式

## 12.1 Reactor模式介绍

Doug Lea在文章“Scalable IO in Java”中对Reactor模式的定义：

Reactor模式由Reactor线程、Handlers处理器两大角色组成，两大角色的职责分别如下：

1. Reactor线程的职责：负责响应IO事件，并且分发到Handlers处理器。这里的IO事件就是NIO中选择器查询出来的通道IO事件。
2. Handlers处理器的职责：非阻塞的执行业务处理逻辑。与IO事件（或者选择键）绑定，负责IO事件的处理，完成真正的连接建立、通道的读取、处理业务逻辑、负责将结果写到通道等。

### 12.1.1 BIO的缺陷

在BIO中，原始的网络服务器程序一般使用一个while循环不断地监听端口是否有新的连接,然后进行处理。示例代码如下：

~~~java
while(true){
	Socket socket = serverSocket.accept();
    handle(socket);
}
~~~

这种方法的最大问题是：如果前一个网络连接的handle(socket)没有处理完，那么后面的新连接无法被服务端接收，于是后面的请求就会被阻塞，导致服务器的吞吐量太低。这对于服务器来说是一个严重的问题。

于是出现了多线程来解决此问题，也就是用一个线程处理一个连接。示例代码如下：

~~~java
ServerSocket serverSocket = new ServerSocket(30000);
while (true){
    //下面这一行代码会一直阻塞等待别人的连接
    Socket socket = serverSocket.accept();
    socketList.add(socket);
    //每当客户端连接后启动一个ServerThread线程为该客户端服务
    new Thread(new ServerThread(socket)).start();
}
~~~

这种模式一定程度上解决了阻塞问题，但是也有缺点，就是对应于大量的连接，需要耗费大量的线程资源，对线程资源要求太高。在系统中，线程是比较昂贵的系统资源。如果线程的数量太多，系统将无法承受。而且，线程的反复创建、销毁、切换也需要代价。因此，在高并发的应用场景下，多线程BIO的缺陷是致命的。

解决上面的这个问题，一个有效途径是使用Reactor模式。用Reactor模式对线程的数量进行控制，做到一个线程处理大量的连接。

### 12.1.2 单线程Reactor模型









