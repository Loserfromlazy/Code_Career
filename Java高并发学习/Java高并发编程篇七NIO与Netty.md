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

- 客户端发送请求：Java客户端程序通过write系统调用将数据复制到内核缓冲区，Linux将内核缓冲区的请求数据通过客户端机器的网卡发送出去。

  在服务端，这份请求数据会从接收网卡中读取到服务端机器的内核缓冲区。服务端获取请求：Java服务端程序通过read系统调用从Linux内核缓冲区读取数据，再送入Java进程缓冲区。

- 服务端业务处理：Java服务器在自己的用户空间中完成客户端的请求所对应的业务处理。服务端返回数据：Java服务器完成处理后，构建好的响应数据将从用户缓冲区写入内核缓冲区，这里用到的是write系统调用，操作系统会负责将内核缓冲区的数据发送出去。

  发送给客户端：服务端Linux系统将内核缓冲区中的数据写入网卡，网卡通过底层的通信协议将数据发送给目标客户端。

### 10.1.4 Socket对象的结构

一般来说socket有一个别名也叫做套接字。socket起源于Unix，都可以用“打开open –> 读写write/read –> 关闭close”模式来操作。Socket就是该模式的一个实现，socket即是一种特殊的文件，一些socket函数就是对其进行的操作（读/写IO、打开、关闭）。说白了Socket是[应用层](https://so.csdn.net/so/search?q=应用层&spm=1001.2101.3001.7020)与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议，而不需要让用户自己去定义什么时候需要指定哪个协议哪个函数。

当进程A执行到创建socket的语句时，操作系统会创建一个由文件系统管理的socket对象，这个对象包含了：

- 发送缓冲区
- 接收缓冲区
- 等待队列,等待队列指向所有等待该socket事件的进程。
- ......

因为socket是一个文件，所以可以用下图来理解：

> 文件句柄也叫文件描述符。在Linux系统中，文件可分为普通文件、目录文件、链接文件和设备文件。文件描述符（File Descriptor）是内核为了高效管理已被打开的文件所创建的索引，是一个非负整数（通常是小整数），用于指代被打开的文件。所有的IO系统调用（包括socket的读写调用）都是通过文件描述符完成的。

![image-20220517145031418](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220517145031418.png)

当执行到接收数据时，操作系统就会将进程A从工作队列移动到该socket的等待队列中：

![image-20220517145717627](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220517145717627.png)

下面我们更详细描述一下内核接收数据的大概过程：

- 首先计算机接收到了对端的数据
- 然后数据由网卡传输到内存中
- 然后网卡通过中断信号，通知CPU有数据到达，CPU执行中断程序
- 中断程序先将网络数据写入对应socket的接收缓冲区（socket缓冲区是在内核中的）中
- 然后唤醒进程A，重新将进程A放入工作队列中

内核接收数据时如何判断数据属于哪一个Socket？网卡接收数据时，会根据socket的数据包格式（源ip，源端口，协议，目标ip，目标端口）判断属于哪个socket。

内核通过IO多路复用来实现监控多个socket的目的。IO多路复用是异步阻塞的IO模型，我们先了解四种IO模型，然后再了解IO多路复用的系统调用。

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

在Linux下，通过调用ulimit命令可以看到一个进程能够打开的最大文件句柄数量。这个命令的具体使用方法是：

~~~bash
ulimit -n
~~~

![image-20220505153614383](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220505153614383.png)

上图是阿里云服务器执行命令后的结果。

ulimit命令是用来显示和修改当前用户进程的基础限制命令，-n选项用于引用或设置当前的文件句柄数量的限制值，Linux系统的默认值为1024。

理论上，1024个文件描述符对绝大多数应用（例如Apache、桌面应用程序）来说已经足够，对于一些用户基数很大的高并发应用则是远远不够的。一个高并发的应用面临的并发连接数往往是十万级、百万级、千万级，甚至像腾讯QQ一样的上亿级。**文件句柄数不够，会导致什么后果呢？当单个进程打开的文件句柄数量超过了系统配置的上限值时会发出“Socket/File:Can't open so many files”的错误提示**。所以，对于高并发、高负载的应用，必须调整这个系统参数，以适应并发处理大量连接的应用场景。可以通过ulimit来设置这两个参数，方法如下：

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

## 10.3 IO多路复用的系统调用

> 这里推荐[IO Multiplexing](https://github.com/Liu-YT/IO-Multiplexing)这个仓库，有每一步的源码分析。想更深入研究可以去这个仓库看一下。

在四种IO模型中最常用的就是IO多路复用模型，或者说是异步阻塞模型，因此我们现在来了解操作系统为IO多路复用提供的系统调用。

> 操作系统的主要功能是为管理硬件资源和为应用程序开发人员提供良好的环境来使应用程序具有更好的兼容性，为了达到这个目的，内核提供一系列具备预定功能的多内核函数，通过一组称为系统调用（system call)的接口呈现给用户。系统调用把应用程序的请求传给内核，调用相应的内核函数完成所需的处理，将处理结果返回给应用程序。

在IO多路复用的模型中，主要有三个系统调用需要了解，`select`,`epoll`,`poll`。其中windows和linux都支持select和poll系统调用，epoll只有linux支持。我们以linux为例学习这三个系统调用。

### 10.3.1 select系统调用

linux和windows内核都支持select系统调用。

select操作1次轮询会遍历1024个文件描述符的IO事件（或者说是1024个socket的IO事件），轮询发生在内核空间。

select核心步骤如下：

- 准备fds数组（fdset是位图数据结构），数组存放着所有需要监视的socket。select函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。（读写和异常）
- 调用select，如果fds数组中的所有socket没有数据，select会阻塞，直到有socket接收数据，或者超时（timeout指定等待时间，如果立即返回设为null即可），select返回，唤醒进程
- select函数返回后遍历fds，通过FD_ISSET判断具体是哪个socket接收到数据（或者说是哪个文件描述符就绪），然后做处理

select原型如下：

~~~
 int select(
          int nfds,
          fd_set *readfds,
          fd_set *writefds,
          fd_set *exceptfds,
          struct timeval *timeout);
~~~

select的简单使用：

![image-20220518133003373](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518133003373.png)

流程如下：

> fd是指文件描述符

进程A调用select后，从进程A的用户空间拷贝fd_set到内核空间，然后遍历所有fd将进程A存入需要监听的socket的等待队列中。

![image-20220518113653491](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518113653491.png)

进入等待队列后，执行for循环（无限轮询），遍历每一个fd（或者说是socket），调用对应文件描述符中的poll回调函数，检测是否就绪，（只有fd或者说socket处于就绪状态或信号中断或出错或超时才退出循环或者说退出轮询）。不就绪就会调用`poll_schedule_timeout`函数，让当前进程睡眠，一直到超时或者有描述符就绪被唤醒（当有socket进行数据接收时，会调用中断程序唤醒进程A）。接着又会再次遍历每个描述符，调用`poll`再次检测。如此循环，直到符合条件才会退出。

![image-20220518113817822](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518113817822.png)

进程A从所有的socket的等待队列中移除，加入到操作系统的工作队列

![image-20220518113948781](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518113948781.png)

对于进程A，当调用select时，会将A存入每个socket的等待队列，当某个socket接收数据时，A被唤醒从多个socket等待队列移除，然后A遍历所有socket（即fds数组），当A处理完数据后，移出工作队列这是需要在遍历所有socket再将A加入等待队列。

**select系统调用缺点：**

- 被监控的fds集合限制为1024太小了
- fds集合需要从用户空间拷贝到内核空间
- 当被监控的fds中某些有数据可读的时候，通知其实可以更加精细一点，就是能够从通知中得到有可读事件的fds列表，而不是需要遍历整个fds来收集。

### 10.3.2 poll系统调用

poll出现是为了代替select，poll不在限制socket数量。

poll的内部与select基本一样，都是需要进行遍历的，但是区别在于底层fds数组的数据结构不一样了，从而实现最大文件句柄数量限制去除了。

> 在select中使用的是fd_set结构（类似bitmap），而poll中使用的是pollfd

pollfd结构：

```
  struct pollfd {
        int     fd;
        short   events;
        short   revents;
  };
```

poll系统调用原型：

```
  int poll(
          struct pollfd filedes[];
          unsigned int nfds;
          int timeout   /* in milliseconds */);
```

poll流程如下：

`poll`的实现与`select`基本差不多。调用`poll`函数，调用`copy_from_user`将`fds`拷贝到内核，`poll_initwait`对每个`fd`将此进程注册到其等待队列，当有事件发生时会唤醒此进程，循环对每个`fd`调用`do_pollfd`，判断是否有事件到来，没有的话进入终端睡眠，有事件到来后将所有`fds`再拷贝回用户层。

poll虽然解决了fds集合大小1024的限制问题，但是，它并没改变大量描述符数组被整体复制于用户态和内核态的地址空间之间，以及个别描述符就绪触发整体描述符集合的遍历的低效问题。

**socket读就绪条件**

1. socket接收缓冲区数据字节数大于等于socket缓冲区低水位标记SO_RCVLOWAT。

   对于TCP和UDP套接字而言，**缓冲区低水位的值默认为**1。那就意味着，默认情况下，只要缓冲区中有数据，那就是可读的。我们可以通过使用SO_RCVLOWAT套接字选项(参见setsockopt函数)来设置该套接字的低水位大小。此种描述符就绪(可读)的情况下，当我们使用read/recv等对该套接字执行读操作的时候，套接字不会阻塞，而是成功返回一个大于0的值（即可读数据的大小）。

2. 该连接的读半部关闭（就是接受了FIN的TCP连接）。对这样的socket的读操作不会阻塞，而是会返回0

3. 该socket是一个listen的监听套接字，且目前完成的连接数不为0，对于这样的套接字进行accept操作不会阻塞

4. 有一个错误socket待处理。对这样的socket的读操作不阻塞并返回-1.

**socket写就绪条件**

1. socket内核中，发送缓冲区的可用字节数大于等于低水位标记SO_SNDLOWAT，此时可以无阻塞的写，并且返回值大于0

   对于TCP和UDP而言，这个低水位SO_SNDLOWAT的值默认为2048，而套接字默认的发送缓冲区大小是8k，这就意味着一般一个套接字连接成功后，就是处于可写状态的。我们可以通过SO_SNDLOWAT套接字选项（参见setsockopt函数）来设置这个低水位。此种情况下，我们设置该套接字为非阻塞，对该套接字进行写操作(如write,send等)，将不阻塞，并返回一个正值（例如由传输层接受的字节数，即发送的数据大小）。

2. 该连接的写半部关闭（主动发送FIN包的TCP连接），对这样的套接字的写操作将会产生SIGPIPE信号。所以我们的网络程序基本都要自定义处理SIGPIPE信号。因为SIGPIPE信号的默认处理方式是程序退出

3. 使用非阻塞的connect已建立连接，或者connect已经失败。（即connect有结果了）

4. 有一个错误的套接字待处理。对这样的套接字的写操作将不阻塞并返回-1

PS：poll系统调用的事件列表如下，图片来自于网络：

![image-20220522155558599](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220522155558599.png)

### 10.3.3 epoll系统调用

select和poll低效的原因是**需要拷贝fds集合和没有按需遍历fds集合**。而在poll中也没有解决这两个问题，poll只是解决了1024的限制因为改变了底层数据结构。这时就需要epoll。

epoll是在Linux 2.6内核中提出的，是之前select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符的限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间之间的数据拷贝只需一次。

epoll是怎么解决这两个性能问题的呢？

1. 拷贝fds集合的问题

   对于IO多路复用，有两件事是必须要做的(对于监控可读事件而言)：1. 准备好需要监控的fds集合；2. 探测并返回fds集合中哪些fd可读了。每次调用select或poll都在重复地准备整个需要监控的fds集合。然而对于频繁调用的select或poll而言，fds集合的变化频率要低得多，我们没必要每次都重新准备(集中处理)整个fds集合。

   于是，epoll引入了epoll_ctl系统调用，将高频调用的epoll_wait和低频的epoll_ctl隔离开。同时，epoll_ctl通过(EPOLL_CTL_ADD、EPOLL_CTL_MOD、EPOLL_CTL_DEL)三个操作来分散对需要监控的fds集合的修改，做到了有变化才变更，将select或poll高频、大块内存拷贝(集中处理)变成epoll_ctl的低频、小块内存的拷贝(分散处理)，避免了大量的内存拷贝。同时，对于高频epoll_wait的可读就绪的fd集合返回的拷贝问题，epoll通过内核与用户空间mmap(内存映射)同一块内存来解决。mmap将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射映射到物理地址），使得这块物理内存对内核和对用户均可见，减少用户态和内核态之间的数据交换。

2. 遍历fds集合的问题

   为了做到只遍历就绪的fd，我们需要有个地方来组织那些已经就绪的fd。为此，epoll引入了一个中间层，一个双向链表(ready_list)，一个单独的睡眠队列(single_epoll_wait_list)，并且，与select或poll不同的是，epoll的process不需要同时插入到多路复用的socket集合的所有睡眠队列中，相反process只是插入到中间层的epoll的单独睡眠队列中，process睡眠在epoll的单独队列上，等待事件的发生。

epoll数据结构：

eventpoll的rbr是一个红黑树，rdlist是一个双向链表，存放已经发生IO事件的socket，等待队列是poll_wait,进程放入这里。

![image-20220518150434281](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518150434281.png)

epoll的流程：

首先会调用epoll_create,创建一个eventpoll对象，创建`eventpoll`对象的初始化操作，获取当前用户信息,是不是`root`，获取最大监听`fd`数目等并且保存到`eventpoll`对象中。初始化等待队列，初始化就绪链表，初始化红黑树的头结点。如下图：

![image-20220518141648086](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518141648086.png)

然后调用epoll_ctl，将需要监听的fd（socket）添加或删除。

> epoll_ctl将epoll_event结构拷贝到内核空间中，并且判断加入的fd是否支持`poll`结构(`epoll，poll，select I/O多路复用必须支持poll操作`),并且从`epfd->file->privatedata`获取eventpoll对象，根据op区分是添加删除还是修改。首先在eventpoll结构中的红黑树查找是否已经存在了相对应的fd，没找到就支持插入操作。
>
> 插入操作时，会创建一个与`fd`对应的`epitem`结构，并且初始化相关成员，比如保存监听的`fd`跟`file`结构之类的。重要的是指定了调用`poll_wait`时的回调函数用于数据就绪时唤醒进程，(其内部，初始化设备的等待队列，将该进程注册到等待队列)完成这一步，我们的`epitem`就跟这个`socket`关联起来了， 当它有状态变化时，会通过`ep_poll_callback()`来通知。最后调用加入的`fd`的`file operation->poll`函数(最后会调用`poll_wait`操作)用于完成注册操作。 最后将`epitem`结构添加到红黑树中。

![image-20220518144818157](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518144818157.png)

如果有就绪事件，或者说是socket接收到数据，那么中断程序就会操作eventpoll的就绪队列rdlist，而不是直接操作进程。比如socket1和2收到数据，那就让这俩socket进入rdlist：

![image-20220518152858806](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220518152858806.png)

然后执行epoll_wait,当程序执行到epoll_wait时，如果rdlist非空则返回，如果rdlist为空，阻塞进程。epoll_wait会计算睡眠时间(如果有)，判断eventpoll对象的链表是否为空，不为空那就返回。并且初始化一个等待队列，把自己挂上去，设置自己的进程状态为可睡眠状态。判断是否有信号到来(有的话直接被中断醒来)，如果啥事都没有那就调用`schedule_timeout`进行睡眠，如果超时或者被唤醒，首先从自己初始化的等待队列删除，然后开始拷贝资源给用户空间了。拷贝资源则是先把就绪事件链表转移到中间链表，然后挨个遍历拷贝到用户空间，并且挨个判断其是否为水平触发（水平触发后面会学习），是的话再次插入到就绪链表。

**epoll的唤醒逻辑如下：**

> 1. 协议数据包到达网卡并被排入socket sk的接收队列
> 2. 睡眠在sk的睡眠队列wait_entry被唤醒，wait_entry_sk的回调函数epoll_callback_sk被执行
> 3. epoll_callback_sk将当前sk插入epoll的ready_list中
> 4. 唤醒睡眠在epoll的单独睡眠队列single_epoll_wait_list的wait_entry，wait_entry_proc被唤醒执行回调函数epoll_callback_proc
> 5. 遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件 
> 6. 将每个sk收集到的事件，通过epoll_wait传入的events数组回传并唤醒相应的process。

### 10.3.4 三种系统调用的比较

通过比较`select`、`poll`和`epoll`处理 I/O 的过程来剖析其中的原因：

- 用户态将文件描述符传入内核的方式
  - `select`：创建3个文件描述符集并拷贝到内核中,分别监听读、写、异常动作。这里受到单个进程可以打开的`fd`数量限制,默认是1024。
  - `poll`：将传入的`struct pollfd`结构体数组拷贝到内核中进行监听。
  - `epoll`：执行`epoll_create`会在内核的高速`cache`区中建立一颗红黑树以及就绪链表(该链表存储已经就绪的文件描述符)。接着用户执行的`epoll_ctl`函数添加文件描述符会在红黑树上增加相应的结点。
- 内核态检测文件描述符是否可读可写的方式
  - `select`：采用轮询方式，遍历所有`fd`，最后返回一个描述符读写操作是否就绪的`mask`掩码，根据这个掩码给`fd_set`赋值。
  - `poll`：同样采用轮询方式，查询每个`fd`的状态，如果就绪则在等待队列中加入一项并继续遍历。
  - `epoll`：采用回调机制。在执行`epoll_ctl`的`add`操作时，不仅将文件描述符放到红黑树上，而且也注册了回调函数，内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。
- 如何找到就绪的文件描述符并传递给用户态
  - `select`：将之前传入的`fd_set`拷贝传出到用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。
  - `poll`：将之前传入的`fd`数组拷贝传出用户态并返回就绪的文件描述符总数。用户态并不知道是哪些文件描述符处于就绪态，需要遍历来判断。
  - `epoll`：`epoll_wait`只用观察就绪链表中有无数据即可，最后将链表的数据返回给数组并返回就绪的数量。内核将就绪的文件描述符放在传入的数组中，所以只用遍历依次处理即可。这里返回的文件描述符是通过`mmap`让内核和用户空间共享同一块内存实现传递的，减少了不必要的拷贝。
- 继续重新监听时如何重复以上步骤
  - `select`：将新的监听文件描述符集合拷贝传入内核中，继续以上步骤。
  - `poll`：将新的`struct pollfd`结构体数组拷贝传入内核中，继续以上步骤。
  - `epoll`：无需重新构建红黑树，直接沿用已存在的即可。

select/poll的时间复杂度是O(n)，epoll的时间复杂度是O(1)

### 10.3.5 ET和LT

epoll执行epoll_wait有两种触发模式LT水平触发和ET边沿触发。

**LT水平触发**

socket接收缓冲区不为空，有数据可读，则读事件一直触发。socket发送缓冲区不满可以继续写入数据，则写事件一直触发。

**ET边沿触发**

socket的接收缓冲区状态变化时触发读事件，即空的接收缓冲区刚接收到数据时触发读事件。socket的发送缓冲区状态变化时触发写事件，即满的缓冲区刚空出空间时触发读事件。

在10.3.3中我们了解了epoll的唤醒逻辑，在epoll_wait唤醒中第五步会遍历epoll的ready_list，挨个调用每个sk的poll逻辑收集发生的事件。在这一步会移除socket而socket从ready_list移除的时机正是区分两种事件模式的本质：

1. LT的逻辑：首先遍历epoll的ready_list，将socket从ready_list中移除，然后调用该socket的poll逻辑收集发生的事件 然后如果该sk的poll函数返回了关心的事件(对于可读事件来说，就是POLL_IN事件)，那么该sk被重新加入到epoll的ready_list中。
2. ET的逻辑：遍历epoll的ready_list，将socket从ready_list中移除，然后调用该socket的poll逻辑（这里的poll不是系统调用而是一个操作或函数，I/O多路复用必须支持poll操作）收集发生的事件

在Java中的selector会根据操作系统的不同采用不同的实现：

- 在linux2.6以后的NIO的selector版本是EPollSelectorProvider，底层使用的是epoll，采用水平触发模式
- 在macos
- 在netty中额外提供了EpollEventLoop，它使用了边缘触发

也就是说除了jdk的epoll实现外，netty实现了自己的epoll版本，此版本采用了边沿触发模式

使用Netty自己的epoll实现：[官方文档：Native transports](https://netty.io/wiki/native-transports.html)

首先导入依赖：

```xml
  <dependencies>
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-transport-native-epoll</artifactId>
      <version>${project.version}</version>
      <classifier>linux-x86_64</classifier>
    </dependency>
    ...
  </dependencies>
```

然后将所有的NIO换成Epoll实现类，如下：

- `NioEventLoopGroup` → `EpollEventLoopGroup`
- `NioEventLoop` → `EpollEventLoop`
- `NioServerSocketChannel` → `EpollServerSocketChannel`
- `NioSocketChannel` → `EpollSocketChannel`

# 十一、Java NIO

> NIO的实例暂时略，可以先看我的博客中的简单的示例和Reactor模式中的示例，后续会进行补充。

## 11.1 简介

Java NIO类库包含以下三个核心组件：Channel（通道）Buffer（缓冲区）Selector（选择器）。JavaNIO属于异步阻塞模型（IO多路复用模型）。

PS：如果要看NIO的源码比如Selector等类是可以直接看到的，但是像SelectorImpl实现类或者SelectorImpl的各平台的实现类就需要去jdk的源码中看，

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

SelectionKey的继承关系如下：

```
SelectionKeyImpl extends AbstractSelectionKey
AbstractSelectionKey extends SelectionKey
```

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

### 11.4.4 Channel、Selector、SelectionKey关系

Channel和Selector可以说是多对一的关系，他们俩和SelectionKey就像是数据库中的两张表和一张中间表的关系。用图表示如下：

![image-20220513151036492](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220513151036492.png)



### 11.4.5 NIO示例

暂略

## 11.5 NIO原理

我们下面来详细的学习一下NIO

### 11.5.1 IO事件在Java和native

Java的IO事件存在SelectionKey中，一共有四个：

~~~java
//通道读事件就绪
public static final int OP_READ = 1 << 0;
//通道写事件就绪
public static final int OP_WRITE = 1 << 2;
//通道对应的socket已经准备好连接
public static final int OP_CONNECT = 1 << 3;
//通道对应的server socket已经准备好接收一个新连接
public static final int OP_ACCEPT = 1 << 4;
~~~

poll系统调用的事件如下（图片来源于网络）：

![image-20220520160628604](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220520160628604.png)

java在Net.java中通过JNI加载不同平台的poll事件的定义值（代码来自于openjdk源码）：

![image-20220520161132661](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220520161132661.png)

这些native方法，也可在Net.c中找到，比如windows平台的Net.c（代码来自于openjdk源码）:

![image-20220520161315187](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220520161315187.png)

那么在NIO中是如何进行事件的翻译的呢？

在channel注册时（流程可以自己去跟），会在SocketChannelImpl#translateInterestOps方法将NIO事件转换成JNI事件（代码来自于OpenJdk），如下：

~~~java
/**
     * Translates an interest operation set into a native poll event set
     */
    public int translateInterestOps(int ops) {
        int newOps = 0;
        if ((ops & SelectionKey.OP_READ) != 0)
            newOps |= Net.POLLIN;
        if ((ops & SelectionKey.OP_WRITE) != 0)
            newOps |= Net.POLLOUT;
        if ((ops & SelectionKey.OP_CONNECT) != 0)
            newOps |= Net.POLLCONN;
        return newOps;
    }
~~~

在查询的时候（流程可以自己跟一遍），会在SocketChannelImpl#translateReadyOps方法，将JNI事件转换成NIO事件：

~~~java
/**
     * Translates native poll revent ops into a ready operation ops
     */
    public boolean translateReadyOps(int ops, int initialOps, SelectionKeyImpl ski) {
        int intOps = ski.nioInterestOps();
        int oldOps = ski.nioReadyOps();
        int newOps = initialOps;

        if ((ops & Net.POLLNVAL) != 0) {
            // This should only happen if this channel is pre-closed while a
            // selection operation is in progress
            // ## Throw an error if this channel has not been pre-closed
            return false;
        }

        if ((ops & (Net.POLLERR | Net.POLLHUP)) != 0) {
            newOps = intOps;
            ski.nioReadyOps(newOps);
            return (newOps & ~oldOps) != 0;
        }

        boolean connected = isConnected();
        if (((ops & Net.POLLIN) != 0) &&
            ((intOps & SelectionKey.OP_READ) != 0) && connected)
            newOps |= SelectionKey.OP_READ;

        if (((ops & Net.POLLCONN) != 0) &&
            ((intOps & SelectionKey.OP_CONNECT) != 0) && isConnectionPending())
            newOps |= SelectionKey.OP_CONNECT;

        if (((ops & Net.POLLOUT) != 0) &&
            ((intOps & SelectionKey.OP_WRITE) != 0) && connected)
            newOps |= SelectionKey.OP_WRITE;

        ski.nioReadyOps(newOps);
        return (newOps & ~oldOps) != 0;
    }
~~~

可以看到，在上面这两个方法中就用到了Net类。

### 11.5.2 SelectionKey原理

根据上面的学习，我们可以了解SelectionKey就像一个纽扣维系着Channel和Selector，SelectionKey代表着一个channel和它注册的Selector之间的关联关系。

我们调用channel()方法会返回相关联的SelectableChannel对象，而selector()方法会返回相关的Selector对象

```java
/**
 * Returns the channel for which this key was created.  This method will
 * continue to return the channel even after the key is cancelled.
 *
 * @return  This key's channel
 */
public abstract SelectableChannel channel();

/**
 * Returns the selector for which this key was created.  This method will
 * continue to return the selector even after the key is cancelled.
 *
 * @return  This key's selector
 */
public abstract Selector selector();
```

除此之外，在SelectionKey（实现类SelectionKeyImpl）中还有着三个重要属性，这里我们主要关注两个interestOps和readyOps。（index属性在选择器中在详细学习）

```java
//代表注册Channel所感兴趣的事件集合。即Socket监听哪些事件
private volatile int interestOps;
//代表着interest集合中从上次调用select()方法以来已经就绪的事件集合。即已经发生了的事件
private int readyOps;
//SelectionKey集合的下标索引，该SelectionKey在注册选择器中存储的SelectionKey集合的下标，当此SelectionKey被撤销时，index为-1
private int index;
```

有这两个属性，也要有存入这俩属性的事件，在SelectionKey使用了四个常量来代表事件：

```java
//通道读事件就绪
public static final int OP_READ = 1 << 0;
//通道写事件就绪
public static final int OP_WRITE = 1 << 2;
//通道对应的socket已经准备好连接
public static final int OP_CONNECT = 1 << 3;
//通道对应的server socket已经准备好接收一个新连接
public static final int OP_ACCEPT = 1 << 4;
```

了解了SelectionKey属性和事件，那么SelectionKey怎么将通道和选择器关联呢？

其实向通道注册事件就可以完成关联，这时就可以得到他连关联的这个选择键，代码如下，如果想注册多个事件可以用位或运算符连接：

~~~java
SelectionKey selectionKey = socketChannel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
~~~

当然SelectionKey除了在通道注册时注册事件，还可以单独进行注册事件，我们只需要调用interestOps()方法即可：

```java
selectionKey.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

> 当然，其实register底层也是调用了interestOps方法，源码如下：
>
> ![image-20220519202330956](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220519202330956.png)

当然除了进行设置事件，SelectionKey还可以获取事件，调用interestOps()方法就可以获取当前SelectionKey感兴趣的事件，然后使用位与操作就可以判断对某种事件是否感兴趣：

```java
int interestOps = selectionKey.interestOps();
boolean isAccept = (interestOps & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT;
//其余同理
```

调用readyOps()方法就可以获取已经就绪事件的集合，当然还定义了以下几个方法用于判断事件是否就绪（也就是说不用向上面自己写位与操作）

```java
public final boolean isReadable() {
    return (readyOps() & OP_READ) != 0;
}
public final boolean isWritable() {
    return (readyOps() & OP_WRITE) != 0;
}

public final boolean isConnectable() {
    return (readyOps() & OP_CONNECT) != 0;
}
public final boolean isAcceptable() {
    return (readyOps() & OP_ACCEPT) != 0;
}
```

SelectionKey除了对事件进行操作，它还可以添加和获取附件：

~~~java
key.attach(obj);//添加附件
Object obj = key.attachment();//获取附件
//当然附件也可以在注册时加上去，register()方法有这个重载：
 public abstract SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException;
~~~

这个方法可以让SelectionKey更加灵活方便，比如Reactor模式中就使用了附件，具体可以在第十二章学习。

之前我们学习SelectionKey的继承关系时，他还有一个中间的继承的类AbstractSelectionKey，这个类中只有一个属性，三个方法：

```java
//标记此SelectionKey是否有效，默认有效
private volatile boolean valid = true;
//返回SelectionKey是否有效
public final boolean isValid() {
    return valid;
}
//将SelectionKey设置为无效
void invalidate() { 
    valid = false;
}
//将SelectionKey从选择器中删除
public final void cancel() {
    synchronized (this) {
        if (valid) {
            valid = false;
            ((AbstractSelector)selector()).cancel(this);
        }
    }
}
```

这些方法属性主要用于维护SelectionKey的有效性，当调用cancel()方法或关闭其通道或关闭其选择器时会导致SelectionKey失效。

当我们调用selectionKey的cancel()方法后，它将被放在相关的选择器的cancelledKeys集合中。注册关系不会立即被取消，但是selectionKey会立即失效。当再次调用select( )方法时（或者一个正在进行的select()调用结束时），cancelledKeys中的被取消的键将被清理掉。

### 11.5.3 Selector原理

Selector是通道的多路复用器，创建时通过open()方法新建一个选择器：

```java
Selector selector = Selector.open();
```

在创建完后可以通过注册方法将通道注册到选择器上，注册的channel必须时非阻塞的。所以FileChannel不适用Selector，因为FileChannel不能切换为非阻塞模式，更准确的来说是因为FileChannel没有继承SelectableChannel。Socket channel可以正常使用。

SelectableChannel中有一个configureBlocking，用于设置通道是否是非阻塞的

```java
public abstract SelectableChannel configureBlocking(boolean block) throws IOException;
```

Selector中select方法可以返回已经准备就绪的通道，比如你对读事件感兴趣，那么select方法就会返回读事件已就绪的通道。

Selector中包含或者说维护了三个Set集合：

- keys：存放注册到Selector的所有的Key。
- selectedKeys：存放已选择的键集，它是检测到registeredKeys中key感兴趣的事件发生后存放key的地方。
- cancelledKeys：其cancel方法调用过的，待反注册的key

除了上面的三个Set集合，Selector还维护了两个Set集合，如下：

![image-20220521220345524](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220521220345524.png)

这两个也可以理解为视图，这两个Set是通过unmodifiableSet和ungrowableSet方法包装而来的，源码如下。

```java
protected SelectorImpl(SelectorProvider var1) {
    super(var1);
    if (Util.atBugLevel("1.4")) {
        this.publicKeys = this.keys;
        this.publicSelectedKeys = this.selectedKeys;
    } else {
        this.publicKeys = Collections.unmodifiableSet(this.keys);
        this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
    }

}
```

在不同的平台有不同的实现类：

针对linux平台：

- PollSelectorImpl
- EPollSelectorImpl

针对windows平台：

- WindowsSelectorImpl

Netty的实现类：

- Netty的NioEventLoop是使用的上面的linux的PollSelectorImpl，但是Netty自己提供了而外的epoll实现，具体的上面已经学习过了。

#### Selector的创建

我们现在来跟一下Selector.open()方法的源码，看一下Selector的创建的原理。

首先我们调用open方法，源码如下:

```java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
```

这里调用了SelectorProvider.provider()创建了一个provider，然后调用了provider的openSelector方法。

我们先看SelectorProvider.provider()方法，源码如下：

```java
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                        if (loadProviderFromProperty())
                            return provider;
                        if (loadProviderAsService())
                            return provider;
                    //这里
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
```

在上面的源码中，我们抓主干，我们会发现provider是DefaultSelectorProvider#create方法创建的，这个方法jdk会在不同的操作系统中调用不同的方法，比如我们在windows平台开发就会跟进windows平台的方法（我们这里就用windows平台举例，因为很方便IDEA可以直接进入反编译的class文件），如下图：

![image-20220521224054961](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220521224054961.png)

我们再回到open方法，提供了provider之后会调用openSelector方法

~~~java
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}
~~~

openSelector方法源码如下：

~~~java
public class WindowsSelectorProvider extends SelectorProviderImpl {
    public WindowsSelectorProvider() {
    }

    public AbstractSelector openSelector() throws IOException {
        return new WindowsSelectorImpl(this);
    }
}
~~~

这里是直接返回了一个windows平台的Selector实现类[WindowsSelectorImpl](https://github.com/openjdk/jdk8/blob/master/jdk/src/windows/classes/sun/nio/ch/WindowsSelectorImpl.java)。

这几个类的关系如下（其实就是jdk在不同的平台下有不同的实现类）：

![image-20220522144148150](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220522144148150.png)

那么既然最后open方法会返回一个WindowsSelectorImpl（Windows平台）那么我们就看一下这个类的构造：

当然我们先来看它的父类构造函数，在AbstractSelector中会初始化provider和cancelledKeys：

```java
public abstract class AbstractSelector extends Selector
{
    // The provider that created this selector
    private final SelectorProvider provider;

    /**
     * Initializes a new instance of this class.
     * 初始化provider
     */
    protected AbstractSelector(SelectorProvider provider) {
        this.provider = provider;//这里是WindowsSelectorProvider
    }

    private final Set<SelectionKey> cancelledKeys = new HashSet<SelectionKey>();
    //......略
}
```

在SelectorImpl中会初始化selectedKeys和keys，同时会将publicKeys和publicSelectedKeys包装出来

```java
public abstract class SelectorImpl extends AbstractSelector {
    protected Set<SelectionKey> selectedKeys = new HashSet();
    protected HashSet<SelectionKey> keys = new HashSet();
    private Set<SelectionKey> publicKeys;
    private Set<SelectionKey> publicSelectedKeys;

    protected SelectorImpl(SelectorProvider var1) {
        super(var1);
        if (Util.atBugLevel("1.4")) {
            this.publicKeys = this.keys;
            this.publicSelectedKeys = this.selectedKeys;
        } else {
            this.publicKeys = Collections.unmodifiableSet(this.keys);
            this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
        }

    }
}
```

然后我们看WindowsSelectorImpl，见下图：

![image-20220522152734456](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220522152734456.png)

这里展示了WindowsSelectorImpl的结构和构造函数，其中辅助线程仅在Windows平台使用，因为一个线程轮询1024个文件描述符，所以需要辅助线程。也正是因为如此，windows平台性能不高，因为要开很多线程。

我们下面来看WindowsSelectorImpl的几个重要成员：

1. FdMap：保存的是文件描述符和SelectionKey的映射，是一个map结构，源码如下：

   ```java
   //在pollArray中将文件描述符映射到它们的索引
   private static final class FdMap extends HashMap<Integer, WindowsSelectorImpl.MapEntry> {
       static final long serialVersionUID = 0L;
   
       private FdMap() {
       }
   
       private WindowsSelectorImpl.MapEntry get(int var1) {
           return (WindowsSelectorImpl.MapEntry)this.get(new Integer(var1));
       }
   
       private WindowsSelectorImpl.MapEntry put(SelectionKeyImpl var1) {
           return (WindowsSelectorImpl.MapEntry)this.put(new Integer(var1.channel.getFDVal()), new WindowsSelectorImpl.MapEntry(var1));
       }
   
       private WindowsSelectorImpl.MapEntry remove(SelectionKeyImpl var1) {
           Integer var2 = new Integer(var1.channel.getFDVal());
           WindowsSelectorImpl.MapEntry var3 = (WindowsSelectorImpl.MapEntry)this.get(var2);
           return var3 != null && var3.ski.channel == var1.channel ? (WindowsSelectorImpl.MapEntry)this.remove(var2) : null;
       }
   }
   ```

2. SubSelector：内部封装了JNI poll系统调用，以及获取SelectionKey的方法。每一个Selector中都有一个SubSelector，SubSelector内部保存了select/poll/epoll获取到的可读文件描述符，可写文件描述符和异常的文件描述符，这样Selector就会有自己单独的就绪文件描述符数组。源码如下：

   ![image-20220522155054224](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220522155054224.png)

3. PollArrayWrapper：内部有一个pollArray用Unsafe类申请一块物理内存，存放注册时的socket句柄fdVal和event的数据结构，其中pollfd共8字节，0~3字节保存socket句柄，4~7字节保存event。这两个信息在8字节数据单元中的偏移量分别由FD_OFFSET和EVENT_OFFSET标识，也就是0和4，源码如下：

   ~~~java
   class PollArrayWrapper {
   
       private AllocatedNativeObject pollArray; // The fd array 底层内存空间
   
       long pollArrayAddress; // pollArrayAddress 内存空间起始位置
   
       @Native private static final short FD_OFFSET     = 0; // fd offset in pollfd 文件描述起始位置
       @Native private static final short EVENT_OFFSET  = 4; // events offset in pollfd 感兴趣事件开始位置
   
       static short SIZE_POLLFD = 8; // sizeof pollfd struct 文件描述符长度+事件长度=4+4
   
       // events masks
       @Native static final short POLLIN     = AbstractPollArrayWrapper.POLLIN;
       @Native static final short POLLOUT    = AbstractPollArrayWrapper.POLLOUT;
       @Native static final short POLLERR    = AbstractPollArrayWrapper.POLLERR;
       @Native static final short POLLHUP    = AbstractPollArrayWrapper.POLLHUP;
       @Native static final short POLLNVAL   = AbstractPollArrayWrapper.POLLNVAL;
       @Native static final short POLLREMOVE = AbstractPollArrayWrapper.POLLREMOVE;
       @Native static final short POLLCONN   = 0x0002;
   
       private int size; // Size of the pollArray 
       PollArrayWrapper(int var1) {
           int var2 = var1 * SIZE_POLLFD;
           this.pollArray = new AllocatedNativeObject(var2, true);
           this.pollArrayAddress = this.pollArray.address();
           this.size = var1;
       }
   }
   ~~~

   AllocatedNativeObject继承NativeObject，下面是NativeObject的构造函数，可以看到是unsafe直接申请的堆外内存（或者说是直接内存，这是因为java和系统调用两方面都要能调用所以使用了堆外内存）。

   ```java
   protected NativeObject(int var1, boolean var2) {
       if (!var2) {
           this.allocationAddress = unsafe.allocateMemory((long)var1);
           this.address = this.allocationAddress;
       } else {
           int var3 = pageSize();
           long var4 = unsafe.allocateMemory((long)(var1 + var3));
           this.allocationAddress = var4;
           this.address = var4 + (long)var3 - (var4 & (long)(var3 - 1));
       }
   
   }
   ```

   pollArray的结构如下图右边，fd是int类型4字节；eventOps和readyOps是short类型是两字节。所以每一个pollArray数组成员是8个字节（左边其实是channelArray是一个选择键数组，在下面的注册中还会再次讲到这个图）

   ![image-20220524110023256](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524110023256.png)

   以上就是关于Selector创建的相关知识，这里只介绍了创建涉及到的类和某些类的关键属性，而它们的作用就是我们接下来要学习的内容。

> 在不同操作系统中DefaultSelectorProvider的实现类不同：
>
> - macosx：KQueueSelectorProvider，源码链接：[DefaultSelectorProvider](https://github.com/openjdk/jdk8/blob/master/jdk/src/macosx/classes/sun/nio/ch/DefaultSelectorProvider.java)
> - linux：PollSelectorProvider或EPollSelectorProvider，源码链接：[DefaultSelectorProvider ](https://github.com/openjdk/jdk8/blob/master/jdk/src/solaris/classes/sun/nio/ch/DefaultSelectorProvider.java)
> - SunOS：DevPollSelectorProvider，源码链接：[DefaultSelectorProvider ](https://github.com/openjdk/jdk8/blob/master/jdk/src/solaris/classes/sun/nio/ch/DefaultSelectorProvider.java)
> - windows:WindowsSelectorProvider，源码链接：[DefaultSelectorProvider](https://github.com/openjdk/jdk8/blob/master/jdk/src/windows/classes/sun/nio/ch/DefaultSelectorProvider.java)
>

#### 注册Channel到Selector

在创建完选择器之后，需要将channel注册到选择器上（代码如下），下面我们来详细的了解这一过程

```java
SelectionKey selectionKey = socketChannel.register(selector, SelectionKey.OP_READ | SelectionKey.OP_WRITE);
```

首先我们看一下注册的整体结构图：

![image-20220523090003990](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220523090003990.png)

通俗的来讲，就是将此socket交由Selector统一轮询管理。

下面我们来看一下注册方法的整体流程：

![image-20220523090732982](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220523090732982.png)

我们下面就跟着流程一步一步查看注册的原理，首先我们从`socketChannel.register`跟进,源码如下：

```java
//这里的第二个参数是事件集合
public final SelectionKey register(Selector sel, int ops)
    throws ClosedChannelException
{
    return register(sel, ops, null);
}
public abstract SelectionKey register(Selector sel, int ops, Object att)
        throws ClosedChannelException;
```

这里register会调用同类中的抽象方法，然后调用抽象方法的实现，源码如下：

```java
public final SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException{
    synchronized (regLock) {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (blocking)
            throw new IllegalBlockingModeException();
        //如果已经注册过，就直接添加事件和附件
        SelectionKey k = findKey(sel);
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        if (k == null) {
            // New registration
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```

在这个方法中如果没注册过，会调用这句代码`k = ((AbstractSelector)sel).register(this, ops, att);`，这句代码会调用`AbstractSelector#register`方法，这同样是一个抽象方法，源码如下：

```java
protected abstract SelectionKey register(AbstractSelectableChannel ch, int ops, Object att);
//抽象实现
protected final SelectionKey register(AbstractSelectableChannel var1, int var2, Object var3) {
    if (!(var1 instanceof SelChImpl)) {
        throw new IllegalSelectorException();
    } else {
        SelectionKeyImpl var4 = new SelectionKeyImpl((SelChImpl)var1, this);
        var4.attach(var3);//添加附件
        synchronized(this.publicKeys) {
            this.implRegister(var4);//进行注册
        }

        var4.interestOps(var2);
        return var4;
    }
}
```

然后我们分两部分继续跟进：

首先进入到`this.implRegister(var4);`方法，此方法是抽象方法，根据不同平台有不同的实现，这里看windows平台实现：

```java
protected abstract void implRegister(SelectionKeyImpl var1);
//抽象方法实现
protected void implRegister(SelectionKeyImpl ski) {
    synchronized (closeLock) {
        if (pollWrapper == null)
            throw new ClosedSelectorException();
        //如果当前channel的数量等于SelectionKey数组的大小，对SelectionKeyImpl数组和pollWrapper数组进行扩容
        growIfNeeded();
        //将选择键加入数组（选择键数组）
        channelArray[totalChannels] = ski;
        //设置此选择键在数组中的位置
        ski.setIndex(totalChannels);
        fdMap.put(ski);//将选择键放入映射map中
        keys.add(ski);//注册选择键
        pollWrapper.addEntry(totalChannels, ski);//socket句柄添加到对应的pollfd
        totalChannels++;
    }
}
```

我们注意一下上面代码的`pollWrapper.addEntry`方法。这个方法里，会将socket存入PollArrayWrapper，PollArrayWrapper在Selector的创建中已经学习过了这内部是一个pollArray，存的是文件描述符和事件，源码如下：

~~~java
void addEntry(int index, SelectionKeyImpl ski) {
	putDescriptor(index, ski.channel.getFDVal());//getFDVal获取文件描述符（操作系统编号）
}
// Access methods for fd structures，计算位置，将fd放入pollArray正确的位置上
void putDescriptor(int i, int fd) {
	pollArray.putInt(SIZE_POLLFD * i + FD_OFFSET, fd);
}
~~~

> 上面代码我们可以看到channelArray和pollWrapper使用的索引是一样的，因此它俩的的位置也是一样的，如下图，也就是说SelectionKey和pollArray的每个成员都能对应上。
>
> ![image-20220524110023256](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524110023256.png)

我们了解完这条分支然后回到`AbstractSelector#register`，如下图

![image-20220524110608956](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524110608956.png)

下面我们继续跟进` var4.interestOps(var2);`方法。

此方法最终会调用`SelectionKeyImpl#nioInterestOps(int)`,调用过程源码如下：

```java
public SelectionKey interestOps(int var1) {
    this.ensureValid();
    return this.nioInterestOps(var1);
}
//参数是事件集合即int ops
public SelectionKey nioInterestOps(int var1) {
    if ((var1 & ~this.channel().validOps()) != 0) {
        throw new IllegalArgumentException();
    } else {
        this.channel.translateAndSetInterestOps(var1, this);
        this.interestOps = var1;
        return this;
    }
}
```

然后我们跟进translateAndSetInterestOps方法，此方法是个抽象方法，我们进入到他的实现类方法：

```java
//SocketChannelImpl
/**
* Translates an interest operation set into a native poll event set
* 将关注的事件翻译成对应的POLL类型本地事件写入到pollWrapper属性中
*/
public void translateAndSetInterestOps(int ops, SelectionKeyImpl sk) {
    int newOps = 0;
    if ((ops & SelectionKey.OP_READ) != 0)
        newOps |= PollArrayWrapper.POLLIN;
    if ((ops & SelectionKey.OP_WRITE) != 0)
        newOps |= PollArrayWrapper.POLLOUT;
    if ((ops & SelectionKey.OP_CONNECT) != 0)
        newOps |= PollArrayWrapper.POLLCONN;
    sk.selector.putEventOps(sk, newOps);
}
```

然后我们跟进putEventOps方法：

~~~java
//WindowsSelectorImpl
public void putEventOps(SelectionKeyImpl sk, int ops) {
    synchronized (closeLock) {
        if (pollWrapper == null)
            throw new ClosedSelectorException();
        // make sure this sk has not been removed yet
        //取出sk在pollArray轮询数组的位置
        int index = sk.getIndex();
        if (index == -1)
            throw new CancelledKeyException();
        pollWrapper.putEventOps(index, ops);//此方法重点是这句代码
    }
}

//继续跟进pollWrapper.putEventOps(index, ops);
//PollArrayWrapper
void putEventOps(int i, int event) {
    //将事件信息存放到pollArray中指定的位置。位置*长度+事件偏移量
    pollArray.putShort(SIZE_POLLFD * i + EVENT_OFFSET, (short)event);
}
~~~

到这里，注册的流程我们就跟完了，我们现在来看一下注册流程中的细节。首先是跟一下在WindowsSelectorImpl#implRegister方法中的growIfNeed扩容逻辑：

~~~java
//WindowsSelectorImpl
private void growIfNeeded() {
    //如果channel数组满了，扩容2倍
    if (channelArray.length == totalChannels) {
        int newSize = totalChannels * 2; // Make a larger array
        SelectionKeyImpl temp[] = new SelectionKeyImpl[newSize];
        System.arraycopy(channelArray, 1, temp, 1, totalChannels - 1);
        channelArray = temp;
        pollWrapper.grow(newSize);
    }
    //达到最大文件描述符（1024）则增加辅助线程，这种方式是仅在windows平台实现的方式，是Windows平台提升poll性能的方式，但是也存在性能问题，比如过多线程造成CPU飙高。
    if (totalChannels % MAX_SELECTABLE_FDS == 0) { // more threads needed
        //将唤醒的文件描述符加入到扩容后的第一个位置
        pollWrapper.addWakeupSocket(wakeupSourceFd, totalChannels);
        totalChannels++;
        threadsCount++;
    }
}
//PollArrayWrapper
void grow(int newSize) {
    //创建新的数组
    PollArrayWrapper temp = new PollArrayWrapper(newSize);
    //将原来数组内容存放到新数组
    for (int i = 0; i < size; i++)
        replaceEntry(this, i, temp, i);
    //释放原来的数组
    pollArray.free();
    //更新引用、大小和地址
    pollArray = temp.pollArray;
    this.size = temp.size;
    pollArrayAddress = pollArray.address();
}
~~~

#### Selector.select()

我们在注册通道，完成一些初始化的工作后，会调用Selector的select方法进行轮询，当轮询调用select()方法时：

- 首先根据cancelledKeys去删除registeredKeys和selectedKeys中的需要取消的key
- 然后调用操作系统去做操作系统级别的select，一旦有registeredKeys感兴趣的事件，则将对应事件的key添加到selectedKey中
- 如果selectedKey已经存在了的key，则将事件添加到key中的readyOps（已经准备好的事件集中）。

我们下面来一步一步跟进分析源码：

首先进入到SelectorImpl#select(long)方法：

~~~java
public int select(long timeout) throws IOException{
    if (timeout < 0)
        throw new IllegalArgumentException("Negative timeout");
    return lockAndDoSelect((timeout == 0) ? -1 : timeout);//此方法为下一步主要流程
}

public int select() throws IOException {
    return select(0);
}
~~~

此方法主要调用了lockAndDoSelect方法，所以我们继续跟进：

~~~java
 private int lockAndDoSelect(long timeout) throws IOException {
     synchronized (this) {
         if (!isOpen())
             throw new ClosedSelectorException();
         synchronized (publicKeys) {
             synchronized (publicSelectedKeys) {
                 return doSelect(timeout);
             }
         }
     }
 }
~~~

这里的核心是doSelect方法，这个方法是抽象方法，我们跟进windows的实现类：

~~~java
protected int doSelect(long timeout) throws IOException {
    if (channelArray == null)
        throw new ClosedSelectorException();
    this.timeout = timeout; // set selector timeout
    processDeregisterQueue();
    if (interruptTriggered) {
        resetWakeupSocket();
        return 0;
    }
    // Calculate number of helper threads needed for poll. If necessary
    // threads are created here and start waiting on startLock
    adjustThreadsCount();
    finishLock.reset(); // reset finishLock
    // Wakeup helper threads, waiting on startLock, so they start polling.
    // Redundant threads will exit here after wakeup.
    startLock.startThreads();
    // do polling in the main thread. Main thread is responsible for
    // first MAX_SELECTABLE_FDS entries in pollArray.
    try {
        begin();
        try {
            //调用本地选择器的poll方法
            subSelector.poll();
        } catch (IOException e) {
            finishLock.setException(e); // Save this exception
        }
        // Main thread is out of poll(). Wakeup others and wait for them
        if (threads.size() > 0)
            finishLock.waitForHelperThreads();
    } finally {
        end();
    }
    // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
    finishLock.checkForException();
    processDeregisterQueue();
    int updated = updateSelectedKeys();//将就绪的文件描述符更新成SelectionKey
    // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
    resetWakeupSocket();
    return updated;
}
~~~

这个doSelect方法中比较最重要的地方有两点一是`subSelector.poll()`调用本地选择器的poll方法，然后poll会调用本地方法poll0()，源码如下：

~~~java
private int poll() throws IOException{ // poll for the main thread
            return poll0(pollWrapper.pollArrayAddress,
                         Math.min(totalChannels, MAX_SELECTABLE_FDS),
                         readFds, writeFds, exceptFds, timeout);
        }

private int poll(int index) throws IOException {
    // poll for helper threads
    return  poll0(pollWrapper.pollArrayAddress +
                  (pollArrayIndex * PollArrayWrapper.SIZE_POLLFD),
                  Math.min(MAX_SELECTABLE_FDS,
                           totalChannels - (index + 1) * MAX_SELECTABLE_FDS),
                  readFds, writeFds, exceptFds, timeout);
}
//参数：pollArray数组地址，文件描述符数量，读、写异常文件描述符数组（也可以理解为事件数组），阻塞时间
private native int poll0(long pollAddress, int numfds,
                         int[] readFds, int[] writeFds, int[] exceptFds, long timeout);
~~~

poll0的源码在[WindowsSelectorImpl.c](https://github.com/openjdk/jdk8/blob/master/jdk/src/windows/native/sun/nio/ch/WindowsSelectorImpl.c)中：大概操作就是将fds指定的fd，解析到readfds，writefds和exceptfds中。等select返回后，结果信息再存放在readfds，writefds和exceptfds中。源码可以自行去查看。

doSelect方法中第二个比较重要的一点是`updateSelectedKeys()`方法,此方法将就绪的文件描述符更新成SelectionKey，源码如下：

~~~java
private int updateSelectedKeys() {
    updateCount++;
    int numKeysUpdated = 0;
    numKeysUpdated += subSelector.processSelectedKeys(updateCount);
    for (SelectThread t: threads) {
        numKeysUpdated += t.subSelector.processSelectedKeys(updateCount);
    }
    return numKeysUpdated;
}
//分别对可读，可写和异常调用processFDSet方法
private int processSelectedKeys(long updateCount) {
    int numKeysUpdated = 0;
    numKeysUpdated += processFDSet(updateCount, readFds,
                                   PollArrayWrapper.POLLIN,
                                   false);
    numKeysUpdated += processFDSet(updateCount, writeFds,
                                   PollArrayWrapper.POLLCONN |
                                   PollArrayWrapper.POLLOUT,
                                   false);
    numKeysUpdated += processFDSet(updateCount, exceptFds,
                                   PollArrayWrapper.POLLIN |
                                   PollArrayWrapper.POLLCONN |
                                   PollArrayWrapper.POLLOUT,
                                   true);
    return numKeysUpdated;
}
//将文件描述符和事件转换成选择键
private int processFDSet(long updateCount, int[] fds, int rOps,
                         boolean isExceptFds)
{
    int numKeysUpdated = 0;
    for (int i = 1; i <= fds[0]; i++) {
        int desc = fds[i];
        if (desc == wakeupSourceFd) {
            synchronized (interruptLock) {
                interruptTriggered = true;
            }
            continue;
        }
        //根据fd导航从fdMap中找到选择键
        MapEntry me = fdMap.get(desc);
        if (me == null)
            continue;
        SelectionKeyImpl sk = me.ski;
        if (isExceptFds &&
            (sk.channel() instanceof SocketChannelImpl) &&
            discardUrgentData(desc))
        {
            continue;
        }

        if (selectedKeys.contains(sk)) { // Key in selected set
            if (me.clearedCount != updateCount) {
                if (sk.channel.translateAndSetReadyOps(rOps, sk) &&
                    (me.updateCount != updateCount)) {
                    me.updateCount = updateCount;
                    numKeysUpdated++;
                }
            } else { // The readyOps have been set; now add
                if (sk.channel.translateAndUpdateReadyOps(rOps, sk) &&//调用translateAndUpdateReadyOps进行转换，并存入sk
                    (me.updateCount != updateCount)) {
                    me.updateCount = updateCount;
                    numKeysUpdated++;
                }
            }
            me.clearedCount = updateCount;
        } else { //。。。略
        }
    }
    return numKeysUpdated;
}
~~~

因为对这几个集合的操作不是线程安全的,所以**一般使用Selector的select()只用单线程**而对于select得到的channel和对应的IO操作,可以新开线程或者使用线程池来处理。这也正是IO复用的意义所在。

#### Selector.wakeup()

我们在调用select方法时，会在底层系统调用进行阻塞，那么NIO就提供了一个wakeup方法，用于唤醒这个阻塞的线程。方法定义如下，是一个抽象方法：

```java
public abstract Selector wakeup();
```

此方法不同的平台有不同的实现，原理如下：

PollSelectorImpl在select过程中的阻塞时间受控于Channel的事件，一旦有事件才返回，所以为了手动控制就额外增加了一个对pipe的读监控，将pipe的文件描述符加到PollArrayWrapper中的第一个位置，如果我们对这个pipe进行写入数据操作，那么pipe的读文件描述符必然会收集到读事件，这样就可以不在阻塞，立即返回。

在Windows上会建立一对自己和自己的loopback的TCP（socket）连接，在Linux上回开一对pipe管道（pipe在linux上一般都是成对打开的），如果想唤醒select，只需要朝自己连接或pipe发点数据，就可以唤醒阻塞的select线程了。

下面我们看一下WindowsSelectorImpl的loopback（回环）连接，在WindowsSelectorImpl的构造方法中对此进行了定义：

~~~java
WindowsSelectorImpl(SelectorProvider sp) throws IOException {
    super(sp);
    pollWrapper = new PollArrayWrapper(INIT_CAP);
	//打开管道
    wakeupPipe = Pipe.open();
    //获取接收端source的fd
    wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

    // Disable the Nagle algorithm so that the wakeup is more immediate
    //获取发送端sink
    SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
    //禁用Nagle算法，即发送消息不能有延迟
    (sink.sc).socket().setTcpNoDelay(true);
    //发送端的fd，用于后续发消息
    wakeupSinkFd = ((SelChImpl)sink).getFDVal();
	//接收端的fd加入poll队列的第一个
    pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
}
~~~

> 注意这里我们禁用了Nagle算法`(sink.sc).socket().setTcpNoDelay(true);`这是为什么呢？因为我们的消息不能有延迟，像唤醒必须要立马进行唤醒。而Nagle算法就是为了尽可能发送大块数据，避免网络中充斥着许多小数据块。Nagle算法的基本定义是任意时刻，最多只能有一个未被确认的小段。 所谓“小段”，指的是小于MSS尺寸的数据块，所谓“未被确认”，是指一个数据块发送出去后，没有收到对方发送的ACK确认该数据已收到，也就是说当我们发送很小的一段数据用于唤醒时，会被Nagle算法因为过小而不被发送，所以我们要禁用Nagle延迟。

那么接下来我们就看一下这个Pipe的在windows上的实现类PipeImpl：

~~~java
class PipeImpl extends Pipe {
    private static final int NUM_SECRET_BYTES = 16;
    private static final Random RANDOM_NUMBER_GENERATOR = new SecureRandom();
    private SourceChannel source;//接收端
    private SinkChannel sink;//发送端
    public SourceChannel source() {
        return this.source;
    }

    public SinkChannel sink() {
        return this.sink;
    }
	//略///
}
~~~

在上面的代码中存在两个重要的属性source接收端和sink发送端。除了这两个属性PipeImpl中还有一个内部类，源码如下，这个类中进行了sink和source的新建：

~~~java
private class LoopbackConnector implements Runnable {

    @Override
    public void run() {
        ServerSocketChannel ssc = null;
        SocketChannel sc1 = null;
        SocketChannel sc2 = null;

        try {
            // Loopback address
            InetAddress lb = InetAddress.getByName("127.0.0.1");
            assert(lb.isLoopbackAddress());
            InetSocketAddress sa = null;
            for(;;) {
                // Bind ServerSocketChannel to a port on the loopback
                // address
                if (ssc == null || !ssc.isOpen()) {
                    //开一个监听套接字
                    ssc = ServerSocketChannel.open();
                    //绑定一个随机端口
                    ssc.socket().bind(new InetSocketAddress(lb, 0));
                    sa = new InetSocketAddress(lb, ssc.socket().getLocalPort());
                }

                // Establish connection (assume connections are eagerly
                // accepted)
                //建立一个套接字连接
                sc1 = SocketChannel.open(sa);
                ByteBuffer bb = ByteBuffer.allocate(8);
                long secret = rnd.nextLong();
                bb.putLong(secret).flip();
                sc1.write(bb);//写入校验密码

                // Get a connection and verify it is legitimate
                //接收连接进行校验
                sc2 = ssc.accept();
                bb.clear();
                sc2.read(bb);
                bb.rewind();
                if (bb.getLong() == secret)//校验结果正确
                    break;
                //校验结果不正确
                sc2.close();
                sc1.close();
            }

            // Create source and sink channels，建立发送端和接收端通道
            source = new SourceChannelImpl(sp, sc1);
            sink = new SinkChannelImpl(sp, sc2);
        } catch (IOException e) {
            try {
                if (sc1 != null)
                    sc1.close();
                if (sc2 != null)
                    sc2.close();
            } catch (IOException e2) {}
            ioe = e;
        } finally {
            try {
                if (ssc != null)
                    ssc.close();
            } catch (IOException e2) {}
        }
    }
}
~~~

这个内部类实际上整体流程如下图：

![image-20220525211237185](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220525211237185.png)

既然我们搞明白了wakeup的原理，那么接下来我们就跟一下wakeup的源码流程，当调用wakeup()方法时，(windows平台)最终会调用`sun.nio.ch.WindowsSelectorImpl#wakeup`：

```java
public Selector wakeup() {
    synchronized(this.interruptLock) {
        if (!this.interruptTriggered) {
            this.setWakeupSocket();
            this.interruptTriggered = true;
        }

        return this;
    }
}
private void setWakeupSocket() {
    this.setWakeupSocket0(this.wakeupSinkFd);
}

private native void setWakeupSocket0(int var1);
```

这里会调用setWakeupSocket方法，而次方法最终会调用本地方法setWakeupSocket0：

~~~c
JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
~~~

我们可以看到底层时调用send方法，发送一个字节，来唤醒poll，所以wakeup可以唤醒阻塞的select方法。

以上就是NIO中Selector的原理。



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

Doug Lea（JUC作者）在《Scalable IO in Java》的文章中实现了一个Reactor反应器模式的参考代码，我们可以一起来看一下：

首先他给出了单线程Reactor的设计图：

![image-20220507125756667](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507125756667.png)

可以看到单线程Reactor模式，Reactor反应器和所有的Handler处理器实例的执行，都执行在同一条线程中。

我们下面来看参考代码，深刻体会上图：

首先Lea给出了Reactor的代码：

![image-20220507153520085](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507153520085.png)

![image-20220507153540222](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507153540222.png)

![image-20220507153555958](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507153555958.png)

![image-20220507153607476](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507153607476.png)

> 这个wakeup作用如下：If another thread is currently blocked in an invocation of the select() or select(long) methods then that invocation will return immediately. 翻译过来就是此方法会唤醒调用select阻塞。但是在单线程下是没有什么用的。

![image-20220507153617490](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220507153617490.png)

关于doug lea大师的代码，我已经加上了注释。这里介绍上述代码重点：

1. 首先Reactor类启动后，会在在线程方法run中轮询获取事件。
2. Reactor获取事件后然后调用分发方法，将轮询获取的事件进行分发
3. 分发方法主要是取出selectKey中的处理器类，然后执行处理器的run方法。（这里不是调用start而是直接调用Thread对象的 **run()方法**不会启动**单独的线程**，而是可以在当前线程中执行。）
4. 取出selectKey中的处理器类主要调用attachment()方法，这个方法主要用于获取attach中放进去的处理器类。
5. 处理器类有Acceptor和Handler两种，Acceptor是Reactor的内部类，主要用于处理接受请求。Handler主要进行读写事件的注册和处理。

下面我们根据Lea大师的单线程Reactor模式简单实现一个显示消息的服务器端，代码如下：

```java
public class SingleThreadServer implements Runnable {

    private ServerSocketChannel serverSocketChannel;
    private Selector selector;
    private SelectionKey selectionKey;

    public SingleThreadServer(int port) {
        try {
            this.serverSocketChannel = ServerSocketChannel.open();
            this.selector = Selector.open();
            serverSocketChannel.socket().bind(new InetSocketAddress(port));
            System.out.println("开始监听");
            serverSocketChannel.configureBlocking(false);
            //selector中注册OP_ACCEPT事件
            selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            //selector中OP_ACCEPT事件绑定Acceptor对象附件
            selectionKey.attach(new Acceptor());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                final Set<SelectionKey> keySet = selector.selectedKeys();
                final Iterator<SelectionKey> iterator = keySet.iterator();
                while (iterator.hasNext()) {
                    final SelectionKey next = iterator.next();
                    dispatch(next);
                }
                keySet.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void dispatch(SelectionKey key) {
        final Runnable attachment = (Runnable) key.attachment();
        if (attachment != null) {
            attachment.run();
        }
    }
	//连接处理器类
    class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                final SocketChannel accept = serverSocketChannel.accept();
                new Handler(accept, selector);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
	//测试方法
    public static void main(String[] args) {
        new Thread(new SingleThreadServer(8999)).start();
    }
}
```

```java
public class Handler implements Runnable{

    private SocketChannel socketChannel;
    private SelectionKey selectionKey;
    final ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    static final int RECIEVING = 0, SENDING = 1;
    int state = RECIEVING;

    public Handler(SocketChannel socketChannel, Selector selector) {
        try {
            this.socketChannel = socketChannel;
            socketChannel.configureBlocking(false);
            // 之后设置感兴趣的IO事件
            this.selectionKey = socketChannel.register(selector, 0);
            selectionKey.attach(this);
            //注册Read就绪事件
            selectionKey.interestOps(SelectionKey.OP_READ);

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        try {
            if (state == SENDING){
                socketChannel.write(byteBuffer);
                byteBuffer.clear();
                selectionKey.interestOps(SelectionKey.OP_READ);
                state = RECIEVING;
            }
            //最开始在RECIEVING状态
            if (state == RECIEVING){
                int length = 0;
                //length>0,还未读取完
                while ((length = socketChannel.read(byteBuffer))>0) {
                    System.out.println(new String(byteBuffer.array(),0,length));
                }
                byteBuffer.flip();
                selectionKey.interestOps(SelectionKey.OP_WRITE);
                state = SENDING;
            }


        } catch (IOException e) {
            e.printStackTrace();
            selectionKey.cancel();
            try {
                socketChannel.finishConnect();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }

    }
}
```

关于连接上面服务器端的客户端类，这里仅仅简单的进行了实现：

```java
public class NioClient {
    public static void main(String[] args) throws IOException {
        SocketChannel channel =SocketChannel.open();
        channel.configureBlocking(false);
        InetSocketAddress address = new InetSocketAddress("127.0.0.1",8999);
        if (!channel.connect(address)){
            while (!channel.finishConnect()){//nio非阻塞的优势
                System.out.println("Client: 链接服务器的同时，干别的事");
            }
        }
        while (true){
            Scanner scanner = new Scanner(System.in);
            final String msg = scanner.nextLine();
            ByteBuffer writebuffer = ByteBuffer.wrap(msg.getBytes());
            //发送数据
            channel.write(writebuffer);
        }

    }
}
```

### 12.1.3 单线程Reactor模式优缺点

此模式基于NIO实现，相较于传统的BIO模式，不需要一直启动线程，避免了上下文的同时切换，服务器效率大大提升。

但是正是由于都在一条线程上，所以当某一个Handler阻塞后会导致其他的Handler都不能执行，并且负责监听的Acceptor其实也不能执行，这就会导致服务器无响应。

因此，在高性能服务器的应用场景中，单线程反应器模式使用很少。

### 12.1.4 多线程Reactor模式

我们还是来看一看Doug Lea关于对多线程Reactor模型的阐述：

![image-20220509171207917](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220509171207917.png)

其实主要是说，首先要适当的增加线程提升效率，然后要升级Reactor的线程，提高查询和分发IO事件的能力，还有升级工作线程（即Handler），使其能快速处理IO事件。

因此最后Lea给出了这样的架构图：

![image-20220509171248833](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220509171248833.png)

总体来说，可以将IOHandler放入独立线程池，分离监听和业务处理，同时拆分Reactor，引入多个选择器，一个线程负责一个选择器的轮询，充分释放系统资源的能力。

下面，我们将根据上面的单线程的显示消息的服务器端进行改造，使其成为多线程的Reactor模式，具体改造思路如下：

1. 增加boss和worker两种选择器，boss负责查询和分发新连接事件，worker负责查询和分发IO事件
2. 创建两种Reactor，bossReactor负责新连接事件的处理，与boss选择器绑定；workerReactor负责IO事件的查询和分发，绑定worker选择器。
3. 服务器Channel注册到boss选择器，所有的channel轮询注册到worker选择器，实现新连接监听和IO读写事件的分离

代码如下（客户端代码跟单线程一样）：

```java
public class MultiThreadServer {
    Selector bossSelector = Selector.open();
    Selector[] workerSelectors = new Selector[2];
    ServerSocketChannel serverSocketChannel;
    AtomicInteger index = new AtomicInteger(0);
    Reactor bossReactor;
    Reactor [] workerReactors = new Reactor[2];

    public MultiThreadServer() throws IOException {
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().bind(new InetSocketAddress("127.0.0.1",8999));
        serverSocketChannel.configureBlocking(false);
        workerSelectors[0] = Selector.open();
        workerSelectors[1] = Selector.open();
        bossReactor = new Reactor(bossSelector);
        workerReactors[0] = new Reactor(workerSelectors[0]);
        workerReactors[1] = new Reactor(workerSelectors[1]);
        SelectionKey selectionKey = serverSocketChannel.register(bossSelector, SelectionKey.OP_ACCEPT);
        selectionKey.attach(new Acceptor());
    }

    class Reactor implements Runnable {

        Selector selector;

        public Reactor(Selector selector) {
            this.selector = selector;
        }

        @Override
        public void run() {
            try {
                while (!Thread.interrupted()) {
                    selector.select(1000);
                    final Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    if (selectionKeys == null || selectionKeys.size() == 0) {
                        continue;
                    }
                    Iterator<SelectionKey> iterator = selectionKeys.iterator();
                    while (iterator.hasNext()) {
                        SelectionKey next = iterator.next();
                        dispatch(next);
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    private void dispatch(SelectionKey next) {
        Runnable attachment = (Runnable) next.attachment();
        attachment.run();

    }

    //启动boss和worker的Reactor线程
    public void start(){
        new Thread(bossReactor).start();
        new Thread(workerReactors[0]).start();
        new Thread(workerReactors[1]).start();
    }
    //连接处理器
    class Acceptor implements Runnable {
        @Override
        public void run() {
            try {
                //获取channel
                SocketChannel socketChannel = serverSocketChannel.accept();
                if (socketChannel!=null){
                    //新建IO处理器处理读写
                    new MutilHandler(socketChannel,workerSelectors[index.getAndIncrement()]);
                }
                //这里的index是为了做一个workerSelectors的负载均衡
                if (index.get() == 2) {
                    index.set(0);
                }
            }catch (Exception e){
                e.printStackTrace();
            }

        }
    }

    public static void main(String[] args) throws IOException {
        new MultiThreadServer().start();
    }
}
```

```java
public class MutilHandler implements Runnable{
    Selector selector;
    SocketChannel socketChannel;
    SelectionKey selectionKey;
    final ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    static final int RECIEVING = 0, SENDING = 1;
    int state = RECIEVING;
    ExecutorService threadPool = Executors.newFixedThreadPool(4);

    public MutilHandler(SocketChannel channel,Selector workerSelector) throws IOException {
        selector = workerSelector;
        socketChannel =channel;
        socketChannel.configureBlocking(false);
        selectionKey = socketChannel.register(selector, 0);
        selectionKey.interestOps(SelectionKey.OP_READ);
        selectionKey.attach(this);
        selector.wakeup();
    }

    @Override
    public void run() {
        threadPool.execute(()->{
            doTask();
        });
    }

    public synchronized void doTask(){
        try {
            if (state == SENDING){
                socketChannel.write(byteBuffer);
                byteBuffer.clear();
                selectionKey.interestOps(SelectionKey.OP_READ);
                state = RECIEVING;
            }
            //最开始在RECIEVING状态
            if (state == RECIEVING){
                int length = 0;
                //length>0,还未读取完
                while ((length = socketChannel.read(byteBuffer))>0) {
                    System.out.println(new String(byteBuffer.array(),0,length));
                }
                byteBuffer.flip();
                selectionKey.interestOps(SelectionKey.OP_WRITE);
                state = SENDING;
            }


        } catch (IOException e) {
            e.printStackTrace();
            selectionKey.cancel();
            try {
                socketChannel.finishConnect();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
        }
    }

}
```

# 十三、Netty

## 13.1 Netty入门

我们先从一个入门案例来简单了解一下netty，案例代码来自于我的博客

服务器端代码

~~~java
public class HelloServer {
    public static void main(String[] args) {
        // 1、启动器，负责装配netty组件，启动服务器
        new ServerBootstrap()
                // 2、创建 NioEventLoopGroup，可以简单理解为 线程池 + Selector
                .group(new NioEventLoopGroup())
                // 3、选择服务器的 ServerSocketChannel 实现
                .channel(NioServerSocketChannel.class)
                // 4、child 负责处理读写，该方法决定了 child 执行哪些操作
            	// ChannelInitializer 处理器（仅执行一次）
            	// 它的作用是待客户端SocketChannel建立连接后，执行initChannel以便添加更多的处理器
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                        // 5、SocketChannel的处理器，使用StringDecoder解码，ByteBuf=>String
                        nioSocketChannel.pipeline().addLast(new StringDecoder());
                        // 6、SocketChannel的业务处理，使用上一个处理器的处理结果
                        nioSocketChannel.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override//读事件
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                System.out.println(msg);
                            }
                        });
                    }
                    // 7、ServerSocketChannel绑定8080端口
                }).bind(8080);
    }
}

~~~

客户端代码

~~~java
public class HelloClient {
    public static void main(String[] args) throws InterruptedException {
        new Bootstrap()
                .group(new NioEventLoopGroup())
                // 选择客户端 Socket 实现类，NioSocketChannel 表示基于 NIO 的客户端实现
                .channel(NioSocketChannel.class)
                // ChannelInitializer 处理器（仅执行一次）
                // 它的作用是待客户端SocketChannel建立连接后，执行initChannel以便添加更多的处理器
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel channel) throws Exception {
                        // 消息会经过通道 handler 处理，这里是将 String => ByteBuf 编码发出
                        channel.pipeline().addLast(new StringEncoder());
                    }
                })
                // 连接服务器指定地址和端口
                .connect(new InetSocketAddress("localhost", 8080))
                // Netty 中很多方法都是异步的，如 connect
                // 这时可以使用 sync 同步方法等待 connect 建立连接完毕
                .sync()
                // 获取 channel 通道对象，可以进行数据读写操作
                .channel()
                // 写入消息并清空缓冲区
                .writeAndFlush("hello world");
    }
}

