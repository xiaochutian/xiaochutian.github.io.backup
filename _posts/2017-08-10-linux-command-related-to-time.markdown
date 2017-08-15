---
layout:     post
title:      "Linux指令-时间相关"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Shell
    - timestamp
---

## 0 一篇一句

**未离开，已想念**

## 1 概述

#### 1.1 缘由

写Shell脚本的时候，或多或少的会碰到与时间相关的处理，在这做个记录。

#### 1.2 相关指令

- date，用来显示或设定系统的日期与时间。[详见](http://www.runoob.com/linux/linux-comm-date.html)
- stat，以文字的格式来显示inode的内容。[详见](http://www.runoob.com/linux/linux-comm-stat.html)

## 2 举几个栗子

#### 2.1 日期与时间戳的相互转换

###### 2.1.1 日期转时间戳
```
date -d "2017-08-08" +%s
```

###### 2.1.2 时间戳转日期
```
date -d @1502035200 "+%Y-%m-%d"
```

#### 2.2 获取文件修改的日期
```
stat filename | grep -i Modify | awk -F' ' '{print $2}'    
```

【步骤】详情如下：

```shell
# 1 获取文件状态
[leappspark@emr-header-1 bin]$ stat play_rate.latest
  File: 'play_rate.latest'
  Size: 1335      	Blocks: 8          IO Block: 4096   regular file
Device: fc01h/64513d	Inode: 669827      Links: 1
Access: (0664/-rw-rw-r--)  Uid: (  505/leappspark)   Gid: (  505/leappspark)
Access: 2017-08-10 15:07:07.935788631 +0800
Modify: 2017-08-10 09:01:59.212324852 +0800
Change: 2017-08-10 09:01:59.212324852 +0800

# 2 从文件状态中获取修改时间
[leappspark@emr-header-1 bin]$ stat play_rate.latest | grep -i Modify
Modify: 2017-08-10 09:01:59.212324852 +0800

# 3 从修改时间中获取日期
[leappspark@emr-header-1 bin]$ stat play_rate.latest | grep -i Modify | awk -F' ' '{print $2}'
2017-08-10

```

#### 2.3 输出区间内的所有日期

详见：[【TypeCodes】shell遍历输出两个日期范围内所有的日期](https://typecodes.com/linux/alldateduringtwodays1.html)

## 3 注意

#### 3.1 timestamp的单位

date命令获得的时间戳是从1970-01-01 00:00:00到当前时间的 **秒数** 。    
但是，在某些地方的timestamp的单位可能是 **毫秒** ，比如：opentsdb中的timestamp

```
# 有问题，这个时间戳（timestamp属性）是以『秒』为单位
sh send_to_opentsdb.sh '{"metric": "blender.alg.rec_count", "value": 120579.0, "timestamp": 1502035200, "tags": {"bucket": "blender.language.en"}}'

# 正确，opentsdb里面时间戳（timestamp属性）以『毫秒』为单位
sh send_to_opentsdb.sh '{"metric": "blender.alg.rec_count", "value": 120579.0, "timestamp": 1502035230000, "tags": {"bucket": "blender.language.en"}}'
```

## 4 参考

[【海王】linux 下查看文件修改时间 等](http://www.cnblogs.com/leaven/archive/2011/09/28/2194199.html)    
[【知识天地】linux在shell中获取时间](http://www.cnblogs.com/mfryf/archive/2012/03/23/2413362.html)    
[【菜鸟教程】Linux date命令](http://www.runoob.com/linux/linux-comm-date.html)    
[【菜鸟教程】Linux stat命令](http://www.runoob.com/linux/linux-comm-stat.html)    
[【菜鸟教程】Shell 教程](http://www.runoob.com/linux/linux-shell.html)    
[【TypeCodes】shell遍历输出两个日期范围内所有的日期](https://typecodes.com/linux/alldateduringtwodays1.html)
