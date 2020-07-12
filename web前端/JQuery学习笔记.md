# JQuery学习笔记

## 一.概述

JQuery的主旨是：用更少的代码，实现更多的功能。

### 1.基本功能

访问和操作DOM

控制页面样式

对页面事件的处理

大量插件在页面中的运用

与Ajax技术的完美结合

### 2.简单应用

#### 2.1访问DOM

DOM(Document Object Model，文本对象模型)，每一个页面都是一个DOM对象，通过传统的JS访问页面中的元素

~~~javascript
<div id="divTmp">lalalal </div>
JS
var tdiv=document.getElementById("divTmp");//获取DOM对象
var cdiv=tdiv.innerHTML;//获取tdiv对象中的内容

JQuery
var tdiv=$("#divTmp")//获取JQuery对象
var cdiv=tdiv.html();
~~~

#### 2.2控制CSS

通过JQuery中的toggleClass(ClassName)方法实现页面样式的动态切换

### 3.jQuery语法

jQuery 语法是通过选取 HTML 元素，并对选取的元素执行某些操作。

基础语法： **$(selector).action()**

- 美元符号定义 jQuery
- 选择符（selector）"查询"和"查找" HTML 元素
- jQuery 的 action() 执行对元素的操作

Query 入口函数:

```
$(document).ready(function(){
    // 执行代码
});
或者
$(function(){
    // 执行代码
});
```

JavaScript 入口函数:

```
window.onload = function () {
    // 执行代码
}
```

jQuery 入口函数与 JavaScript 入口函数的区别：

-  jQuery 的入口函数是在 html 所有标签(DOM)都加载之后，就会去执行。
-  JavaScript 的 window.onload 事件是等到所有内容，包括外部图片之类的文件加载完后，才会执行。

