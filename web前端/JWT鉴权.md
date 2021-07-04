# JWT鉴权学习笔记

# 一、概述

JSON Web Token(JWT)是一个非常轻巧的规范，这个规范允许我们使用JWT在用户和服务器之间传递安全可靠的消息。

JWT实际上是一个字符串，它由三部分组成，头部、载荷与签名。

**头部**

头部用来描述该JWT最基本的信息，例如其类型以及签名所用的算法，也可以被表示成json对象。

~~~json
{"typ":"JWT","alg","HS256"}
~~~

在头部指明了签名算法是HS256，进行BASE64编码，编码后如下

~~~
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
~~~

**载荷** playload

载荷就是存放有效信息的地方，这个有效信息包含三个部分

1. 标准中注册的声明

   ~~~
   iss：jwt签发者
   sub：jwt所面向的用户
   aud：接受jwt的一方
   exp：jwt的过期时间，这个过期时间必须大于签发时间
   nbf：定义在什么时间之前，jwt是不可用的
   iat：jwt的签发时间
   jti：jwt的唯一身份表示，主要用来作为一次性token
   ~~~

2. 公共的声明

   公共的声明可以添加任何信息，一般添加用户的相关信息或其他业务所需要的必要信息。但不建议添加敏感信息。

3. 私有的声明

   私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。这个指的就是自定义的claim。这些claim跟JWT标准规定的claim区别在于：JWT规定的claim，JWT的接收方在拿到JWT之后，都知道怎么对这些标准的claim进行验证(还不知道是否能够验证)；而private claims不会验证，除非明确告诉接收方要对这些claim进行验证以及规则才行。

定义一个playload进行base64编码得到jwt的第二部分

~~~json
{"sub":"123456789","name":"John","admin":true}
~~~

~~~
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
~~~

**签证**

jwt第三部分是一个签证信息，这个信息由三部分组成

> header(base64后)
>
> playload(base64后)
>
> secret

这个部分需要base64加密后的header和base64加密后的playload使用.连接组成的字符串，然后通过header中声明的加密方式进行加盐secret组合加密，然后就构成了jwt的第三部分。

将这三部分用.连接成一个完整的字符串,构成了最终的jwt:

~~~
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
~~~

secret是保存在服务器端的，jwt的签发生成也是在服务器端的，secret就是用来进行jwt的签发和jwt的验证，所以，它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。

# 二、Java实现

JJWT是一个提供端到端的JWT创建和验证的Java库。永远免费和开源(Apache License，版本2.0)，JJWT很容易使用和理解。它被设计成一个以建筑为中心的流畅界面，隐藏了它的大部分复杂性。

## 2.1 token的创建

导入依赖

~~~xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.6.0</version>
</dependency>
~~~

创建CreateJwtTest类

~~~java
public class CreateJwtTest {
    public static void main(String[] args) {
        JwtBuilder builder = Jwts.builder().setId("666")
                .setSubject("小李")
                .setIssuedAt(new Date())//设置签发时间
                .signWith(SignatureAlgorithm.HS256,"loserfromlazy");//设置签名密钥
        System.out.println(builder.compact());
    }
}
~~~

## 2.2 token的解析

我们刚才已经创建了token ，在web应用中这个操作是由服务端进行然后发给客户端，客户端在下次向服务端发送请求时需要携带这个token（这就好像是拿着一张门票一样），那服务端接到这个token 应该解析出token中的信息（例如用户id）,根据这些信息查询数据库返回相应的结果。

创建ParseJwtTest类

~~~java
public class ParseJwtTest {
    public static void main(String[] args) {
        String token ="eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI2NjYiLCJzdWIiOiLlsI_mnY4iLCJpYXQiOjE2MjUyOTIzMjUsImV4cCI6MTYyNTI5MjM4NH0.DwetWOWkrZvdAVZ29rJDBh_Ojs25Qii9sMnm2flmp-c";
        Claims claims = Jwts.parser().setSigningKey("loserfromlazy").parseClaimsJws(token).getBody();
        System.out.println(claims.getId());
        System.out.println(claims.getSubject());
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");

        System.out.println("签发时间"+simpleDateFormat.format(claims.getIssuedAt()));
        System.out.println("当前时间"+simpleDateFormat.format(new Date()));
    }
}
~~~

> 如果将token稍微修改就会发现报错，所以解析token也是验证token

## 2.3 token过期验证

一个token并不是永久有效的，所以过期时间校验很有必要
创建CreateJwtTest2类

~~~java
public class CreateJwtTest2 {
    public static void main(String[] args) {
        //设置过期时间一分钟
        long now =System.currentTimeMillis();
        long exp = now +1000*60;
        JwtBuilder builder = Jwts.builder().setId("666")
                .setSubject("小李")
                .setIssuedAt(new Date())//设置签发时间
                .signWith(SignatureAlgorithm.HS256,"loserfromlazy")//设置签名密钥
                .setExpiration(new Date(exp));//设置过期时间
        System.out.println(builder.compact());
    }
}
~~~

