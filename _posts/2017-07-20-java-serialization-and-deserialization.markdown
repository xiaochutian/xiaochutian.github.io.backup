---
layout:     post
title:      "Java序列化和反序列化"
subtitle:   ""
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Serialization
    - Java
---

## 0 一篇一句

**最舒服的关系就是『凡事不用猜』**

## 1 概述

#### 1.1 概念介绍

对象序列化(Object Serialization)可以把对象转换成一种 **『可存储』** 或者 **『可传输』** 的形式。    
> 注：不想听『哔哔』的可以直接跳到[『总结』](#sum)

#### 1.2 使用场景

**【序列化场景】** 把对象序列化成二进制对象后：**存储** 到文件中；**存储** 到内存，数据库中（Redis, Mongo, Mysql等）；RCP（Remote Procedure Call Protocol）远程过程调用协议中，把对象 **传输** 到网络上    
**【反序列化场景】** 把二进制对象反序列化成对象：从文件反序列化得到对象；从内存，数据库反序列化得到对象；RPC中，从网络反序列化得到对象

#### 1.3 起因
- 推荐服务器，需要记录给用户推荐的历史，使用History类记录相关信息。    
- 第一版的推荐服务器，没有做反馈统计，所以History中只记录了视频的ID，并没有记录视频是使用哪种算法推荐出来的。    
- 新版的推荐服务器，需要在 **History中加入一个新的字段bucket** ，来记录该视频是由哪种算法推荐出来。

## 2 serialVersionUID

#### 2.1 serialVersionUID的作用

###### 2.1.1 简而言之

serialVersionUID 就是 **控制版本是否兼容** 的，若我们认为修改的 『实体类』 是向后兼容的，则不修改 serialVersionUID；反之，则提高 serialVersionUID的值。

###### 2.1.2 详细解释（我的理解）

*serialVersionUID是用来在序列化和反序列化的过程中，一个类的 **『唯一标识』** 。*    

**不同的类，应该serialVersionUID不相同**    
比如，Apple类的serialVersionUID，如果跟Orange类的serialVersionUID相同的话。那么相互序列化和反序列化，将不会报错，属性都被赋值成默认值。    
但是，在实际的问题领域，Apple和Orange是完全不同的东西，不应该被赋值成默认值，而应该直接报错。

> K.I.S.S.说：应该尽早发现问题。因为，越晚发现问题，花费的时间和精力就越多。（编译器的问题，总比运行时的问题容易解决）

**同一个类，多个不兼容版本之间，serialVersionUID应该不相同**    
比如，要更新一个大版本，类整体要重构，确定不兼容以前的版本。则修改serialVersionUID。

###### 2.1.3 详细解释（官方文档）

[Serializable官方文档](http://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html)上如是说：
>The serialization runtime associates with each serializable class a version number, called a serialVersionUID, which is used during deserialization to verify that the sender and receiver of a serialized object have loaded classes for that object that are compatible with respect to serialization. If the receiver has loaded a class for the object that has a different serialVersionUID than that of the corresponding sender's class, then deserialization will result in an InvalidClassException. A serializable class can declare its own serialVersionUID explicitly by declaring a field named "serialVersionUID" that must be static, final, and of type long:

> ANY-ACCESS-MODIFIER static final long serialVersionUID = 42L;

>If a serializable class does not explicitly declare a serialVersionUID, then the serialization runtime will calculate a default serialVersionUID value for that class based on various aspects of the class, as described in the Java(TM) Object Serialization Specification. However, it is strongly recommended that all serializable classes explicitly declare serialVersionUID values, since the default serialVersionUID computation is highly sensitive to class details that may vary depending on compiler implementations, and can thus result in unexpected InvalidClassExceptions during deserialization. Therefore, to guarantee a consistent serialVersionUID value across different java compiler implementations, a serializable class must declare an explicit serialVersionUID value. It is also strongly advised that explicit serialVersionUID declarations use the private modifier where possible, since such declarations apply only to the immediately declaring class--serialVersionUID fields are not useful as inherited members. Array classes cannot declare an explicit serialVersionUID, so they always have the default computed value, but the requirement for matching serialVersionUID values is waived for array classes.


**翻译**    
- serialVersionUID 是 Java 为每个序列化类产生的版本标识，可用来保证在反序列时，发送方发送的和接受方接收的是可兼容的对象。
- 如果接收方接收的类的 serialVersionUID 与发送方发送的 serialVersionUID 不一致，进行反序列时会抛出 [***InvalidClassException***](http://docs.oracle.com/javase/8/docs/api/java/io/InvalidClassException.html)。序列化的类可显式声明 serialVersionUID 的值，如下:
```
ANY-ACCESS-MODIFIER static final long serialVersionUID = 1L;
```
- 当显式定义 serialVersionUID 的值时，Java 根据类的多个方面(具体可参考 Java 序列化规范)动态生成一个默认的 serialVersionUID 。
- 尽管这样，还是建议你在每一个序列化的类中显式指定 serialVersionUID 的值，因为不同的 jdk 编译很可能会生成不同的 serialVersionUID 默认值，进而导致在反序列化时抛出 InvalidClassExceptions 异常。
- 所以，为了保证在不同的 jdk 编译实现中，其 serialVersionUID 的值也一致，可序列化的类 **必须显式指定** serialVersionUID 的值。
- 另外，serialVersionUID 的修饰符最好是 private，因为 serialVersionUID 不能被继承，所以建议使用 private 修饰 serialVersionUID。

#### 2.2 推荐服务器修改

【类的组织结构】
1. 序列化的是CardHistory类，CardHistory类中设置了serialVersionUID
2. CardHistory类中，包含了List<History>。
3. History类中 **没有设置** serialVersionUID，并且需要在History类中添加属性。

【计划的思路】（进行数据迁移）
1. 使用老的，不带serialVersionUID的History反序列化出来
2. 使用HistoryNew，带bucketId，并且serialVersionUID=xxx存到新的mongo库
3. 使用新的，带serialVersionUID=xxx和bucketId的History反序列化（做验证）

【实际情况】    
修改完History类，直接就可以反序列化老的数据。并不需要迁移数据。    

【思考】    
为什么，CardHistory的serialVersionUID没有改，里面的List<History>的History的serialVersionUID改了（老的版本没有显式的声明serialVersionUID，肯定跟新版的serialVersionUID不相同）。但是还是可以序列化？    

【猜想】    
序列化的是CardHistory类，并且CardHistory类是设置了serialVersionUID，所以JVM认为CardHistory还是以前那个CardHistory。如果，以前有用History直接序列化到存储中，应该是不能用新版的History反序列化的。

## 3 其他

#### 3.1 transient关键字

Java规范中对transient关键字的解释：
> The transient marker is not fully specified by The Java Language Specification but is used in object serialization to mark member variables that should not be serialized.

简言之就是：在对象序列化过程中，标记不该被序列化的成员变量。

**【使用场景】**    
1. 某些类型的对象，其状态是瞬时的，这样的对象是无法保存其状态的。例如一个Thread对象或一个FileInputStream对象 ，对于这些字段，我们必须用transient关键字标明，否则编译器将报措。
2. 一些敏感信息（如密码，银行卡号等），为了安全起见，不希望在网络操作（主要涉及到序列化操作，本地序列化缓存也适用）中被传输。

**【语法】**    
1. 只能修饰成员变量：非静态成员变量和静态成员变量
2. 不能修饰类、方法和本地变量
3. 静态成员，不管是否被transient修饰，都不会被序列化。（因为，这些信息都存在了类的描述部分，供类的所有实例共享）

> 注：被transient修饰就一定不能被序列化吗？答案是否定的。如果，实现的是Externalizable接口的话，可能存在特殊情况。详见[【ImportNew】Java transient关键字使用小记，第3部分](http://www.importnew.com/21517.html)

#### 3.2 IDEA设置serialVersionUID的提示

Idea里面，implements Serializable，不自动生成VID    
Ctrl + Alt + s，搜索inspection serializable。    
把Serializable class without 'serialVersionUID'，打勾，并且设置成黄色（警告级别）或者红色（Error级别）

#### 3.3 Serializable与Externalizable的区别

1. Serializable，JVM会有默认的方式实现对象的序列化。而Externalizable必须用户自己实现。
2. 在早期的Java版本，Externalizable可能性能更好。但是，在JVM大大优化的今天，基本可以弃用Externalizable。
3. 更多内容待续：Externalizable里面有两个接口readExternal和writeExternal。但是Serializable里面没有接口（为何会有readObject、writeObject、readResolve和writeResolve）。

## 4 总结 {#sum}
1. 对象序列化(Object Serialization)可以把对象转换成一种 『可存储』 或者 『可传输』 的形式。
2. serialVersionUID用来在『序列化』和『反序列化』过程中作为类的唯一标识。可以用来区分 **不同的类** ，或者同一个类的多个 **不兼容版本** 。
3. 如果，类实现了Serializable接口。那么记得定义serialVersionUID，养成好习惯。
4. transient用来修饰 **不该被序列化的 成员变量**

## 5 参考

[【Oracle Doc】Serializable官方文档](http://docs.oracle.com/javase/8/docs/api/java/io/Serializable.html)  
[【GitHub】serialVersionUID 有什么作用？该如何使用？](https://github.com/giantray/stackoverflow-java-top-qa/blob/master/contents/what-is-a-serialversionuid-and-why-should-i-use-it.md)  
[【ImportNew】关于 Java 对象序列化您不知道的 5 件事](http://www.importnew.com/16151.html)
[【ImportNew】在Java中如何使用transient](http://www.importnew.com/12611.html)    
[【ImportNew】Java transient关键字使用小记](http://www.importnew.com/21517.html)    
[【StackOverflow】What is the difference between Serializable and Externalizable in Java?](https://stackoverflow.com/questions/817853/what-is-the-difference-between-serializable-and-externalizable-in-java)    
[【Codingeek】7 DIFFERENCES BETWEEN SERIALIZABLE AND EXTERNALIZABLE INTERFACE IN JAVA](http://www.codingeek.com/java/io/differences-serializable-externalizable-interface-java-tutorial/)