![img](https://www.runoob.com/wp-content/uploads/2019/05/20171231003829544.jpeg)

## 二.JQuery选择器

jQuery选择器继承了CSS与Path语言的部分语法，允许通过标签名、属性名对DOM进行快速的选择

jQuery的优势：代码更简单、完善的检测机制

### 元素选择器

jQuery 元素选择器基于元素名选取元素。

在页面中选取所有 <p> 元素:

$("p")

**实例**

用户点击按钮后，所有 <p> 元素都隐藏：

~~~js
$(document).ready(function(){
  $("button").click(function(){
    $("p").hide();
  });
});
~~~

### #id 选择器

jQuery #id 选择器通过 HTML 元素的 id 属性选取指定的元素。

页面中元素的 id 应该是唯一的，所以您要在页面中选取唯一的元素需要通过 #id 选择器。

通过 id 选取元素语法如下：

$("#test")

**实例**

当用户点击按钮后，有 id="test" 属性的元素将被隐藏：

~~~javascript
$(document).ready(function(){
  $("button").click(function(){
    $("#test").hide();
  });
});
~~~

### .class 选择器

jQuery 类选择器可以通过指定的 class 查找元素。

语法如下：

$(".test")

**实例**

用户点击按钮后所有带有 class="test" 属性的元素都隐藏：

~~~javascript
$(document).ready(function(){
  $("button").click(function(){
    $(".test").hide();
  });
});
~~~

### 更多实例

| 语法                     | 描述                                                    |
| :----------------------- | :------------------------------------------------------ |
| $("*")                   | 选取所有元素                                            |
| $(this)                  | 选取当前 HTML 元素                                      |
| $("p.intro")             | 选取 class 为 intro 的 <p> 元素                         |
| $("p:first")             | 选取第一个 <p> 元素                                     |
| $("ul li:first")         | 选取第一个 <ul> 元素的第一个 <li> 元素                  |
| $("ul li:first-child")   | 选取每个 <ul> 元素的第一个 <li> 元素                    |
| $("[href]")              | 选取带有 href 属性的元素                                |
| $("a[target='_blank']")  | 选取所有 target 属性值等于 "_blank" 的 <a> 元素         |
| $("a[target!='_blank']") | 选取所有 target 属性值不等于 "_blank" 的 <a> 元素       |
| $(":button")             | 选取所有 type="button" 的 <input> 元素 和 <button> 元素 |
| $("tr:even")             | 选取偶数位置的 <tr> 元素                                |
| $("tr:odd")              | 选取奇数位置的 <tr> 元素                                |

### 其他选择器

~~~javascript
$("#id", ".class")  复合选择器
$(div p span)       层级选择器 //div下的p元素中的span元素
$(div>p)            父子选择器 //div下的所有p元素
$(div+p)            相邻元素选择器 //div后面的p元素(仅一个p)
$(div~p)            兄弟选择器  //div后面的所有p元素(同级别)
$(.p:last)          类选择器 加 过滤选择器  第一个和最后一个（first 或者 last）
$("#mytable td:odd")      层级选择 加 过滤选择器 奇偶（odd 或者 even）
$("div p:eq(2)")    索引选择器 div下的第三个p元素（索引是从0开始）
$("a[href='www.baidu.com']")  属性选择器
$("p:contains(test)")        // 内容过滤选择器，包含text内容的p元素
$(":emtyp")        //内容过滤选择器，所有空标签（不包含子标签和内容的标签）parent 相反
$(":hidden")       //所有隐藏元素 visible 
$("input:enabled") //选取所有启用的表单元素
$(":disabled")     //所有不可用的元素
$("input:checked") //获取所有选中的复选框单选按钮等
$("select option:selected") //获取选中的选项元素
~~~

## 三.jQuery事件

### 3.1事件机制

严格来说，事件在触发后被分为两个阶段，一个是捕获（Capture），另一个则是冒泡（Bubbling）；但有些遗憾的是，大多数浏览器并不是都支持捕获阶段，jQuery也不支持。因此在事件触发后，往往执行冒泡过程。所谓的冒泡其实质就是事件执行中的顺序。

所以事件冒泡的走向是由子节点向父节点去触发同名事件。

阻止事件冒泡

~~~javascript
 $("body").click(function(event){
            alert("我是body");
            event.stopPropagation();    //  阻止事件冒泡
        });
~~~

### 3.2页面载入事件

详见 一.3

### 3.3事件绑定

在上面我们采用

~~~javascript
$(function(){
	$("#btnShow").click(function(){
		//执行
	})
})
~~~

绑定事件

除了上面的写法外，jquery还可以用bind()方法进行事件的绑定，其语法格式如下

> bing(type ,[data] ,fn)

其中参数type为一个或多个类型的字符串，如“click"或“change”，也可以自定义类型；可以被参数type调用的类型包括blur、focus、load、resize、scroll、unload、click、dblclick、mousedown、mouseup、mousemove、mouseover、mouseout、mouseenter、mouseleave、change、select、submit、keydown、keypress、keyup、error。

参数data是作为event.data属性值传递给事件对象的额外数据对象。

参数fn是绑定到每个选择元素的事件中的处理函数。示例：

~~~javascript
$(function(){
	$("#btnBind").bind("click",function(){
		$(this).attr("disabled","disabled");//按钮不可用
	})
})
~~~

如果要绑定多个事件，可以用空格隔开

~~~javascript
$(function(){
	$("#btnBind").bind("click mouseout",function(){
		$(this).attr("disabled","disabled");//按钮不可用
	})
})
~~~

用映射绑定不同的事件

```javascript
$(selector).bind({event:function, event:function, ...})
```

~~~javascript
$(function(){
	$(".txt").bind({
        focus:function{
        $("#divTip").show().html("执行focus事件");
    },
  		change:function(){
    		$("#divTip").show().html("执行change事件");
	}
    })
})
~~~

### 3.4事件切换

#### 3.4.1hover()

调用jQuery中的hover()方法可以使元素在鼠标悬停与鼠标移出的事件中进行切换，该方法在实现运用中，也可以通过jQuery中的事件mouseenter与mouseleave进行替换。下列代码是等价的。

hover()功能是当鼠标移动到所选的元素上面时，执行指定的第一个函数；当鼠标移出这个元素时，执行指定的第二个函数，其语法格式如下：

> hover(over,out)

参数over为鼠标移到元素时触发的函数，参数out为鼠标移出元素时触发的函数。下面举例说明。

~~~javascript
$("a").hover(function(){
    //code1    
},function(){
 	//code2   
})
~~~

~~~javascript
$("a").mouseenter(function(){
	//code1
})
$("a").mouseleave(function(){
    //code2
})
~~~

#### 3.4.2toggle()

在toggle()方法中，可以依次调用N个指定的函数，直到最后一个函数，然后重复对这些函数轮番调用。toggle()方法的功能是每次单击后依次调用函数，请注意到“依次”这两个字，说明该方法在调用函数时并非随机或指定调用，而是通过函数设置的前后顺序进行调用，其调用的语法格式如下：

> toggle(fn,fn2,fn3,......)

其中fn，fn2......为被单击时依次调用的函数 eg：单击变换图片

~~~javascript
$(function(){
    $("img").toggle(
    	function(){
            $("img").attr("src","/xxx.jpg");
            $("img").attr("title",this.src);
        },function(){
            $("img").attr("src","/xxx.jpg");
            $("img").attr("title",this.src);
        }
    )
})
~~~

### 3.5常用事件

常见 DOM 事件：

| 鼠标事件   | 键盘事件 | 表单事件 | 文档/窗口事件 |
| :--------- | :------- | :------- | :------------ |
| click      | keypress | submit   | load          |
| dblclick   | keydown  | change   | resize        |
| mouseenter | keyup    | focus    | scroll        |
| mouseleave |          | blur     | unload        |
| hover      |          |          |               |

#### click()

click() 方法是当按钮点击事件被触发时会调用一个函数。

该函数在用户点击 HTML 元素时执行。

在下面的实例中，当点击事件在某个 <p> 元素上触发时，隐藏当前的 <p> 元素：

~~~
$("p").click(function(){
  $(this).hide();
});
~~~

#### dblclick()

当双击元素时，会发生 dblclick 事件。

dblclick() 方法触发 dblclick 事件，或规定当发生 dblclick 事件时运行的函数：

~~~
$("p").dblclick(function(){
  $(this).hide();
});
~~~

#### mouseenter()

当鼠标指针穿过元素时，会发生 mouseenter 事件。

mouseenter() 方法触发 mouseenter 事件，或规定当发生 mouseenter 事件时运行的函数：

~~~
$("#p1").mouseenter(function(){
    alert('您的鼠标移到了 id="p1" 的元素上!');
});
~~~

#### mouseleave()

当鼠标指针离开元素时，会发生 mouseleave 事件。

mouseleave() 方法触发 mouseleave 事件，或规定当发生 mouseleave 事件时运行的函数：

~~~
$("#p1").mouseleave(function(){
    alert("再见，您的鼠标离开了该段落。");
});
~~~

#### mousedown()

当鼠标指针移动到元素上方，并按下鼠标按键时，会发生 mousedown 事件。

mousedown() 方法触发 mousedown 事件，或规定当发生 mousedown 事件时运行的函数：

~~~
$("#p1").mousedown(function(){
    alert("鼠标在该段落上按下！");
});
~~~

#### mouseup()

当在元素上松开鼠标按钮时，会发生 mouseup 事件。

mouseup() 方法触发 mouseup 事件，或规定当发生 mouseup 事件时运行的函数：

~~~
$("#p1").mouseup(function(){
    alert("鼠标在段落上松开。");
});
~~~

#### focus()

当元素获得焦点时，发生 focus 事件。

当通过鼠标点击选中元素或通过 tab 键定位到元素时，该元素就会获得焦点。

focus() 方法触发 focus 事件，或规定当发生 focus 事件时运行的函数：

~~~
$("input").focus(function(){
  $(this).css("background-color","#cccccc");
});
~~~

#### blur()

当元素失去焦点时，发生 blur 事件。

blur() 方法触发 blur 事件，或规定当发生 blur 事件时运行的函数：

~~~
$("input").blur(function(){
  $(this).css("background-color","#ffffff");
});
~~~

#### keypress,keydown,keyup

-  1.keydown：在键盘上按下某键时发生,一直按着则会不断触发（opera浏览器除外）, 它返回的是键盘代码;
-  2.keypress：在键盘上按下一个按键，并产生一个字符时发生, 返回ASCII码。注意: **shift、alt、ctrl**等键按下并不会产生字符，所以监听无效 ,换句话说, 只有按下能在屏幕上输出字符的按键时keypress事件才会触发。若一直按着某按键则会不断触发。
-  3.keyup：用户松开某一个按键时触发, 与keydown相对, 返回键盘代码.

案例1:获取按键代码或字符的ASCII码

```
$(window).keydown( function(event){
   // 通过event.which可以拿到按键代码.  如果是keypress事件中,则拿到ASCII码.
} );
```

案例2:传递数据给事件处理函数

语法:

```
jQueryObject.keypress( [[ data ,]  handler ] );
```

-  data: 通过event.data传递给事件处理函数的任意数据;
-  handler: 指定的事件处理函数;

举例:

```
// 只允许按下的字母键生效, 65~90是所有大写字母的键盘代码范围.
var validKeys = { start: 65, end: 90  };
$("#keys").keypress( validKeys, function(event){
    var keys = event.data;  //拿到validKeys对象.
    return event.which >= keys.start && event.which <= keys.end;
} );
```

## 四.jQuery动画

## 五.jQuery Ajax

Ajax是Asynchronous JavaScript and XML的缩写，其核心是通过XMLHttpRequest对象，以一种异步的方式，向服务器发送数据请求，并通过该对象接收请求返回的数据，从而完成人机交互的数据操作。这种利用Ajax技术进行的数据交互并不局限于Web动态页面，在普通的静态HTML页面中同样可以实现，因此，在这样的背景下，Ajax技术在页面开发中得以广泛使用。在jQuery中，同样有大量的函数与方法为Ajax技术提供支持。

### 5.1加载异步数据

#### 5.1.1传统js方法

~~~javascript
var objXmlHttp =null;
function CreateXMLHTTP(){
    //根据浏览器不同，返回实体对象
    if(window.ActiveXObject){
        objXmlHttp=new ActiveXObject("Microsoft.XMLHTTP");
    }
    else{
        if(window.XMLHttpRequest){
            objXmlHttp=new XMLHttpRequest();
        }
        else{
            alert("初始化XMLHTTP错误");
        }
    }
}
function GetSendData(){
    //初始化内容
    document.getElementById("divTip").innerHTML="<img alt='' title='正在加载中' src='/xxx.jpg'/>";
    //设置发送地址变量并赋初值
    var strSendUrl="b.html?date="+Date();
    //实例化
    CreateXMLHTTP();
    //open方法初始化
    objXmlHttp.open("GET",strSendUrl,true);
    objXmlHttp.onreadystatechange=function(){
        //回调事件函数
        if(objXmlHttp.readyState==4){//如果请求完成加载
            if(objXmlHttp.status==200){//若果响应成功
                //获取显示数据
                document.getElementById("divTip").innerHTML=objXmlHttp.responseText;
            }
        }
        objXmlHttp.send(null);
    }
}
~~~

#### 5.1.2jQuery中的load()方法

在传统的JavaScript中，使用XMLHttpRequest对象异步加载数据；而在jQuery中，使用load()方法可以轻松实现获取异步数据的功能。其调用的语法格式为：

> load(url,[data],[callback])

其中参数url为被加载的页面地址，可选项[data]参数表示发送到服务器的数据，其格式为key/value。另一个可选项[callback]参数表示加载成功后，返回至加载页的回调函数。下面举例说明。load()方法实现异步获取数据

~~~javascript
$(function(){
	$("Button1").click(function(){
        $("#divTip").load("b.thml");
    })
})
~~~

说明 在load ()方法中，参数url还可以用于过滤页面中的某类别的元素，如代码“$("#divTip").load("b.html .divContent")”，则表示获取页面b.html中类别名为“ivContent”的全部元素。

#### 5.1.3jQuery中全局函数getJSON()

虽然使用load()方法可以很快地加载数据到页面中，但有时需要对获取的数据进行处理，如果将用load()方法获取的内容进行遍历，也可以进行数据的处理，但这样获取的内容必须先插入页面中，然后才能进行，因此，执行的效率不高。

为了解决这个问题，我们采用将要获取的数据另存为一种轻量极的数据交互格式，即JSON文件格式，由于这种格式很方便计算机的读取，因而颇受开发者的青睐。在jQuery中，专门有一个全局函数getJSON()，用于调用JSON格式的数据，其调用的语法格式为：

> $.getJSON(url,[data],[callback])

参数url表示请求的地址，可选项[data]参数表示发送到服务器的数据，其格式为key/value。另外一个可选项[callback]参数表示加载成功时执行的回调函数。下面举例说明。全局函数getJSON()实现异步获取数据

~~~javascript
$(function(){
    $("#Button1").click(function(){//按钮单击事件
        $.getJSON("UserInfo.json",function(data){
            $("divTip").empty();//清空标记中的内容
            var strHTML="";
            //遍历获取数据
            $.each(data,function(InfoIndex,Info){
                strHtML+="姓名"+Info["name"]+"<br>";
                strHtML+="性别"+Info["sex"]+"<br>";
                strHtML+="邮箱"+Info["email"]+"<br>";
            })
            $("#divTip").html(strHTML);
        })
    })
})
~~~

UserInfo.json

~~~json
[
    {
        "name":"陶宏达",
        "sex":"man",
        "email":"xxx"
    },
    {
        "name":"李登辉",
        "sex":"woman",
        "email":"xxx"
    }
]
~~~

#### 5.1.4jQuery中的全局函数getScript()

在jQuery中，除通过全局函数getJSON获取.json格式的文件内容外，还可以通过另外一个全局函数getScript()获取.js文件的内容。其实，在页面中获取.js文件的内容有很多方法，如下列代码：

~~~javascript
<script type="text/javascript" src="xxx/xxx.js"></script>

动态设置：
$("<script type="text/javascript" src="xxx/xxx.js"></script>").appendTo("head");
~~~

但这样的调用方法并不是最理想的。在jQuery中，通过全局函数getScript()加载.js文件后，不仅可以像加载页面片段一样轻松地注入脚本，而且所注入的脚本自动执行，大大提高了页面的执行效率。函数getScript()的调用格式如下所示：

> $.getScript(url ,[callback])

参数url为等待加载的JS文件地址，可选项[callback]参数表示加载成功时执行的回调函数。

#### 5.1.5jQuery中异步加载XML文档

在前几节中，我们通过jQuery中的各种全局函数，实现了不同格式数据的异步加载，如html、json、js格式数据。在日常的页面开发中，有时也会遇到使用XML文档保存数据的情况，对于这种格式的数据，jQuery中使用全局函数$.get()进行访问，其调用的语法格式为：

> $.get(url,[data],[callback],[type])

其中参数url表示等待加载的数据地址，可选项[data]参数表示发送到服务器的数据，其格式为key/value，可选项[callback]参数表示加载成功时执行的回调函数，可选项[type]参数表示返回数据的格式，如html、xml、js、json、text等。下面举例说明。

~~~javascript
$(function(){
    $("#Button1").click(function(){//按钮单击事件
        $.get("UserInfo.xml",function(data){
            $("divTip").empty();//清空标记中的内容
            var strHTML="";
            //遍历获取数据
           $(data).find("User").each(function(){
               var $strUser=$(this);
               strHTML+="姓名"+$strUser.find("name").text()+"<br>";
               strHTML+="性别"+$strUser.find("sex").text()+"<br>";
               strHTML+="邮箱"+$strUser.find("email").text()+"<br>";
           })
            $("#divTip").html(strHTML);
        })
    })
})
~~~

### 5.2请求服务器数据

#### 5.2.1$.get()

$.get()其调用的语法格式为：

> $.get(url,[data],[callback],[type])

在上面的6.1.5小节中，通过调用全局函数$.get()，实现了XML文档的加载。除加载数据外，$.get()函数还可以实现数据的请求。下面通过一个示例介绍$.get()函数带参请求服务器中的数据。

~~~javascript
$(function(){
    $("#Button1").click(function(){//按钮单击事件
        $.get("UserInfo.aspx",
              {name:encodeURI($("#txtname").val())},
              function(data){
            	$("#divTip").empty().html(data);
            
        })
    })
})
~~~

UserInfo.aspx

~~~asp
//....处理数据
Response.write(strHTML);
~~~

**如果参数的值是中文格式，必须通过使用encodeURI()进行转码，当然，在客户端接收时，使用decodeURI()进行解码即可。**

#### 5.2.2$.post()

$.post()也是带参数向服务器发出数据请求。全局函数$.post()与$.get()在本质上没有太大的区别，所不同的是，GET方式不适合传递数据量较大的数据，同时，其请求的历史信息会保存在浏览器的缓存中，有一定的安全风险。而以POST方式向服务器发送数据请求，则不存在这两个方面的不足。

$.post()函数调用的语法格式如下所示：

> $.post(url,[data],[callback],[type])

~~~javascript
$(function(){
    $("#Button1").click(function(){//按钮单击事件
        $.post("UserInfo.aspx",
              {name:encodeURI($("#txtname").val()),
               sex:encodeURI($("#selsex").val()
              },
              function(data){
            	$("#divTip").empty().html(data);
            
        })
    })
})
~~~

aspx略

### 5.3$.ajax()

除了可以使用全局性函数load()、get()、post()实现页面的异步调用和与服务器交互数据外，在jQuery中，还有一个功能更为强悍的最底层的方法$.ajax()，该方法不仅可以很方便地实现上述三个全局函数完成的功能，还更多地关注实现过程中的细节。

## 六 .jQuery DOM

### 6.1访问元素

#### 6.1.1属性操作

获取元素属性的语法格式如下：

> attr()

$("img").attr("title");

attr()方法不仅可以获取元素的属性值，还可以设置元素的属性，其设置属性语法格式如下所示：

> attr(key ,value)

如果要设置多个属性，也可以通过attr()方法实现，其语法格式如下所示：

> attr({key0 :value0,key1 :value1 })

$("img").attr("src","xxx/xx/jpg");

attr()方法还可以绑定一个function()函数，通过该函数返回的值作为元素的属性值，其语法格式如下所示：

> attr(key,function(index))

其中，参数index为当前元素的索引号，整个函数返回一个字符串作为元素的属性值。

~~~javascript
$("img").attr("src",function(){
	return "Images/img0"+Math.floor(Math.random()*2+1)+".jpg"
});
~~~

使用removeAttr()方法可以将元素的属性删除，其使用的语法格式为：

> removeAttr(name)

$("img").removeAttr("title");

#### 6.1.2内容操作

获得内容 - text()、html() 以及 val()

三个简单实用的用于 DOM 操作的 jQuery 方法：

- text() - 设置或返回所选元素的文本内容
- html() - 设置或返回所选元素的内容（包括 HTML 标记）
- val() - 设置或返回表单字段的值

~~~javascript
$("#btn1").click(function(){
  alert("Text: " + $("#test").text());
});
$("#btn2").click(function(){
  alert("HTML: " + $("#test").html());
});
$("#btn1").click(function(){
  alert("值为: " + $("#test").val());
});
~~~

通过val()方法还可以获取select标记中的多个选项值，其语法格式如下所示：

> val(),join(",")

#### 6.1.3样式操作

在jQuery中，可以通过css()方法为某个指定的元素设置样式值，其语法格式如下所示：

> css(name ,value)

$("p").click(function(){
  $(this).css("font-weight","bold");
});

通过addClass()方法增加元素类别的名称，其语法格式如下：

> addClass(class)
>
> addclass(class0 class1)

~~~html
<style type="text/css">
    .cls1{font-style:italic }
    .cls2{background-color:#eee }
</style>
<script>$("p").click(function(){
  $(this).addClass(cls1 cls2);
});
</script>
其余略
~~~

通过toggleClass()方法切换不同的元素类别，其语法格式如下：

> toggleClass(class)

其中参数class为类别名称，其功能是当元素中含有名称为class的CSS类别时，删除该类别，否则增加一个该名称的CSS类别。

$("p").click(function(){
  $(this).toggleClass("clsP");
});

与增加CSS类别的addClass()方法相对应，removeClass()方法则用于删除类别，其语法格式如下：

> removeClass([class])

其中，参数class为类别名称，该名称是可选项，当选该名称时，则删除名称是class的类别，有多个类别时用空格隔开。如果不选名称，则删除元素中的所有类别。

~~~javascript
 $(p).removeClass("clsP");//删除clsP类别
$(p).removeClass("clsP cls1");//删除clsP 和cls1类别
$(p).removeClass();//删除所以类别
~~~

