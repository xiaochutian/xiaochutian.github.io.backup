---
layout:     post
title:      "搭建Elasticsearch集群"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Elasticsearch
    - Cluster
    - Setup
    - 运维
---

## 0 一篇一句

**爱一个人之前，请先自给自足**    

## 1 概述

#### 1.1 要求

搭建一个多节点的[Elasticsearch](https://www.elastic.co/products/elasticsearch)集群，并且安装相关插件。

#### 1.2 软件版本

|--
| 组件 | 版本
|-|-
| Elasticsearch | 2.3.3
| Kibana | 4.5.1
| elasticsearch-analysis-jieba | 2.3.3

#### 1.3 服务器列表

|--
| 编号 | IP
|-|-
| Server1 | 10.3.122.36
| Server2 | 10.3.122.35

#### 1.4 好用的工具和插件

|---
| 编号 | 工具或插件 | 作用
|-|-|-
| 1 | [Kibana](https://www.elastic.co/products/kibana) | Elastic的可视化工具
| 2 | [Sense](https://www.elastic.co/guide/en/sense/current/index.html) | Elasticsearch的DSL查询工具
| 3 | [Jieba](https://github.com/huaban/elasticsearch-analysis-jieba) | 中文分词插件
| 4 | [delete-by-query](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/delete-by-query-usage.html) | Elasticsearch根据query删除文档的插件
| 5 | [elasticsearch-head](https://mobz.github.io/elasticsearch-head/) | 集群管理工具
| 6 | [bigdesk](https://github.com/hlstudio/bigdesk) | Elasticsearch集群监控工具
| 7 | [servicewrapper](https://github.com/elastic/elasticsearch-servicewrapper) | Elasticsearch的服务化组件
| 8 | [Elasticsearch-SQL](https://github.com/NLPchina/elasticsearch-sql) | 让Elasticsearch支持SQL语法的查询

Elasticsearch 2.3 插件管理：[https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/index.html](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/index.html)

#### 1.5 步骤

1. 下载安装elasticsearch
2. 下载安装kibana
3. 使用kibana，安装es的sense查询
4. 使用es，安装es的结巴分词插件
5. 使用es，安装es的delete-by-query插件
6. 使用es，安装es的集群状态监控插件
7. 安装es的bigdesk插件
8. 安装es的servicewrapper插件

## 2 详细步骤

#### 2.1 下载安装elasticsearch
Elasticsearch安装目录：/data/devtools/elasticsearch    

1. 下载解压
2. 创建backups目录
3. 修改配置文件 config/elasticsearch.yml

###### 2.1.1 下载解压，并创建backups目录
```bash
# 创建 Elasticsearch 的安装目录，下载并解压
mkdir -p /data/devtools
cd /data/devtools
wget "https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.3.3/elasticsearch-2.3.3.tar.gz"
tar zxvf elasticsearch-2.3.3.tar.gz
ln -s elasticsearch-2.3.3 elasticsearch

# 创建 backups 目录，并编辑 config/elasticsearch.yml 文件（详见本文 2.1.2节）
cd /data/devtools/elasticsearch
mkdir backups
cd config
mv elasticsearch.yml elasticsearch.yml.bk
vim elasticsearch.yml
```

###### 2.1.2 Server1和Server2的 config/elasticsearch.yml 配置

> 说明：两个配置文件 **不同** 的地方：node.name和 network.host    
discovery.zen.ping.unicast.hosts两个配置文件 **相同**，通过这个地址来发现集群中的机器    

**配置详情如下**

```bash
# 10.3.122.36上的配置文件/data/devtools/elasticsearch/config/elasticsearch.yml

# 集群相关配置
cluster.name: leappDev-es
node.name: leappDev-es-node-1
path.repo: /data/devtools/elasticsearch/backups
network.host: 10.3.122.36
index.analysis.analyzer.default.type: jieba

http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.zen.ping.unicast.hosts: ["10.3.122.36"]

# 让查询支持Groovy脚本，function score使用
script.groovy.sandbox.enabled: true
script.inline: on
script.indexed: on
script.search: on
script.engine.groovy.inline.aggs: on

# 这段配置，视实际情况而定（慢查询日志的配置）
index.search.slowlog.threshold.query.warn: 1000ms
index.search.slowlog.threshold.query.info: 100ms
index.search.slowlog.threshold.query.debug: 20ms
index.search.slowlog.threshold.query.trace: 1ms

index.search.slowlog.threshold.fetch.warn: 500s
index.search.slowlog.threshold.fetch.info: 200ms
index.search.slowlog.threshold.fetch.debug: 50ms
index.search.slowlog.threshold.fetch.trace: 1ms

index.indexing.slowlog.threshold.index.warn: 10s
index.indexing.slowlog.threshold.index.info: 5s
index.indexing.slowlog.threshold.index.debug: 2s
index.indexing.slowlog.threshold.index.trace: 500ms
index.indexing.slowlog.level: info
index.indexing.slowlog.source: 1000

# 默认的慢查询日志的配置
# index.search.slowlog.threshold.query.warn: 10s
# index.search.slowlog.threshold.query.info: 5s
# index.search.slowlog.threshold.query.debug: 2s
# index.search.slowlog.threshold.query.trace: 500ms
#
# index.search.slowlog.threshold.fetch.warn: 1s
# index.search.slowlog.threshold.fetch.info: 800ms
# index.search.slowlog.threshold.fetch.debug: 500ms
# index.search.slowlog.threshold.fetch.trace: 200ms
```

```bash
# 10.3.122.35上的配置文件/data/devtools/elasticsearch/config/elasticsearch.yml

# 集群相关配置
cluster.name: leappDev-es
node.name: leappDev-es-node-2
path.repo: /data/devtools/elasticsearch/backups
network.host: 10.3.122.35
index.analysis.analyzer.default.type: jieba

http.cors.enabled: true
http.cors.allow-origin: "*"

discovery.zen.ping.unicast.hosts: ["10.3.122.36"]

 # 让查询支持Groovy脚本，function score使用
script.groovy.sandbox.enabled: true
script.inline: on
script.indexed: on
script.search: on
script.engine.groovy.inline.aggs: on

# 这段配置，视实际情况而定（慢查询日志的配置）
index.search.slowlog.threshold.query.warn: 1000ms
index.search.slowlog.threshold.query.info: 100ms
index.search.slowlog.threshold.query.debug: 20ms
index.search.slowlog.threshold.query.trace: 1ms

index.search.slowlog.threshold.fetch.warn: 500s
index.search.slowlog.threshold.fetch.info: 200ms
index.search.slowlog.threshold.fetch.debug: 50ms
index.search.slowlog.threshold.fetch.trace: 1ms

index.indexing.slowlog.threshold.index.warn: 10s
index.indexing.slowlog.threshold.index.info: 5s
index.indexing.slowlog.threshold.index.debug: 2s
index.indexing.slowlog.threshold.index.trace: 500ms
index.indexing.slowlog.level: info
index.indexing.slowlog.source: 1000

# 默认的慢查询日志的配置
# index.search.slowlog.threshold.query.warn: 10s
# index.search.slowlog.threshold.query.info: 5s
# index.search.slowlog.threshold.query.debug: 2s
# index.search.slowlog.threshold.query.trace: 500ms
#
# index.search.slowlog.threshold.fetch.warn: 1s
# index.search.slowlog.threshold.fetch.info: 800ms
# index.search.slowlog.threshold.fetch.debug: 500ms
# index.search.slowlog.threshold.fetch.trace: 200ms
```

#### 2.2 下载安装kibana

```bash
cd /data/devtools
wget "https://download.elastic.co/kibana/kibana/kibana-4.5.1-linux-x64.tar.gz"
tar zxvf kibana-4.5.1-linux-x64.tar.gz
ln -s kibana-4.5.1-linux-x64.tar.gz kibana
```

#### 2.3 使用kibana，安装es的sense查询

```bash
cd /data/devtools/kibana
./bin/kibana plugin --install elastic/sense
```

#### 2.4 使用es，安装es的结巴分词插件

```bash
cd /data/devtools/elasticsearch
bin/plugin install https://github.com/huaban/elasticsearch-analysis-jieba/releases/download/v2.3.3/elasticsearch-analysis-jieba-2.3.3-bin.zip
```

#### 2.5 使用es，安装es的delete-by-query插件

```bash
cd /data/devtools/elasticsearch
bin/plugin install delete-by-query
```

#### 2.6 使用es，安装es的集群状态监控插件

```bash
cd /data/devtools/elasticsearch
bin/plugin install mobz/elasticsearch-head
```

#### 2.7 安装es的bigdesk插件

详见[【Elasticsearch插件Bigdesk安装】](/2017/08/18/setting-up-elasticsearch-plugin-bigdesk/){:target="_blank_"}

#### 2.8 安装es的servicewrapper插件

详见[【Elasticsearch插件servicewrapper安装】](/2017/08/18/setting-up-elasticsearch-plugin-servicewrapper/){:target="_blank_"}


## 3 状态查看

Kibana地址：[http://10.3.122.36:5601/app/kibana](http://10.3.122.36:5601/app/kibana)    
Sense地址：[http://10.3.122.36:5601/app/sense](http://10.3.122.36:5601/app/sense)    
ES集群监控：[http://10.3.122.36:9200/_plugin/head/](http://10.3.122.36:9200/_plugin/head/)    
ES集群状态JSON：[http://10.3.122.36:9200/_cluster/stats](http://10.3.122.36:9200/_cluster/stats)    
ES集群健康状态JSON：[http://10.3.122.36:9200/_cluster/health?pretty](http://10.3.122.36:9200/_cluster/health?pretty)    
ES节点状态JSON：[http://10.3.122.36:9200/_nodes/process?pretty](http://10.3.122.36:9200/_nodes/process?pretty)    
ES节点状态JSON：[http://10.3.122.36:9200/_nodes/stats?pretty](http://10.3.122.36:9200/_nodes/stats?pretty)    
ES的pending_tasks：[http://10.3.122.36:9200/_cluster/pending_tasks](http://10.3.122.36:9200/_cluster/pending_tasks)    

## 4 启动与停止

```bash
# ES start
cd /data/devtools/elasticsearch && ./bin/elasticsearch -d
# ES stop
kill -9 XXX

# kibana start
cd /data/devtools/kibana && nohup ./bin/kibana -e 'http://10.3.122.36:9200/' > kibana.log &
# kibana stop
kill -9 XXX

# 在ES安装了servicewrapper插件之后，可以使用service命令启动和重启
service elasticsearch start
service elasticsearch stop
```

## 5 参考

[【elastic】Kibana](https://www.elastic.co/products/kibana)    
[【elastic】Elasticsearch](https://www.elastic.co/products/elasticsearch)    
[【elastic】Installing and Running Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/guide/current/running-elasticsearch.html)    
[【elastic】Sense](https://www.elastic.co/guide/en/sense/current/index.html)    
[【elastic】Elasticsearch Plugins and Integrations](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/index.html)    
[【elastic】Delete By Query Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/2.0/plugins-delete-by-query.html)    
[【elastic】Using Delete-by-Query](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/delete-by-query-usage.html)    
[【elastic】Configuring Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)    

[【GitHub】huaban/elasticsearch-analysis-jieba](https://github.com/huaban/elasticsearch-analysis-jieba)    
[【GitHub】elasticsearch-head](https://mobz.github.io/elasticsearch-head/)    
[【GitHub】hlstudio/bigdesk](https://github.com/hlstudio/bigdesk)    
[【GitHub】elastic/elasticsearch-servicewrapper](https://github.com/elastic/elasticsearch-servicewrapper)    
[【GitHub】NLPchina/elasticsearch-sql](https://github.com/NLPchina/elasticsearch-sql)    

[【Official website】Bigdesk](http://bigdesk.org/)    
[【Loggly】9 tips on ElasticSearch configuration for high performance](https://www.loggly.com/blog/nine-tips-configuring-elasticsearch-for-high-performance/)    
[【CSDN】elasticsearch.yml基本配置说明](http://blog.csdn.net/lu_wei_wei/article/details/51263153)    
[【CSDN】搭建Elasticsearch 5.4分布式集群](http://blog.csdn.net/napoay/article/details/52202877)    
