# 服务端渲染技术NUXT

## 一、简介

**服务端渲染**

服务端渲染又称SSR  (Server Side Render)是在服务端完成页面的内容，而不是在客户端通过AJAX获取数据。

​	与传统 SPA（Single-Page Application - 单页应用程序）相比，服务器端渲染(SSR)的优势主要在于：**更好的 SEO**，由于搜索引擎爬虫抓取工具可以直接查看完全渲染的页面。

​	请注意，截至目前，Google 和 Bing 可以很好对同步 JavaScript 应用程序进行索引。在这里，同步是关键。如果你的应用程序初始展示 loading 菊花图，然后通过 Ajax 获取内容，抓取工具并不会等待异步完成后再行抓取页面内容。也就是说，如果 SEO 对你的站点至关重要，而你的页面又是异步获取内容，则你可能需要服务器端渲染(SSR)解决此问题。

​	更快的内容到达时间(time-to-content)，特别是对于缓慢的网络情况或运行缓慢的设备。无需等待所有的 JavaScript 都完成下载并执行，才显示服务器渲染的标记，所以你的用户将会更快速地看到完整渲染的页面。通常可以产生更好的用户体验，并且对于那些「内容到达时间(time-to-content)与转化率直接相关」的应用程序而言，服务器端渲染(SSR)至关重要。

**NUXT**

Nuxt.js 是一个基于 Vue.js 的轻量级应用框架,可用来创建服务端渲染 (SSR) 应用,也可充当静态站点引擎生成静态站点应用,具有优雅的代码结构分层和热加载等特性。

## 二、NUXT环境搭建

（1）我们从网站上下载模板的压缩包 starter-template-master.zip 解压，修改template目录目录的package.json中的名称

（2）在命令提示符下进入该目录下的template目录 

（3）安装依赖

```
cnpm install
```

（4）修改package.json 

```
 ......
 "name": "tensquare",
 ......
```

（5）修改nuxt.config.js

```
......
title: 'XXX'
......
```

（6）测试运行

```sh
npm run dev
```

## 三、NUXT目录结构

（1）资源目录 assets

 用于组织未编译的静态资源如 LESS、SASS 或 JavaScript。

（2）组件目录 components

用于组织应用的 Vue.js 组件。Nuxt.js 不会扩展增强该目录下 Vue.js 组件，即这些组件不会像页面组件那样有 asyncData 方法的特性。

（3）布局目录 layouts

用于组织应用的布局组件。

（4）页面目录 pages

用于组织应用的路由及视图。Nuxt.js 框架读取该目录下所有的 .vue 文件并自动生成对应的路由配置。

（5）插件目录 plugins

用于组织那些需要在 根vue.js应用 实例化之前需要运行的 Javascript 插件。

（6）nuxt.config.js 文件

nuxt.config.js 文件用于组织Nuxt.js 应用的个性化配置，以便覆盖默认配置。

## 四、快速入门

### 4.1 定义布局

我们通常的网站头部和尾部都是相同的，我们可以把头部为尾部提取出来，形成布局页

修改layouts目录下default.vue

```html
<template>
  <div>
      <header>NUXT入门小Demo</header>
        <nuxt/>
      <footer>--黑马程序员--</footer>
  </div>
</template>
```

<**nuxt**/>为内容的区域

### 4.2 页面路由

在pages目录下创建文件夹

> --pages
>
> ----gathering
>
> ----recruit

recruit目录创建index.vue

```html
<template>
  <div>
    招聘列表
  </div>
</template>
```

gathering目录创建index.vue

```html
<template>
  <div>
    活动列表
  </div>
</template>
```

**NUXT的路由是根据目录自动生成的，无需手写。**

修改default.vue，header中添加导航链接

```html
      <router-link to="/">首页</router-link>
      <router-link to="/recruit">招聘</router-link>
      <router-link to="/gathering">活动</router-link>
```

点击导航链接，测试路由效果

### 4.3 数据渲染

（1）安装axios，用于异步获取数据

```sh
cnpm install axios --save
```

（2）修改gathering目录的index.vue

```html
<template>
  <div>
    活动列表
    <div v-for="(item,index) in items" :key="index" >{{item.name}}</div>
  </div>
</template>
<script>
import axios from 'axios'
export default {
    asyncData () {
        return axios.get('http://192.168.184.133:7300/mock/5af314a4c612520d0d7650c7/gathering/gathering')
           .then( res => {
                return { items: res.data.data }
           })
    }
}
</script>
```

asyncData是用于异步加载数据的方法，可以在组件的初始化之前被调用。因为如此，所以在方法内没法通过this来引用组建的实例对象。

### 4.4 动态路由

如果我们需要根据ID查询活动详情，就需要使用动态路由。NUXT的动态路由是以下划线开头的vue文件，参数名为下划线后边的文件名 

创建pages/gathering/item/_id.vue

```html
<template>
  <div>
    活动详情
    {{item.id}}
    <hr>
    {{item.name}}

  </div>
</template>
<script>
import axios from 'axios'
export default {
    asyncData( {params} ){
        //params.id
        return axios.get(`http://192.168.184.133:7300/mock/5af314a4c612520d0d7650c7/gathering/gathering/${params.id}`).then(
            res =>{              
              return {item: res.data.data}
            }
        )
    }
}
</script>
```

我们在地址栏输入 [http://localhost:3000/gathering/item/1](http://localhost:3000/gathering/item/1) 即可看到运行结果

在活动列表页点击链接进入详情页

```html
    活动列表
    <div v-for="(item,index) in items" :key="index">
        <nuxt-link :to="'/gathering/item/'+item.id">{{item.name}}</nuxt-link>
    </div>
```

目前 `nuxt-link` 的作用和 [`router-link`](https://router.vuejs.org/zh-cn/api/router-link.html) 一致    ，都是进行路由的跳转。



