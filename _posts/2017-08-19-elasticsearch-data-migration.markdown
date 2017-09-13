---
layout:     post
title:      "Elasticsearch数据迁移"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Elasticsearch
    - 运维
---

## 0 一篇一句

**Later Equals Never**    

## 1 概述

云服务器整体从『云服务商A』迁移到『云服务商B』。    
所以，基于Elasticsearch的搜索服务也得迁移。

#### 1.1 需要做的工作

1. 迁移服务：在『云服务商B』部署分布式的Elasticsearch集群
2. 迁移数据：从『云服务商A』导出，并导入『云服务商B』
3. 修改配置：迁移后，搜索服务的地址需要更新为『云服务商B』上的Elasticsearch的地址

#### 1.2 服务器情况

云服务商A：10.19.118.18（Centos7.0 64位，8核16G，20GB系统盘+200GB SSD）    
云服务商B：10.3.122.36,10.3.122.35（2核8G，50GB硬盘）

> 单节点服务，变成集群服务

#### 1.3 需要迁移的数据

- amaze_20170228（线上库）
- amaze_20170512（下个版本的线上库）
- amaze_search_20170220（搜索历史库）

## 2 迁移步骤

1. 保证可以同时访问两个云服务商的网络
2. 安装迁移工具[elasticdump](https://www.npmjs.com/package/elasticdump)
3. 编写迁移脚本
4. 执行数据迁移
5. 在云服务商B的Elasticsearch上创建别名
6. 验证正确性和完整性

#### 2.1 保证可以同时访问两个云服务商的网络

做法：给云服务商A的机器开启外网IP，并连接云服务商B的VPN

#### 2.2 安装迁移工具elasticdump

```bash
# 安装elasticdump
cd /data/devtools/elasticsearch
sudo npm install elasticdump -g
```

#### 2.3 编写迁移脚本

```bash
# 编写dump脚本
vim dump_data_from_ucloud.sh
#!/bin/bash
if [ $# -lt 1 ]; then
    echo "USAGE: sh dump_data_from_ucloud.sh index_name"
    exit 1
fi
index_name=$1
elasticdump \
  --input=http://123.59.54.15:9200/${index_name} \
  --output=http://10.3.122.36:9200/${index_name} \
  --type=analyzer
elasticdump \
  --input=http://123.59.54.15:9200/${index_name} \
  --output=http://10.3.122.36:9200/${index_name} \
  --type=mapping
elasticdump \
  --input=http://123.59.54.15:9200/${index_name} \
  --output=http://10.3.122.36:9200/${index_name} \
  --type=data
```

#### 2.4 执行数据迁移

```bash
# 执行脚本，迁移线上库（amaze_20170228）和新版的库，vid放在_id里，并且只保存title, tags, desp, depracated_word（amaze_20170512）
sh dump_data_from_ucloud.sh amaze_20170228
sh dump_data_from_ucloud.sh amaze_20170512
sh dump_data_from_ucloud.sh amaze_search_20170220
```

#### 2.5 在云服务商B的Elasticsearch上创建别名

打开 http://10.3.122.36:5601/app/sense 的控制台，执行创建alias

```
# Sense的DSL查询
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "amaze_20170228",
        "alias": "amaze_product"
      },
      "add": {
        "index": "amaze_search_20170220",
        "alias": "amaze_search"
      }
    }
  ]
}
```

#### 2.6 验证正确性和完整性

由 2.6.1节的正确性验证 和 2.6.2节的完整性验证，可以证明迁移成功！

###### 2.6.1 正确性验证

```
# Sense的DSL查询
GET amaze_20170512/video/_search
GET amaze_product/video/_search
GET amaze_search/history/_search
```

> 查询可以正确执行，并返回数据

###### 2.6.2 完整性验证

对比：在『云服务商A上的Elasticsearch』和『云服务商B的Elasticsearch』上，**需要迁移的库中的文档数量**。    
分别在两个云服务商的Elasticsearch的Sense平台执行以下查询，并进行对比。

```
GET amaze_20170512/video/_count
GET amaze_product/video/_count
GET amaze_search/history/_count
```

> 发现两个云服务商各个库的的文档数量分别对应相同

## 3 失败的尝试

使用[【Elasticsearch-Exporter】](https://github.com/mallocator/Elasticsearch-Exporter)，进行数据迁移。    
先从云服务商A的ES导出到文件，再导入到云服务商B的ES

#### 3.1 安装Elasticsearch-Exporter

npm install elasticsearch-exporter --production    
npm安装的是1.4版本，1.4版本，不能导出到文件（2.0版本才有这个功能）。    
尝试，给云服务商A开个外网，直接用1.4版本导。   

```bash
# 安装Elasticsearch-Exporter
npm install elasticsearch-exporter --production
cd node_modules/elasticsearch-exporter/
```

#### 3.2 使用Elasticsearch-Exporter迁移数据

偶尔可以成功的导出一些数据，但是，总不能完整的导出整个库的数据。

```bash

# 好像偶尔可以成功一点点，但是会莫名中断
[root@10-19-118-18 elasticsearch-exporter]# node exporter.js -a localhost -i amaze_product -t video -g amaze_product
Elasticsearch Exporter - Version 1.4.0
Reading source statistics from ElasticSearch
Reading mapping from ElasticSearch
Storing type mapping in meta file amaze_product.meta
Mapping is now ready. Starting with 0 queued hits.
Processed 100 of 63122 entries (0%)
The source driver has not reported any documents that can be exported. Not exporting.
Number of calls:    3
Fetched Entries:    100 documents
Processed Entries:  100 documents
Source DB Size:     63122 documents
```

## 4 参考

[【kiyanpro】Export Index Using Elasticsearch Dump](http://blog.kiyanpro.com/2016/03/11/elasticsearch/Export-Index-Using-Elasticsearch-Dump/)    
[【GitHub】taskrabbit/elasticsearch-dump](https://github.com/taskrabbit/elasticsearch-dump)    
[【npm】elasticdump](https://www.npmjs.com/package/elasticdump)    
[【GitHub】mallocator/Elasticsearch-Exporter](https://github.com/mallocator/Elasticsearch-Exporter)    
[【npm】elasticsearch-exporter](https://www.npmjs.com/package/elasticsearch-exporter)    
[【CSDN】Elasticsearch索引迁移的三种方式](http://blog.csdn.net/laoyang360/article/details/65449407)    
