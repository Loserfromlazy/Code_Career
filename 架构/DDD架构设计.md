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

### 2.1 案例分析

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
    public void createOrder(String orderNo, String director, String phone, Date orderTerm) {
        if (orderNo == null || "".equals(orderNo)) {
            throw new ValidateException("invalid parameter");
        }
        //省略其他校验

        //业务逻辑，根据工单号获取工单类型名称
        String prefix = "";
        String[] typeArray = new String[]{"REPAIR ", "INSTALL", "OTHER"};
        for (String typePrefix : typeArray) {
            if (orderNo.startsWith(typePrefix)) {
                prefix = typePrefix;
            }
        }
        String orderType = orderMapper.findOrderTypeByPrefix(prefix);

        //处理实体类
        Order order = new Order();
        order.setOrderNo(orderNo);
        order.setOrderType(orderType);
        order.setDirector(director);
        order.setPhone(phone);
        order.setOrderTerm(orderTerm);

        //保存数据库
        orderMapper.insertOrder(order);
    }
}

```

日常的代码基本都是类似上面的形式，下面就从四个维度分析一下上面代码的问题：

- 接口的清晰度（可读性）
- 数据验证和错误处理
- 业务代码逻辑清晰度
- 可测试性

1. **接口的清晰度**：

   在Java中，方法的参数名在编译时是会丢失的，所以方法实际上是这样的：`createOrder(String, String, String , Date)`，如果传入的参数是`createOrder(”随便的订单号“, "随便填的负责人", "随便填的手机号", 2023-01-01)`，这种编译器是无法发现的，也很难通过code review发现，只能是在运行时被发现。

   在实际场景中，一般都通过方法名称命名来区分，如下：

   ```java
   Order findByOrderNo(String OrderNo);
   Order findByDirector(String director);
   Order findByOrderNoAndDirector(String OrderNo,String director);
   ```

   当然这种方式如果参数多了，不仅要保证参数正确，还要保证顺序正确。

2. **数据验证和错误处理**：

   在上面的代码中有类似这样的代码：

   ```java
   if (orderNo == null || "".equals(orderNo)) {
       throw new ValidateException("invalid parameter");
   }
   ```

   在日常开发中很常见，这样的代码是有弊端的，假设有多个类似的接口和入参，就要重复很多遍，且如果参数要拓展校验，比如orderNo又有其他的校验：

   ```java
   if (orderNo == null || "".equals(orderNo) || isValidNo(orderNo)) {
       throw new ValidateException("invalid parameter");
   }
   ```

   就需要这样扩展代码，而如果有很多地方有这个参数，而有一个地方忘记修改，那么就会造成bug。

   > 这是一个DRY原则被违背导致经常会发生的问题：DRY原则就是Don't Repeat Youself ，不要重复自己，可以理解为不要写重复的代码。

   在传统的Java开发中，可以使用BeanValidation注解或者类似的ValidationUtils工具类解决参数校验问题，但是这类工具一样有些问题：

   - BeanValidation注解：通常只能解决简单的校验，且新逻辑产生时，如果某些地方忘记加注解，还是会造成一样的问题。
   - ValidationUtils工具类：当大量的校验逻辑集中在一个类时，会违背单一性原则，导致代码混乱且不可维护。

3. **业务代码逻辑清晰度**:

   上面的代码有这么一段胶水代码：

   ```java
   //业务逻辑，根据工单号获取工单类型名称
   String prefix = "";
   String[] typeArray = new String[]{"REPAIR ", "INSTALL", "OTHER"};
   for (String typePrefix : typeArray) {
       if (orderNo.startsWith(typePrefix)) {
           prefix = typePrefix;
       }
   }
   String orderType = orderMapper.findOrderTypeByPrefix(prefix);
   ```

   常见的处理这种代码的方式是抽离出来，变成独立的一个或多个方法：

   ```java
   private String getOrderType(String orderNo){
       String prefix = "";
       String[] typeArray = new String[]{"REPAIR ", "INSTALL", "OTHER"};
       for (String typePrefix : typeArray) {
           if (orderNo.startsWith(typePrefix)) {
               prefix = typePrefix;
           }
       }
       return orderMapper.findOrderTypeByPrefix(prefix);
   }
   ```

   原方法变成这样：

   ```java
   //业务逻辑，根据工单号获取工单类型名称
   String orderType = getOrderType(orderNo);
   ```

   如果这个方法需要复用，可能还会抽取工具类，如果项目中有大量的静态工具类，那么核心业务逻辑就变得异常难找。

4. **可测试性**:

   为了保证代码质量，一般每个方法的每个入参都需要有TC覆盖，我们上面的方法需要以下的TC：

   > TC：Test Case 测试用例，包含输入、期望输出等

   | 条件\入参              | orderNo | director | phone | orderTerm |
   | ---------------------- | ------- | -------- | ----- | --------- |
   | 入参为null             |         |          |       |           |
   | 入参为空               |         |          |       |           |
   | 入参不符合要求（多个） |         |          |       |           |

   一般来说，假设一个方法有N个参数，每个参数有M个校验逻辑，那么就至少要有`N*M`个TC。假设有P个方法都用到了某字段，那么就一共需要`P*N*M`个TC才能完全覆盖所有数据验证问题。而这个成本非常高，所以日常开发中不可能全被覆盖到，而这个地方才是最容易出问题的地方。这时降低测试成本等于提高代码质量。

### 2.2 提高代码质量

我们来看一下上面的业务，根据创建的工单号匹配对应的工单类型，分析一下就会发现这里订单类型的概念被隐藏到了代码中，这里可以分析一下，取工单的类型前缀这个逻辑是属于工单还是属于创建工单，如果都不是很贴切，说明这个逻辑应该属于一个独立的概念，所以这里引入第一个原则：`Make Implicit Concepts Expecit`将隐性的概念显性化。下面我们将工单号的概念显性化，代码如下：

```java
public class OrderNo {

