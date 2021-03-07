# Mock.js

# 一、简介

Mock.js （官网http://mockjs.com/）是一款模拟数据生成器，旨在帮助前端攻城师独立于后端进行开发，帮助编写单元测试。提供了以下模拟功能：

根据数据模板生成模拟数据

模拟 Ajax 请求，生成并返回模拟数据

基于 HTML 模板生成模拟数据

**Mock.js具有以下特点：**

前后端分离

让前端攻城师独立于后端进行开发。

增加单元测试的真实性

通过随机数据，模拟各种场景。

开发无侵入

不需要修改既有代码，就可以拦截 Ajax 请求，返回模拟的响应数据。

用法简单

符合直觉的接口。

数据类型丰富

支持生成随机的文本、数字、布尔值、日期、邮箱、链接、图片、颜色等。

方便扩展

支持支持扩展更多数据类型，支持自定义函数和正则。

# 二、快速入门

**安装mockjs**

~~~sh
cnpm install mockjs
~~~

需求：生成列表数据，数据为5条

新建demo1.js

~~~js
let Mock=require('mockjs')
let data=Mock.mock({
    'list|5':[
        {
            'id':1,
            'name':'测试'
        }
    ]
})
console.log(JSON.stringify(data,null,2 ))
~~~

我们在本例中产生了5条相同的数据，这些数据都是相同的，如果我们需要让这些数据是按照一定规律随机生成的，需要按照Mock.js的语法规范来定义。

Mock.js 的语法规范包括两部分：

1.数据模板定义规范（Data Template Definition，DTD）

2.数据占位符定义规范（Data Placeholder Definition，DPD）

运行结果：

~~~
{
    "list": [
        {
            "id": 1,
            "name": "测试"
        },
        {
            "id": 1,
            "name": "测试"
        },
        {
            "id": 1,
            "name": "测试"
        },
        {
            "id": 1,
            "name": "测试"
        },
        {
            "id": 1,
            "name": "测试"
        }
    ]
}
~~~

## 2.1 数据模板定义规范DTD

数据模板中的每个属性由 3 部分构成：属性名、生成规则、属性值

~~~
// 属性名   name
// 生成规则 rule
// 属性值   value
'name|rule': value
~~~

属性名 和 生成规则 之间用竖线 | 分隔。

生成规则 是可选的。生成规则的含义需要依赖属性值的类型才能确定。属性值 中可以含有 @占位符。属性值 还指定了最终值的初始值和类型

生成规则 有 7 种格式：

'name|min-max': value

'name|count': value

'name|min-max.dmin-dmax': value

'name|min-max.dcount': value

'name|count.dmin-dmax': value

'name|count.dcount': value

'name|+step': value

### 2.1.1属性值是字符串

（1）'name|count': string

通过重复 string 生成一个字符串，重复次数等于 count

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id': 1,
        'name':'测试',
        'phone|11':'1'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（2）'name|min-max': string

通过重复 string 生成一个字符串，重复次数大于等于 min，小于等于 max

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id': 1,
        'name|2-4':'测试',
        'phone|11':'1'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.1.2属性值是数字

（1）'name|+1': number

属性值自动加 1，初始值为 number。

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（2）'name|min-max': number

生成一个大于等于 min、小于等于 max 的整数，属性值 number 只是用来确定类型

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（3）'name|min-max.dcount': value     生成一个浮点数，整数部分大于等于 min、小于等于 max，小数部分为dcount位

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（4）'name|min-max.dmin-dmax': number

生成一个浮点数，整数部分大于等于 min、小于等于 max，小数部分保留 dmin 到 dmax 位。

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'money2|1000-5000.2-4':0,
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.1.3属性值是boolean

（1）'name|1': boolean

随机生成一个布尔值，值为 true 的概率是 1/2，值为 false 的概率同样是 1/2

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'status|1':true
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（2）'name|min-max': value

随机生成一个布尔值，值为 value 的概率是 min / (min + max)

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'status|1':true,
        'default|1-3':true
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.1.4属性值是Object

（1）'name|count': object

从属性值 object 中随机选取 count 个属性。

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'status|1':true,
        'default|1-3':true,
        'detail|2':{'id':1,'date':'2005-01-01','content':'记录'}
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