~~~

此案例的流程（图片来自于我的个人博文[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)）：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211012093559832.png)



上面的案例都是用netty写的，主要展示了常用组件和netty的简单写法，其实就是使用了netty的各个组件组成了服务器端和客户端代码。我们将在下面一一学习netty的各个组件的用法。

## 13.2 Netty的Reactor模式

在学习Nettu的各个组件前，我们先了解一下netty的Reactor模式。reactor模式我们在上面第十二章已经学习过了。Reactor模式的流程如下：

![image-20220523164206406](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220523164206406.png)

上面的流程其实总结下来就是四步：

1. 注册通道
2. 查询事件
3. 分发事件
4. 处理事件

下面我们来了解一下Netty为Reactor模式提供的

### 13.2.1 Netty中的Channel组件

Netty中不直接使用NIO的Channel组件，而是进行了自己的封装。Netty中实现了一系列的Channel组件，这么做是为了支持多种通信协议，除了NIO，Netty还提供了Java阻塞式（BIO）的处理通道。

Netty对于每一种协议基本上都有NIO和OIO(即阻塞式的)两个版本，常见的通道如下：

- NioSocketChannel：异步非阻塞TCP socketc传输通道
- NioServerSocketChannel：异步非阻塞TCP socket服务端监听通道
- NioDatagramChannel：异步非阻塞的UDP传输通道
- NioSctpChannel：异步非阻塞Sctp传输通道
- NioSctpServerChannel：异步非阻塞Sctp服务器端监听通道
- OioSocketChannel：同步阻塞式TCP Socket传输通道
- OioServerSocketChannel：同步阻塞式TCP Socket服务器端监听通道
- OioDatagramChannel：同步阻塞式UDP传输通道
- OioSctpChannel：同步阻塞式Sctp传输通道
- OioSctpServerChannel：同步阻塞式Sctp服务器端监听通道

