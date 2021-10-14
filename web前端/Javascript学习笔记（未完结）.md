# Javascript学习笔记

# 第一部分 JavaScript核心

### JS正则表达式



# 第二部分 客户端JavaScript

## 一.Window对象（BOM）

Window对象是所有客户端JavaScript特性和API的主要接入点。它表示Web浏览器的一个窗口或窗体，并且可以用标识符window来引用它。

### 1.1 计时器

setTimeout（）和setInterval（）可以用来注册在指定的时间之后单次或重复调用的函数。因为它们都是客户端JavaScript中重要的全局函数，所以定义为Window对象的方法，但作为通用函数，其实不会对窗口做什么事情。Window对象的setTimeout（）方法用来实现一个函数在指定的毫秒数之后运行。setTimeout（）返回一个值，这个值可以传递给clearTimeout（）用于取消这个函数的执行。

**setInterval() 方法**

setInterval() 间隔指定的毫秒数不停地执行指定的代码

> window.setInterval("*javascript function*",*milliseconds*); //

window.setInterval() 方法可以不使用 window 前缀，直接使用函数 setInterval()。

setInterval() 第一个参数是函数（function）。第二个参数间隔的毫秒数

~~~
setInterval(function(){alert("Hello")},3000); //每三秒弹出 "hello" 
~~~

**clearInterval()** 

方法用于停止 setInterval() 方法执行的函数代码。

> window.clearInterval(*intervalVariable*); 

window.clearInterval()方法可以不使用window前缀，直接使用函数clearInterval()

要使用 clearInterval() 方法, 在创建计时方法时你必须使用全局变量：

> myVar=setInterval("*javascript function*",*milliseconds*);

~~~html
<p id="demo"></p>
<button onclick="myStopFunction()">停止</button>
<script>
var myVar=setInterval(function(){myTimer()},1000);
function myTimer(){
    var d=new Date();
    var t=d.toLocaleTimeString();
    document.getElementById("demo").innerHTML=t;
}
function myStopFunction(){
    clearInterval(myVar);
}
</script>
~~~

**setTimeout() 方法**

> myVar= window.setTimeout("*javascript function*", *milliseconds*);

setTimeout() 方法会返回某个值。在上面的语句中，值被储存在名为 myVar 的变量中。假如你希望取消这个 setTimeout()，你可以使用这个变量名来指定它。

setTimeout() 的第一个参数是含有 JavaScript 语句的字符串。这个语句可能诸如 "alert('5 seconds!')"，或者对函数的调用，诸如 alertMsg。

第二个参数指示从当前起多少毫秒后执行第一个参数。

提示：1000 毫秒等于一秒。

~~~
setTimeout(function(){alert("Hello")},3000);
~~~

**clearTimeout()** 

方法用于停止执行setTimeout()方法的函数代码。

~~~js
var myVar;
 
function myFunction()
{
    myVar=setTimeout(function(){alert("Hello")},3000);
}
 
function myStopFunction()
{
    clearTimeout(myVar);
}
~~~

### 1.2 浏览器定位和导航

Window对象的location属性引用的是Location对象，它表示该窗口中当前显示的文档的URL，并定义了方法来使窗口载入新的文档。

Document对象的location属性也引用到Location对象：

> window.location===document.location //总是返回true

Document对象也有一个URL属性，是文档首次载入后保存该文档的URL的静态字符串。如果定位到文档中的片段标识符（如#table-of-contents），Location对象会做相应的更新，而document.URL属性却不会改变。

**解析URL**

Window对象的location属性引用的是Location对象，它表示该窗口中当前显示的文档的URL。Location对象的href属性是一个字符串，后者包含URL的完整文本。Location对象的toString（）方法返回href属性的值，因此在会隐式调用toString（）的情况下，可以使用location代替location.href。



### 1.3 浏览历史

### 1.4 浏览器和屏蔽信息

### 1.5 对话框

### 1.6 错误处理

### 1.7 作为window对象属性的文档元素

### 1.8 多媒体和窗口

## 二.脚本化文档（DOM）

## 三.脚本化CSS

## 四.事件处理

## 五.脚本化HTTP

