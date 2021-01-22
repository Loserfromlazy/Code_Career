# Vue.js学习笔记

Vue.js是一个构建数据驱动的web 界面的渐进式框架。Vue.js 的目标是通过尽可能简单的API 实现响应的数据绑
定和组合的视图组件。它不仅易于上手，还便于与第三方库或既有项目整合。

## 一.概述

### 1.1MVVM模式

MVVM是Model-View-ViewModel的简写。它本质上就是MVC 的改进版。MVVM 就是将其中的View 的状态和行为
抽象化，让我们将视图UI 和业务逻辑分开
MVVM模式和MVC模式一样，主要目的是分离视图（View）和模型（Model）

Vue.js 是一个提供了MVVM 风格的双向数据绑定的Javascript 库，专注于View 层。它的核心是MVVM 中的VM，
也就是ViewModel。ViewModel负责连接View 和Model，保证视图和数据的一致性，这种轻量级的架构让前端
开发更加高效、便捷

### 1.2快速入门

**不使用return包裹的数据会在项目的全局可见，会造成变量污染**
**使用return包裹后数据中变量只在当前组件中生效，不会影响其他组件**

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="js/vuejs-2.5.16.js">
    </script>
</head>
<body>
	<div id="app">
        {{message}}
    </div>
    <script>
    	new Vue({
            el:"#app",//表示当前vue对象接管了div区域
            data:{
                message:"hello world" //注意不要以分号结尾
            }
        })
    </script>
</body>
</html>
~~~

### 1.3插值表达式

数据绑定最常见的形式就是使用“Mustache”语法(双大括号) 的文本插值，Mustache 标签将会被替代为对应数据对
象上属性的值。无论何时，绑定的数据对象上属性发生了改变，插值处的内容都会更新。

Vue.js都提供了完全的JavaScript表达式支持

~~~
{{number+1}}
{{ok?'yes':'no'}}
~~~

这些表达式会在所属Vue 实例的数据作用域下作为JavaScript 被解析。有个限制就是，每个绑定都只能包含单个
表达式，所以下面的例子都不会生效。

~~~
<!-- 这是语句，不是表达式-->
{{ var a = 1 }}
<!-- 流控制也不会生效，请使用三元表达式-->
{{ if (ok) { return message } }}
~~~

## 二 模板语法

### 2.1插值

**不使用return包裹的数据会在项目的全局可见，会造成变量污染**
**使用return包裹后数据中变量只在当前组件中生效，不会影响其他组件**

**文本**

数据绑定最常见的形式就是使用 {{...}}（双大括号）的文本插值：

~~~html
<div id="app">
  <p>{{ message }}</p>
</div>
~~~

**html**

使用 v-html 指令用于输出 html 代码：

~~~html
<div id="app">
    <div v-html="message"></div>
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    message: '<h1>aaa</h1>'
  }
})
</script>
~~~

**属性**

HTML 属性中的值应使用 v-bind 指令。

以下实例判断 class1 的值，如果为 true 使用 class1 类的样式，否则不使用该类：

~~~html
<div id="app">
  <label for="r1">修改颜色</label><input type="checkbox" v-model="use" id="r1">
  <br><br>
  <div v-bind:class="{'class1': use}">
    v-bind:class 指令
  </div>
</div>
    
<script>
new Vue({
    el: '#app',
  data:{
      use: false
  }
});
</script>
~~~

**表达式**

Vue.js 都提供了完全的 JavaScript 表达式支持。

~~~html
<div id="app">
    {{5+5}}<br>
    {{ ok ? 'YES' : 'NO' }}<br>
    {{ message.split('').reverse().join('') }}
    <div v-bind:id="'list-' + id">aaa</div>
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    ok: true,
    message: 'AAA',
    id : 1
  }
})
</script>
~~~

### 2.2 指令

指令是带有 v- 前缀的特殊属性。

指令用于在表达式的值改变时，将某些行为应用到 DOM 上。如下例子： v-if 指令将根据表达式 seen 的值(true 或 false )来决定是否插入 p 元素。