一般来说，服务器端编程用到最多的通信协议还是TCP协议，其对应的Netty传输通道类型为NioSocketChannel类，其对应的Netty服务器监听通道类型为NioServerSocketChannel。但是服务端监听通道和传输通道的API都类似。NioSocketChannel的继承关系如下（IDEA可使用Ctrl+H）：

![image-20220524095244162](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524095244162.png)

在Netty的NioSocketChannel内部封装了一个Java NIO的SelectableChannel成员，所有IO操作最终都会落地到Java NIO的SelectableChannel底层通道。

### 13.2.2 Netty中的Reactor反应器

在反应器模式中，一个反应器会有一个事件处理线程负责事件的查询和分发。该线程不断轮询，通过Selector选择器不断查询注册过的IO事件，如果有则分发给Handler业务处理器。

在Netty中Reactor反应器组件有多个实现类，每个实现类都与其Channel通道类型相匹配。比如对于NioSocketChannel通道，Netty的反应器类为NioEventLoop。

NioEventLoop类有两个重要的属性：一个是Thread线程类成员，一个是JavaNIO的选择器的成员属性。NioEventLoop的继承关系如下：

![image-20220524100206205](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524100206205.png)

我们可以看到NioEventLoop与Reactor模式的思路是一致的，一个NioEventLoop拥有一个Thread线程，负责一个Java Nio Selector选择器的IO事件轮询。