（2）'name|min-max': object

从属性值 object 中随机选取 min 到 max 个属性

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'status|1':true,
        'default|1-3':true,
        'detail|2-3':{'id':1,'date':'2005-01-01','content':'记录'}
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.1.5属性值是数组

（1）'name|count': array

通过重复属性值 array 生成一个新数组，重复次数为 count

（2）'name|min-max': array

通过重复属性值 array 生成一个新数组，重复次数大于等于 min，小于等于 max。

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|5-10': [{
        'id|+1': 1,
        'name|2-3':'测试',
        'phone|11':'1',
        'point|122-500':0,
        'money|3000-8000.2':0,
        'status|1':true,
        'default|1-3':true,
        'detail|2-3':{'id':1,'date':'2005-01-01','content':'记录'}
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

## 2.2 数据占位符模板定义规范DPD

Mock.Random 是一个工具类，用于生成各种随机数据。

Mock.Random 的方法在数据模板中称为『占位符』，书写格式为 @占位符(参数 [, 参数]) 。

内置方法列表：

| **Type**      | **Method**                                                   |
| ------------- | ------------------------------------------------------------ |
| Basic         | boolean, natural, integer, float, character, string, range, date, time, datetime, now |
| Image         | image, dataImage                                             |
| Color         | color                                                        |
| Text          | paragraph, sentence, word, title, cparagraph, csentence, cword, ctitle |
| Name          | first, last, name, cfirst, clast, cname                      |
| Web           | url, domain, email, ip, tld                                  |
| Address       | area, region                                                 |
| Helper        | capitalize, upper, lower, pick, shuffle                      |
| Miscellaneous | guid, id                                                     |

### 2.2.1 基本方法

可以生成随机的基本数据类型

string 字符串	integer 整数	date 日期

~~~js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        'id|+1': 1,
        'name':'@string',
        'point':'@integer',
        'birthday':'@date'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
~~~

### 2.2.2 图像方法

image 随机生成图片地址

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        'id|+1': 1,
        'name':'@string',
        'point':'@integer',
        'birthday':'@date',
        'pic':'@image'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.2.3 文本方法

@title: 标题

@cword(100)  :文本内容 参数为字数

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        'id|+1': 1,
        'name':'@string',
        'point':'@integer',
        'birthday':'@date',
        'pic':'@image',
        'title':'@title',
        'content':'@cword(100)'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.2.4 名称方法

cname :中文名称	cfirst:中文姓氏	Last:英文姓氏

~~~js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        // 属性 id 是一个自增数，起始值为 1，每次增 1
        'id|+1': 1,
        'name':'@cname',
        'ename':'@last',
        'cfirst':'@cfirst',
        'point':'@integer',
        'birthday':'@date',
        'pic':'@image',
        'title':'@title',
        'content':'@cword(100)'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
~~~

### 2.2.5 网络方法

可以生成url ip email等网络相关信息

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        'id|+1': 1,
        'name':'@cname',
        'ename':'@last',
        'cfirst':'@cfirst',
        'point':'@integer',
        'birthday':'@date',
        'pic':'@image',
        'title':'@title',
        'content':'@cword(100)',
        'url':"@url",
        'ip':"@ip",
        'email':"@email"
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

### 2.2.6 地址方法

@region 区域	@county 省市县

```js
// 使用 Mock
let Mock = require('mockjs')
let data = Mock.mock({
    'list|10': [{
        'id|+1': 1,
        'name':'@cname',
        'ename':'@last',
        'cfirst':'@cfirst',
        'point':'@integer',
        'birthday':'@date',
        'pic':'@image',
        'title':'@title',
        'content':'@cword(100)',
        'url':"@url",
        'ip':"@ip",
        'email':"@email",
        'area':'@region',
        'address':'@county(true)'
    }]
})
// 输出结果
console.log(JSON.stringify(data,null,2))
```

# EasyMock

## 一、简介

Easy Mock 是杭州大搜车无线团队出品的一个极其简单、高效、可视化、并且能快速生成模拟数据的`在线 mock 服务`。以项目管理的方式组织 Mock List，能帮助我们更好的管理 Mock 数据。









