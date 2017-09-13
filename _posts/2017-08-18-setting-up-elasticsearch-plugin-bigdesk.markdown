---
layout:     post
title:      "Elasticsearch插件Bigdesk安装"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Elasticsearch
    - plugin
---

## 0 一篇一句

**有个同事质疑说我写这些东西没意义。我不以为然。    
写得不好就更加需要练习，不能选择逃避。    
有了量变的积累，才有可能有质变的突破。**    

## 1 概述

> Bigdesk 是一个 ES 集群监控工具，可以检测到集群状态、各节点信息，包括 JVM、Thread Pools、OS、Process、Indices 等信息

## 2 Bigdesk与Elasticsearch版本对应表

|---
| Bigdesk | Elasticsearch
|-|-
| 2.5.0	| 1.3.0 ... 1.3.x
| 2.4.1 (2.4.0)	| 1.0.0.RC1 ... 1.2.x
| n/a	| 1.0.0.Beta1 ... 1.0.0.Beta2
| 2.2.3	| 0.90.10 ... 0.90.x
| 2.2.2 (2.2.1) | 0.90.0 ... 0.90.9
| 2.1.0	| 0.20.0 ... 0.20.x
| 2.0.0	| 0.19.0 ... 0.20.x
| 1.0.0	| 0.17.0 ... 0.18.x

## 3 安装

#### 3.1 Elasticsearch 1.3.x 以下版本（包含1.3.x）的安装方式

```bash
# ${ES_HOME}代表ES安装的根目录
cd ${ES_HOME}
./bin/plugin -install lukas-vlcek/bigdesk/<bigdesk_version>
```

#### 3.2 Elasticsearch 1.3.x 以上版本（不包含1.3.x）的安装方式

```bash
# 下载并解压bigdesk到plugins目录下
cd /data/devtools/elasticsearch/plugins
wget https://github.com/lukas-vlcek/bigdesk/archive/master.zip
unzip master.zip
mkdir -p bigdesk/_site
mv bigdesk-master/* bigdesk/_site/
rm -rf bigdesk-master
rm master.zip

# 修改配置，添加ES的插件描述文件
cd bigdesk
vim plugin-descriptor.properties
description = bigdesk
version = 2.5.0
name = bigdesk
site = true

# 修改bigdesk文件中的版本判断
vim _site/js/store/BigdeskStore.js
# 修改142行，把『major == 1 &&』删除
# 原来：142         return (major == 1 && minor >= 0 && maintenance >= 0 && (build != 'Beta1' || build != 'Beta2'));
# 改后：142         return (minor >= 0 && maintenance >= 0 && (build != 'Beta1' || build != 'Beta2'));

# 重启ES
kill -9 ${ES_PID}
cd /data/devtools/elasticsearch
bin/elasticsearch -d

# 测试插件加载成功
bin/plugin list
```

## 4 相关地址

#### 4.1 Bigdesk插件地址

http://10.3.122.36:9200/_plugin/bigdesk/#nodes    
http://10.3.122.35:9200/_plugin/bigdesk/#nodes

###### 4.1.1 JVM和Thread Pools监控

![1](/img/in-post/post-setting-up-elasticsearch-plugin-bigdesk/1.png)    

###### 4.1.2 OS、Process和HTTP & Transport监控

![2](/img/in-post/post-setting-up-elasticsearch-plugin-bigdesk/2.png)    

###### 4.1.3 Indices和File system监控

![3](/img/in-post/post-setting-up-elasticsearch-plugin-bigdesk/3.png)

#### 4.2 Bigdesk的官网和github地址

官网地址：[http://bigdesk.org/](http://bigdesk.org/)    
github地址：[https://github.com/lukas-vlcek/bigdesk](https://github.com/lukas-vlcek/bigdesk)

#### 4.3 重点参考

[【CSDN】小怪兽的技术博客 - Elasticsearch 2.4.1 Bigdesk 插件安装](http://www.cnblogs.com/wangxiaoqiangs/p/6430354.html)

## 5 参考

[【Bigdesk】官网](http://bigdesk.org/)    
[【Bigdesk】github](https://github.com/lukas-vlcek/bigdesk)    
[【CSDN】Elasticsearch 2.4.1 Bigdesk 插件安装](http://www.cnblogs.com/wangxiaoqiangs/p/6430354.html)    
[【21运维】Elasticsearch5.X elasticsearch-head插件和bigdesk安装](http://www.21yunwei.com/archives/5285)    
