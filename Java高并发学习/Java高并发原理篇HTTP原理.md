# Java高并发原理篇HTTP原理

转载请声明！！！切勿剽窃他人成果。本文如有错误欢迎指正，感激不尽。

> 参考资料：Java高并发核心编程卷一尼恩编著、以及菜鸟等互联网资源

# 一、HTTP原理与Web服务器

### 1.1 高性能Web应用架构

一般来说，高性能的IM应用还需要高性能的Web应用来进行配合。高并发、大流量的WEB应用，QPS在10万甚至千万每秒，所以使用高并发HTTP通信技术提升内部各节点的通信性能，对于提升分布式系统整体的吞吐量有着非常大的作用。

我们先来看10w级别的Web应用架构：

![image-20220805105906938](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805105906938.png)



百万级别的Web应用架构：

![image-20220805110518046](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220805110518046.png)

1.2 HTTP应用层协议

1.3 HTTP协议的演进

1.4 基于Netty实现简单的Web服务器

# 二、高并发HTTP的核心原理

# 三、WebSocket原理

# 四、SSL/TLS核心原理