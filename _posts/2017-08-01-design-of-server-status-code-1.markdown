---
layout:     post
title:      "服务器状态码设计（一）"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Server
    - Design
---

## 0 一篇一句

**每多学一点知识，就少写一行代码**

## 1 概述

#### 1.1 服务器状态码介绍

服务器状态码，用于表示服务器状态。返回给客户端之后，客户端可以通过状态码 **『判断请求是否成功执行』** ，或者根据状态码查找 **『请求失败的原因』** 。

#### 1.2 状态码返回的形式

状态码有多种表现形式。可以复用HTTP的状态码[【rfc2616】](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)[【wikipedia】](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)，也可以在自定义的协议中加入状态码。

#### 1.3 修改之前的状态

服务器的响应结果封装到一个Result对象里面，包含：**code**, **msg** 和 **data**  。    
状态码（code）为0表示正常返回，非0表示异常。

###### 1.3.1 Result类

以下，为两种不同的Result类的格式：    
一种是使用了泛型来限制data类型，另一种是使用Object类型的data。    
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result implements Serializable {
    private static final long serialVersionUID = -5460510668301615639L;
    private Integer code;
    private String msg;
    private Object data;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> implements Serializable {
    private static final long serialVersionUID = 5460510668301615639L;
    private Integer code;
    private String msg;
    private T data;
}
```

> 注：@Data, @NoArgsConstructor, @AllArgsConstructor这三个注解是lombok项目注解。    
> 可以取代Getter和Setter，还有一些基础的构造函数等。

###### 1.3.2 Result类属性说明

|---
| 编号 | 属性 | 说明
|-|-|-
| 1 | code | 状态码
| 2 | msg | 状态码对应的解释
| 3 | data | 返回的数据

###### 1.3.3 Result实例

```json
{
  "msg": "success",
  "code": 0,
  "data": [
    {
      "type": 1,
      "date": "2017-06-23",
      "rightCount": 15,
      "wrongCount": 6,
      "duration": 300
    }
  ]
}
```

## 2 借鉴的例子

#### 2.1 Facebook

自定义错误代码，HTTP状态码统一为200。
```
{
  "error": {
    "message": "(#803) Some of the aliases you requested do not exist: me1",
    "type": "OAuthException",
    "code": 803
  }
}
```

#### 2.2 新浪

使用自定义错误代码，错误的HTTP状态码是400。[『详见』](http://open.weibo.com/wiki/Error_code)

#### 2.3 腾讯
使用自定义错误代码，错误的HTTP状态码统一是200。[『详见』](http://wiki.open.qq.com/wiki/%E5%85%AC%E5%85%B1%E8%BF%94%E5%9B%9E%E7%A0%81%E8%AF%B4%E6%98%8E)

#### 2.4 阿里巴巴

自定义错误代码，不需要关心HTTP状态码。
```
{
  "error": {
    "message": "(#803) Some of the aliases you requested do not exist: me1",
    "type": "OAuthException",
    "code": 803
  }
}
```

## 3 我的设计 {#myDesign}

**自定义错误代码，HTTP状态码统一为200。**

#### 3.1 我的思考 {#noHttpStatusCode}

为什么我不复用HTTP状态码？
1. HTTP状态码可能不够用。跟项目业务相关的错误细节无法反应出来。
2. HTTP状态码应该属于HTTP协议的定义之中。项目中服务器与客户端的协议，应该是HTTP的上层协议。上层协议不应该知道和依赖底层协议的细节。

#### 3.2 协议设计

HTTP状态码统一为200。Result对象里面的code为0表示正常返回，非0表示异常返回。    
异常详情如下：
1. 4XXXX，表示客户端错误
2. 5XXXX，表示服务器（api）错误
3. 6XXXX，表示服务器（dubbo）错误

###### 3.2.1 客户端错误

|---
| code | 说明
|-|-
| 40000 | 客户端错误，通用
| 40001 | 客户端参数非法
| 40002 | 客户端参数json不符合格式要求

###### 3.2.2 服务器（api）错误

|---
| code | 说明
|-|-
| 50000 | 服务器（api）错误，通用

###### 3.2.3 服务器（dubbo）错误

|---
| code | 说明
|-|-
| 60000 | 服务器（dubbo）错误，通用

## 4 其他思考

问：是不是所有的API（包括服务器与客户端交互的，**也包括服务器内部的** ），都用Result<T>封装？

**【优势】**    
1.可以统一错误码和错误消息的处理，可以透传。    
2.可以快速定位异常位置，大大提高debug效率。

**【劣势】**    
1.这样相当于把所有后端的错误码，都返回给了前端，直接在页面上显示。暴露了过多信息。    
2.会牺牲后端的性能。

**【折衷】**    
1.ErrorCode还是照样按照规则设计。但是，不用全部接口使用Result封装，并且透传。而是，在报错的地方，logger.error()打印出状态码和报错信息。    
2.在api的部分，隐藏服务器报错详情。统一返回给客户端模糊处理后的报错信息和ErrorCode。

## 5 参考
[【微信】公众号的状态码](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1433747234)    
[【淘宝】开放平台-错误码一览表](https://open.taobao.com/doc2/detail.htm?articleId=114&docType=1&treeId=null)    
[【腾讯】开放平台-公共返回码](http://wiki.open.qq.com/wiki/%E5%85%AC%E5%85%B1%E8%BF%94%E5%9B%9E%E7%A0%81%E8%AF%B4%E6%98%8E)    
[【微博】开放平台-ErrorCode说明](http://open.weibo.com/wiki/Error_code)    
[【狐狸糊涂】RESTful实践：如何设计API的错误消息](https://my.oschina.net/foxty/blog/382344)    
[【wikipedia】List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)    
[【w3】RFC2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)    
[【知乎】在API程序设计开发中错误码如何规划设计](https://www.zhihu.com/question/24091286)    
[【北漂周】随意定义错误码，你还在这样干？](http://blog.csdn.net/yzzst/article/details/54799971)    
[【Scarletsky】Restful API 中的错误处理](https://scarletsky.github.io/2016/11/30/error-handling-in-restful-api/)    
[【简书】Restful API设计思路及实践](http://www.jianshu.com/p/265397f812d4)    
[【阮一峰】RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)    
[【w3ctech】学习设计接口api](https://www.w3ctech.com/topic/1348)
