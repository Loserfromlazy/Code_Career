# Java注解学习笔记

## 一、简介

### 1.1 Java Annotation的定义与分类

Java 注解（Annotation）又称 Java 标注，是 JDK5.0 引入的一种注释机制。

Java 语言中的类、方法、变量、参数和包等都可以被标注。和 Javadoc 不同，Java 标注可以通过反射获取标注内容。在编译器生成类文件时，标注可以被嵌入到字节码中。Java 虚拟机可以保留标注内容，在运行时可以获取到标注内容 。 当然它也支持自定义 Java 标注。

**注解本身不会有任何的作用，需要有其他代码或工具的支持才有用。**



**Java一共定义了7个注解，3个在 java.lang 中是作用在代码中的，剩下 4 个在 java.lang.annotation 中，他们主要作用是元注解也就是作用在注解上的注解。**

**作用在代码的注解是**

- @Override - 检查该方法是否是重写方法。如果发现其父类，或者是引用的接口中并没有该方法时，会报编译错误。
- @Deprecated - 标记过时方法。如果使用该方法，会报编译警告。
- @SuppressWarnings - 指示编译器去忽略注解中声明的警告。

作用在其他注解的注解(或者说 元注解)是:

- @Retention - 标识这个注解怎么保存，是只在代码中，还是编入class文件中，或者是在运行时可以通过反射访问。
- @Documented - 标记这些注解是否包含在用户文档中。
- @Target - 标记这个注解应该是哪种 Java 成员。
- @Inherited - 标记这个注解是继承于哪个注解类(默认 注解并没有继承于任何子类)

从 Java 7 开始，额外添加了 3 个注解:

- @SafeVarargs - Java 7 开始支持，忽略任何使用参数为泛型变量的方法或构造函数调用产生的警告。
- @FunctionalInterface - Java 8 开始支持，标识一个匿名函数或函数式接口。
- @Repeatable - Java 8 开始支持，标识某注解可以在同一个声明上使用多次。

下面我们将以Junit的@Test为例简单了解一下注解：

~~~java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Test {
    long timeout() default 0L;
}
~~~

可以看到@Test上有两个元注解@Target()`和`@Retention()

可以看到定义**注解的格式**为：

~~~java
修饰符 @interface 注解名 {   
    注解元素的声明1 
    注解元素的声明2   
}
~~~

注解中元素的声明：

~~~java
type elementName();
type elementName() default value;  // 带默认值
~~~

### 1.2 Annotation架构

架构如图：（图片来自菜鸟教程）

