# Swagger学习笔记

# 一、RESTful

## 1.简介

RESTful架构，就是目前最流行的一种互联网软件架构。它结构清晰、符合标准、易于理解、扩展方便，所以正得到越来越多网站的采用。

**要理解RESTful架构，最好的方法就是去理解Representational State Transfer这个词组到底是什么意思，它的每一个词代表了什么涵义。**

（1）**资源（Resources）**

REST的名称"表现层状态转化"中，省略了主语。"表现层"其实指的是"资源"（Resources）的"表现层"。

**所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息。**它可以是一段文本、一张图片、一首歌曲、一种服务，总之就是一个具体的实在。你可以用一个URI（统一资源定位符）指向它，每种资源对应一个特定的URI。要获取这个资源，访问它的URI就可以，因此URI就成了每一个资源的地址或独一无二的识别符。

所谓"上网"，就是与互联网上一系列的"资源"互动，调用它的URI。

（2）**表现层（Representation）**

"资源"是一种信息实体，它可以有多种外在表现形式。**我们把"资源"具体呈现出来的形式，叫做它的"表现层"（Representation）。**

比如，文本可以用txt格式表现，也可以用HTML格式、XML格式、JSON格式表现，甚至可以采用二进制格式；图片可以用JPG格式表现，也可以用PNG格式表现。

URI只代表资源的实体，不代表它的形式。严格地说，有些网址最后的".html"后缀名是不必要的，因为这个后缀名表示格式，属于"表现层"范畴，而URI应该只代表"资源"的位置。它的具体表现形式，应该在HTTP请求的头信息中用Accept和Content-Type字段指定，这两个字段才是对"表现层"的描述。

（3）**状态转化（State Transfer）**

访问一个网站，就代表了客户端和服务器的一个互动过程。在这个过程中，势必涉及到数据和状态的变化。

互联网通信协议HTTP协议，是一个无状态协议。这意味着，所有的状态都保存在服务器端。因此，**如果客户端想要操作服务器，必须通过某种手段，让服务器端发生"状态转化"（State Transfer）。而这种转化是建立在表现层之上的，所以就是"表现层状态转化"。**

客户端用到的手段，只能是HTTP协议。具体来说，就是HTTP协议里面，四个表示操作方式的动词：GET、POST、PUT、DELETE。它们分别对应四种基本操作：**GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资源，DELETE用来删除资源。**

综合上面的解释，我们总结一下什么是RESTful架构：

　　（1）每一个URI代表一种资源；

　　（2）客户端和服务器之间，传递这种资源的某种表现层；

​    　（3）客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。

## 2.常见错误

（1）URI包含动词

```
POST /accounts/1/transfer/500/to/2
```

正确的写法是把动词transfer改成名词transaction

（2）URI包含版本

```
http://www.example.com/app/1.0/foo

http://www.example.com/app/1.1/foo

http://www.example.com/app/2.0/foo
```

因为不同的版本，可以理解成同一种资源的不同表现形式，所以应该采用同一个URI。版本号可以在HTTP请求头信息的Accept字段中进行区分

```
Accept: vnd.example-com.foo+json; version=1.0

Accept: vnd.example-com.foo+json; version=1.1

Accept: vnd.example-com.foo+json; version=2.0
```

# 二、Swagger

## 2.1简介

随着互联网技术的发展，现在的网站架构基本都由原来的后端渲染，变成了：前端渲染、先后端分离的形态，而且前端技术和后端技术在各自的道路上越走越远。 
前端和后端的唯一联系，变成了API接口；API文档变成了前后端开发人员联系的纽带，变得越来越重要，`swagger`就是一款让你更好的书写API文档的框架。

SwaggerEditor安装与启动（SwaggerEditor有在线版推荐使用）

> 下载并解压，全局安装http-server(http-server是一个简单的零配置命令行http服务器)
>
> ```
> npm install -g http-server
> ```
>
> 启动swagger-editor
>
> ```
>http-server swagger-editor
> ```
> 
> 浏览器打开：http://localhost:8080  

**语法规则**

（1）固定字段

