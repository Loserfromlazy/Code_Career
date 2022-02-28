# Web安全防护学习笔记

转载请声明作者和出处！

本文如有错误，欢迎指正，感激不尽。

HTTP相关基础知识见HTTP学习笔记。

请不要攻击他人服务器或计算机系统，本文仅是对web安全的学习。

> 《刑法》
> 第二百八十五条　【非法侵入计算机信息系统罪;非法获取计算机信息系统数据、非法控制计算机信息系统罪;提供侵入、非法控制计算机信息系统程序、工具罪】违反国家规定，侵入国家事务、国防建设、尖端科学技术领域的计算机信息系统的，处三年以下有期徒刑或者拘役。
>
> 违反国家规定，侵入前款规定以外的计算机信息系统或者采用其他技术手段，获取该计算机信息系统中存储、处理或者传输的数据，或者对该计算机信息系统实施非法控制，情节严重的，处三年以下有期徒刑或者拘役，并处或者单处罚金;情节特别严重的，处三年以上七年以下有期徒刑，并处罚金。
>
> 提供专门用于侵入、非法控制计算机信息系统的程序、工具，或者明知他人实施侵入、非法控制计算机信息系统的违法犯罪行为而为其提供程序、工具，情节严重的，依照前款的规定处罚。
>
> 第二百八十六条　【破坏计算机信息系统罪】违反国家规定，对计算机信息系统功能进行删除、修改、增加、干扰，造成计算机信息系统不能正常运行，后果严重的，处五年以下有期徒刑或者拘役;后果特别严重的，处五年以上有期徒刑。
>
> 违反国家规定，对计算机信息系统中存储、处理或者传输的数据和应用程序进行删除、修改、增加的操作，后果严重的，依照前款的规定处罚。
>
> 故意制作、传播计算机病毒等破坏性程序，影响计算机系统正常运行，后果严重的，依照第一款的规定处罚。
>
> 《治安管理处罚法》
> 第二十九条 有下列行为之一的，处五日以下拘留；情节较重的，处五日以上十日以下拘留：
> (一)违反国家规定，侵入计算机信息系统，造成危害的；
> (二)违反国家规定，对计算机信息系统功能进行删除、修改、增加、干扰，造成计算机信息系统不能正常运行的；
> (三)违反国家规定，对计算机信息系统中存储、处理、传输的数据和应用程序进行删除、修改、增加的；
> (四)故意制作、传播计算机病毒等破坏性程序，影响计算机信息系统正常运行的。

# 一、跨站脚本攻击（`XSS`）

`XSS：Cross Site Scripting`,为了不和`CSS`混淆所以叫`XSS`。

恶意攻击者往Web页面里插入恶意Script代码，当用户浏览该页时，嵌入其中的Script代码就会被执行，从而达到恶意攻击用户的目的。在一开始这种攻击演示案例是跨域的所以叫跨站脚本。但是发展到今天，由于JavaScript的强大功能且基于网站前端应用的复杂化，是否跨域已经不再重要。但是由于历史原因，`XSS`名字被保存了下来。

`XSS`长期被列为客户端Web安全的头号大敌。因为`XSS`破坏力强大，且产生的场景复杂，难以一次性解决。现在针对不同场景的`XSS`,需要区分情景对待。

攻击原理：

Web应用程序混淆了用户提交的数据和JavaScript脚本的代码边界，导致浏览器把用户的输入当成JavaScript代码来执行。

## 1.1 攻击演示及问题

攻击演示：

服务和页面搭建一开始是使用`SpringBoot+vue`实现:

后台页面

WebDemo.java

```java
//简单替代数据库
private static List<String> names = new ArrayList<>();

@GetMapping("/addName")
private String addName(@RequestParam("name") String name) {
    System.out.println(name);
    names.add(name);

    return "zhanshi";
}

@GetMapping("/getName")
@ResponseBody
private List<String> getName() {
    return names;
}
```

前端页面

zhanshi.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="https://cdn.bootcss.com/vue/2.5.2/vue.min.js"></script>
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
</head>
<body>
<div id="app">
    <form action="/demo/addName" method="GET">
        <input name="name">
        <input type="submit" value="submit">
    </form>
    <table>
        <thead>
        <th>序号</th>
        <th>留言</th>
        </thead>
        <tbody>
        <tr v-for="(item, i) in names">
            <td v-text="i"></td>
            <td v-html="item"></td>
        </tr>
        </tbody>
    </table>
</div>
</body>

<script>
    new Vue({
        el: "#app",//表示当前vue对象接管了div区域
        data: {
            message: "hello world", //注意不要以分号结尾
            names:[]
        },
        mounted() {
            axios.get("/demo/getName")
                .then(response => {
                    //console.log(response.data)
                    this.names = response.data
                    console.log(this.names)
                })
                .catch(function (error) { //处理失败请求
                    console.log(error);
                });
        }
    })
