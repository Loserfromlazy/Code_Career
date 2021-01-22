# Node.js学习笔记

Node.js 就是运行在服务端的 JavaScript。

Node.js 是一个基于Chrome JavaScript 运行时建立的一个平台。

Node.js是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。

# 一、Node.js安装

1.下载地址：[https://nodejs.org/en/download/](https://nodejs.org/en/download/)

2.选择目录进行安装

3.测试 在cmd中输入命令

~~~
node -v
~~~

显示当前node的版本

# 二、快速入门

## 2.1 控制台输出

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

## 2.2 使用函数

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

## 2.3 模块化编程

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

## 2.4 创建web服务器



## 2.5 理解服务端渲染

## 2.6 接收参数













