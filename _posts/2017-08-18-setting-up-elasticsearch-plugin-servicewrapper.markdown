---
layout:     post
title:      "Elasticsearch插件servicewrapper安装"
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

**我亦无他，唯手熟尔**    

## 1 概述

**servicewrapper可以让运维人员用service命令来管理Elasticsearch**    

> 虽然，[官方GitHub](https://github.com/elastic/elasticsearch-servicewrapper)上说明不支持Elasticsearch 2.x    
但是，参考『CSDN-天纯蓝』的博客[『Elasticsearch-2.3.x填坑之路』](http://www.cnblogs.com/skyblue/p/5504595.html)以让servicewrapper支持Elasticsearch 2.3.x

## 2 安装过程

```bash
# 下载工程，把service目录copy到${ES_HOME}/bin下面。并安装service
cd /data/devtools/elasticsearch
mkdir downloads
cd downloads
git clone https://github.com/elastic/elasticsearch-servicewrapper.git
cp -r elasticsearch-servicewrapper/service ../bin/
sudo /data/devtools/elasticsearch/bin/service/elasticsearch install

# ！！！修改service目录下的elasticsearch.conf文件，开始！！！ #
### 修改其中的ES_HOME和ES_HEAP_SIZE 第1，2行 ###
vim /data/devtools/elasticsearch/bin/service/elasticsearch.conf
# 原始
set.default.ES_HOME=<Path to Elasticsearch Home>
set.default.ES_HEAP_SIZE=1024
#改后
set.default.ES_HOME=/data/devtools/elasticsearch
set.default.ES_HEAP_SIZE=6144

### 修改启动类 第58行 ###
# 原始
wrapper.app.parameter.1=org.elasticsearch.bootstrap.ElasticsearchF
# 改后
wrapper.app.parameter.1=org.elasticsearch.bootstrap.Elasticsearch
wrapper.app.parameter.2=start

### 修改root启动权限 第49行 ###
# 追加一行：
wrapper.java.additional.10=-Des.insecure.allow.root=true
# ！！！修改service目录下的elasticsearch.conf文件，结束！！！ #

# 设置elasticsearch的security。修改elasticsearch.yml文件，添加配置（追加）
vim /data/devtools/elasticsearch/config/elasticsearch.yml
security.manager.enabled: false

# 用service目录下的elasticesearch进行启动和重启（安装了服务之后，可以直接用service命令）
service elasticsearch stop
service elasticsearch start
```

## 3 可用命令

|---
| Parameter	| Description
|-|-
| console	| Run the elasticsearch in the foreground.
| start	| Run elasticsearch in the background.
| stop	| Stops elasticsearch if its running.
| install	| Install elasticsearch to run on system startup (init.d / service).
| remove	| Removes elasticsearch from system startup (init.d / service).

## 4 停止和启动

```bash
service elasticsearch stop
service elasticsearch start
```

> 注1：在首次安装之后，直接使用service elasticsearch stop的话，会失败。因为，之前没有用service启动，没有记录PID。    
解决：先kill，然后再用service elasticesearch start。之后，就可以用service elasticsearch stop    
注2：服务的安装需要sudo，使用不需要sudo

## 5 参考

[【GitHub】elastic/elasticsearch-servicewrapper](https://github.com/elastic/elasticsearch-servicewrapper)    
[【CSDN】Elasticsearch-2.3.x填坑之路](http://www.cnblogs.com/skyblue/p/5504595.html)    