~~~html
<div id="app">
    <p v-if="seen">现在你看到我了</p>
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    seen: true
  }
})
</script>
~~~

**参数**

参数在指令后以冒号指明。例如， v-bind 指令被用来响应地更新 HTML 属性：

~~~html
<div id="app">
    <pre><a v-bind:href="url">aaa</a></pre>
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    url: 'http://www.baidu.com'
  }
})
</script>
~~~

在这里 href 是参数，告知 v-bind 指令将该元素的 href 属性与表达式 url 的值绑定。

另一个例子是 v-on 指令，它用于监听 DOM 事件：

```
<a v-on:click="doSomething">
```

在这里参数是监听的事件名。

**修饰符**

修饰符是以半角句号 **.** 指明的特殊后缀，用于指出一个指令应该以特殊方式绑定。例如，.prevent 修饰符告诉 **v-on** 指令对于触发的事件调用 event.preventDefault()

```
<form v-on:submit.prevent="onSubmit"></form>
```

### 2.3用户输入

在 input 输入框中我们可以使用 v-model 指令来实现双向数据绑定：

~~~html
<div id="app">
    <p>{{ message }}</p>
    <input v-model="message">
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    message: 'AAA!'
  }
})
</script>
~~~

**v-model** 指令用来在 input、select、textarea、checkbox、radio 等表单控件元素上创建双向数据绑定，根据表单上的值，自动更新绑定的元素的值。

按钮的事件我们可以使用 v-on 监听事件，并对用户的输入进行响应。

### 2.4 过滤器

Vue.js 允许你自定义过滤器，被用作一些常见的文本格式化。由"管道符"指示, 格式如下：

```
<!-- 在两个大括号中 -->
{{ message | capitalize }}
<!-- 在 v-bind 指令中 -->
<div v-bind:id="rawId | formatId"></div>
```

**过滤器函数接受表达式的值作为第一个参数**。

以下实例对输入的字符串第一个字母转为大写：

~~~html
<div id="app">
  {{ message | capitalize }}
</div>
    
