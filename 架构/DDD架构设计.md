# DDD架构设计

> 写在前面，本文是DDD架构的学习笔记，参考资料是尼恩发布的文章（详见参考资料章节），仅保存在Git仓库学习使用。
>
> 原文中的使用的例子均在学习后在本文中进行了替换以便加深学习印象并巩固知识。

## 一、概述

DDD，就像REST一样，它是一种设计思想，缺乏完整的规范，这使得新手在实践中常感到困难。对于架构师而言，降低系统的复杂性一直都是一项挑战。

从技术角度来看，有许多业界大咖一直在尝试从技术层面来降低系统的复杂性：

- 1994 年的 GoF 设计模式

- 1999 年的 Martin Fowler 的代码重构

  《重构：改善既有代码设计》是世界级的编程大师Martin Fowler的作品。Martin Fowler被誉为软件开发“教父”，ThoughtWorks公司的首席科学家，同时他也是敏捷开发的开拓者和创始人。他的作品包括《重构：改善既有代码设计》以及其他杰出的著作，如《企业应用架构模式》、《领域特定语言》、《NoSQL精粹》和《分析模式：可重用对象模型》等。

- 2003 年的企业集成模式 EIP（Enterprise Integration Patterns）

  EIP（Enterprise Integration Patterns，企业集成模式）是集成领域的圣经，它为许多消息中间件和ESB提供了理论基础。各种消息概念和术语，如消息代理、消息通道、消息端点、消息路由、消息转换、消息增强、信息分支、消息聚合、消息分解、消息重排等，都来源于EIP，并在《企业集成模式：设计、构建及部署消息传递解决方案》一书中得到详细解释。

这些著作通过一系列设计模式和范例来帮助降低系统的复杂性，但是从技术角度解决问题并不能从业务角度解决问题。

因此，除了从技术角度出发，还有许多大师从业务角度出发，探索降低软件复杂性的根本方法：

- 2003 年 Eric Evans 的《领域驱动设计》
- Vaughn Vernon 的《实现领域驱动设计》
- Uncle Bob 的《清洁架构》等书籍

这些书籍都是从业务角度出发，为全球各地从事业务开发的开发者提供了一套完整的架构思路。

DDD作为一种软件架构的方法论 ，从2003年就诞生了。但是，却一直到微服务盛行的20年后，才开始普及和火爆，这是因为：

- 领域驱动设计DDD是一种架构思维，而非具体的开发框架或者软件技术的基础设施，因此在代码层面缺乏足够的约束，缺乏足够的，使用多业务场景的开发基础脚手架，使得DDD实际上手门槛极高。
- 从业务视角出发的设计方法论，和从技术视角出发的设计方法论，存在着很多的理论冲突。

举个例子 [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)（贫血域模型）是 Martin Fowler在他个人博客里描述的一个**Anti-pattern反模式**。

但是，贫血域模型这种反模式在实际应用中却频繁出现。比如，基于数据库技术和MVC的四层应用架构（UI、业务、数据访问、数据库）在一定程度上与DDD的概念存在冲突，导致大多数人在业务分析时使用DDD的建模，而在技术实现时使用MVC框架。此外，一些流行的ORM工具，如Hibernate和Entity Framework，实际上助长了这种MVC的贫血模型的传播。

在2012年之前，早期的应用架构主要局限于单机加负载均衡，通常通过MVC提供REST接口供外部调用，使用SOAP或Web Services进行应用之间的RPC调用。在单体架构中，应用之间的RPC调用受到外部依赖协议的限制，外部依赖的变更频繁。

这时候，"防腐层"（Anti-Corruption Layer）的概念应运而生，它解决了如何将核心业务逻辑与外部依赖隔离的问题。防腐层的概念特别适用于解决外部依赖频繁变更的情况。

到了2014年，面向服务的架构（SOA）开始流行，微服务概念逐渐兴起。如何将单体应用合理拆分为多个微服务成为热门话题，而DDD中的限界上下文（Bounded Context）为微服务的拆分提供了合理的方法论框架。

如今，在一切皆服务的时代（XAAS），DDD的思想使我们能够冷静思考，哪些内容可以以服务的方式拆分，哪些逻辑需要聚合，以降低维护成本，而不仅仅是简单追求开发效率。

综上，那么什么是DDD？用《基于DDD和微服务的中台架构与实现》一书中的概念来说就是：

DDD首先从业务领域入手，划分 业务领域边界，采用事件风暴工作坊方法，分析并提取业务场景中的实体、值对象、聚合根、聚合、领域事件等领域对象，根据限界上下文边界构建领域模型，将领域模型作为微服务设计的输入，进而完成微服务洋细设计。

用DDD方法设计出来的微服务，业务和应用边界非常清晰，符合“高内聚，低耦合”的设计原则，可以轻松适应业务模型变化和微服务架构演进。

## 二、Domain Primitive

了解DDD之前先了解下DP（Domain Primitive，原子领域）。

> DP的概念来自[Secure by Design](https://livebook.manning.com/book/secure-by-design/chapter-5/1?spm=a2c6h.12873639.article-detail.7.108512fe7Pwoew)作者是于Dan Bergh Johnsson & Daniel Deogun

DP的定义是在一个特定领域里，拥有精准定义的、可自我验证的、拥有行为的Value Object。DP在DDD里是无处不在的，它可以说是一切模型、方法、架构的基础。就像Integer、String一样。

下面通过一个简单的例子了解下DP的概念。业务逻辑如下，需要根据创建的工单号匹配对应的工单类型。

一个简单的代码实现如下：

首先是实体类：

```java
public class Order {
    public Integer id;
    public String name;
    public String phone;
    public String address;
    public String email;
}
```

然后是业务类的代码：

```java
public class OrderServiceImpl implements OrderService {

    private OrderMapper orderMapper;


    @Override
    public void createOrder(Order order) {
        if (order.getOrderNo()== null || "".equals(order.getOrderNo())){
            throw new ValidateException("invalid parameter");
        }
        //省略其他校验

        //业务逻辑，根据工单号获取工单类型名称
        String orderNo = order.getOrderNo();
        String [] typeArray = new String[]{"REPAIR ","INSTALL","OTHER"};
        for (String typePrefix : typeArray) {
            if (orderNo.startsWith(typePrefix)){
                order.setOrderType(typePrefix);
            }
        }

        //处理实体类
        Order orderSave = new Order();
        orderSave.setOrderNo(order.getOrderNo());
        orderSave.setOrderType(order.getOrderType());
        orderSave.setDirector(order.getDirector());
        orderSave.setPhone(order.getPhone());
        orderSave.setOrderTerm(order.getOrderTerm());

        //保存数据库
        orderMapper.insertOrder(orderSave);
    }
}
```

日常的代码基本都是类似上面的形式，下面就从四个维度分析一下上面代码的问题：

- 接口的清晰度（可读性）
- 数据验证和错误处理
- 业务代码逻辑清晰度
- 可测试性



## 参考资料

- [《阿里DDD大佬：从0到1，带大家精通DDD》](https://mp.weixin.qq.com/s/_FKXKC-M22FyCv9K7slISg)

  > 这是个长文章，上面链接是第一篇的链接地址。文章来自尼恩老师技术自由圈公众号。