    private final String orderNo;

    public String getOrderNo() {
        return orderNo;
    }

    public OrderNo(String orderNo) {
        if (orderNo == null) {
            throw new ValidateException("参数不能为空");
        }else if ("".equals(orderNo)){
            throw new ValidateException("参数不能为空串");
        }
        this.orderNo = orderNo;
    }

    public String getPrefix(){
        String prefix = "";
        String[] typeArray = new String[]{"REPAIR ", "INSTALL", "OTHER"};
        for (String typePrefix : typeArray) {
            if (orderNo.startsWith(typePrefix)) {
                prefix = typePrefix;
            }
        }
        return prefix;
    }
}
```

这里有几个很重要的点：

1. 首先orderNo是final的，确保OrderNo是一个不可变的Value Object。
2. 其次校验逻辑都放在构造函数中，确保OrderNo一被创建出来后，一定是校验通过的。
3. 然后把getOrderType变成OrderNo类的一个方法，将orderType的前缀变成了一个OrderNo的计算属性。

这样生成了OrderNo类之后，其实是生成了一个Type(数据类型)和一个Class（类），Type是指在今后的代码里通过OrderNo去显性的表示工单号这个概念，Class表示可以把所有跟工单号有关的逻辑都放到一个类里。使用了DP之后，其他类代码如下：

```java
@Data
public class Order {
    //工单编号
    public OrderNo orderNo;
    //工单类型
    public String orderType;
    //工单负责人电话
    public String phone;
    //工单负责人
    public String director;
    //工单期限
    public Date orderTerm;
}
//-------------------------分割线--------------------------------
@Override
public void createOrder(@NotNull OrderNo orderNo,@NotNull String director,@NotNull String phone,@NotNull Date orderTerm) {
    //业务逻辑，根据工单号获取工单类型名称
    String orderType = orderMapper.findOrderTypeByPrefix(orderNo.getPrefix());


    //处理实体类
    Order order = new Order();
    order.setOrderNo(orderNo);
    order.setOrderType(orderType);
    order.setDirector(director);
    order.setPhone(phone);
    order.setOrderTerm(orderTerm);

    //保存数据库
    orderMapper.insertOrder(order);
}
```

下面我们在重新用上面的四个维度分析一下这段代码：

1. **接口的清晰度**：

   假设我们的参数都用DP重构后，`createOrder(OrderNo orderNo,Director director,Phone phone,OrderTerm orderTerm)`,这样的接口很清晰，即使用之前那种传参`createOrder(new Order(”随便的订单号“), new Director("随便填的负责人"), new Phone("随便填的手机号"),new OrderTerm(2023-01-01))`也能在编译期被及时发现。

   同样的，查询方法也可以充分使用方法重载：

   ```java
   Order find(OrderNo orderNo);
   Order find(Director director);
   Order find(OrderNo OrderNo,Director director);
   ```

2. **数据验证和错误处理**：

   ```java
   public void createOrder(@NotNull OrderNo orderNo,@NotNull Director director,@NotNull Phone phone,@NotNull OrderTerm orderTerm) {
   }
   ```

   这里可以看到，因为DP的特性，重构后的方法里不会有任何的数据验证的逻辑，只要是能够带到入参里的一定是正确的，把数据验证的工作量前置到了调用方，而调用方本来就是应该提供合法数据的，所以更加合适。

   同时使用DP的另一个好处就是代码遵循了DRY原则和单一性原则，如果未来需要修改OrderNo的校验逻辑，只需要在一个文件里修改即可，所有使用到了OrderNo的地方都会生效。

3. **业务代码的清晰度**：

   ```java
   String orderType = orderMapper.findOrderTypeByPrefix(orderNo.getPrefix());
   ```

   除了不再需要校验数据之外，原来的胶水代码被改为了OrderNo的一个计算属性getPrefix，让代码清晰度大大提升。

   而且胶水代码通常都不可复用，但是使用了DP后，变成了可复用、可测试的代码。

4. **可测试**：

   | 条件\入参              | orderNo | director | phone | orderTerm |
   | ---------------------- | ------- | -------- | ----- | --------- |
   | 入参为null             |         |          |       |           |
   | 入参为空               |         |          |       |           |
   | 入参不符合要求（多个） |         |          |       |           |

   当把OrderNo抽取之后，再来看一下TC：首先 PhoneNumber 本身还是需要 个测试用例，但是由于我们只需要测试单一对象，每个用例的代码量会大大降低，维护成本降低。且每个方法里的每个参数，现在只需要覆盖为null的情况就可以了，其他的case不可能发生（因为只要不是null就一定是合法的）。所以，单个方法的TC从`N*M`变成了`N+M`，多个方法的TC数量从`N*M*P`变成了`N+M+P`。

   > 这里其实本质上就是

### 2.3 DP的进阶使用

上面2.2章节中介绍了DP的第一个原则，将隐形的概念显性化，下面将通过转账的案例介绍另两个原则即：

- Make Implicit Context Expecit 将隐性的上下文显性化
- Encapsulate Multi-Object Behavior 封装多对象行为

下面通过转账功能让A用户支付X元给B：

```java
public void sendMessage(String message,Long userId){
    unifiedMessageService.send(message,"Chinese",userId);
}
```



## 参考资料

- [《阿里DDD大佬：从0到1，带大家精通DDD》](https://mp.weixin.qq.com/s/_FKXKC-M22FyCv9K7slISg)

  > 这是个长文章，上面链接是第一篇的链接地址。文章来自尼恩老师技术自由圈公众号。