![img](https://www.runoob.com/wp-content/uploads/2019/08/28123151-d471f82eb2bc4812b46cc5ff3e9e6b82.jpg)

从中，我们可以看出：

**1 个 Annotation 和 1 个 RetentionPolicy 关联。**

可以理解为：每1个Annotation对象，都会有唯一的RetentionPolicy属性。

**1 个 Annotation 和 1~n 个 ElementType 关联。**

可以理解为：对于每 1 个 Annotation 对象，可以有若干个 ElementType 属性。

**Annotation 有许多实现类，包括：Deprecated, Documented, Inherited, Override 等等。**

## 二、元注解

### 2.1 @Target注解

`@Target`注解用于限制注解能在哪些项上应用，没有加`@Target`的注解可以应用于任何项上。

在`java.lang.annotation.ElementType`类中可以看到所有`@Target`接受的项

- `TYPE` 在【类、接口、注解】上使用
- `FIELD` 在【字段、枚举常量】上使用
- `METHOD` 在【方法】上使用
- `PARAMETER` 在【参数】上使用
- `CONSTRUCTOR` 在【构造器】上使用
- `LOCAL_VARIABLE` 在【局部变量】上使用
- `ANNOTATION_TYPE` 在【注解】上使用
- `PACKAGE` 在【包】上使用
- `TYPE_PARAMETER` 在【类型参数】上使用 Java 1.8 引入
- `TYPE_USE` 在【任何声明类型的地方】上使用 Java 1.8 引入

通过`@Test`的代码我们可以得出@Test只能在方法上使用，但如果想要要支持多项，则传入多个值。

```less
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface MyAnnotation { ... }
```

> 此外元注解也是注解，也符合注解的语法，如`@Target`注解。
> `@Target(ElementType.ANNOTATION_TYPE)`表明`@Target`注解只能使用在注解上。
>
> ~~~java
> @Documented
> @Retention(RetentionPolicy.RUNTIME)
> @Target(ElementType.ANNOTATION_TYPE)
> public @interface Target {
>     ElementType[] value();
> }
> ~~~

### 2.2 @Retention注解

`@Retention`指定注解应该保留多长时间，默认是`RetentionPolicy.CLASS`。
在`java.lang.annotation.RetentionPolicy`可看到所有的项

- `SOURCE` 不包含在类文件中
- `CLASS` 包含在类文件中，不载入虚拟机
- `RUNTIME` 包含在类文件中，由虚拟机载入，可以用反射API获取

从@Test的注解中我们可以看到它会载入到虚拟机，可以通过代码获取

### 2.3 @Document注解

主要用于归档工具的识别。被注解的元素能被javadoc文档化

### 2.4 @Inherited注解

添加了`@Inherited`注解的注解，所注解的类的子类也将拥有这个注解

举个例子，自定义一个注解MyAnnotation

```less
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface MyAnnotation { ... }
```

然后将他标注在父类上

```angelscript
@MyAnnotation 
class Parent { ... }
```

这时子类`Child`会把加在`Parent`上的`@MyAnnotation`继承下来

```scala
class Child extends Parent { ... }
```

> PS：
> 在接口上添加注解，然后类实现了接口，类不会拥有接口上的注解。
> 抽象类添加了注解，并且这个注解是可继承的，那么抽象类的子类拥有抽象类注解。

## 三、Java内置的注解

### `@Override`注解

告诉编译器这个是个覆盖父类的方法。如果父类删除了该方法，则子类会报错。

### `@Deprecated`注解

表示被注解的元素已被弃用。

### `@SuppressWarnings`注解

告诉编译器忽略警告。

### `@FunctionalInterface`注解

Java 1.8 引入的注解。该注释会强制编译器`javac`检查一个接口是否符合函数接口的标准。

## 四、注解中元素的类型

在注解中支持的元素的类型为

- 8种基本数据类型（`byte`，`short`，`char`，`int`，`long`，`float`，`double`，`boolean`）
- `String`
- `Class`
- `enum`
- 注解类型
- 数组（所有上边类型的数组）

自定义注解示例：

首先定义一个枚举供注解使用

```java
public enum Status {
    GOOD,
    BAD
}
```

定义一个注解MyAnnotation

```java
@Target(ElementType.ANNOTATION_TYPE)
public @interface MyAnnotation {
    int val();
}
```

定义一个注解MyAnnotation1

```java
@Target(ElementType.TYPE)
public @interface MyAnnotation1 {
    boolean boo() default false;
    Class<?> cla() default Void.class;
    Status enu() default Status.GOOD;
    MyAnnotation anno() default @MyAnnotation(val = 1);
    String [] arr();
}
```

定义一个测试类

```java
@MyAnnotation1(arr = {"1","2"})
public class Test {
}
```

> PS：注解元素无默认值时必须要传值。

## 五、在Spring中使用自定义注解

### 5.1 需求分析

假设程序需要接收不同的指令CMD，然后根据不同的指令调用不同的处理类Handler，我们可以使用Map来存储命令和处理类的映射关系，但是项目是由多个成员开发，所以希望开发人员只关注Handler的实现，不需要主动去Map中注册CMD和Handler的映射。

最终效果如下：

```java
@CmdMapping(Cmd.LOGIN)
public class LoginHandler implements ICmdHandler {
    @Override
    public void handle() {
        System.out.println("handle login request");
    }
}

@CmdMapping(Cmd.LOGOUT)
public class LogoutHandler implements ICmdHandler {
    @Override
    public void handle() {
        System.out.println("handle logout request");
    }
}
```

开发人员增加自己的`Handler`，只需要创建新的类并注上`@CmdMapping(Cmd.Xxx)`即可。

### 5.2 实现

具体的实现是使用Spring和自定义注解

首先定义@CmdMapping

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface CmdMapping {
    int value();
}
```

> `@CmdMapping`中有一个`int`类型的元素`value`，用于指定`CMD`。这里做成一个单值注解。
> 这里还加了`Spring`的`@Component`注解，因此注解了`@CmdMapping`的类也会被Spring创建实例。

然后定义CMD接口，存储命令

```java
public interface Cmd {
    int REGISTER =1;
    int LOGIN =2;
    int LOGOUT =3;
}
```

处理类接口

```java
public interface ICmdHandler {
    void handler();
}
```

注解是不起作用的，需要其他的支持，下面是让注解生效的部分：

```java
@Component
public class HandlerDispatcherServlet implements InitializingBean , ApplicationContextAware {

    private ApplicationContext applicationContext;
    private Map<Integer,ICmdHandler> handlers = new HashMap<>();

    public void handle(int cmd){
        handlers.get(cmd).handler();
    }
    /**
     * 继承InitializingBean接口的方法，InitializingBean接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候都会执行该方法。
     **/
    @Override
    public void afterPropertiesSet() throws Exception {
        // 获取所有的beanName
        String[] beanNames = this.applicationContext.getBeanNamesForType(Object.class);
        for (String beanName : beanNames) {
            if (ScopedProxyUtils.isScopedTarget(beanName)) {
                continue;
            }
            // 通过beanName获取Bean类型
            Class<?> beanType = this.applicationContext.getType(beanName);

            if (beanType != null) {
                // 是否带有CmdMapping注解
                CmdMapping annotation = AnnotatedElementUtils.findMergedAnnotation(
                        beanType, CmdMapping.class);

                if(annotation != null) {
                    // 获取bena，在handlers中注册
                    handlers.put(annotation.value(), (ICmdHandler) applicationContext.getBean(beanType));
                }
            }
        }
    }
    /**
     * 继承ApplicationContextAware接口的方法，实现了这个接口的bean，当spring容器初始化的时候，会自动的将ApplicationContext注入进来
     **/
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

这部分代码主要工作全是Spring在做，我们只是维护Map

然后定义三个ICmdHandler的实现类，此处以一个为例

```java
@CmdMapping(Cmd.LOGIN)
public class LoginHandler implements ICmdHandler {
    @Override
    public void handler() {
        System.out.println("handle login request");
    }
}
```

测试类

```java
@ComponentScan("com.learn.testMyAnno")
public class Test {
    public static void main(String[] args) {
        //使用AnnotationConfigApplicationContext可以实现基于Java的配置类加载Spring的应用上下文。避免使用application.xml进行配置。相比XML配置，更加便捷。
        AnnotationConfigApplicationContext context
                = new AnnotationConfigApplicationContext(Test.class);

        HandlerDispatcherServlet bean = context.getBean(HandlerDispatcherServlet.class);

        bean.handler(Cmd.REGISTER);
        bean.handler(Cmd.LOGIN);
        bean.handler(Cmd.LOGOUT);

        context.close();
    }
}
```





