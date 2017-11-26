---
layout:     post
title:      "操作系统的大小写敏感性"
subtitle:   "引发的问题及解决方案"
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - OS
    - File System
---

## 0 一篇一句

> The Only Thing That Is Constant Is Change     
                                —— Heraclitus

## 1 问题

#### 1.1 nodejs后端部署报错

最近，在学习用Vue做公司CMS（Content Management System）的前端，并且，使用expressjs来写对应的后端。我在本机（MacOS）写好功能之后，尝试使用jenkins部署到服务器（CentOS）上。但是，同样的代码，在我的Mac上可以运行，部署到服务器后运行不了。报错如下：

```javascript
module.js:471
    throw err;
    ^

Error: Cannot find module 'Mongoose-sequence'
    at Function.Module._resolveFilename (module.js:469:15)
    at Function.Module._load (module.js:417:25)
    at Module.require (module.js:497:17)
    at require (internal/module.js:20:19)
    at Object.<anonymous> (/home/service/server/db/schema/CommentModel.js:4:49)
    at Module._compile (module.js:570:32)
    at Object.Module._extensions..js (module.js:579:10)
    at Module.load (module.js:487:32)
    at tryModuleLoad (module.js:446:12)
    at Function.Module._load (module.js:438:3)
    at Module.require (module.js:497:17)
    at require (internal/module.js:20:19)
    at Object.<anonymous> (/home/service/server/db/helper/CommentHelper.js:3:22)
    at Module._compile (module.js:570:32)
    at Object.Module._extensions..js (module.js:579:10)
    at Module.load (module.js:487:32)
```

/home/service/server/db/schema/CommentModel.js的代码如下
```javascript
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
var db_1 = require("../db");
var AutoIncrement = require('Mongoose-sequence')(db_1.default);
var Schema = db_1.default.Schema;
var CommentSchema = new Schema({
    _id: Number,
    nextNodeId: Number,
    content: Object,
    history: Object,
}, {
    _id: false,
    versionKey: false,
    timestamps: true
});
CommentSchema.plugin(AutoIncrement, { inc_field: '_id' });
exports.default = db_1.default.model('comment', CommentSchema);
//# sourceMappingURL=CommentModel.js.map
```


报错信息显示，找不到『Mongoose-sequence』这个模块。但是，明明package.json和node_modules里面都有这个组件。

```json
{
  "name": "server",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@types/body-parser": "^1.16.8",
    "@types/multer": "^1.3.6",
    "body-parser": "^1.18.2",
    "cors": "^2.8.4",
    "deep-object-diff": "^1.0.4",
    "express": "^4.16.2",
    "mongoose": "^4.13.4",
    "mongoose-sequence": "^4.0.1",
    "multer": "^1.3.0",
    "nodemon": "^1.12.1",
    "reflect-metadata": "^0.1.10",
    "routing-controllers": "^0.7.6"
  },
  "devDependencies": {
    "@types/express": "^4.0.39",
    "nodemon": "^1.12.1"
  }
}

```

#### 1.2 报错原因猜想

###### 1.2.1 jenkins部署导致的问题

*『排除方法』*：直接从gitlab上clone代码到服务器，并尝试启动。    
*『结果』*：仍然无法启动，说明不是jenkins部署导致的问题。

###### 1.2.2 各种软件的版本的问题

*『排除方法』*：把两个环境的版本号统一。    
*『结果』*：仍然无法启动，说明不是软件版本导致的问题。

> 注：依赖的安装，使用cnpm，cnpm依赖了npm和nodejs。   

```bash
# 本机MacOS的各软件版本
➜  xiaochutian.github.io git:(dev) ✗ cnpm -v
cnpm@5.1.1 (/usr/local/lib/node_modules/cnpm/lib/parse_argv.js)
npm@5.5.1 (/usr/local/lib/node_modules/cnpm/node_modules/npm/lib/npm.js)
node@8.9.0 (/usr/local/Cellar/node/8.9.0/bin/node)
npminstall@3.2.1 (/usr/local/lib/node_modules/cnpm/node_modules/npminstall/lib/index.js)
prefix=/usr/local
darwin x64 16.6.0
registry=http://registry.npm.taobao.org
```

```bash
# 服务器CentOS的各软件版本
[cm@iZuf69v86f97q1wn4lqh3jZ cm-server]$ cnpm -v
cnpm@5.1.1 (/usr/lib/node_modules/cnpm/lib/parse_argv.js)
npm@5.5.1 (/usr/lib/node_modules/cnpm/node_modules/npm/lib/npm.js)
node@8.9.1 (/usr/local/bin/node)
npminstall@3.2.1 (/usr/lib/node_modules/cnpm/node_modules/npminstall/lib/index.js)
prefix=/usr/local
linux x64 3.10.0-514.6.2.el7.x86_64
registry=http://registry.npm.taobao.org
```

###### 1.2.3 某台服务器环境的问题

