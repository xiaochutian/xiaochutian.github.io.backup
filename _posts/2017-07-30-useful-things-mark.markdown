---
layout:     post
title:      "有用的文章，待整理"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - 综合
---

## 0 一篇一句

**每天进步一点点**

## 1 概述

[【简书】JavaEE技术文章 ——MyBatis Mapper $与#的区别](http://www.jianshu.com/p/7481670a90e2)    
$是字符串拼接，#是占位符。
当，tableName的时候，必须使用$

```
@Select("SELECT COUNT(*) FROM ${tableName} WHERE user_id=#{userId}")
```

[【Spring ApplicationContext配置文件的加载过程】]（http://www.javacoder.cn/?p=337）

jenkins

dubbo

跳板机

配置多个机器的ssh免密

shell脚本

常用linux指令

压力测试 jmeter

jsBridge进行 前端和客户端的交互

回归测试框架

maven-shade-plugin

shutdownhook来关闭dubbo

## 5 参考

【维基百科】IEEE754标准。[英文](https://en.wikipedia.org/wiki/IEEE_floating_point)，[中文](https://zh.wikipedia.org/wiki/IEEE_754)    
[【北邮人论坛】为什么Double.MIN_VALUE设计为正数？](http://bbs.cloud.icybee.cn/article/Java/51063)    
[【北邮人论坛】double强制类型转换为int的实现过程](http://bbs.cloud.icybee.cn/article/Java/40040)    
[【北邮人论坛】double计算，看不明白，请教！](http://bbs.cloud.icybee.cn/article/Java/47757)