</script>
</html>
```

在输入框中输入：`<img src='x' onerror='alert(1)'>`,点击提交按钮后会弹出alert。表明XSS攻击成功。

然而在我实验`<script>alert('hey!you are attacked');</script> `或`<script>window.location.href='http://www.baidu.com'</script>`

这种script代码时vue却不生效了，后来查阅资料发现 html5中innerHTML对有可能产生的安全问题进行了处理，原因如下图，即innerHTML会处理掉script标签：

![image-20220228124925200](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220228124925200.png)

所以这时我们用thymeleaf代替vue，使用thymeleaf的`th:utext`不会对script代码进行处理。代码如下：

```java
@GetMapping("/addNameThymeleaf")
private String addNameThymeleaf(@RequestParam("name") String name,Model model) {
    System.out.println(name);
    names.add(name);
    model.addAttribute("names",names);
    return "zhanshi1";
}
```

前端输入数据页面

webdemo1.html

```html
<body>
    <form action="/demo/addNameThymeleaf" method="GET">
        <input name="name">
        <input type="submit" value="submit">
    </form>
</body>
```

前端展示数据页面

zhanshi1.html

```html
<body>
<table>
    <thead>
    <th>序号X</th>
    <th>留言</th>
    </thead>
    <tbody>
    <tr th:each="item,stat : ${names}">
        <td th:text="${stat.index}"></td>
        <td th:utext="${item}"></td>
    </tr>
    </tbody>
</table>
</body>
```

这时实验以下XSS攻击均可成功：

~~~
恶意脚本植入
<img src='x' onerror='alert(1)'>
<script>alert('hey!you are attacked');</script>
流量劫持恶意跳转
<script>window.location.href='http://www.baidu.com'</script>
~~~

## 1.2 分类

XSS分为

- 存储型XSS
- 反射型XSS
- DOM型XSS

**存储型XSS**

存储型也称持久性XSS。攻击者上传的包含恶意JS脚本的信息被Web应用程序保存到了数据库，Web应用程序在生成新的页面时如果包含了这个恶意JS脚本，会导致访问该网页的浏览器解析执行该恶意脚本。这种攻击类型一般常见于博客、论坛等网站中。

存储型是最危险的一种跨站脚本，比反射型XSS和Dom型XSS都更有隐蔽性，任何允许用户存储数据的web程序都可能存在存储型XSS漏洞。若某个页面遭到攻击，所有访问该页面的用户也会被XSS攻击。

攻击步骤：

![XSS攻击流程20220228](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/XSS%E6%94%BB%E5%87%BB%E6%B5%81%E7%A8%8B20220228.png)

存储型 XSS(又被称为持久性XSS)攻击常见于带有用户保存数据的网站功能，如论坛发帖、商品评论、用户私信、留言等功能。

常见的存储型XSS攻击：

1. 简单攻击

   `<script>alert('hello');</script>`

2. 获取cookie信息

   在上面1.1的例子的Controller中加入如下代码：

   ```java
   @GetMapping("/login")
   private void login(HttpServletResponse response){
       long now =System.currentTimeMillis();
       long exp = now +1000*60*30;
       JwtBuilder builder = Jwts.builder().setId("111")
                   .setSubject("222")
               //设置签发时间
                   .setIssuedAt(new Date())
               //设置签名密钥
                   .signWith(SignatureAlgorithm.HS256,"loserfromlazy")
                   .setExpiration(new Date(exp))
                   .claim("roles","admin");
           String token = builder.compact();
       Cookie cookie=new Cookie("cookie",token);
       cookie.setPath("/");
       cookie.setMaxAge(24*60*60);
       response.setHeader("Access-Control-Allow-Credentials", "true");
       response.addCookie(cookie);
   }
   ```

   在zhanshi1.html中加入如下代码：

   ```html
   <script type="text/javascript" src="http://code.jquery.com/jquery-latest.js"></script>
   <script>
       $(function(){
           $.ajax({
               url:'/demo/login',
               type:'get',
               success:function(data){
                   console.log(data);
               },
               error:function(){
                   console.log('请求出错！');
               }
           })
       });
   </script>
   ```

   这时我们在输入框输入`<script>alert(document.cookie)</script>`,然后刷新zhanshi1.html接可以看到我们的存入cookie的token：

   ![image-20220228160436231](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/picgo/image-20220228160436231.png)

   

3. 窃取cookie

   

4. 绕过浏览器同源策略

   

5. 通过cookie入侵

   