<script>
new Vue({
  el: '#app',
  data: {
    message: 'loserfromlazy'
  },
  filters: {
    capitalize: function (value) {
      if (!value) return ''
      value = value.toString()
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
  }
})
</script>
~~~

过滤器可以串联：

```
{{ message | filterA | filterB }}
```

过滤器是 JavaScript 函数，因此可以接受参数：

```
{{ message | filterA('arg1', arg2) }}
```

这里，message 是第一个参数，字符串 'arg1' 将传给过滤器作为第二个参数， arg2 表达式的值将被求值然后传给过滤器作为第三个参数。

### 2.5 缩写

**v-bind 缩写**

Vue.js 为两个最为常用的指令提供了特别的缩写：

```
<!-- 完整语法 -->
<a v-bind:href="url"></a>
<!-- 缩写 -->
<a :href="url"></a>
```

**v-on 缩写**

```
<!-- 完整语法 -->
<a v-on:click="doSomething"></a>
<!-- 缩写 -->
<a @click="doSomething"></a>
```



## 三.VueJS常用的系统指令

### 3.1 v-on （Vue.js事件处理器）

可以用v-on指令监听DOM事件，并在触发时运行一些JS代码

#### 3.1.1 v-on:click

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="js/vuejs-2.5.16.js"></script>
</head>
<body>
<div id="app">
    {{message}}
    <button v-on:click="fun1('good')">
        点击改变
    </button>
</div>
<script>
    new Vue({
        el:"#app",//表示当前vue对象接管了div区域
        data:{
            message:"hello world"
        },
        methods:{
        fun1:function(msg){
        	this.message=msg;
    	}
    }
    });
</script>
</body>
</html>
~~~

#### 3.1.2 v-on:keydown

~~~html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="js/vuejs-2.5.16.js"></script>
</head>
<body>
<div id="app">
    {{message}}
    <input type="text" v-on:keydown="fun2('good',$event)">
</div>
<script>
    new Vue({
        el:"#app",//表示当前vue对象接管了div区域
        data:{
            message:"hello world"
        },
        methods:{
        fun2:function(msg,event){
        	if(!((event.keyCode>=48&&event.keyCode<=57)||event.keyCode==8||event.keyCode==46)){
                event.preventDefault();
            }
    	}
    }
    });
</script>
</body>
</html>
~~~

#### 3.1.3 v-on:mouseover

~~~html
<div id="app">
    <div v-on:mouseover="fun1" id="div">
        <textarea v-on:mouseover="fun2($event)">这是一个文件域</textarea>
    </div>
</div>
<script>
    new Vue({
        el:"#app",//表示当前vue对象接管了div区域
        methods:{
       		fun1:function(){
                alert("div");
            },
            fun2:function(event){
                alert("textarea");
                event.stopPropagation();//阻止冒泡
            }
    	}
    });
</script>
~~~

#### 3.1.4 事件修饰符

Vue.js 为v-on 提供了事件修饰符来处理DOM 事件细节，如：event.preventDefault() 或
event.stopPropagation()。
Vue.js通过由点(.)表示的指令后缀来调用修饰符。

- .stop 阻止事件冒泡，不让他向外去执行函数，到此为止
- .prevent 阻止组件本来应该发生的事件，转而去执行自己定义的事件
- .capture  网页是默认按照冒泡方式去触发函数的，但是当我们使用.capture修饰符时，网页就会按照捕获的方式触发函数，也就是从外向内执行，**但是这个时候一定要注意，.capture修饰符一定要写在外层才能生效**
- .self    当前元素自身时触发处理函数时才会触发函数，原理：是根据event.target确定是否当前元素本身，来决定是否触发的事件/函数
- .once  加上此修饰符之后相应的函数只能触发一次，无论你点击多少下，函数就只触发一次

~~~html
<div id="app">
<form @submit.prevent action="http://www.loserfromlazy.cn" method="get">
<input type="submit" value="提交"/>
</form>
<div @click="fun1">
<a @click.stop href="http://www.loserfromlazy.cn">itcast</a>
</div>
</div>
<script>
new Vue({
el:'#app', //表示当前vue对象接管了div区域
methods:{
fun1:function(){
alert("hello itcast");
}
}
});
</script>
~~~

#### 3.1.5 按键修饰符

Vue 允许为v-on 在监听键盘事件时添加按键修饰符

全部的按键别名

- .enter
- .tab
- .delete 捕获delete和backspace键
- .esc
- .space
- .up
- .down
- .left
- .right
- .ctrl
- .alt
- .shift
- .meta

~~~html
<div id="app">
<input type="text" v-on:keyup.enter="fun1">
</div>
<script>
new Vue({
el:'#app', //表示当前vue对象接管了div区域
methods:{
fun1:function(){
alert("你按了回车");
}
}
});
</script>
~~~

### 3.2 v-text 和v-html

~~~html
<div id="app">
    <div v-text="message"></div>	<!--将文本嵌入浏览器中显示<h1>loserfromlazy</h1> -->
<div v-html="message"></div>	<!--将html嵌入 浏览器中显示 loserfromlazy //此为h1标题 -->
</div>
<script>
new Vue({
el:'#app', //表示当前vue对象接管了div区域
data:{
message:"<h1>loserfromlazy</h1>"
}
});
</script>
~~~

### 3.3 v-bind

~~~html
<div id="app">
<font size="5" v-bind:color="ys1">传智播客</font>
<font size="5" :color="ys2">黑马程序员</font>
<hr>
<a v-bind={href:"http://www.loserfromlazy.com/index/"+id}>itcast</a>
</div>
<script>
new Vue({
el:'#app', //表示当前vue对象接管了div区域
data:{
ys1:"red",
ys2:"green",
id:1
}
});
</script>
~~~

### 3.4 v-model(Vue.js 表单)

你可以用v-model指令在表单控件元素上创建双向数据绑定

v-model会根据控件类型自动选取正确的方法，以下是**input和textarea元素**实现双向绑定

~~~html
<div id="app">
    <p>input元素</p>
    <input v-model="message" palceholder="编辑" >
    <p>消息是：{{message}}</p>
	<p>textarea元素</p>
    <textarea v-model="message2" placeholder="多行文本"></textarea>
</div>
<script>
	new Vue({
        el:"#app",
        data:{
            message:"loserfromlazy",
            message2:"loserfromlazy/rwww.baidu.com"
        }
    })
</script>
~~~

**复选框**(复选框如果是一个为逻辑值，如果是多个则绑定到同一个数组)

~~~html
<div id="app">
  <p>单个复选框：</p>
  <input type="checkbox" id="checkbox" v-model="checked">
  <label for="checkbox">{{ checked }}</label>
    
  <p>多个复选框：</p>
  <input type="checkbox" id="runoob" value="Runoob" v-model="checkedNames">
  <label for="runoob">Runoob</label>
  <input type="checkbox" id="google" value="Google" v-model="checkedNames">
  <label for="google">Google</label>
  <input type="checkbox" id="taobao" value="Taobao" v-model="checkedNames">
  <label for="taobao">taobao</label>
  <br>
  <span>选择的值为: {{ checkedNames }}</span>
</div>
<script>
	new Vue({
        el:"#app",
        data:{
            checked:false,
            chackedNames: []
        }
    })
</script>
~~~

**单选按钮**

~~~html
<div id="app">
  <input type="radio" id="runoob" value="Runoob" v-model="picked">
  <label for="runoob">Runoob</label>
  <br>
  <input type="radio" id="google" value="Google" v-model="picked">
  <label for="google">Google</label>
  <br>
  <span>选中值为: {{ picked }}</span>
</div>
 
<script>
new Vue({
  el: '#app',
  data: {
    picked : 'Runoob'
  }
})
</script>
~~~

**select列表**（）

~~~html
<div id="app">
   <select v-model="selected" name="fruit">
    <option v-for="option in optionsList" :value='option.value'>{{option.key}}</option>
  </select>

  <div id="output">
      选择的网站是: {{selected}}
  </div>
</div>
 
<script>
new Vue({
  el: '#app',
  data: {
        optionsList:[{
            key:'选择',
            value:'Select'
            },{
            key:'淘宝',
            value:'taobao'
            },{
            key:'阿里巴巴',
            value:'alibaba'
            },{
            key:'拼多多',
            value:'pinduoduo'
        }],
        selected: 'Select'
  }
})
</script>
~~~

**表单用的修饰符**

**.lazy**

在默认情况下， v-model 在 input 事件中同步输入框的值与数据，但你可以添加一个修饰符 lazy ，从而转变为在 change 事件中同步：

> ```
> <input v-model.lazy="msg" >
> ```

**.number**

如果想自动将用户的输入值转为 Number 类型（如果原值的转换结果为 NaN 则返回原值），可以添加一个修饰符 number 给 v-model 来处理输入值：

> ```
> <input v-model.number="age" type="number">
> ```

**.trim**

如果要自动过滤用户输入的首尾空格，可以添加 trim 修饰符到 v-model 上过滤输入：

> ```
> <input v-model.trim="msg">
> ```

### 3.5 v-for（Vue.js循环语句）

v-for 指令需要以 **site in sites** 形式的特殊语法

v-for绑定数据到数组

~~~html
<div id="app">
  <ol>
    <li v-for="site in sites">
      {{ site.name }}
    </li>
  </ol>
</div>
 
<script>
new Vue({
  el: '#app',
  data: {
    sites: [
      { name: 'Google' },
      { name: 'Taobao' }
    ]
  }
})
</script>
~~~

v-for循环整数

~~~html
<div id="app">
  <ul>
    <li v-for="n in 10">
     {{ n }}
    </li>
  </ul>
</div>
~~~

v-for迭代对象

~~~html
<div id="app">
  <ul>
    <li v-for="value in object">
    {{ value }}
    </li>
  </ul>
</div>
 
<script>
new Vue({
  el: '#app',
  data: {
    object: {
      name: '菜鸟教程',
      url: 'http://www.runoob.com',
      slogan: '学的不仅是技术，更是梦想！'
    }
  }
})
</script>
你也可以提供第二个的参数为键名：
<div id="app">
  <ul>
    <li v-for="(value, key) in object">
    {{ key }} : {{ value }}
    </li>
  </ul>
</div>
第三个参数为索引：
<div id="app">
  <ul>
    <li v-for="(value, key, index) in object">
     {{ index }}. {{ key }} : {{ value }}
    </li>
  </ul>
</div>

~~~

### 3.6 v-if 和v-show（Vue.js条件语句）

~~~html
<div id="app">
<span v-if="flag">传智播客</span>
<span v-show="flag">itcast</span>
<button @click="toggle">切换</button>
</div>
<script>
	new Vue({
       el:"#app",
       data:{
           flag:false,
       },
       method:{
           toggle:function(){
               this.flag=!this.flag;
           }
       }
    });
</script>
~~~



**区别:**

在切换 **v-if** 块时，Vue.js 有一个局部编译/卸载过程，因为 **v-if** 之中的模板也可能包括数据绑定或子组件。**v-if**是真实的条件渲染，因为它会确保条件块在切换当中合适地销毁与重建条件块内的事件监听器和子组件。

**v-if** 也是惰性的：如果在初始渲染时条件为假，则什么也不做——在条件第一次变为真时才开始局部编译（编译会被缓存起来）。

相比之下，**v-show** 简单得多——元素始终被编译并保留，只是简单地基于 CSS 切换。

一般来说，**v-if** 有更高的切换消耗而 **v-show** 有更高的初始渲染消耗。因此，如果需要频繁切换 **v-show** 较好，如果在运行时条件不大可能改变 **v-if** 较好。

## 四 Vue.js生命周期

- ![Lifecycle](http://cn.vuejs.org/images/lifecycle.png)

## 五.Vue.js Ajax（简单）

### 5.1 vue ajax(vue-resource)

Vue 要实现异步加载需要使用到 vue-resource 库。

Vue.js 2.0 版本推荐使用 axios 来完成 ajax 请求。

### 5.2 vue ajax(axios)

**安装**：

```
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
```

> ```
> $ npm install axios          使用npm
> $ bower install axios		 使用bower
> $ yarn add axios			 使用yarm
> ```

#### 5.2.1 get请求

**response.data** 读取 JSON 数据：

~~~html
<div id="app">
    {{info}}
</div>
<script>
	new Vue({
      el:"#app",
      data(){
          return {
              info: null
          }
      },
      mounted(){
          axios.get("https://www.runoob.com/try/ajax/json_demo.json")
          	.then(response => (this.info =response))   
          //.then(response => (this.info = response.data.sites))
          	.catch(function(error){ //处理失败请求
             console.log(error); 
          });
      }
    })
</script>
~~~

**get传参**

~~~js
// 直接在 URL 上添加参数 ID=12345
axios.get('/user?ID=12345')
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
 
// 也可以通过 params 设置参数：
axios.get('/user', {
    params: {
      ID: 12345
    }
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
~~~

#### 5.2.2 post请求

~~~html
<div id="app">
  {{ info }}
</div>
<script type = "text/javascript">
new Vue({
  el: '#app',
  data () {
    return {
      info: null
    }
  },
  mounted () {
    axios
      .post('https://www.runoob.com/try/ajax/demo_axios_post.php')
      .then(response => (this.info = response))
      .catch(function (error) { // 请求失败处理
        console.log(error);
      });
  }
})
</script>
~~~

**post传参**

~~~js
axios.post('/user', {
    firstName: 'Fred',        // 参数 firstName
    lastName: 'Flintstone'    // 参数 lastName
  })
  .then(function (response) {
    console.log(response);
  })
  .catch(function (error) {
    console.log(error);
  });
~~~







