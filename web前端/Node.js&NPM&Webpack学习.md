# Node.js学习笔记

Node.js 就是运行在服务端的 JavaScript。

Node.js 是一个基于Chrome JavaScript 运行时建立的一个平台。

Node.js是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。

## 一、Node.js安装

1.下载地址：[https://nodejs.org/en/download/](https://nodejs.org/en/download/)

2.选择目录进行安装

3.测试 在cmd中输入命令

~~~
node -v
~~~

显示当前node的版本

## 二、快速入门

### 2.1 控制台输出

创建demo1.js

~~~js
var a=1;
var b=2;
console.log(a+b);
~~~

在当前目录的cmd下输入命令

~~~shell
node demo.js
~~~

输出结果为3

### 2.2 使用函数

创建文本文件demo2.js

```js
var c=add(100,200);
console.log(c);
function add(a,b){
	return a+b;
}
```

命令提示符输入命令

```sh
node demo2.js
```

运行后看到输出结果为300

### 2.3 模块化编程

在nodejs中，应用由模块组成，nodejs中采用commonJS模块规范。

1. 一个js文件就是一个模块
2. 每个模块都是一个独立的作用域，在这个而文件中定义的变量、函数、对象都是私有的，对其他文件不可见。

**node中模块分类**

- 1 核心模块

  由 node 本身提供，不需要单独安装（npm），可直接引入使用

- 2 第三方模块

  由社区或个人提供，需要通过npm安装后使用

- 3 自定义模块

  由我们自己创建，比如：tool.js 、 user.js

**基本使用**

先引入、在使用

**模块导入**

- 通过`require("fs")`来加载模块
- 如果是第三方模块，需要先使用npm进行下载
- 如果是自定义模块，需要加上相对路径`./`或者`../`,可以省略`.js`后缀，如果文件名是`index.js`那么index.js也可以省略。
- 模块可以被多次加载，但是只会在第一次加载

**模块导出**

- 在模块的内部，`module`变量代表的就是当前模块，它的`exports`属性就是对外的接口，加载某个模块，加载的就是`module.exports`属性，这个属性指向一个空的对象。

~~~js

//module.exports指向的是一个对象，我们给对象增加属性即可。
//module.exports.num = 123;
//module.exports.age = 18;
 
//通过module.exports也可以导出一个值，但是多次导出会覆盖
module.exports = '123';
module.exports = "abc";
~~~

**示例**

创建文本文件demo3_1.js

```js
exports.add=function(a,b){
	return a+b;
}
```

创建文本文件demo3_2.js

```js
var demo= require('./demo3_1');
console.log(demo.add(400,600));
```

我们在命令提示符下输入命令

```sh
node demo3_2.js
```

结果为1000

### 2.4 创建web服务器

创建文本文件demo4.js

~~~js
var http = require('http');
http.createServer(function (request,response){
    //发送HTTP头
    //HTTP 状态：200 : OK
    //内容类型：text/plain
    response.writeHead(200,{'Content-Type':'text/plain'});
    //发送响应数据
    response.end('HEllo World\n');
}).listen(8888)
console.log('Server running at http://127.0.0.1:8888/');
~~~

http为node内置的web模块

在cmd下输入命令

~~~sh
node demo4.js
~~~

服务启动后打开浏览器输入网址查看输出结果

### 2.5 理解服务端渲染

创建demo5.js

~~~js
var http = require('http');
http.createServer(function (request, response) {
    // 发送 HTTP 头部 
    // HTTP 状态值: 200 : OK
    // 内容类型: text/plain
    response.writeHead(200, {'Content-Type': 'text/plain'});
    // 发送响应数据 "Hello World"
	for(var i=0;i<10;i++){
		response.write('Hello World\n');
	}  
	response.end('');	
}).listen(8888);
// 终端打印如下信息
console.log('Server running at http://127.0.0.1:8888/');
~~~

启动服务

~~~sh
node demo5.js
~~~

输入网址查看结果

我们右键“查看源代码”发现，并没有我们写的for循环语句，而是直接的10条Hello World ，这就说明这个循环是在服务端完成的，而非浏览器（客户端）来完成。这与我们原来的JSP很是相似。

### 2.6 接收参数

创建demo6.js

~~~js
var http = require('http');
var url = require('url');
http.createServer(function(request, response){
    response.writeHead(200, {'Content-Type': 'text/plain'});
    // 解析 url 参数
    var params = url.parse(request.url, true).query;
    response.write("name:" + params.name);
    response.write("\n");
    response.end();
}).listen(8888);
console.log('Server running at http://127.0.0.1:8888/');
~~~

我们在命令提示符下输入命令

```sh
node demo6.js
```

在浏览器测试结果

# 包资源管理器NPM

npm全称Node Package Manager，他是node包管理和分发工具。其实我们可以把NPM理解为前端的Maven .

我们通过npm 可以很方便地下载js库，管理前端工程.

最近版本的node.js已经集成了npm工具，在命令提示符输入 npm -v 可查看当前npm版本

## 一、NPM命令

### 1.1 初始化工程

init命令是工程初始化命令

建立文件夹，cmd进入该文件夹，执行命令初始化

~~~sh
npm init
~~~

按照提示输入相关信息，如果是用默认值则直接回车即可。

name: 项目名称

version: 项目版本号

description: 项目描述

keywords: {Array}关键词，便于用户搜索到我们的项目

最后会生成package.json文件，这个是包的配置文件，相当于maven的pom.xml

我们之后也可以根据需要进行修改。

### 1.2 本地安装

install用于安装某个模块，如果我们想安装express模块（node的web框架）命令如下

~~~sh
npm install express
~~~

出现的是黄色的警告命令，可以忽略，这时是成功的执行了该命令。

在该目录下出现了一个叫node_modules文件夹和package-lock.json

node_modules文件夹用于存放下载的js库（相当于maven的本地仓库）

package-lock.json是当 node_modules 或 package.json 发生变化时自动生成的文件。这个文件主要功能是确定当前安装的包的依赖，以便后续重新安装的时候生成相同的依赖，而忽略项目开发过程中有些依赖已经发生的更新。我们再打开package.json文件，发现刚才下载的express已经添加到依赖列表中了.

关于版本号：

指定版本：比如1.2.2，遵循“大版本.次要版本.小版本”的格式规定，安装时只安装指定版本。

波浪号（tilde）+指定版本：比如~1.2.2，表示安装1.2.x的最新版本（不低于1.2.2），但是不安装1.3.x，也就是说安装时不改变大版本号和次要版本号。

插入号（caret）+指定版本：比如ˆ1.2.2，表示安装1.x.x的最新版本（不低于1.2.2），但是不安装2.x.x，也就是说安装时不改变大版本号。需要注意的是，如果大版本号为0，则插入号的行为与波浪号相同，这是因为此时处于开发阶段，即使是次要版本号变动，也可能带来程序的不兼容。

latest：安装最新版本。

### 1.3 全局安装

刚刚是本地安装，会将js库安装在当前目录下，使用全局安装会将库安装在全局目录下

如果你不知道你的全局目录在哪里，执行命令

```sh
npm root -g
```

比如全局安装jQuery，输入以下命令

~~~sh
npm install jquery -g
~~~

### 1.4 批量下载

我们从网上下载某些代码，发现只有package.json,没有node_modules文件夹，这时我们需要通过命令重新下载这些js库.

进入目录（package.json所在的目录）输入命令

```sh
npm install
```

此时，npm会自动下载package.json中依赖的js库.

### 1.5 淘宝NPM镜像

有时npm下载很慢所以可以安装一个淘宝镜像来加快下载速度。

输入命令，进行全局安装淘宝镜像

~~~sh
npm install -g cnpm -- registry=https://registry.npm.taobao.org
~~~

安装后可以使用以下命令来查看cnpm的版本

~~~sh
cnpm -v
~~~

使用cnpm

~~~sh
cnpm install 需要下载的js库
~~~

### 1.6 运行工程

如果想运行某个工程，则使用run命令

~~~sh
npm run dev
~~~

在package.json中定义的脚本：

dev是开发测试阶段，build是构建编译工程，lint是运行js代码检测

### 1.7 编译工程

接下来，测试一个代码的编译.编译后就可以将工程部署到nginx中啦~

编译后的代码会放在dist文件夹中，首先先删除dist文件夹中的文件,进入命令提示符输入命令

```sh
npm run build
```

生成后我们会发现只有个静态页面，和一个static文件夹

这种工程我们称之为单页Web应用（single page web application，SPA），就是只有一张Web页面的应用，是加载单个HTML 页面并在用户与应用程序交互时动态更新该页面的Web应用程序。

这里其实是调用了webpack来实现打包的

# Webpack

## 一、简介

Webpack 是一个前端资源加载/打包工具。它将根据模块的依赖关系进行静态分析，然后将这些模块按照指定的规则生成对应的静态资源。

 ![](https://mypic-12138.oss-cn-beijing.aliyuncs.com/blog/node/1-1.jpg)

​	从图中我们可以看出，Webpack 可以将多种静态资源 js、css、less 转换成一个静态文件，减少了页面的请求。 

## 二、安装

全局安装

~~~sh
npm install webpack -g
npm install webpack-cli -g
~~~

安装后查看版本号

~~~sh
webpack -v
~~~

## 三、快速入门

### 3.1 JS打包

在项目文件夹jsdemo下创建src文件夹

在src文件夹下创建bar.js

~~~js
exports.info=function(str){
    document.write(str);
}
~~~

src下创建logic.js

~~~js
exports.add=function(a,b){
    return a+b;
}
~~~

src下创建main.js

~~~js
var bar= require('./bar.js');
var logic=require('./logic');
bar.info('hello,world'+logic.add(100,200));
~~~

创建配置文件webpack.config.js，该文件与src处于同级

~~~js
var path = require("path");
module.exports = {
	entry: '.\src\main.js',
	output: {
		path: path.resolve(__dirname, '.\dist'),
		filename: 'bundle.js'
	}
};
~~~

以上代码的意思是：读取当前目录下src文件夹中的main.js（入口文件）内容，把对应的js文件打包，打包后的文件放入当前目录的dist文件夹下，打包后的js文件名为bundle.js

执行编译命令

`webpack`

执行后查看bundle.js会发现里面包含了上面两个js文件的内容

在jsdemo下创建index.html引用bundle.js

~~~html
<!doctype html>
<html>
  <head>  
  </head>
  <body>   
    <script src="dist/bundle.js"></script>
  </body>
</html>
~~~

打开html发现有内容

### 3.2 CSS打包

1.安装style-loader和css-loader

Webpack 本身只能处理 JavaScript 模块，如果要处理其他类型的文件，就需要使用 loader 进行转换。

Loader 可以理解为是模块和资源的转换器，它本身是一个函数，接受源文件作为参数，返回转换的结果。这样，我们就可以通过 require 来加载任何类型的模块或文件，比如 CoffeeScript、 JSX、 LESS 或图片。首先我们需要安装相关Loader插件，css-loader 是将 css 装载到 javascript；style-loader 是让 javascript 认识css

~~~sh
cnpm install style-loader css-loader --save-dev
~~~

2.修改webpack.config.js

~~~js
var path = require("path");
module.exports = {
	entry: './src/main.js',
	output: {
		path: path.resolve(__dirname, './dist'),
		filename: 'bundle.js'
	},
	module: {
		rules: [  
            {  
                test: /\.css$/,  
                use: ['style-loader', 'css-loader']
            }  
        ]  
	}
};
~~~

3.在src下创建css文件夹，文件夹下创建css1

~~~css
body{
	background:red;
}
~~~

4.修改main.js，引入css1.css

~~~css
require('./css1.css')
~~~

重新运行webpack，查看index.html

















