在Netty中一个EventLoop可以对应多个channel，他们之间是一对多的关系。

### 13.2.3 Netty中的Handler处理器

在Netty中，EventLoop反应器内部有一个线程负责NIO选择器的轮询，然后进行对应的事件分发。事件分发的目标就是Netty的handler处理器。Netty的Handler处理器有两大类：一是ChannelInboundHandler入站处理器，第二类是ChannelOutboundHandler出站处理器，两者都继承了ChannelHandler处理器接口，继承关系如下图：

![image-20220524101723456](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220524101723456.png)

Netty的入站流程以JavaNIO的OP_READ事件为例：在通道中发生了都时间后，会被EventLoop查到，然后分发给ChannelInboundHandler入站处理器，调用对应的入站处理的read方法。而后在read方法的具体实现中，可以从通道中读取数据。

Netty中的入站处理触发的方向为：由通道触发，ChannelInboundHandler入站处理器负责接收（或者执行）。Netty中的入站处理，不仅仅是OP_READ输入事件的处理，还包括从通道底层触发，由Netty通过层层传递，调用ChannelInboundHandler入站处理器进行的其他某个处理。

Netty中的出站处理指的是从ChannelOutboundHandler处理器到通道的某次IO操作，例如，在应用程序完成业务处理后，可以通过ChannelOutboundHandler出站处理器将处理的结果写入底层通道。它的最常用的一个方法就是write()方法，把数据写入到通道。Netty中的出站处理，不仅仅包括Java NIO的OP_WRITE可写事件，还包括Netty自身从处理器到通道的方向的其他操作。OP_WRITE可写事件是Java NIO的概念，它和Netty的出站处理的在概念不是一个维度，**Netty的出站处理是应用层维度的**。

