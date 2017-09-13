---
layout:     post
title:      "timestamp引发的血案"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - timestamp
    - mysql
---

## 0 一篇一句

**以终为始，以终为始，以终为始**    

## 1 是什么
今天，在加一个小功能的时候，瞄了一眼数据库中的数据。
![WrongDataInMysql](/img/in-post/post-things-about-timestamp/wrong-data-in-mysql.png)
- 这个表记录的是用户的流水信息，其中 **create_at** 字段表示流水产生的时间
- create_at由客户端产生，序列化成json之后传给server，再由server持久化到mysql中
- mysql的表中，create_at字段使用的 **TIMESTAMP** 类型
- 这些测试数据，是在2017年7月的时候，测试客户端所产生的。

> 注：与客户端约定，时间戳以秒为单位。

**那么问题来了：为什么create_at的日期不对？**

## 2 为什么

#### 2.1 确认持久化层没问题
使用这个数据跑@Test，数据可以成功插入，但是mysql中create_at还是 \'1970-01-18 XX:XX:XX\'
```
// otherProperties代表其他的属性，不需要关注，略去
{
  "createAt": 1499074940,
  "otherProperties": "..."
}
```

#### 2.2 发现问题，修改原始数据

###### 2.2.1 分析create_at的规律
查看多条数据的create_at字段，发现虽然都是 \'1970-01-18\' 开头，但是，后面的数据是有不同的， **并不是完全一样** 。
这表示，客户端发来的值，确实已经存入了mysql。（如果，客户端发来的值就是null的话，数据库的TIMESTAMP类型字段，会取数据库的当前时间作为字段的默认值，就会是 \'2017\' 开头的数据了。）

**不同记录之间的create_at是不相同的，说明客户端传过来的数据在某种程度上存入了数据库，只是数值不准确**