*『排除方法』*：使用另一台服务器，并配置相同的环境。    
*『结果』*：仍然无法启动，说明不是某台服务器环境导致的问题。

#### 1.3 真正的问题

前一天晚上下班后，在公司折腾了一个多小时，仍然没有找到新的思路，于是决定先回家。回家之后又想这问题，是不是expressjs需要安装特殊的依赖什么的？第二天，到公司后就问了问同事，得到答案说不需要安装其他依赖。

于是，依然没有思路的我，又重新打开了错误日志。看到了找不到的那个模块的名字：**『Mongoose-sequence』**。我注意到首字母是大写，并联想到，大多数nodejs的模块在import的时候都是*全部小写*。同时，我还联想到，以前处理MySQL的表结构迁移的时候（从Windows迁移到Ubuntu），遇到过类似的由于操作系统的大小写敏感性引起的问题。

**于是，我猜想，可能是由于大小写问题导致的。**

## 2 解决

**把require中的『Mongoose-sequence』改成『mongoose-sequence』**

## 3 拓展

#### 3.1 require的工作原理

require究竟是怎么工作的？怎么根据引号里面的内容，找到对应的文件？    
详见阮一峰博文：[【require() 源码解读】](http://www.ruanyifeng.com/blog/2015/05/require.html)
的第一部分 **require() 的基本用法**

#### 3.2 操作系统的文件系统大小写

**Linux、Solaris、BSD及其他类Unix操作系统使用的是大小写敏感的文件系统，而Windows和Mac OS X（默认安装）的文件系统则是大小写不敏感的文件系统。即用文件名README、readme以及Readme（混合大小写）进行访问，在Linux等操作系统上访问的是不同的文件，而在Windows和Mac OS X上则指向同一个文件。**

我本机MacOS的磁盘文件系统信息查看：打开磁盘工具 --> 右键磁盘，查看信息
![DiskInfoOfMyMac](/img/in-post/post-case-sensitivity-of-os/disk-info-of-my-mac.jpeg)


#### 3.3 MySQL数据库中的大小写敏感性

MySQL的配置文件中，有一个『lower_case_table_names』参数，用来表示大小写策略。不同操作系统的数值以及对应的策略如下表所示：

|---
| 操作系统 | 值 | 大小写是否敏感
|-|-|-
| Linux | 0 | 敏感。0表示使用指定的大小写字母在硬盘上保存表名和数据库名，并且区分字母大小写。（保存和使用都区分大小写）
| Windows | 1 | 不敏感。1表示表名在硬盘上以小写保存，MySQL将所有表名转换为小写在存储和查找表上，不区分字母大小写。（保存和使用都不区分大小写）
| MacOS | 2 | 不敏感。2表示表名和数据库名在硬盘上使用指定的大小写字母进行保存，但MySQL将它们转换为小写在查找表上，不区分字母大小写。（保存区分大小写，使用不区分大小写）

在Linux下MySQL数据库名、表名、列名、别名大小写规则是这样的：
1. 数据库名与表名是严格区分大小写的；
2. 表的别名是严格区分大小写的；
3. 列名与列的别名在所有的情况下均是忽略大小写的；
4. 变量名也是严格区分大小写的；

> 注：Mac OS X 操作系统通常不是大小写敏感的，但是如果Mac 操作系统使用了 UFS 磁盘卷，那么这个时候是大小写敏感。

## 4 经验总结

1. 使用find + replace的时候得注意。最好，使用IDE提供的refactor功能。
2. 文件名全部使用小写字母和连词线（all-lowercase-with-dashes），是一种值得推广的正确做法。详见阮一峰博文：[【为什么文件名要小写？】](http://www.ruanyifeng.com/blog/2017/02/filename-should-be-lowercase.html)   
3. 数据库命名使用全部小写字母加下划线（all_lowercase_with_underline）

## 5 参考

[【阮一峰的网络日志】require() 源码解读](http://www.ruanyifeng.com/blog/2015/05/require.html)    
[【阮一峰的网络日志】为什么文件名要小写？](http://www.ruanyifeng.com/blog/2017/02/filename-should-be-lowercase.html)    
[【简书-东皇Amrzs】数据库大小写敏感问题](http://www.jianshu.com/p/ce3786435c46)    
[【简书-JavaQ】细说MySQL区分字母大小写](http://www.jianshu.com/p/336bbb6780d9)    
[【CSDN-zyz511919766】MySQL数据库中的大小写敏感性](http://blog.csdn.net/zyz511919766/article/details/50297815)    
[【worldhello】文件名大小写问题](http://www.worldhello.net/gotgit/08-git-misc/030-case-insensitive.html)    
[【MariaDB】标识符对大小写敏感的情况](https://mariadb.com/kb/zh-cn/identifier-case-sensitivity/)    
[【简书-CoderKo1o】深入解析Mac OS X & iOS 操作系统 学习笔记（十七）](http://www.jianshu.com/p/bddf86e31b19)    