无论是入站还是出站，Netty都提供了各自的默认适配器实现：ChannelInboundHandler的默认实现为ChannelInboundHandlerAdapter（入站处理适配器）。ChannelOutboundHandler的默认实现为ChanneloutBoundHandlerAdapter（出站处理适配器）。这两个默认的通道处理适配器，分别实现了基本的入站操作和出站操作功能。如果要实现自己的业务处理器，不需要从零开始去实现处理器的接口，只需要继承通道处理适配器即可。

### 13.2.4 Netty的Pipeline通道处理流水线

在Netty中反应器模式的各个组件的关系是这样的：

- 反应器和通道之间是一对多的关系，一个反应器可以查询很多个通道的IO事件。
- 通道和Handler处理器之间是多对多的关系，一个通道的IO事件可以被多个Handler处理，一个Handler也可以绑定多个通道，处理多个通道的IO事件。

那么通道和Handler之间的绑定关系Netty是如何实现的呢？在Netty中，设计了一个特殊的组件，就是ChannelPipeline，它像一条管道，将绑定到一个通道的多个Handler处理器实例串在一起，形成一条流水线。ChannelPipeline（通道流水线）的默认实现，实际上被设计成一个双向链表。所有的Handler处理器实例被包装成了双向链表的节点，被加入到了ChannelPipeline。

以入站处理为例。每一个来自通道的IO事件，都会进入一次ChannelPipeline通道流水线。在进入第一个Handler处理器后，这个IO事件将按照既定的从前往后次序，在流水线上不断地向后流动，流向下一个Handler处理器。

在向后流动的过程中，会出现3种情况：

- 如果后面还有其他Handler入站处理器，那么IO事件可以交给下一个Handler处理器，向后流动。
- 如果后面没有其他的入站处理器，这就意味着这个IO事件在此次流水线中的处理结束了。
- 如果在中间需要终止流动，可以选择不将IO事件交给下一个Handler处理器，流水线的执行也被终止了。

Netty的流水线不是单向的，而是双向的，而普通的流水线基本都是单向的。Netty是这样规定的：入站处理器Handler的执行次序，是从前到后；出站处理器Handler的执行次序，是从后到前。总之，IO事件在流水线上的执行次序，与IO事件的类型是有关系的。除了流动的方向与IO操作类型有关之外，流动过程中所经过的处理器类型，也是与IO操作的类型有关。入站的IO操作只能从Inbound入站处理器类型的Handler流过；出站的IO操作只能从Outbound出站处理器类型的Handler流过。如下图，图片来自于我自己的博客：

![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20211013111316423.png)

下面我们从Netty的BootStrap和EventLoopGroup组件入手逐步学习Netty的每一个组件。

## 13.3 BootStrap和EventLoopGroup

### 13.3.1 概念

Bootstrap类是Netty提供的一个便利的工厂类，可以通过它来完成Netty的客户端或服务器端的Netty组件的组装，以及Netty程序的初始化和启动执行。当然，Netty的官方解释是，完全可以不用这个Bootstrap引导类，可以一点点去手动创建通道、完成各种设置和启动、并且注册到EventLoop反应器然后开始事件的轮询和处理，但是这个过程会非常麻烦。通常情况下，还是使用这个便利的Bootstrap工具类会效率更高。

Netty有一个抽象接口，然后下面有两个BootStrap的实现类，一个是服务器端使用的，另一个是客户端使用的：

![image-20220525212836647](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220525212836647.png)

两者的API都差不多，因此这里以ServerBootStrap为例进行学习。

当然学习BootStrap组件之前，我们再了解一下父子通道和EventLoopGroup的概念：

在netty中，每一个NioSocketChannel都是对JavaNIO通道的封装，再往下对应着socket文件描述符，理论上来说socket文件描述符分为两类：

- 连接监听型：连接监听的socketFd，处于服务器端，负责接收客户端的连接。
- 数据传输型：负责数据传输，同一条TCP的Socket传输链路，在客户端和服务器端都有一个对应的SocketFd。

在Netty中，将有接收关系的监听通道和传输通道，叫做父子通道。其中，负责服务器连接监听和接收的监听通道，也叫父通道，异步非阻塞的服务器端监听通道NioServerSocketChannel，所封装的Linux底层的文件描述符就是“连接监听类型”的socket描述符。对应于每一个接收到的传输类通道，也叫子通道而异步非阻塞的传输通道NioSocketChannel，所封装的Linux的文件描述符，是“数据传输类型”的socket描述符。

然后我们再来了解一下EventLoopGroup。上面我们了解了EventLoop，这其实是Netty对Selector的封装，那么Reactor的多线程模式Netty是如何实现的呢？

其实是使用EventLoopGroup轮询组。多个EventLoop线程放在一起，可以组成一个EventLoopGroup轮询组。反过来说，EventLoopGroup轮询组就是一个多线程版本的反应器，其中的单个EventLoop线程对应于一个子反应器（SubReactor）。我们在开发的时候，一般都是创建EventLoopGroup，创建时可以指定参数，参数为内部线程的个数。如果使用无参构造函数，则默认是CPU核数的两倍。

为了及时接收处理新连接，在Netty服务器端，一般用两个EventLoopGroup轮询组，一个负责新连接的监听和接收，另一个负责IO事件的轮询和分发。两个组的分工如下：

- 新连接监听和接收的EventLoopGroup轮询组，负责完成查询通道的新连接IO事件，这组可以比喻为Boss轮询组
- IO事件的轮询和分发的EventLoopGroup轮询组，负责查询所有子通道的IO事件，并且执行对应的Handler处理器完成IO处理——例如数据的输入和输出，这组可以比喻为Worker轮询组

Netty中的Reactor模式的示例图如下：



![image-20220527131823116](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220527131823116.png)

### 13.3.2 启动流程详解

Netty的BootStrap的启动流程，就是Netty组件的组装配置，以及服务端（或客户端）的启动流程。主要分8个步骤，我们这里用ServerBootStrap进行演示。

开始流程之前首先创建一个ServerBootStrap

~~~java
ServerBootstrap bootstrap = new ServerBootstrap();
~~~

下面的启动主要流程：

1. 创建Reactor反应器轮询组

   ```java
   EventLoopGroup bossGroup = new NioEventLoopGroup();
   EventLoopGroup workerGroup = new NioEventLoopGroup();
   bootstrap.group(bossGroup,workerGroup)//设置线程组
   ```

   如果不需要进行新连接事件和输出事件进行分开监听，就不一定非得配置两个轮询组，可以仅配置一个EventLoopGroup反应器轮询组。具体的配置方法是调用`bootstrap.group(workerGroup)`。在这种模式下，新连接监听IO事件和数据传输IO事件可能被挤在了同一个线程中处理。这样会带来一个风险：新连接的接受被更加耗时的数据传输或者业务处理所阻塞。所以，在服务器端，建议设置成两个轮询组的工作模式.

2. 设置IO通道类型

   ```java
   bootstrap.group(bossGroup,workerGroup)//设置线程组
   		 .channel(NioServerSocketChannel.class)//使用NioServerSocketChannel作为服务器端通道实现
   ```

   如果确实指定Bootstrap的IO模型为BIO类型，可以配置为OioServerSocketChannel.class类即可。由于NIO的优势巨大，通常不会在Netty中使用BIO。

3. 设置监听端口

   ```java
   bootstrap.localAddress(new InetSocketAddress(port));
   //或者是在绑定监听端口时设置端口号。绑定监听端口为下面第六步
   bootstrap.bind(9999);
   ```

4. 设置传输通道的配置选项

   ```java
   bootstrap.option(ChannelOption.SO_BACKLOG,128)//设置线程队列中等待连接的个数
   bootstrap.childOption(ChannelOption.SO_KEEPALIVE,true)//保持活动连接状态
   ```

   这里用到了Bootstrap的option(…) 选项设置方法。对于服务器的Bootstrap而言，这个方法的作用是：给父通道（Parent Channel）通道设置一些与传输协议相关的选项。如果要给子通道（Child Channel）设置一些通道选项，则需要用另外一个childOption(…)设置方法.

5. 装配子通道的Pipeline流水线

   每一个通道都用一条ChannelPipeline流水线。它的内部有一个双向的链表。装配流水线的方式是：将业务处理器ChannelHandler实例包装之后加入双向链表中。

   装配子通道的Handler流水线调用引导类的childHandler()方法，该方法需要传入一个ChannelInitializer通道初始化类的实例作为参数。每当父通道成功接收一个连接，并创建成功一个子通道后，就会初始化子通道，此时这里配置的ChannelInitializer实例就会被调用.

   ```java
   //它代表需要初始化的通道类型，这个类型需要和前面的引导类中设置的传输通道类型，保持一一对应起来
   bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {//创建一个通道初始化对象
       //往Pipeline链中添加自定义业务处理handler
       @Override
       protected void initChannel(SocketChannel ch) throws Exception {
           ch.pipeline().addLast("decoder",new ProtobufDecoder(BookMessage.Book.getDefaultInstance()));
           ch.pipeline().addLast(new NettyServerHandler());
           System.out.println("========Server is Ready========");
       }
   });
   ```

   为什么仅装配子通道的流水线,因为父通道也就是NioServerSocketChannel的内部业务处理是固定的：接受新连接后，创建子通道，然后初始化子通道，所以不需要特别的配置，由Netty自行进行装配。当然，如果需要完成特殊的父通道业务处理，可以类似的使用ServerBootstrap的handler(ChannelHandler handler)方法，为父通道设置ChannelInitializer初始化器。

6. 开始绑定服务器新连接的监听端口

   ```java
   ChannelFuture channelFuture = bootstrap.bind(9999).sync();
   ```

   b.bind()方法的功能：返回一个端口绑定Netty的异步任务channelFuture。上面的代码中并没有给channelFuture异步任务增加回调监听器，而是调用sync阻塞channelFuture异步任务，直到端口绑定任务执行完成。除了阻塞异步任务还可以设置监听器，以获得Netty中的IO操作的真正结果。

7. 自我阻塞，直至服务器关闭

   ```java
   channelFuture.channel().closeFuture().sync();
   ```

   如果要阻塞当前线程直到通道关闭，可以使用通道的closeFuture()方法，以获取通道关闭的异步任务。当通道被关闭时，closeFuture实例的sync()方法会返回。

8. 关闭EventLoopGroup

   ```java
   bossGroup.shutdownGracefully();
   workerGroup.shutdownGracefully();
   ```

### 13.3.3 ChannelOPtion选项

无论是对于NioServerSocketChannel父通道类型，还是对于NioSocketChannel子通道类型，都可以设置一系列的ChannelOption通道选项。ChannelOption类中定义了一系列的选项，下面介绍一些常见的选项：

1. SO_RCVBUF和SO_SNDBUF

   TCP传输选项，每个套接字(TCP socket)，在内核中都有一个发送缓冲区和一个接收缓冲区，这两个选项就是用来设置TCP连接的这两个连接的缓冲区大小。TCP的全双工工作模式以及TCP的滑动窗口对两个独立的缓冲区都有依赖。

2. TCP_NODELAY

   TCP传输选项，如果设置为true表示立即发送数据。此选项用于启用关闭Nagle算法，如果要求高实时性，有数据发送时就马上发送。TCP_NODELAY在Netty中默认为true，操作系统默认为False。

3. SO_KEEPALIVE

   TCP传输选项,表示是否开启TCP协议的心跳机制。true表示保持心跳，默认为false。启用该功能时，TCP会主动探测空闲连接的有效性。可以将此功能视为TCP的心跳机制，需要注意的是：默认的心跳间隔是7200秒即2小时。Netty默认关闭该功能。

4. SO_REUSEADDR

   TCP传输选项，如果为true时表示地址复用，默认值为false。有四种情况需要用到这个参数设置：

   - 当有一个地址和端口相同的连接socket1处于TIME_WAIT状态时，而又希望启动一个新的连接socket2要占用该地址和端口；
   - 有多块网卡或用IP Alias技术的机器在同一端口启动多个进程，但每个进程绑定的本地IP地址不能相同；
   - 同一进程绑定相同的端口到多个socket（套接字）上，但每个socket绑定的IP地址不同；
   - 完全相同的地址和端口的重复绑定，但这只用于UDP的多播，不用于TCP。 

5. SO_LINGER

   TCP传输选项，可以用来控制socket.close()方法被调用后的行为，包括延迟关闭时间。如果此选项设置为-1，表示socket.close()方法在调用后立即返回，但操作系统底层会将发送缓冲区的数据全部发送到对端；如果设置为0，表示socket.close()方法在调用后立即返回，但操作系统底层会将发送缓冲区的数据全部放弃，直接向对端发送RST包，对端将收到复位错误；如果设置为非0整数，表示socket.close()方法在调用后阻塞，直到延迟时间到来，发送缓冲区的数据发送完毕，若超时，则对端会收到复位错误。

   SO_LINGER，表示禁用此功能。

6. SO_BACKLOG

   此为TCP传输选项，表示服务器端接收连接的队列长度，如果队列已满，客户端连接将被拒绝。服务端在处理客户端新连接请求时（三次握手），是顺序处理的，所以同一时间只能处理一个客户端连接，多个客户端来的时候，服务端将不能处理的客户端连接请求放在队列中等待处理，队列的大小通过SO_BACKLOG指定。

   具体来说，服务端对完成第二次握手的连接放在一个队列（暂时称A队列），如果进一步完成第三次握手，再把的连接从A队列移动到新队列（暂时称B队列），接下来应用程序会通过accept方法取出握手成功的连接，而系统则会将该连接从B队列移除。 **A队列和B队列的长度之和是SO_BACKLOG指定的值**，当A和B队列的长度之和大于SO_BACKLOG值时，新连接将会被TCP内核拒绝，所以，如果SO_BACKLOG过小，可能会出现accept速度跟不上，A和.B两队列满了，导致新客户端无法连接。如果连接建立频繁，服务器处理新连接较慢，可以适当调大这个参数。

7. SO_BROADCAST

   此为TCP传输选项，表示设置为广播模式

## 13.4 Channel和Future

### 13.4.1 Channel的重要属性和方法

通道是Netty的核心概念之一，代表着网络连接，由它负责同对端进行网络通信，可以写入数据到对端，也可以从对端读取数据。