修改ParseJwtTest类

~~~java
public class ParseJwtTest {
    public static void main(String[] args) {
        String token ="eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI2NjYiLCJzdWIiOiLlsI_mnY4iLCJpYXQiOjE2MjUyOTIzMjUsImV4cCI6MTYyNTI5MjM4NH0.DwetWOWkrZvdAVZ29rJDBh_Ojs25Qii9sMnm2flmp-c";
        Claims claims = Jwts.parser().setSigningKey("loserfromlazy").parseClaimsJws(token).getBody();
        System.out.println(claims.getId());
        System.out.println(claims.getSubject());
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");

        System.out.println("签发时间"+simpleDateFormat.format(claims.getIssuedAt()));
        System.out.println("过期时间"+simpleDateFormat.format(claims.getExpiration()));
        System.out.println("当前时间"+simpleDateFormat.format(new Date()));
    }
}
~~~

如果是过期token会报io.jsonwebtoken.ExpiredJwtException异常。

## 2.4 自定义Claims

上述代码只存储了id和subject两个值，若想存更多则需要自定义claims

创建CreateJwtTest3类

~~~java
public class CreateJwtTest3 {
    public static void main(String[] args) {
        //设置过期时间一分钟
        long now =System.currentTimeMillis();
        long exp = now +1000*60;
        JwtBuilder builder = Jwts.builder().setId("666")
                .setSubject("小李")
                .setIssuedAt(new Date())//设置签发时间
                .signWith(SignatureAlgorithm.HS256,"loserfromlazy")//设置签名密钥
                .setExpiration(new Date(exp))
                .claim("roles","admin")
                .claim("logo","logo.jpg");
        System.out.println(builder.compact());
    }
}

~~~

修改ParseJwtTest类

~~~java
public class ParseJwtTest {
    public static void main(String[] args) {
        String token ="eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI2NjYiLCJzdWIiOiLlsI_mnY4iLCJpYXQiOjE2MjUzNjU5MzksImV4cCI6MTYyNTM2NTk5OSwicm9sZXMiOiJhZG1pbiIsImxvZ28iOiJsb2dvLmpwZyJ9.gWOG0m5e5MrOHaChR_J9xRQ5z01bv6eveot_6A7Jkc0";
        Claims claims = Jwts.parser().setSigningKey("loserfromlazy").parseClaimsJws(token).getBody();
        System.out.println(claims.getId());
        System.out.println(claims.getSubject());
        System.out.println(claims.get("roles"));
        System.out.println(claims.get("logo"));
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");

        System.out.println("签发时间"+simpleDateFormat.format(claims.getIssuedAt()));
        System.out.println("过期时间"+simpleDateFormat.format(claims.getExpiration()));
        System.out.println("当前时间"+simpleDateFormat.format(new Date()));
    }
}
~~~

# 三、场景实例

## 3.1 创建工具类

~~~java
@Component
@ConfigurationProperties(prefix = "jwt.config")//用于导入配置文件信息
public class JwtUtil {
    private String key;//密钥

    private long ttl;//过期时间

    public String getKey() {
        return key;
    }

    public void setKey(String key) {
        this.key = key;
    }

    public long getTtl() {
        return ttl;
    }

    public void setTtl(long ttl) {
        this.ttl = ttl;
    }

    /**
     * 创建token
     * @param id
     * @param subject
     * @param roles
     * @return
     */
    public String createJwt(String id ,String subject,String roles){
        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);
        JwtBuilder builder = Jwts.builder().setId(id)
                .setSubject(subject)
                .setIssuedAt(now)
                .signWith(SignatureAlgorithm.HS256,key).claim("roles",roles);
        if (ttl>0){
            builder.setExpiration(new Date(nowMillis+ttl));
        }
        return builder.compact();
    }
    
    public Claims parseJWT(String jwtStr){
        return Jwts.parser().setSigningKey(key)
                .parseClaimsJws(jwtStr)
                .getBody();
    }
}
~~~

> PS:如果在使用@ConfigurationProperties(prefix = "xxx")注解报错Springboot configuration annotation processor cannot find class错误时需要导入依赖
>
> ~~~xml
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-configuration-processor</artifactId>
>     <optional>true</optional>
> </dependency>
> ~~~

application.yml

~~~yml
jwt:
  config:
    key: loserfromlazy
    ttl: 360000
~~~

## 3.2 用户/管理员登录后签发token

在启动类加入bean

~~~java
@Bean
public JwtUtil jwtUtil(){
    return new JwtUtil();
}
~~~

登录签发token

~~~java
@Controller
public class LoginController {

    @Autowired
    private JwtUtil jwtUtil;

