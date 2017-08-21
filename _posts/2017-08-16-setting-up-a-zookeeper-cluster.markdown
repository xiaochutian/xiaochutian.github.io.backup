---
layout:     post
title:      "Zookeeper安装"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Zookeeper
    - Setup
---

## 0 一篇一句

**君生我未生**    

## 1 概述

#### 1.1 Zookeeper是什么

> ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

#### 1.1 安装要求
- 三台服务器（Zookeeper需要奇数个节点）
- 服务器安装好java 1.6以上版本

#### 1.2 服务器IP

```
10.1.14.160
10.2.87.190
10.3.122.28
```

## 2 安装步骤

1. 下载zookeeper稳定版本（[3.4.6版本](http://apache.fayea.com/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz)）。[所有版本列表](http://zookeeper.apache.org/releases.html)
2. 解压
3. 编辑配置文件conf/zoo.cfg
4. 创建data目录和log目录。（与zoo.cfg中配置的一致）
5. 修改hosts文件
6. 在data目录下，添加与机器对应的myid文件
7. 开启zk

```bash
# 修改hosts
vim /etc/hosts
10.1.14.160 zoo1
10.2.87.190 zoo2
10.3.122.28 zoo3

# 创建zk安装目录
mkdir -p /data/zk
cd /data/zk

# 下载zk稳定版本
wget "http://apache.fayea.com/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz"
tar zxvf zookeeper-3.4.6
cd zookeeper-3.4.6

# 创建数据目录和日志目录
mkdir data
mkdir log

# 创建myid文件
# 这个1是机器的ID，与zoo.cfg中『server.1』对应
# 在zoo1上是1，在zoo2上是2，在zoo3是3
echo "1" >> data/myid

# 创建zk配置文件
vim conf/zoo.cfg
tickTime=2000
dataDir=/data/zk/zookeeper-3.4.6/data
dataLogDir=/data/zk/zookeeper-3.4.6/log
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888

# 开启并测试状态
cd bin
sh zkServer.sh start
sh zkServer.sh status
```

## 3 参考

[【百度百科】Zookeeper](https://baike.baidu.com/item/zookeeper/4836397?fr=aladdin)       
[【Apache】Zookeeper Docs](https://zookeeper.apache.org/doc/r3.4.6/zookeeperStarted.html)       