###### 2.2.2 猜想是单位问题，并验证
联想之前遇到的坑，并查找[timestamp资料](https://en.wikipedia.org/wiki/Unix_time)。
- timestamp代表的是从 1970-01-01 00:00:00 开始，到当前时间的 **秒数** 。
- createAt属性，在反序列化的时候，使用的类是 java.sql.Timestamp （构造函数是以 **毫秒** 为单位的long型变量）    

**猜想是由于单位不一致造成的！    
于是，把客户端传过来的 createAt \* 1000 。相当于，把秒变成了毫秒。    
结果，存到数据库的值，由 1970/1/18 16:24:34 变成了 2017/7/3 17:42:20**
```
// otherProperties代表其他的属性，不需要关注，略去
{
  "createAt": 1499074940000,
  "otherProperties": "..."
}
```

#### 2.3 查找原因
- 客户端传来的 createAt 以 **『秒』** 为单位
- 反序列化的时候，使用的 java.sql.Timestamp 类。其[构造函数](https://docs.oracle.com/javase/8/docs/api/java/sql/Timestamp.html#Timestamp-long-)，以 **『毫秒』** 为单位。
- mysql中的TIMESTAMP类型，使用4字节存储，32bit的有符号整型，范围是 -2147483648 ~ 2147483647。以 **『秒』** 为单位。
- java.sql.Timestamp类，认为 createAt 为1499074940毫秒，即1499074秒。所以，存入mysql之后时间为 1970/1/18 16:24:34

> 注：可以使用[在线Unix时间戳工具](http://tool.chinaz.com/Tools/unixtime.aspx)进行验证

## 3 怎么做

#### 3.1 选取修复方案 {#solutions}

|---
|  | 方案名称 | 方案详情 | 是否选取 & 原因
|-|-|-|-
| 1 | 客户端修复 | 客户端把createAt * 1000，直接传毫秒值 | 否，iOS和android两端都要改，并且已经发出去的版本无法修复。
| 2 | 服务器在反序列化时修复 | 反序列化的时候把createAt * 1000 | 否，每个包含createAt的实体类都得处理，不合适。
| 3 | 服务器使用int存储时间戳 | **mysql中create_at的类型由TIMESTAMP改成INT(10)，实体类中createAt的类型由java.sql.Timestamp改成Integer** | 是，可以提高mysql的性能。mysql只管存储，谁需要这个数据谁自己去解析。

#### 3.2 进行修复

根据[3.1节](#solutions)中的修复方案比较，最终选取了方案3：在服务器使用int存储时间戳。    
大体思路如下：
1. 修改mysql中create_at的类型，TIMESTAMP --> INT(10)
    - 在mysql的表中，新建类型为INT(10)的列
    - 使用sql脚本，把create_at列的值，恢复到create_at_new中
    - 删除create_at列
    - 把create_at_new列，重命名为create_at
2. 修改实体类中createAt的类型，java.sql.Timestamp --> java.lang.Integer
3. 重新部署maven仓库和服务器

###### 3.2.1 无损恢复 or 有损恢复
mysql中的TIMESTAMP的精度是秒。    
所以，当 1499074940 变成 1499074 存入mysql之后，精度就已经丢失了，**只能进行有损恢复**。    
恢复方法：1499074 * 1000 = 1499074000    
误差范围：999秒

###### 3.2.2 停机恢复 or 不停机恢复

|---
| 方案 | 优势 | 劣势
|-|-|-
| 停机恢复 | 不需要新加字段，后期维护不会产生疑惑 | 恢复的时候，服务不可用
| 不停机恢复 | 恢复的时候，服务仍然可用，客户端无感知 | 需要在数据库新建一个字段，在实体类中新建一个字段，并且把实体类中的旧字段映射到新字段上。 **后期维护的时候，可能看到会比较疑惑**

权衡利弊，选择在夜深人静的时候，**停机更新**。

###### 3.2.2 恢复过程
恢复步骤
1. 修改数据库表（所有流水和日流水表）
2. 修改实体类（所有流水实体类）
3. 重新部署（maven和服务器）

恢复的mysql脚本（对于一种流水表）
```
# 添加INT(10)类型的列create_at_new
ALTER TABLE `earpro_common`.`serial_all_seventh_chord` ADD COLUMN `create_at_new` INT(10) NOT NULL AFTER `create_at`;

# 把create_at列的值，恢复到create_at_new中（有损恢复）
UPDATE `earpro_common`.`serial_all_seventh_chord` SET create_at_new=UNIX_TIMESTAMP(create_at) * 1000;

# 把旧的create_at列删除，并把create_at_new重命名成create_at
ALTER TABLE `earpro_common`.`serial_all_seventh_chord` DROP COLUMN `create_at`,CHANGE COLUMN `create_at_new` `create_at` INT(10) NOT NULL ;
```


## 4 总结

#### 4.1 重新认识timestamp

时间戳（timestamp）是指格林威治时间1970-01-01 00:00:00（北京时间1970-01-01 08:00:00）起，至现在的总 **『秒数』**。    

> [Wikipedia](https://en.wikipedia.org/wiki/Unix_time)上如是说：Unix time (also known as POSIX time or epoch time) is a system for describing instants in time, defined as the number of seconds that have elapsed since 00:00:00 Coordinated Universal Time (UTC), Thursday, 1 January 1970, minus the number of leap seconds that have taken place since then.

- timestamp使用32bit的signed int来表示，范围是 -2147483648 ~ 2147483647（2 ^ 31 -1）。    
- 由于，timestamp代表的是秒数，所以必须大于0，于是范围变成 0 ~ 2147483647    
- 起始时间是1970-01-01 00:00:00(UTC)，所以，最久能正常表示的时间是2038-01-19 03:14:07(UTC)（北京时间2038-01-19 11:14:07）

**超过这个时间之后，32bit的时间戳会发生越界，这就引发了[『2038年问题』](https://en.wikipedia.org/wiki/Year_2038_problem)**

#### 4.2 java.sql.Timestamp 与 mysql的TIMESTAMP类型

###### 4.2.1 java.sql.Timestamp

- java.sql.Timestamp为java.util.Date的子类。    
- 有两个构造函数，推荐使用的是使用long类型的毫秒数为参数的构造函数。
<img src="/img/in-post/post-things-about-timestamp/timestamp-class-info.jpeg" width="400" align="middle"/>
![TimestampConstructors](/img/in-post/post-things-about-timestamp/timestamp-constructors.jpeg)

###### 4.2.2 mysql的TIMESTAMP类型

- 使用32bit的signed int存储
- 代表从1970-01-01 00:00:00(UTC)到现在的总秒数
- 范围：1970-01-01 00:00:00(UTC) ~ 2038-01-19 03:14:07(UTC)

> 注：mysql5.6.4以后的版本，支持定义TIMESTAMP类型精确到微秒（[详见](https://dev.mysql.com/doc/refman/5.7/en/fractional-seconds.html)）。     
> 只需要在建表的时候类型TIMESTAMP(6)    
> 只能是 TIMESTAMP(n)，其中 0 <= n <= 6

#### 4.3 mysql存时间

可以使用DATETIME，TIMESTAMP和INT。到底应该如何选择呢？（若细聊，可再开一篇）

|---
| 方案 | 适用场景 | 优势 | 劣势
|-|-|-|-
| INT | 有大量时间范围查询的时候 | 数据库只管存储，不管解释，可以减小数据库压力。 | 无法使用mysql提供的时间函数
| TIMESTAMP | 与时区相关或者需要自动更新的时候，比如记录最近更新时间 | 时区转换，可自动更新 | 范围有限制
| DATETIME | 记录原始的创建时间 | 不会自动改变，除非手动重新赋值 | 占的空间多（8字节），查询比另外两种慢

## 5 参考

[【Wikipedia】Unix time](https://en.wikipedia.org/wiki/Unix_time)       
[【Oracle】Java Docs](https://docs.oracle.com/javase/8/docs/api/java/sql/Timestamp.html)       
[【Chinaz】站长工具，Unix时间戳](http://tool.chinaz.com/Tools/unixtime.aspx)       
[【Wikipedia】Year 2038 problem](https://en.wikipedia.org/wiki/Year_2038_problem)       
[【segmentfault】mysql数据库的timestamp为什么从1970到2038的某一时间？某一时间是指什么时间？过了这个时间之后怎么办？](https://segmentfault.com/q/1010000004418596)       
[【知乎】如何在mysql中优雅的解决精确到毫秒的问题？](https://www.zhihu.com/question/20859370)    
[【segmentfault】Mysql时间字段格式如何选择，TIMESTAMP，DATETIME，INT？](https://segmentfault.com/q/1010000000121702)       
[【CSDN-souldak】mysql中timestamp,datetime,int类型的区别与优劣](http://blog.csdn.net/souldak/article/details/11737799)       
[【mysql Docs】Date and Time Functions](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-functions.html)    
[【mysql Docs】Fractional Seconds in Time Values](https://dev.mysql.com/doc/refman/5.7/en/fractional-seconds.html)