Netty通道的抽象类AbstractChannel的构造函数：

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;//父通道
    id = newId();
    unsafe = newUnsafe();//新建一个底层的NIO通道，完成实际IO操作
    pipeline = newChannelPipeline();//新建一条通道流水线
}
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```

AbstractChannel内部有一个pipeline属性，表示处理器的流水线，Netty在通道初始化时，将pipeline初始化成DefaultChannelPipeline的实例。因此，每个通道都会有一条ChannelPipeline处理器流水线。

内部还有一个parent父通道属性，保持通道的父通道。对于监听类的通道其父通道是null，对于传输类通道，其父通道是接收到该连接的监听通道。几乎所有的Netty通道实现类都继承了AbstractChannel抽象类，都有上面的parent和pipeline属性。

学习了通道的重要属性，我们接下来学习一下重要方法：

- ChannelFuture connect(SocketAddress address)

  连接远程服务器，方法的参数为远程服务器的地址，调用后会立即返回，其返回值为执行连接操作的异步任务ChannelFuture，此方法在客户端的传输通道使用

- ChannelFuture bind(SocketAddress address)

  绑定监听地址，开始监听新的客户端连接。此方法在服务器的新连接监听和接收通道使用。

- ChannelFuture close()

  关闭通道连接，返回连接关闭的ChannelFuture异步任务。如果需要在连接正式关闭后执行操作，则需要设置回调方法，或者调用sync方法阻塞当前线程

- Channel read()

  读取通道数据，并且启动入站处理。就是从内部的JavaNIO的Channel读取数据，然后启用通道内部的pipeline流水线，开启数据读取的入站处理。此方法返回通道本身，方便链式调用

- ChannelFuture write(Object o)

  启动出战的流水处理，把处理后的最终数据鞋到底层通道，此方法的返回值是出站处理的异步任务。

- ChannelFuture flush()

  此方法会将缓冲区的数据写出到对端。调用前面的write出站处理时，并不能将数据写出到对端，write操作大部分情况下是写入操作系统的缓冲区，操作系统根据缓冲区的情况决定什么时候把数据写到对端。而执行flush方法会立即将数据写到对端。

### 13.4.2 EmbeddedChannel嵌入式通道

在Netty的实际开发中，底层通信传输的基础工作Netty已经替大家完成。实际上，更多的工作是设计和开发ChannelHandler业务处理器。处理器开发完成后，需要投入单元测试。

一般单元测试的大致流程是：

- 需要将Handler业务处理器加入到通道的Pipeline流水线中；
- 接下来先后启动Netty服务器、客户端程序；
- 相互发送消息，测试业务处理器的效果

这个过程是十分繁琐的，所以Netty提供了一个专用通道EmbeddedChannel。EmbeddedChannel仅仅是模拟入站与出站的操作，底层不进行实际的传输，不需要启动Netty服务器和客户端。除了不进行传输之外，EmbeddedChannel的其他的事件机制和处理流程和真正的传输通道是一模一样的。因此，使用EmbeddedChannel，开发人员可以在单元测试用例中方便、快速地进行ChannelHandler业务处理器的单元测试。



为了模拟数据的发送和接收，EmbeddedChannel提供了一组专门的方法，如下表：

| 名称            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| writeInbound()  | 向通道写入入站数据，模拟真实通道收到数据的场景。也就是说，这些写入的数据会被流水线上的入站处理器所处理到。 |
| readInbound()   | 从EmbeddedChannel中读取入站数据，返回经过流水线的最后一个入站处理器完成后的入站数据。没有数据返回null |
| writeOutbound() | 向通道写入出站数据，模拟真实通道发送数据。也就是说，这些写入的数据会被流水线上的出站处理器处理。 |
| readOutbound()  | 从EmbeddedChannel中读取出站数据，返回经过流水线的最后一个入站处理器完成后的出站数据。没有数据返回null |
| finish()        | 结束EmbeddedChannel，会调用通道的close方法                   |

上面表格中最重要的两个方法如下，writeInbound和writeOutBound：

- writeInbound

  用于测试入站处理器。在测试入站处理器时（例如测试一个解码器），需要读取入站（Inbound）数据。可以调用writeInbound方法，向EmbeddedChannel写入一个入站数据（如二进制ByteBuf数据包）模拟底层的入站包，从而被入站处理器处理到，达到测试的目的。

- writeOutBound

  用于测试出站处理器。在测试出站处理器时（例如测试一个编码器），需要有出站的（Outbound）数据进入到流水线。可以调用writeOutbound方法，向模拟通道写入一个出站数据（如二进制ByteBuf数据包），该包将进入处理器流水线，被待测试的出站处理器所处理

总之，EmbeddedChannel类既拥有通道的通用接口和方法，又增加了一些单元测试的辅助方法。具体用法后续示例中进行体现。

## 13.5 Handler

暂略，后续进行整理，此部分可以先看我的博文[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)中的第2.5节。

## 13.6 Pipeline

暂略，后续进行整理，此部分可以先看我的博文[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)中的第2.5节。

## 13.7 ByteBuf

暂略，后续进行整理，此部分可以先看我的博文[NIO与Netty](https://www.cnblogs.com/yhr520/p/15384520.html)中的第2.6节。

## 13.8 Netty编码解码

Netty从底层Java通道读到ByteBuf二进制数据，传入Netty 通道的流水线，随后开始入站处理。

在入站处理过程中，需要将ByteBuf二进制类型，解码成Java POJO对象。这个解码过程，可以通过Netty的Decoder（解码器）去完成。在出站处理过程中，业务处理后的结果（出站数据），需要从某个Java POJO对象，编码为最终的ByteBuf二进制数据，然后通过底层Java通道发送到对端。在编码过程中，需要用到Netty的Encoder（编码器）去完成数据的编码工作.

### 13.8.1 Decoder原理

Decoder是一个Inbound入站处理器，解码器负责处理入站数据。它主要将上一个入站处理器的输入数据，进行数据解码或格式转换，然后发送到下一个入站处理器。一个标准的解码器是将输入类型为ByteBuf缓冲区的数据进行节码，输出一个Java POJO对象。netty内置了这个解码器（ByteToMessageDecoder）。Netty中的解码器都是入站处理器类型，几乎都直接或间接地实现了入站处理的超级接口ChannelInboundHandler。

#### **ByteToMessageDecoder处理流程**

ByteToMessageDecoder是一个非常重要的解码器基类，它是一个抽象类，实现了解码器的基础逻辑和流程。ByteToMessageDecoder继承自ChannelInboundHandlerAdapter适配器，用于完成从ByteBuf到JavaPojo的解码。它的流程大概如下：

- 首先，将上一站传过来的输入到ByteBuf中的数据进行解码，解码出一个`List<Object>`对象列表
- 然后，迭代`List<Object>`列表，逐个将Java POJO对象传入下一站InBound处理器

流程如下：

![image-20220527210541866](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220527210541866.png)

ByteToMessageDecoder是个抽象类，不能实例化对象，所以解码需要具体的实现类。ByteToMessageDecoder的解码方法是decode，这是一个抽象方法，也就是说具体的解码过程（如ByteBuf中的字节数据变成什么样的Object）需要子类完成。所以ByteToMessageDecoder仅仅提供一个整体框架，他会调用子类方法，完成具体的二进制字节解码，然后获取子类解码的Object，放入自己内部的结果列表`List<Object>`中。最终父类会负责将`List<Object>`中的元素一个一个传递给下一站。所以也可以说ByteToMessageDecoder使用了模板模式。

ByteToMessageDecoder子类要做的就是从入站ByteBuf解码出来的所有Object实例加入到父类的`List<Object>`中。如果要实现自己的解码器，首先要继承ByteToMessageDecoder然后实现其基类的decode方法，将ByteBuf到目标pojo的解码逻辑写入此方法，然后解码完成后，将解码后的Object加入到父类的`List<Object>`中。剩下的工作都由父类去完成。在流水线的处理过程中，父类知执行完子类decode后，会将`List<Object>`一个一个的传入下一个InBound入站处理器。

#### **自定义Byte2IntegerDecoder整数解码器**

这个示例是简单的整数解码器，其功能是将ByteBuf缓冲区中的字节，解码成Integer整数类型。根据上面的流程，我们现在进行实现：

```java
public class Byte2IntegerDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        while (in.readableBytes()>=4){//再读数据前，需要对数据进行验证是否可读，这里是4的原因是int为4字节
            int i = in.readInt();
            System.out.println("解码出一个整数"+i);
            out.add(i);
        }

    }
}
```

可以看到我们继承ByteToMessageDecoder后，实现了自己的解码逻辑：从buf中读取整形放入父类的集合中。有了自己的解码器之后，我们现在实现一个handler功能是读取上一站的入站数据，把它转换成整数，并且输出到Console控制台上，代码如下：

```java
public class IntegerProcessHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Integer integer = (Integer) msg;
        System.out.println("整数"+integer);
    }
}
```

下面我们进行测试，这里使用我们之前介绍过的EmbeddedChannel，向里面放入我们自己的解码器和handler，然后向buffer中写入数据，在调用EmbeddedChannel将数据入站。这里调用writeInbound方法，模拟入站数据的写入，向嵌入式通道EmbeddedChannel写入100次ByteBuf入站缓冲；每一次写入仅仅包含一个整数。模拟入站数据，会被流水线上的两个入站处理器所接收和处理。接着，这些入站的二进制字节被解码成一个一个的整数，然后逐个地输出到控制台上。

```java
@Test
public void Byte2IntegerDecoderTest(){
    ChannelInitializer initializer = new ChannelInitializer() {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast(new Byte2IntegerDecoder());
            ch.pipeline().addLast(new IntegerProcessHandler());
        }
    };
    EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 100; i++) {
        ByteBuf buffer = Unpooled.buffer();
        buffer.writeInt(i);
        embeddedChannel.writeInbound(buffer);
    }
}
```

运行结果：

~~~
解码出一个整数0
整数0
解码出一个整数1
整数1
解码出一个整数2
...略
~~~

#### **ReplayingDecoder解码器**

上面的自定义解码器有一个问题，那就是在读取时需要对ByteBuf的长度进行检查，如果有足够的字节，才进行整数的读取。这个工作其实可以交给Netty去做，使用ReplayingDecoder基类编写解码器，就不用进行长度检测了。

ReplayingDecoder是ByteToMessageDecoder的子类，作用是：

- 在读取ByteBuf缓冲区之前，需要检查是否有足够的字节
- 有，则正常读取；没有则会停止解码

我们现在将创建Byte2IntegerReplayDecoder类继承ReplayingDecoder：

```java
public class Byte2IntegerReplayDecoder extends ReplayingDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        int i = in.readInt();
        System.out.println("解码出一个整数" + i);
        out.add(i);
    }
}
```

继承ReplayingDecoder实现一个解码器，就不用编写长度判断的代码。这是因为它的内部定义了一个新的二进制缓冲区类ReplayingDecoderBuffer，对ByteBuf缓冲区进行了装饰。该装饰器的特点是：在缓冲区真正读数据之前，首先进行长度的判断：如果长度合格，则读取数据；否则，抛出ReplayError。ReplayingDecoder捕获到ReplayError后，会留着数据，等待下一次IO事件到来时再读取。

简单来讲，ReplayingDecoder对输入的ByteBuf进行了偷梁换柱，在将外部传入的ByteBuf缓冲区传给子类之前，换成了自己装饰过的ReplayingDecoderBuffer缓冲区。也就是说，在Byte2IntegerReplayDecoder中的decode方法所得到的实参in的值，它的直接类型并不是原始的ByteBuf类型，而是ReplayingDecoderBuffer类型。ReplayingDecoderBuffer类型继承了ByteBuf类型，包装了ByteBuf类型的大部分读取方法。ReplayingDecoderBuffer对ByteBuf类型的读取方法做什么样的功能增强呢？主要是进行二进制数据长度的判断，如果长度不足，则抛出异常。这个异常会反过来被ReplayingDecoder基类所捕获，将解码工作停掉。

```java
//ReplayingDecoderBuffer的readInt方法
@Override
public int readInt() {
    checkReadableBytes(4);
    return buffer.readInt();
}
```

实质上，ReplayingDecoder的作用远远不止于进行长度判断，它更重要的作用是用于分包传输的应用场景。

在底层通讯协议中，数据是分包进行传输的，也就是说一份数据可能会有很多个包，比如发送端发送4个字符串，接收端可能会接收到3个ByteBuf数据缓冲，如下图。在Java阻塞式IO中，由于读取不完整的信息就会一直阻塞，但是在NIO中如何保证接受的数据完整就成了一个大问题。在Netty中，理论上可以通过ReplayingDecoder解决。

![image-20220528132700615](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220528132700615.png)

这个图是以字符串为例，为了简单，我们下面用整形举个例子，这个例子的场景是整数序列解码，并且将它们两两一组进行相加，重点是，解码过程中需要保持发送时的次序。

在实现这个例子前，我们先来了解ReplayingDecoder一个很重要的属性——state成员属性，因为要完成这个例子我们需要这个属性。该成员属性的作用：保存当前解码器在解码过程中的所处阶段。我们下面看一下ReplayingDecoder重要属性和构造方法：

```java
private final ReplayingDecoderByteBuf replayable = new ReplayingDecoderByteBuf();//缓冲区装饰类
private S state;//解码过程中所处的阶段，类型是泛型，默认为Object
private int checkpoint = -1;//读指针检查点，默认为-1

/**
 * Creates a new instance with no initial state (i.e: {@code null}).
 */
protected ReplayingDecoder() {
    this(null);
}

/**
 * Creates a new instance with the specified initial state.
 */
protected ReplayingDecoder(S initialState) {
    state = initialState;
}
```

在上面的Byte2IntegerReplayDecoder中我们继承时没有给泛型，但是我们要完成整数序列解码要用到泛型，或者说要用到state属性，因为整数序列的解码工作不可能通过一次完成，要完成两个整数的提取并相加就需要解码两次，每一次解码只能解码出一个整数，只有在第二个整数提取之后，然后才能求和，整个解码的工作才算完成，这里边存在了两个阶段，具体的阶段需要使用state来记录。代码实现如下：

```java
public class IntegerAddDecoder extends ReplayingDecoder<IntegerAddDecoder.PHASE> {

    enum PHASE{
        PHASE_1,PHASE_2;
    }
    public IntegerAddDecoder(){
        //初始化父类的state属性为PHASE_1，表示现在是解码的第一个阶段
        super(PHASE.PHASE_1);
    }
    private int first;
    private int second;
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        switch (state()){
            case PHASE_1:
                //读取数据，第一步仅提取第一个整数
                first = in.readInt();
                //设置状态为第二步
                checkpoint(PHASE.PHASE_2);
                break;
            case PHASE_2:
                //读取第二个数据
                second = in.readInt();
                //第二步还需要计算两者的结果
                Integer sum = first + second;
                //和作为解码结果
                out.add(sum);
                //设置状态为第一步
                checkpoint(PHASE.PHASE_1);
                break;
            default:
                break;
        }
    }
}
```

在上面这个例子中我们使用了两个父类的函数，分别是state()和checkpoint()，其中state()用于获取当前阶段的值而checkpoint()则用于更新当前状态。当然除了更新当前状态checkpoint()还可以用于设置“读指针检查点”。

读指针检查点就是ReplayingDecoder的重要成员，用于暂存内部的ReplayingDecoderBuffer装饰器的readerIndex读指针，有点类似mark标记。当读数据时，一旦缓冲区可读的二进制数据不够，缓冲区装饰器ReplayingDecoderBuffer在抛出ReplayError异常之前，会把readerIndex读指针的值，还原到之前通过checkpoint（…）方法设置的“读指针检查点”。于是乎，在ReplayingDecoder下一次重新读取时，还会从“读指针检查点”的位置开始读取。

回到上面的例子，该方法就是如此，每一次读取完成后，会切换阶段并且保持当前的读指针检查点，以便于可读数据不足后帮助读指针恢复。这个例子的测试方法与自定义Byte2IntegerDecoder整数解码器类似，这里就不再赘述。

#### **字符串的分包解码器**

通过整数分包的例子，我们对ReplayingDecoder的分阶段解码已经有了一个很好的了解。现在我们就来了解字符串的分包传输。但是与整数不同的是，字符串的长度不固定，那么如何获取这个字符串的长度信息呢？在Netty中，可以采用Header-Content内容传输协议，该协议内容主要如下：

1. 在协议的Head部分放置字符串的字节长度，Head部分用一个int描述即可。
2. 在协议的Content部分，放置的是字符串的字节数组。

在实际的传输过程中，一个Header—Content内容包，在发送端会被编码成一个ByteBuf包，到达对端后，可能被分为很多个ByteBuf接受包，对于这些参差不齐的ByteBuf就可以用ReplayingDecoder获得Header-Content内容，代码如下：

首先创建字符串解码器，具体的过程与数字解码器相同，只是多读取了字符串的长度而已。

```java
public class StringReplayingDecoder extends ReplayingDecoder<StringReplayingDecoder.PHASE> {