| 字段名       | 类型                                         | 描述                                                         |
| ------------ | -------------------------------------------- | ------------------------------------------------------------ |
| swagger      | string                                       | 必需的。使用指定的规范版本。                                 |
| info         | Info Object                                  | 必需的。提供元数据API。                                      |
| host         | string                                       | 主机名或ip服务API。                                          |
| basePath     | string                                       | API的基本路径                                                |
| schemes      | [string]                                     | API的传输协议。 值必须从列表中:"http","https","ws","wss"。   |
| consumes     | [string]                                     | 一个MIME类型的api可以使用列表。值必须是所描述的Mime类型。    |
| produces     | [string]                                     | MIME类型的api可以产生的列表。  值必须是所描述的Mime类型。    |
| paths        | [路径对象](#pathsObject)                     | 必需的。可用的路径和操作的API。                              |
| definitions  | [定义对象](#definitionsObject)               | 一个对象数据类型生产和使用操作。                             |
| parameters   | [参数定义对象](#parametersDefinitionsObject) | 一个对象来保存参数,可以使用在操作。 这个属性不为所有操作定义全局参数。 |
| responses    | [反应定义对象](#responsesDefinitionsObject)  | 一个对象响应,可以跨操作使用。 这个属性不为所有操作定义全球响应。 |
| externalDocs | [外部文档对象](#externalDocumentationObject) | 额外的外部文档。                                             |
| summary      | string                                       | 什么操作的一个简短的总结。 最大swagger-ui可读性,这一领域应小于120个字符。 |
| description  | string                                       | [详细解释操作的行为。GFM语法可用于富文本表示。](https://help.github.com/articles/github-flavored-markdown) |
| operationId  | string                                       | 独特的字符串用于识别操作。 id必须是唯一的在所有业务中所描述的API。 工具和库可以使用operationId来唯一地标识一个操作,因此,建议遵循通用的编程的命名约定。 |
| deprecated   | boolean                                      | 声明该操作被弃用。 使用声明的操作应该没有。 默认值是false。  |

（2）字段类型与格式定义

| 普通的名字 | type    | format    | 说明                                |
| ---------- | ------- | --------- | ----------------------------------- |
| integer    | integer | int32     | 签署了32位                          |
| long       | integer | int64     | 签署了64位                          |
| float      | number  | float     |                                     |
| double     | number  | double    |                                     |
| string     | string  |           |                                     |
| byte       | string  | byte      | base64编码的字符                    |
| binary     | string  | binary    | 任何的八位字节序列                  |
| boolean    | boolean |           |                                     |
| date       | string  | date      | 所定义的full-date- - - - - -RFC3339 |
| dateTime   | string  | date-time | 所定义的date-time- - - - - -RFC3339 |
| password   | string  | password  | 用来提示用户界面输入需要模糊。      |

## 2.2 案例城市API文档

数据库字段：

| 字段名 | 字段含义 | 字段类型 | 备注              |
| ------ | -------- | -------- | ----------------- |
| id     | ID       | 文本     |                   |
| name   | 城市名称 | 文本     |                   |
| ishot  | 是否热门 | 文本     | 0：非热门 1：热门 |

| 字段名    | 字段含义 | 字段类型 | 备注              |
| --------- | -------- | -------- | ----------------- |
| id        | ID       | 文本     |                   |
| labelname | 标签名称 | 文本     |                   |
| state     | 状态     | 文本     | 0：无效 1：有效   |
| count     | 使用数量 | 整形     |                   |
| fans      | 关注数   | 整形     |                   |
| recommend | 是否推荐 | 文本     | 0：不推荐 1：推荐 |

| 字段名  | 字段含义 | 字段类型 | 备注 |
| ------- | -------- | -------- | ---- |
| userid  | 用户ID   | 文本     |      |
| labelid | 标签ID   | 文本     |      |

### 新增城市

编写新增城市的API , post提交城市实体

URL： /city

Method: post

~~~yaml
swagger: '2.0'
info:
  version: "1.0.0"
  title: 基础模块-城市API
basePath: /base
host: api.tensquare.com
paths:
  /city:
    post:
      summary: 新增城市
      parameters:
        - name: "body"
          in: "body"
          description: 城市实体类
          required: true
          schema:
            $ref: '#/definitions/City'
      responses:
        200:
          description: 成功
          schema:
            $ref: '#/definitions/ApiResponse'
definitions:
  City: 
    type: object
    properties: 
      id: strin
        type: string
        description: "ID"
      name:
        type: string
        description: "名称"
      ishot:
        type: string
        description: 是否热门
  ApiResponse: 
    type: object
    properties: 
      flag: 
        type: boolean
        description: 是否成功
      code:
        type: integer
        format: int32
        description: 返回码
      message:
        type: string
        description: 返回信息
~~~

### 修改城市

与/city: 同一级别创建/city/{cityId} : 

Method: put

~~~yaml
  /city/{cityId}:
    put:
      summary: 修改城市
      parameters:
        - name: cityId
          in: path
          description: 城市ID
          required: true
          type: string
        - name: body
          in: body
          description: 城市
          schema:
            $ref: '#/definitions/City'
      responses:
        200:
          description: 成功响应
          schema:
            $ref: '#/definitions/ApiResponse'  
~~~

### 删除城市

删除城市地址为/city/{cityId} ，与修改城市的地址相同，区别在于使用delete方法提交请求（/city/{cityId} 下增加delete）

~~~yaml
delete:
      summary: 根据ID删除
      description: 返回是否成功
      parameters:
        - name: cityId
          in: path
          description: 城市ID
          required: true
          type: string
      responses:
        '200':
          description: 成功
          schema:
            $ref: '#/definitions/ApiResponse'
~~~

### 跟据ID查询城市

URL: /city/{cityId}   

Method: get    

返回的内容结构为： {flag:true,code:20000, message:"查询成功",data: {.....} }

data属性返回的是city的实体类型

代码实现如下：

（1）在definitions下定义城市对象的响应对象

```yaml
  ApiCityResponse:
    type: "object"
    properties:
      code:
        type: "integer"
        format: "int32"
      flag:
        type: "boolean"
      message:
        type: "string"
      data:
        $ref: '#/definitions/City'
```

（2）/city/{cityId} 下新增get方法API

```yaml
    get:
      summary: 根据ID查询
      description: 返回一个城市
      parameters:
        - name: cityId
          in: path
          description: 城市ID
          required: true
          type: string
      responses:
        '200':
          description: 操作成功
          schema:
            $ref: '#/definitions/ApiCityResponse'
```

### 城市列表

URL: /city  

Method: get    

返回的内容结构为： {flag:true,code:20000, message:"查询成功",data:[{.....},{.....},{.....}] }

data属性返回的是city的实体数组  

实现步骤如下：

（1）在definitions下定义城市列表对象以及相应对象

```yaml
  CityList:
    type: "array"
    items: 
      $ref: '#/definitions/City'
  ApiCityListResponse:
    type: "object"
    properties:
      code:
        type: "integer"
        format: "int32"
      flag:
        type: "boolean"
      message:
        type: "string"
      data:
        $ref: '#/definitions/CityList'
```

（2）在/city增加get

```yaml
    get:
      summary: "城市全部列表"
      description: "返回城市全部列表"
      responses:
        200:
          description: "成功查询到数据"
          schema: 
            $ref: '#/definitions/ApiCityListResponse'
```

### 根据条件查询城市列表

代码如下：

```yaml
  /city/search:
    post:
      summary: 城市列表(条件查询)
      parameters:
        - name: body
          in: body
          description: 查询条件
          required: true
          schema:
            $ref: "#/definitions/City"
      responses:
        200:
          description: 查询成功
          schema:
            $ref: '#/definitions/ApiCityListResponse'
```

### 城市分页列表

实现如下：

（1）在definitions下定义城市分页列表响应对象

```yaml
  ApiCityPageResponse:
    type: "object"
    properties:
      code:
        type: "integer"
        format: "int32"
      flag:
        type: "boolean"
      message:
        type: "string"
      data:
        properties:
          total:
            type: "integer"
            format: "int32"
          rows:
            $ref: '#/definitions/CityList'
```

（2）新增节点

```yaml
  /city/search/{page}/{size}:
    post:
      summary: 城市分页列表
      parameters:
        - name: page
          in: path
          description: 页码
          required: true
          type: integer
          format: int32
        - name: size
          in: path
          description: 页大小
          required: true
          type: integer
          format: int32
        - name: body
          in: body
          description: 查询条件
          required: true
          schema:
            $ref: "#/definitions/City"
      responses:
        200:
          description: 查询成功
          schema:
            $ref: '#/definitions/ApiCityPageResponse'
```

## 2.3 SwaggerUI

SwaggerUI是用来展示Swagger文档的界面，以下为安装步骤

（1）在本地安装nginx  

（2）下载SwaggerUI源码   https://swagger.io/download-swagger-ui/

（3）解压，将dist文件夹下的全部文件拷贝至 nginx的html目录

（4）启动nginx

（5）浏览器打开页面  http://localhost即可看到文档页面 .

（6）我们将编写好的yml文件也拷贝至nginx的html目录，这样我们就可以加载我们的swagger文档了









