    public Result login(@RequestBody Map<String,String> loginMap){
        Admin admin = adminService.findByLoginNameAndPwd(loginMap.get("loginname"),loginMap.get("password"));
        if(admin != null){
            //生成token
            String token jwtUtil.createJWT(admin.getId(),admin.getLoginname(),"admin");
            Map map = new HashMap();
            map.put("token",token);
            map.put("name",admin.getLoginname());
            return new Result(true,StatusCode.OK,"登录成功",map);
        }else{
            return new Result(false,StatusCode.LOGINERROR,"用户名或密码错误");
        }
    }
}
~~~

预期结果：

~~~json
"flag": true,
"code": 20000,
"message": "登录成功",
"data": {
    "token": ".......",
    "name": "admin"
}
~~~

## 3.3 功能鉴权

需求：用户或管理员在执行某一动作时必须有权限，否则不能执行

前后端约定：前端请求微服务时需要添加头信息Authorization，内容为Brarer+空格+token

> 以vue为例
>
> ~~~js
> // 添加请求拦截器
> axios.interceptors.request.use(function (config) {
>   // 在发送请求之前做些什么
>   // 判断是否存在token,如果存在将每个页面header添加token
>   if (store.state.token) {
>     config.headers.common['Authorization'] = "Bearer " + store.state.token
>   }
>   return config
> }, function (error) {
>   router.push('/login')
>   return Promise.reject(error)
> })
> ~~~

后端代码：

修改Controller，以删除为例，判断请求中的头信息，提取token并验证权限

~~~java
@Autowierd
private HttpServletResquest request;

@RequestMapping(value="/{id}",method =RequestMethod.DELETE)
public Result delete(@PathVariable String id){
    String authHeader = request.getHeader("Authorization");//获取头部信息
    if(authHeader==null){
        return new Result(false,StatusCode.ACCESSERROR,"权限不足");
    }
    if(!authHeader.startWith("Bearer ")){
        return new Result(false,StatusCode.ACCESSERROR,"权限不足");
    }
    String token = authHeader.substring(7);//提取token
    Claims claims = jwtUtils.parseJWT(token);
    if(claims==null){
        return new Result(false,StatusCode.ACCESSERROR,"权限不足");
    }
    if(!"admin".equals(claims.get("roles"))){
        return new Result(false,StatusCode.ACCESSERROR,"权限不足");
    }
    
    userService.deleteById(id);
    return new Result(true,StatusCode.OK,"删除成功")
}
~~~

## 3.4 使用拦截器实现token鉴权

如果每一个方法写一段代码，太过冗余，所以使用拦截器实现

### 3.4.1 添加拦截器

Spring为我们提供了org.springframework.web.servlet.handler.HandlerInterceptorAdapter这个适配器，继承此类可以实现自己的拦截器。这有三个方法，分别实现预处理、后处理（调用了Service并返回ModelAndView，但未进行页面渲染）、返回处理（已经渲染了页面）

在preHandle中，可以实现编码、安全控制等处理

在postHandle中，有机会修改ModelAndView

在afterCompletion中，可以根据ex是否为null判断是否发生异常，进行日志记录

1.创建拦截器类。

~~~java
@Component
public class JwtFilter extends HandlerInterceptorAdapter{
    @Autowired
    private JwtUtil jwtUtil;
    
    @Override
    public boolean preHandle(HttpServletRequest request,HttpServletResponse response,Object handler)throws Exception{
        Sysytem.out.println("经过了拦截器");
        return true;
    }
}
~~~

2.配置拦截器类

~~~java
@Configuration
public class ApplicationConfig extends WebMvcConfigurationSupport{
    @Autowired
	private JwtFilter jwtFilter;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry){
        registry.addInterceptor(jwtFilter)
            .addPathPatterns("/**")
            .excludePathPatterns("/**/login");
    }
}
~~~

### 3.4.2 拦截器验证token

修改JwtFilter

~~~java
@Component
public class JwtFilter extends HandlerInterceptorAdapter{
    @Autowired
    private JwtUtil jwtUtil;
    
    @Override
    public boolean preHandle(HttpServletRequest request,HttpServletResponse response,Object handler)throws Exception{
        Sysytem.out.println("经过了拦截器");
        final String authHeader = request.getHeader("Authorization");
        if(authHeader != null && authHeader.startWith("Bearer ")){
            final String token =authHeader.subString(7);
            Claims claims =jwtUtil.parseJWT(token);
            if("admin".equals(claims.get("roles"))){
                request.setAttribute("admin_claims",claims);
            }
            if("user".equals(claims.get("roles"))){
                request.setAttribute("user_claims",claims);
            }
        }
        return true;
    }
}
~~~

修改controller

~~~java
@Autowierd
private HttpServletResquest request;

@RequestMapping(value="/{id}",method =RequestMethod.DELETE)
public Result delete(@PathVariable String id){
    Claims claims = (Claims) request.getAttribute("admin_claims");
    if(claims==null){
        return new Result(true,StatusCode.ACCESSERROR,"无权访问");
    }
    u
    return new Result(true,StatusCode.OK,"删除成功")
}
~~~