    enum PHASE{
        PHASE_1,PHASE_2
    }
    private int length;
    private byte[] inBytes;
    public StringReplayingDecoder(){
        super(PHASE.PHASE_1);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        switch (state()){
            case PHASE_1:
                //读取字符串长度
                length = in.readInt();
                //新建byte数组
                inBytes = new byte[length];
                checkpoint(PHASE.PHASE_2);
                break;
            case PHASE_2:
                //读取字符串内容
                in.readBytes(inBytes,0,length);
                //结果保存
                out.add(new String(inBytes, StandardCharsets.UTF_8));
                checkpoint(PHASE.PHASE_1);
                break;
            default:
                break;
        }
    }
}
```

然后我们可以编写一个打印字符串的handler类（用于模拟在流水线的数据）：

```java
public class StringProcessHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("打印字符串: "+ msg);
    }
}
```

最后编写一个测试类：

```java
Test
public void StringReplayingDecoderTest(){
    ChannelInitializer initializer = new ChannelInitializer() {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast(new StringReplayingDecoder());
            ch.pipeline().addLast(new StringProcessHandler());
        }
    };
    EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
    Random random = new Random();
    byte[] bytes = "发送的字符串".getBytes(StandardCharsets.UTF_8);
    for (int i = 0; i < 100; i++) {
        int nextInt = random.nextInt(3)+1;
        ByteBuf buffer = Unpooled.buffer();
        buffer.writeInt(bytes.length*nextInt);
        for (int j = 0; j < nextInt; j++) {
            buffer.writeBytes(bytes);
        }
        embeddedChannel.writeInbound(buffer);
    }
}
```

然而，在实际的开发中，不太建议继承这个类，原因如下：

1. 不是所有的ByteBuf操作都被ReplayingDecoderBuffer装饰类所支持，可能有些ByteBuf方法在ReplayingDecoder的decode实现方法中被使用时就会抛出ReplayError异常。
2. 在数据解码逻辑复杂的应用场景，ReplayingDecoder在解码速度上相对较差。因为在ByteBuf中长度不够时，ReplayingDecoder会捕获一个ReplayError异常，这时会把ByteBuf中的读指针还原到之前的读指针检查点（checkpoint），然后结束这次解析操作，等待下一次IO读事件。在网络条件比较糟糕时，一个数据包的解析逻辑会被反复执行多次，此时解析过程是一个消耗CPU的操作，所以解码速度上相对较差。所以，ReplayingDecoder更多的是应用于数据解析逻辑简单的场景

所以我们现在用ByteToMessageDecoder来改写StringReplayingDecoder，代码如下：

```java
public class StringByteDecoder extends ByteToMessageDecoder {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        //可读字节小于4，消息头没读满
        if (in.readableBytes()<4){
            return;
        }
        //在真正读取缓冲区消息前，调用markReaderIndex设置mark标记
        in.markReaderIndex();
        int length = in.readInt();
        //如果剩余消息长度不够，需要reset指针
        if (in.readableBytes()<length){
            //读指针reset到消息头的readIndex位置处
            in.resetReaderIndex();
            return;
        }
        byte [] bytes = new byte[length];
        in.readBytes(bytes, 0, length);
        out.add(new String(bytes,StandardCharsets.UTF_8));
    }
}
```

#### **MessageToMessage解码器**

上面的解码器都是将二进制ByteBuf转换成Pojo对象。实际上还存在着将Pojo对象转换成另一种Pojo对象的解码器，在这种应用场景下的Decoder解码器，需要继承`MessageToMessageDecoder<T>`类。其中泛型T就是指定入站消息的JavaPOJO类型。MessageToMessageDecode使用了模板模式，也有一个decode抽象方法。下面将给出一个将Integer转换成String的解码器实现：

```java
public class Int2StringDecoder extends MessageToMessageDecoder<Integer> {
    @Override
    protected void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out) throws Exception {
        out.add(String.valueOf(msg));
    }
}
```

测试类：

```java
@Test
public void Byte2IntegerDecoderTest(){
    ChannelInitializer initializer = new ChannelInitializer() {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast(new IntegerAddDecoder());
            ch.pipeline().addLast(new Int2StringDecoder());
            ch.pipeline().addLast(new StringProcessHandler());
        }
    };
    EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 100; i++) {
        ByteBuf buffer = Unpooled.buffer();
        buffer.writeInt(i);
        embeddedChannel.writeInbound(buffer);
    }
}
```

### 13.8.2 常用的内置Decoder

Netty提供了很多开箱即用的解码器，一般情况下能满足很多场景，下面学习几个常用的解码器：

1. 固定长度数据包解码器—FixedLengthFrameDecoder

   适用场景：每个接收到的数据包的长度，都是固定的，例如 100个字节。在这种场景下，只需要把这个解码器加到流水线中，它会把入站ByteBuf数据包拆分成一个个长度为100的数据包，然后发往下一个channelHandler入站处理器。（这里的一个数据包指的是一个ByteBuf）

2. 行分割数据包解码器—LineBasedFrameDecoder

   每个ByteBuf数据包，使用换行符（或者回车换行符）作为数据包的边界分割符。在这种场景下，只需要把这个LineBasedFrameDecoder解码器加到流水线中，Netty会使用换行分隔符，把ByteBuf数据包分割成一个一个完整的应用层ByteBuf数据包，再发送到下一站。下面给出示例：

   ```java
   public class NettyDecoder {
       static String split = "\r\n";//发送\r\n 或者\n 都会进行解析
       static String content= "hello world";
   
       @Test
       public void testLineBasedFrameDecoder(){
           ChannelInitializer initializer = new ChannelInitializer() {
               @Override
               protected void initChannel(Channel ch) throws Exception {
                   //LineBasedFrameDecoder支持配置一个最大长度值，如果读取到最大值还没发现换行符就会抛出异常。
                   ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                   ch.pipeline().addLast(new StringDecoder());
                   ch.pipeline().addLast(new StringProcessHandler());
               }
           };
           EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
           Random random = new Random();
           for (int i = 0; i < 100; i++) {
               int nextInt = random.nextInt(3)+1;
               ByteBuf byteBuf = Unpooled.buffer();
               for (int j = 0; j < nextInt; j++) {
                   byteBuf.writeBytes(content.getBytes(StandardCharsets.UTF_8));
               }
               byteBuf.writeBytes(split.getBytes(StandardCharsets.UTF_8));
               embeddedChannel.writeInbound(byteBuf);
           }
       }
   
   }
   ```

3. 自定义分隔符数据包解码器—DelimiterBasedFrameDecoder

   DelimiterBasedFrameDecoder是LineBasedFrameDecoder按照行分割的通用版本。不同之处在于，这个解码器更加灵活，可以自定义分隔符，而不是局限于换行符。如果使用这个解码器，那么所接收到的数据包，末尾必须带上对应的分隔符。DelimiterBasedFrameDecoder解码器不仅可以使用换行符，还可以将其他的特殊字符作为数据包的分隔符，例如制表符“\t”。其构造方法如下：

   ```java
   public DelimiterBasedFrameDecoder(
       int maxFrameLength, //解码的数据包的最大长度
       boolean stripDelimiter, //解码后的数据包是否去掉分隔符，一般选择是
       ByteBuf delimiter) {//分隔符
       this(maxFrameLength, stripDelimiter, true, delimiter);
   }
   ```

   这个的例子略，整体与上面的示例没有不同，可以根据上面的例子testLineBasedFrameDecoder自行修改。

4. 自定义长度数据包解码器—LengthFieldBasedFrameDecoder

   这是一种基于灵活长度的解码器。在ByteBuf数据包中，加了一个长度域字段，保存了原始数据包的长度。解码的时候，会按照这个长度进行原始数据包的提取。传输内容中的Length Field长度字段的值，是指存放在数据包中要传输内容的字节数。普通的基于Header-Content协议的内容传输，尽量用内置的LengthFieldBasedFrameDecoder来解码。

   下面给出示例：

   ```java
   public class NettyDecoder {
       static String content = "hello world";
       @Test
       public void testLengthFieldBasedFrameDecoder() {
           //参数：1. maxFrameLength 发送数据包的最大长度，即一个数据包最大可发送1024个字节
           //2.lengthFieldOffset 长度字段偏移量，指的是长度字段位于整个数据包内部字节数据中的下标索引值
           //3.lengthFieldLength 长度字段所占的字节数。例如长度字段是一个int，则该字节位4
           //4.lengthAdjustment 长度矫正值。加入协议中包含了长度字段，协议版本号、魔数等，那么解码时就需要进行长度矫正。
           //长度矫正的计算公式：内容字段偏移量-长度字段偏移量-长度的字节数。在本例中为 4 - 0 - 4 = 0
           //5.initialBytesToStrip丢弃的起始字节数。在有效的数据字段前面，如果还有一些其他字段的字节，最为最终解析的结果可以丢弃。
           //比如，本例子中，前面有四个字节的长度字段，最终的结果不需要这个字段，所以可以丢弃。
           LengthFieldBasedFrameDecoder decoder = new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4);
           ChannelInitializer initializer = new ChannelInitializer() {
               @Override
               protected void initChannel(Channel ch) throws Exception {
                   ch.pipeline().addLast(decoder);
                   ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));
                   ch.pipeline().addLast(new StringProcessHandler());
               }
           };
           EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
           Random random = new Random();
           for (int i = 0; i < 100; i++) {
               ByteBuf byteBuf = Unpooled.buffer();
               String str = "第" + i + "次发送" + content;
               byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
               byteBuf.writeInt(bytes.length);
               byteBuf.writeBytes(bytes);
               embeddedChannel.writeInbound(byteBuf);
           }
   
           try {
               Thread.sleep(Integer.MAX_VALUE);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   
   
   }
   ```

   本例中的Head-Content的大小如下图所示：

   ![image-20220529144137217](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220529144137217.png)

   Head-Content协议是最为简单的内容传输协议。而在实际使用过程中，则没有那么简单，除了长度和内容，在数据包中还可能包含了其他字段，例如，包含了协议版本号：

   ![image-20220529144820111](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220529144820111.png)

   那么这个时候，构造函数参数该如何计算呢？参数计算如下：

   1. maxFrameLength 可以为1024，自定
   2. lengthFieldOffset  0，表示字段长度在整个数据包在第一个位置
   3. lengthFieldLength 4，表示长度字段占四个字节
   4. lengthAdjustment 2，计算方法：内容字段偏移量-长度字段偏移量-长度的字节数 = 6 - 0 - 4 = 2
   5. initialBytesToStrip 6，最终的content字段需要抛弃前面的6个字节的长度和版本

   我们再举个复杂的例子：

   ![image-20220529145135992](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220529145135992.png)

   这时参数计算如下：

   1. maxFrameLength 可以为1024，自定
   2. lengthFieldOffset  2，表示字段长度在整个数据包在第2个字节的位置
   3. lengthFieldLength 4，表示长度字段占四个字节
   4. lengthAdjustment 4，计算方法：内容字段偏移量-长度字段偏移量-长度的字节数 = 10 - 2 - 4 = 4
   5. initialBytesToStrip 10，最终的content字段需要抛弃前面的10个字节的长度和版本和魔数

### 13.8.3 Encoder原理

在Netty的业务处理完成后，业务处理的结果往往是某个Java POJO对象，需要编码成最终的ByteBuf二进制类型，通过流水线写入到底层的Java通道，这就需要用到Encoder编码器。

编码器是一个Outbound出站处理器，负责处理“出站”数据；其次，编码器将上一站Outbound出站处理器传过来的输入（Input）数据进行编码或者格式转换，然后传递到下一站ChannelOutboundHandler出站处理器。

编码器与解码器相呼应，Netty中的编码器负责将“出站”的某种Java POJO对象编码成二进制ByteBuf，或者转换成另一种Java POJO对象。编码器是ChannelOutboundHandler的具体实现类。一个编码器将出站对象编码之后，编码后数据将被传递到下一个ChannelOutboundHandler出站处理器，进行后面出站处理。由于最后只有ByteBuf才能写入到通道中去，因此可以肯定通道流水线上装配的第一个编码器一定是把数据编码成了ByteBuf类型（因为出栈顺序是从后向前的，可以见我在13.2.4中的图片）

#### **MessageToByteEncoder**

MessageToByteEncoder是一个非常重要的编码器基类，它位于Netty的`io.netty.handler.codec`包中。MessageToByteEncoder的功能是将一个Java POJO对象编码成一个ByteBuf数据包。它是一个抽象类，仅仅实现了编码的基础流程，在编码过程中，通过调用encode抽象方法来完成。但是，它的encode编码方法是一个抽象方法，没有具体的encode编码逻辑实现，实现encode抽象方法的工作需要子类去完成。下面举个例子：

```java
public class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
        out.writeInt(msg);
        System.out.println("integer:"+msg);
    }
}
```

上面的例子编码完成后将msg写入out，将输出的ByteBuf数据包发送到下一站。同理，下面给出测试类：

```java
@Test
public void testIntegerToByteEncoder(){
    ChannelInitializer initializer = new ChannelInitializer() {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            ch.pipeline().addLast(new Integer2ByteEncoder());
        }
    };
    EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 100; i++) {
        //x
        embeddedChannel.write(i);
    }
    embeddedChannel.flush();
    //读取模拟出战数据包
    ByteBuf byteBuf = embeddedChannel.readOutbound();
    while (byteBuf!=null){
        System.out.println(byteBuf.readInt());
        byteBuf = embeddedChannel.readOutbound();
    }
}
```

#### **MessageToMessageEncoder**

如果需要将POJO对象转换成POJO对象，那么需要继承另外一个Netty的重要编码器——MessageToMessageEncoder编码器，并实现它的encode抽象方法。在子类的encode方法实现中，完成原POJO类型到目标POJO类型的转换逻辑。在encode实现方法中，编码完成后，将解码后的目标对象加入到encode方法中的实参List输出容器即可。下面给出代码示例：

```java
public class String2IntegerEncoder extends MessageToMessageEncoder<String> {
    @Override
    protected void encode(ChannelHandlerContext ctx, String msg, List<Object> out) throws Exception {
        final char[] chars = msg.toCharArray();
        for (char aChar : chars) {
            //ascii 48 是0的编码；57 是9的编码
            if (aChar>=48 && aChar<=57){
                final String s = String.valueOf(aChar);
                out.add(Integer.valueOf(s));
            }
        }
    }
}
```

下面给出测试用例：

```java
@Test
public void testMessageToMessageEncoder(){
    ChannelInitializer initializer = new ChannelInitializer() {
        @Override
        protected void initChannel(Channel ch) throws Exception {
            //流水线出站从后至前，出站时需要将Integer转换成ByteBuf
            ch.pipeline().addLast(new Integer2ByteEncoder());
            ch.pipeline().addLast(new String2IntegerEncoder());
        }
    };
    EmbeddedChannel embeddedChannel = new EmbeddedChannel(initializer);
    for (int i = 0; i < 100; i++) {
        System.out.println("i am number"+i);
        embeddedChannel.write("i am number"+i);
    }
    embeddedChannel.flush();
    ByteBuf buf = embeddedChannel.readOutbound();
    while (buf != null){
        System.out.println("buf= "+buf.readInt());
        buf = embeddedChannel.readOutbound();
    }
}
```

### 13.8.4 解码器与编码器的结合

在实际的开发中，由于数据的入站和出站关系紧密，因此编码器和解码器的关系很紧密。编码和解码更是一种紧密的、相互配套的关系。在流水线处理时，数据的流动往往一进一出，进来时解码，出去时编码。所以，在同一个流水线上，加了某种编码逻辑，常常需要加上一个相对应的解码逻辑。编码器和解码器，分开实现在两个不同的类中，导致的一个结果是，相互配套的编码器和解码器在加入到通道的流水线时，常常需要分两次添加。现在问题是：具有相互配套逻辑的编码器和解码器能否放在同一个类中呢？这就要用到Netty的新类型——Codec（编解码器）类型。

#### **ByteToMessageCodec**

完成POJO到ByteBuf数据包的编解码器基类，叫做`ByteToMessageCodec<I>`，它是一个抽象类。从功能上说，继承它就等同于继承了ByteToMessageDecoder解码器和MessageToByteEncoder编码器这两个基类。编解码器ByteToMessageCodec同时包含了编码encode和解码decode两个抽象方法，这两个方法都需要我们自己实现.

```java
public class Byte2IntegerCodec extends ByteToMessageCodec<Integer> {
    @Override
    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
        out.writeInt(msg);
        System.out.println("write msg,msg="+msg);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        if (in.readableBytes()>=4){
            final int i = in.readInt();
            System.out.println("int:"+i);
            out.add(i);
        }
    }
}
```

同理，还有MessageToMessageCodec，这个实例略。

#### **CombinedChannelDuplexHandler**

上面的Codec是通过继承实现的，继承是需要将编码和解码强制放到一个类中，除此之外还可以通过组合的方式实现。Netty中提供了一个新的组合器CombinedChannelDuplexHandler基类。下面给出示例：

```java
public class IntegerHandler extends CombinedChannelDuplexHandler<Byte2IntegerDecoder,Integer2ByteEncoder> {
    public IntegerHandler() {
        super(new Byte2IntegerDecoder(),new Integer2ByteEncoder());
    }
}
```

仅仅只需要继承CombinedChannelDuplexHandler即可，不需要像ByteToMessageCodec那样，把编码逻辑和解码逻辑都挤在同一个类中了，还是复用原来的分开的编码器和解码器实现代码。总之，使用CombinedChannelDuplexHandler可以保证有了相反逻辑关系的encoder编码器和decoder解码器，既可以结合使用，又可以分开使用，十分方便。

## 13.9 序列化和反序列化

“序列化”和“反序列化”一定会涉及POJO的编码和格式化（Encoding & Format），目前我们可选择的编码方式有：

- 使用JSON。将Java POJO对象转换成JSON结构化字符串。基于HTTP协议，在Web应用、移动开发方面等，这是常用的编码方式，因为JSON的可读性较强。但是它的性能稍差。
- 基于XML。和JSON一样，数据在序列化成字节流之前都转换成字符串。可读性强，性能差，异构系统、Open API类型的应用中常用。
- 使用Java内置的编码和序列化机制，可移植性强，性能稍差，无法跨平台（语言）。
- 开源的二进制的序列化/反序列化框架，例如Apache Avro，Apache Thrift、Protobuf等。前面的两个框架和Protobuf相比，性能非常接近，而且设计原理如出一辙；其中Avro在大数据存储（RPC数据交换、本地存储）时比较常用；Thrift的亮点在于内置了RPC机制，所以在开发一些RPC交互式应用时，客户端和服务器端的开发与部署都非常简单。

其中Protobuf是一个高性能、易扩展的序列化框架，性能比较高，其性能的有关的数据可以参看官方文档。Protobuf本身非常简单，易于开发，而且结合Netty框架，可以非常便捷地实现一个通信应用程序。反过来，Netty也提供了相应的编解码器，为Protobuf解决了有关Socket通信中“半包、粘包”等问题。

### 13.9.1 粘包和半包

关于粘包和拆包详细内容可以见我的博客[NIO和Netty](https://www.cnblogs.com/yhr520/p/15384520.html)的**第三部分Netty的粘包和半包**，这里就不再重复整理一遍笔记了。

这里补充一下粘包和半包的底层过程：

底层网络是以二进制字节报文的形式来传输数据的。

- 读数据的过程大致为：当IO可读时，Netty会从底层网络将二进制数据读到ByteBuf缓冲区中，再交给Netty程序转成Java POJO对象。
- 写数据的过程大致为：编码器将一个Java类型的数据转换成底层能够传输的二进制ByteBuf缓冲数据。

在发送端Netty的应用层进程缓冲区，程序以ByteBuf为单位来发送数据，但是到了底层操作系统内核缓冲区，底层会按照协议的规范对数据包进行二次封装，封装成传输层TCP层的协议报文，再进行发送。在接收端收到传输层的二进制包后，首先复制到内核缓冲区，Netty读取ByteBuf时才复制到应用的用户缓冲区。

在接收端，当Netty程序将数据从内核缓冲区复制到用户缓冲区的ByteBuf时，问题就来了：

- 首先，每次读取底层缓冲的数据容量是有限制的，当TCP内核缓冲区的数据包比较大时，可能会将一个底层包分成多次ByteBuf进行复制，进而造成用户缓冲区读到的是半包。
- 当TCP内核缓冲区的数据包比较小时，一次复制的却不止一个内核缓冲区包，进而造成用户缓冲区读到的是粘包

基本思路是，在接收端，Netty程序需要根据自定义协议，将读取到的进程缓冲区ByteBuf，在应用层进行二次组装，重新组装我们应用层的数据包。接收端的这个过程通常也称为分包，或者叫做拆包。

在Netty中分包的方法，主要有两种方法：

1. 可以自定义解码器分包器：基于ByteToMessageDecoder或者ReplayingDecoder，定义自己的用户缓冲区分包器。
2. 使用Netty内置的解码器。如可以使用Netty内置的LengthFieldBasedFrameDecoder自定义长度数据包解码器，对用户缓冲区ByteBuf进行正确的分包。

### 13.9.2 JSON协议通信

现在的Web应用基本上都会使用到JSON，所以关于JSON的介绍这里就不再赘述了。Java处理JSON数据有三个比较流行的开源类库有：阿里的FastJson、谷歌的Gson和开源社区的Jackson。关于这三个类库和API的使用这里也不过多介绍。我们这里仅学习Netty的关于JSON的相关知识。

#### **JSON的编解码器**

JSON本质上其实也就是字符串，所以传时JSON与传输普通文本其实是一样的，在Netty的Head-Content协议中，解码过程如下：

先使用Netty内置的LengthFieldBasedFrameDecoder解码Head-Content二进制数据包，解码出Content字段的二进制内容。然后使用StringDecoder字符串解码器（Netty内置的解码器）将二进制内容解码成JSON字符串。最后，使用自定义业务解码器JsonMsgDecoder解码器将JSON字符串解码成自定义的POJO业务对象。

同理编码过程如下：

先使用Netty内置StringEncoder编码器将JSON字符串编码成二进制字节数组。然后使用Netty内置LengthFieldPrepender编码器将二进制字节数组编码成Head-Content二进制数据包。

> Netty内置LengthFieldPrepender编码器的作用：在数据包的前面加上内容的二进制字节数组的长度。这个编码器和LengthFieldBasedFrameDecoder解码器是一对，常常配套使用。
>
> LengthFieldPrepender的构造函数如下：
>
> ```java
> public LengthFieldPrepender(int lengthFieldLength) {
>     this(lengthFieldLength, false);
> }
> //第一个参数lengthFieldLength表示Head长度字段所占用的字节数，
> //第二个参数lengthIncludesLengthFieldLength表示Head字段的总长度值是否包含长度字段自身的字节数
> public LengthFieldPrepender(int lengthFieldLength, boolean lengthIncludesLengthFieldLength) {
>     this(lengthFieldLength, 0, lengthIncludesLengthFieldLength);
> }
> ```

#### **JSON的传输案例**

下面给出案例，服务端接收客户端的数据包，解码成JSON，然后转换成POJO，客户端将POJO转换成JSON字符串，编码后发送到服务器端。当然我们这里的服务端只有InBound处理器，也就是说不会收到消息后不会有消息到对端。



首先编写传输的消息实体类：

```java
@Data
public class JsonMsg {
    //id Field(域)
    private int id;
    //content Field(域)
    private String content = "Json测试消息";

    public static JsonMsg parseFromJson(String json) {
        return JSONObject.parseObject(json, JsonMsg.class);
    }

    public String convertToJson() {
        return JSONObject.toJSONString(this);
    }

    public JsonMsg(int id) {
        this.id = id;
    }

    public JsonMsg() {
        this.id = RandomUtil.randomInt(100);
    }

}
```

然后编写自定义服务端业务处理Handler：

```java
public class JsonMsgHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String json = (String) msg;
        JsonMsg jsonMsg = JsonMsg.parseFromJson(json);
        System.out.println("收到一个数据包："+jsonMsg);
    }
}
```

然后编写服务端代码：

```java
public class JsonServer {
    public void start() {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        try {
            serverBootstrap
                    .group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(8999)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
                            ch.pipeline().addLast(new StringDecoder(StandardCharsets.UTF_8));
                            ch.pipeline().addLast(new JsonMsgHandler());
                        }
                    });
            ChannelFuture channelFuture = serverBootstrap.bind().sync();
            final ChannelFuture closeFuture = channelFuture.channel().closeFuture();
            closeFuture.sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        new JsonServer().start();
    }
}
```

然后编写客户端代码：java



```java
public class JsonClient {
    static String content = "发送测试数据";
    public void start(){
        Bootstrap bootstrap = new Bootstrap();
        EventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            bootstrap.group(workGroup)
                    .channel(NioSocketChannel.class)
                    .remoteAddress("localhost",8999)
                    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(new LengthFieldPrepender(4));
                            ch.pipeline().addLast(new StringEncoder(StandardCharsets.UTF_8));
                        }
                    });
            ChannelFuture future = bootstrap.connect();
            future.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isSuccess()){
                        System.out.println("连接成功");
                    }else {
                        System.out.println("连接失败");
                    }
                }
            });
            future.sync();
            Channel channel = future.channel();
            for (int i = 0; i < 100; i++) {
                JsonMsg msg = new JsonMsg();
                msg.setId(i);
                msg.setContent("发送测试数据");
                channel.writeAndFlush(msg.convertToJson());
            }
            channel.flush();

            ChannelFuture closeFuture = channel.closeFuture();
            closeFuture.sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            workGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        new JsonClient().start();
    }
}
```

### 13.9.3 Protobuf协议通信使用

#### **介绍**

Protobuf全称是Google Protocol Buffer，Google提出的一种数据交换的格式，是一套类似JSON或者XML的数据传输格式和规范，用于不同应用或进程之间进行通信。Protobuf具有以下特点：

- 语言无关，平台无关

  Protobuf支持Java、 C++,、Python、JavaScript等多种语言，支持跨多个平台。

- 高效

  比XML更小（3~10倍），更快（20 ~ 100倍），更为简单。

- 扩展性，兼容性好

  可以更新数据结构，而不影响和破坏原有的旧程序。

是Protobuf更加适合于高性能、快速响应的数据传输应用场景，它是一种二进制的协议，本身不具有可读性，只有反序列化后才能可读，正因为二进制所以序列化后体积比JSON和XML小，更适合高性能的通信服务器。微信的传输协议就采用了Protobuf协议。

#### **控制台生成POJO和Builder**

我们先编写一个proto文件：

~~~protobuf
//头部声明
syntax = "proto3";
package com.learn.netty.protocol;
//开始Java选项配置
option java_package = "com.learn.netty.protocol";
option java_outer_classname = "MsgProtos";
//消息定义
message Msg{
	uint32 id =1;
	string content = 2;
}
~~~

在“proto”文件中，使用message关键字来定义消息的结构体。在生成“proto”对应的Java代码时，每个具体的消息结构体将对应于一个最终的Java POJO类。结构体的字段对应到POJO类的属性。

在编写完proto文件后我们可以利用文件生成POJO和Builder，proto有两种方式控制台命令和maven插件。一般使用maven插件，当然我们这里也介绍一下命令行的方式。

想使用首先要安装protobuf，官方Github下载对应的安装包，我们这里就以windows为例，下载完后解压，**然后将bin目录配置到环境变量中**，解压完后如下图所示：

![image-20220530171803401](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220530171803401.png)

然后我们输入命令`protoc.exe --java_out=. 1.proto `生成Java类：其中`--java_out=.`表示Java类的输出路径是当前路径，1.proto是我们编写好的proto文件

![image-20220530172056503](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220530172056503.png)

#### **maven插件生成POJO和Builder**

但是更加常用的是Maven插件，使用protobuf-maven-plugin插件，可以非常方便的生成消息的POJO类和Builder的Java代码。具体操作如下：

先在pom中增加插件配置：

```xml
<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.6.1</version>

    <configuration><!-- proto文件目录 -->
        <protoSourceRoot>${project.basedir}/src/main/proto</protoSourceRoot>
        <!-- 生成的Java文件目录 -->
        <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
        <clearOutputDirectory>false</clearOutputDirectory>
        <!--<outputDirectory>${project.build.directory}/generated-sources/protobuf</outputDirectory>-->
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
                <goal>test-compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

然后再pom中增加依赖：

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.19.1</version>
</dependency>
```

然后我们再proto文件目录中将之前的文件拷贝进去：

```protobuf
syntax = "proto3";
package com.learn.protocol;

option java_package  = "com.learn.protobuf.protocol";
option java_outer_classname = "MsgProtos";

message Msg{
 uint32 id =1;
 string content = 2;
}
```

目录结构如下：

![image-20220531104900396](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220531104900396.png)

然后我们使用插件进行编译：

![image-20220531104925026](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220531104925026.png)

这时再去看生成目录下就有文件了，我这里是protocol包下

> PS：以上在win11上会编译失败，输出乱码，暂未找到原因。

#### Protobuf的序列化和反序列化示例

序列化和反序列化之前需要使用Builder构造者构建POJO消息对象

Protobuf为每个message消息结构体生成的Java类中，包含了一个POJO类、一个Builder类。构造POJO消息，首先使用POJO类的newBuilder静态方法获得一个Builder构造者，其次POJO每一个字段的值，需要通过Builder构造者的setter方法去设置。字段值设置完成之后，使用构造者的build()方法构造出POJO消息对象。示例代码如下：

```java
@SpringBootTest
class ProtobufApplicationTests {
    public static MsgProtos.Msg buildMsg(){
        final MsgProtos.Msg.Builder builder = MsgProtos.Msg.newBuilder();
        builder.setId(1001);
        builder.setContent("测试数据");
        return builder.build();
    }
}
```

构建完消息对象后就需要进行序列化，以下是三种序列化和反序列化的三种方式：

1. 序列化和反序列化第一种方式

   这种方式主要是通过Protobuf的POJO对象的toByteArray方法将POJO对象转换成字节数组，然后调用Protobuf的POJO对象的parseFrom进行反序列化：测试方法如下：

   ```java
   @Test
   public void SAD1() throws IOException {
       MsgProtos.Msg msg = buildMsg();
       //将对象序列化为二进制数组
       byte[] bytes = msg.toByteArray();
       ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
       outputStream.write(bytes);
       //反序列化
       byte[] byteArray = outputStream.toByteArray();
       MsgProtos.Msg msgGet = MsgProtos.Msg.parseFrom(byteArray);
       System.out.println(msgGet.getId());
       System.out.println(msgGet.getContent());
   }
   ```

2. 序列化和反序列化第二种方式

   第二种方式是通过调用Protobuf生成的POJO对象的writeTo方法将POJO对象的二进制字节写出到输出流。通过调用Protobuf生成的POJO对象的parseFrom方法，Protobuf从输入流中读取二进制码然后反序列化。这种方式在阻塞式场景中是没问题的，但是这种方式在异步操作的NIO应用场景中，存在粘包/半包的问题。下面是示例：

   ```java
   @Test
   public void SAD2() throws IOException {
       MsgProtos.Msg msg = buildMsg();
       //将对象序列化为二进制码流
       ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
       msg.writeTo(outputStream);
       ByteArrayInputStream inputStream = new ByteArrayInputStream(outputStream.toByteArray());
   
       //从二进制码流反序列化到protobuf对象
       MsgProtos.Msg msgGet = MsgProtos.Msg.parseFrom(inputStream);
       System.out.println(msgGet.getId());
       System.out.println(msgGet.getContent());
   }
   ```

3. 序列化和反序列化第三种方式

   这种方式通过调用Protobuf生成的POJO对象的writeDelimitedTo（OutputStream）方法在序列化的字节码之前添加了字节数组的长度。这一点类似于前面介绍的Head-Content协议，只不过Protobuf做了优化，长度的类型不是固定长度的int类型，而是可变长度varint32类型。

   ```java
   @Test
   public void SAD3() throws IOException {
       MsgProtos.Msg msg = buildMsg();
       //将对象序列化为二进制码流
       ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
       msg.writeDelimitedTo(outputStream);
       ByteArrayInputStream inputStream = new ByteArrayInputStream(outputStream.toByteArray());
   
       //从二进制码流反序列化到protobuf对象
       MsgProtos.Msg msgGet = MsgProtos.Msg.parseDelimitedFrom(inputStream);
       System.out.println(msgGet.getId());
       System.out.println(msgGet.getContent());
   }
   ```

### 13.9.4 Protobuf编码解码实践

#### **Netty内置的Protobuf基础编解码器**

Netty默认支持Protobuf的编码与解码，内置了一套基础的Protobuf编码和解码器。Netty内置的基础Protobuf编码器/解码器为：ProtobufEncoder编码器和ProtobufDecoder解码器。此外，还提供了一组简单的解决半包问题的编码器和解码器。

1. ProtobufEncoder编码器

   这个解码器实现逻辑非常简单，直接使用了Protobuf POJO类实例的toByteArray()方法将自身编码成二进制字节，然后放入Netty的Bytebuf缓冲区，发送到下一站，源码如下：

   ```java
   @Sharable
   public class ProtobufEncoder extends MessageToMessageEncoder<MessageLiteOrBuilder> {
       @Override
       protected void encode(ChannelHandlerContext ctx, MessageLiteOrBuilder msg, List<Object> out)
               throws Exception {
           if (msg instanceof MessageLite) {
               out.add(wrappedBuffer(((MessageLite) msg).toByteArray()));
               return;
           }
           if (msg instanceof MessageLite.Builder) {
               out.add(wrappedBuffer(((MessageLite.Builder) msg).build().toByteArray()));
           }
       }
   }
   ```

2. ProtobufDecoder解码器

   ProtobufDecoder与ProtobufEncoder相互对应，只不过使用的时候，ProtobufDecoder需要指定一个Protobuf POJO类实例，作为解码的原型参考，解码时根据原型实例找到对应的Parser解析器，将二进制字节解码成Protobuf POJO类实例。在Java NIO通信中，仅仅使用以上这组编码器和解码器，传输过程中会存在粘包/半包的问题。Netty也提供了配套的Head-Content类型的Protobuf编码器和解码器，在二进制码流之前加上二进制字节数组的长度。

3. ProtobufVarint32LengthFieldPrepender长度编码器

   这个编码器的作用是，在ProtobufEncoder生成的字节数组之前，前置一个varint32数字，表示序列化的二进制字节数量或者长度

4. ProtobufVarint32FrameDecoder长度解码器

   ProtobufVarint32FrameDecoder和ProtobufVarint32LengthFieldPrepender相互对应，其作用是，根据数据包中长度域（varint32类型）中的长度值，解码一个足额的字节数组，然后将字节数组交给下一站的解码器ProtobufDecoder。varint32是一种紧凑的表示数字的方法，它不是一种固定长度（如32位）的数字类型。varint32它用一个或多个字节来表示一个数字，值越小的数字，使用的字节数越少，值越大使用的字节数越多。varint32根据值的大小自动进行收缩，这能减少用于保存长度的字节数。也就是说，varint32与int类型的最大区别是：varint32用一个或多个字节来表示一个数字，而int是固定长度的数字。varint32不是固定长度，所以为了更好地减少通信过程中的传输量，消息头中的长度尽量采用varint格式。

#### **Protobuf传输案例**

这个案例整体上和JSON的案例差不多，下面给出代码：

```java
public class ProtobufServer {

    public void start(){
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            serverBootstrap
                    .group(bossGroup,workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(8999)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    //protobufDecoder仅负责编码，不支持读半包，所以在之前要有读半包处理器。
                    //三种方式：
                    // 使用netty提供ProtobufVarint32FrameDecoder
                    // 继承netty提供的通用半包处理器 LengthFieldBasedFrameDecoder
                    // 继承ByteToMessageDecoder类，自己处理半包
                    ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
                    ch.pipeline().addLast(new ProtobufDecoder(MsgProtos.Msg.getDefaultInstance()));
                    ch.pipeline().addLast(new MyProtoHandler());
                }
            });
            ChannelFuture channelFuture = serverBootstrap.bind().sync();
            System.out.println(" 服务器启动成功，监听端口: " + channelFuture.channel().localAddress());
            ChannelFuture closeFuture = channelFuture.channel().closeFuture();
            closeFuture.sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }

    }

    static class MyProtoHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            MsgProtos.Msg protoMsg = (MsgProtos.Msg) msg;
            System.out.println("收到一个 MsgProtos.Msg 数据包 :");
            System.out.println("protoMsg.getId()=" + protoMsg.getId());
            System.out.println("protoMsg.getContent()=" + protoMsg.getContent());
        }
    }

    public static void main(String[] args) {
        new ProtobufServer().start();
    }
}
```

```java
public class ProtobufClient {
    static String content = "测试消息";

    public void start() {
        Bootstrap bootstrap = new Bootstrap();
        //创建reactor 线程组
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            bootstrap.group(workerLoopGroup);
            bootstrap.channel(NioSocketChannel.class);
            bootstrap.remoteAddress("127.0.0.1", 8999);
            bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
                    ch.pipeline().addLast(new ProtobufEncoder());
                }
            });
            ChannelFuture f = bootstrap.connect();
            f.addListener((ChannelFuture futureListener) ->
            {
                if (futureListener.isSuccess()) {
                    System.out.println("客户端连接成功!");

                } else {
                    System.out.println("客户端连接失败!");
                }

            });
            f.sync();
            Channel channel = f.channel();
            //发送 Protobuf 对象
            for (int i = 0; i < 1000; i++) {
                MsgProtos.Msg user = build(i, i + "->" + content);
                channel.writeAndFlush(user);
                System.out.println("当前报文个数" + i);
            }
            channel.flush();
            ChannelFuture closeFuture = channel.closeFuture();
            closeFuture.sync();

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            workerLoopGroup.shutdownGracefully();
        }

    }

    //构建ProtoBuf对象
    public MsgProtos.Msg build(int id, String content) {
        MsgProtos.Msg.Builder builder = MsgProtos.Msg.newBuilder();
        builder.setId(id);
        builder.setContent(content);
        return builder.build();
    }

    public static void main(String[] args) throws InterruptedException {
        new ProtobufClient().start();
    }
}
```

### 13.9.5 Protobuf协议语法

在Protobuf中，通信协议的格式是通过“.proto”文件定义的。一个`.proto`文件有两大组成部分：头部声明、消息结构体的定义。头部声明的部分，主要包含了协议的版本、包名、特定语言的选项设置等；消息结构体部分，可以定义一个或者多个消息结构体。在Java中用protobuf编译器编译后，会生成Java中的POJO类和Builder类，然后就可以进行获取设置字段和消息序列化和反序列化。

**proto文件头部声明**

~~~protobuf
//开始声明
syntax = "proto3";
package com.myim.protocol
//开始java选项配置
option java_package = "com.myim.protocol"
option java_outer_classname = "MsgProto"; 
~~~

下面来进行介绍：

1. syntax版本号

   对于proto文件，第一个非空非注释行必须注明protobuf语法版本，默认是proto2

2. package包

   和Java类似，protobuf通过package指定包名，避免消息冲突（即同名但包名不同可共存）。

   通过package，还可以实现消息的引用，比如在proto文件中引用其他proto文件的实体类：

   ```protobuf
   //第一个文件 ，其他略...
   package com.myim.protocol
   message Msg{
   	//...
   }
   //第二个文件 ，其他略...
   package com.myim.protocol
   message Msg{
   	com.111.Msg =1;
   }
   ```

3. option配置

   不是所有的option 配置选项都会生效，option选项是否生效与“.proto”文件使用的一些特定的语言场景有关。在Java语言中，以“java_”打头的“option”选项会生效.

   - 选项“option java_package”表示Protobuf编译器在生成Java POJO消息类时，生成在此选项所配置的Java包名下。如果没有该选项，则会以头部声明中的package作为Java包名。

   - 选项“option java_multiple_files”表示在生成Java类时的打包方式，具体来说，有以下两种方式：

     - 方式1：一个消息对应一个独立的Java类。

     - 方式2：所有的消息都作为内部类，打包到一个外部类中。

       此选项的值，默认为false，表示使用外部类打包的方式。如果设置 true，则使用第一种方式生成Java类，则一个消息对应一个POJO Java类，多个消息结构体会对应到多个类。

   - 选项“option java_outer_classname”表示在Protobuf编译器在生成Java POJO消息类时，如果采用的是上面的方式2（全部POJO类都作为内部类打包在同一个外部类中），则以此选项所配置的值，作为唯一外部类的类名

**proto的消息结构体和消息字段**

定义一个protobuf消息结构体的关键字为message。一个消息结构体有一个或多个消息字段组合而成，例如：

~~~protobuf
message Msg{
	uint32 id =1;
	string content = 2;
}
~~~

protobuf 消息字段的格式为：

`限定修饰符|数据类型|字段名称|=|分配标识号`其中：

1. 限定修饰符：

   - repeated限定修饰符：表示该字段可以包含0-N个元素值，相当于Java的List
   - singular限定修饰符：默认值，表示该字段可以包含0-1个元素值
   - reserved限定修饰符：指定保留字段名称和分配标识号，用于将来的拓展

2. 数据类型

3. 字段名称

   字段名称的命名与Java语言的成员变量命名方式几乎是相同的。Protobuf建议字段的命名以下划线分割，例如使用first_name形式，而不是的驼峰式firstName

4. 分配标识号

   在消息定义中，每个字段都有唯一的一个数字标识符，可以理解为字段编码值，叫做分配标识号（Assigning Tags）。通过该值，通信双方才能互相识别对方的字段。当然，相同的编码值，它的限定修饰符和数据类型必须相同。分配标识号是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变。

   > 分配标识号的取值范围为 1~232（4294967296）。其中编号[1，15]之内的分配标识号，时间和空间效率都是最高的。为什么呢？[1，15]之内的标识号，在编码的时候只会占用一个字节，[16，2047]之内的标识号则要占用2个字节。所以那些频繁出现的消息字段，应该使用 [1，15]之内的标识号。切记：要为将来有可能添加的、频繁出现的字段预留一些标识号。另外，[1900，2000]之内的标识号，为Protobuf内部保留值，建议不要在自己的项目中使用。

   标识号的特点是：一个消息结构体中的标识号是可以不连续的；在同一个消息结构体中，不同的字段不能使用相同的标识号

**proto的数据类型**

| proto Type | 说明                                                     | 对应的JavaType |
| ---------- | -------------------------------------------------------- | -------------- |
| double     | 双精度浮点                                               | double         |
| float      | 单精度浮点                                               | float          |
| int32      | 使用变长编码，负值效率低，字段如果为负值可以用sint64代替 | int            |
| uint32     | 使用变长编码的32位整型                                   | int            |
| uint64     | 使用变长编码的64位整型                                   | long           |
| sint32     | 使用变长编码、有符号的32位整型。在负值上比int32高效      | int            |
| sint64     | 使用变长编码、有符号的64位整型。在负值上比int32高效      | long           |
| fixed32    | 总是四个字节，如果数值总是比2^28大的话，比uint32高效     | int            |
| fixed64    | 总是四个字节，如果数值总是比2^28大的话，比uint32高效     | long           |
| sfixed32   | 总是4个字节                                              | int            |
| sfixed64   | 总是8个字节                                              | long           |
| Bool       | 布尔                                                     | boolean        |
| String     | 字符串，编码必须是UTF8或者7-bit ASCII编码的文本          | String         |
| Bytes      | 可能包含任意顺序的字节数据                               | ByteString     |

> 变长编码的类型（如int32）表示打包的字节并不是固定，而是根据数据的大小或者长度来定。例如int32，如果数值比较小，在0~127时，使用一个字节打包。
>
> fixed32的打包效率比int32的效率高，但是使用的空间一般比int32多。因此定长编码属于时间效率高，变长编码属于空间效率高

**proto的其他规范**

1. import声明

   在需要多个消息结构体时，`.poto`文件可以像Java语言的类文件一样，按照模块进行分开设计，所以一个项目可能有多个`.poto`文件，一个文件在需要依赖其他`.poto`文件的时候，可以通过import进行导入。导入的操作和Java的import的操作大致相同。

2. 嵌套消息

   `.proto`文件支持嵌套消息，消息中既可以包含另一个消息实例作为其字段，也可以在消息中定义一个新的消息

   ~~~protobuf
   message Outer{
   	message MA{
   		message Inner{
   			int64 intVal = 1;
   			bool booleanVal = 2;
   		}
   	}
   	message MB{
   		message Inner{
   			int64 intVal = 1;
   			bool booleanVal = 2;
   		}
   	}
   }
   ~~~

3. enum枚举

   枚举的定义和Java相同，但是有一些限制。枚举值必须大于等于0的整数。另外，需要使用分号`;`分隔枚举变量，而不是Java语言中的逗号`,`

   ~~~protobuf
   enum VoipProtocol{
       H323 = 1;
       SIP = 2;
       MGCP = 3;
       H248 = 4;
   }
   ~~~

### 13.9.6 序列化反序列化、编码解码之间的关系

#### **序列化和反序列化的原理**

在之前的分布式Java程序中，需要进行远程调用，把本地的结果对象通过网络传输到远程的服务。比如客户端调用服务端的用户查询接口（API），服务端查询到Pojo对象后通过远程调用把查询的结果通过TCP连接传输到客户端。在TCP连接上只能通过数据包传输二进制字节流，假如在服务端的JVM中User对象的name属性为“张三”，age属性值为18，那么在JVM的User对象大致如下图：

![image-20220616125607566](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220616125607566.png)

虽然在内存中User对象是以字节的形式存储的，但是其存储在内存中的属性值，并不是具体的值，而是其值的内存地址，如果服务端把这些内存数据直接传输到客户端，客户端凭借这些内存地址是肯定找不到张三的值，也找不到18的值。这时候就需要进行序列化，把Java对象转换成字节序列，用来进行远程传输或保存到硬盘上。但是这些字节需要同时还需要恢复成Java对象，因此还需要配套的反序列化。

目前几种序列化和反序列化的方式：

1. Java内置的序列化能力

   Java的OutputStream的子类ObjectOutputStream类的`WriteObject(Object object)`方法就可以把对象写入到二进制字节流，对于对象的属性而言，序列化的字节流里面，不会有用户空间的地址，而是会替换成对应的值。

2. JSON

   将对象先转换成JSON字符串，然后再进一步提取字符串中的字节序列。JSON是一种轻量级数据交换格式，易于阅读和编写，也易于机器解析和生成。Google和Gson和阿里的FastJson库等都可以进行JSON的序列化和反序列化

3. 将对象转换成更紧凑的二进制数据

   JSON和Java自带的序列化都不能最大化进行压缩，所以可以使用类似Protobuf这种开源的序列化组件，根据官方说明，Protobuf更小更快。

   当然除了Protobuf，还有Thrift和Avro组件也是类似的序列化组件。这里不过多介绍，可自行百度。

#### **编码和解码的原理**

编码和解码是非常广泛的概念，在计算机领域，通过Java的内置的序列化功能，把POJO对象序列化为二进制字节码，属于编码操作。反过来，把二进制字节码恢复到POJO对象，属于解码操作。

**因此序列化可以理解为编码操作的一种，反序列化可以理解为解码操作的一种**

但是编码和解码不一定非得是二进制数据，也可以是其他的POJO对象或者中间数据。比如可以先把POJO对象序列化（通过FastJson编码）成JSON字符串，再把JSON字符串序列化（编码）成二进制数据。再比如说，如果要进行加密传输，那么在序列化之后，还需要做加密的编码处理，最终数据为加密了的二进制数据。



























