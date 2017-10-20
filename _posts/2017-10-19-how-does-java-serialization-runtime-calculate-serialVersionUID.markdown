---
layout:     post
title:      "Java序列化运行时计算serialVersionUID的方式"
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

**匀速会比冲刺走得更远**

## 1 serialVersionUID概述

在之前的博文[《Java序列化和反序列化》](/2017/07/20/java-serialization-and-deserialization/){:target="blank"}中，提到过serialVersionUID。简而言之，serialVersionUID是Java序列化和反序列化过程中，一个类的 **唯一标识**。不同的类需要使用不同的serialVersionUID， 相同类的不兼容版本使用不同的serialVersionUID。

[serialVersionUID的详细介绍](/2017/07/20/java-serialization-and-deserialization/#serialVersionUID){:target="blank"}部分提到：
> If a serializable class does not explicitly declare a serialVersionUID, then the serialization runtime will calculate a default serialVersionUID value for that class based on various aspects of the class, as described in the Java(TM) Object Serialization Specification.

**那么，当一个可序列化的类没有显式定义serialVersionUID的时候：    
1.Java序列化运行时是如何生成默认serialVersionUID的？    
2.如果要保证类向前兼容，可以修改什么？不能修改什么？**

## 2 默认serialVersionUID的计算

查看 **[Java Object Serialization Specification][1]** 中 Class Descriptors部分 的4.6节 **[Stream Unique Identifiers][2]** 。其中，详细定义了Java序列化运行时生成默认serialVersionUID的方式。


```java
private static final long serialVersionUID = 3487495895819393L;
```

stream-unique identifier（serialVersionUID，之后缩写为SUID）是一个综合的哈希值，与类名，接口名，成员方法和成员变量相关。一个类的SUID值必须在除第一个版本之外的所有版本都设置。它可以显式的在类里面定义，但不是必须的。这个值对于可兼容的类来说是固定的（相同的）。如果一个类的SUID没有显式定义，那么它默认为那个类的哈希值。 **动态代理类和枚举类型的serialVersionUID总是为0L。** 数组类不能声明一个显式的serialVersionUID，所以它们总是具有默认的计算值，但是对于数组类，放弃了匹配serialVersionUID值的要求。

> 注意 - 强烈建议所有可序列化的类都明确声明serialVersionUID值，因为默认的serialVersionUID计算对类详细信息非常敏感，这些细节可能因编译器实现而有所不同，因此可能会在反序列化期间导致意外的serialVersionUID冲突，导致反序列化失败。

Externalizable类的初始版本必须输出将来可扩展的流数据格式。 readExternal方法的初始版本必须能够读取所有未来版本的writeExternal方法的输出格式。    

反映类定义的字节流的签名被用来计算serialVersionUID。 国家标准与技术研究所（NIST）安全散列算法（SHA-1）用于计算流的签名。 前两个32位数量用于形成64位散列。 java.lang.DataOutputStream用于将原始数据类型转换为字节序列。 输入流的值由类的Java虚拟机（VM）规范定义。    

类修饰符可以包括ACC_PUBLIC，ACC_FINAL，ACC_INTERFACE和ACC_ABSTRACT标志; 其他标志被忽略，不影响serialVersionUID计算。 类似地，对于字段修饰符，当计算serialVersionUID值时，仅使用ACC_PUBLIC，ACC_PRIVATE，ACC_PROTECTED，ACC_STATIC，ACC_FINAL，ACC_VOLATILE和ACC_TRANSIENT标志。 对于构造函数和方法修饰符，仅使用ACC_PUBLIC，ACC_PRIVATE，ACC_PROTECTED，ACC_STATIC，ACC_FINAL，ACC_SYNCHRONIZED，ACC_NATIVE，ACC_ABSTRACT和ACC_STRICT标志。 名称和描述符以java.io.DataOutputStream.writeUTF方法使用的格式编写。

**流中的项目顺序如下：**
1. 类名。
2. 类修饰符写为32位整数。
3. 按名称排序的每个接口的名称。
4. 对于按字段名排序的类的每个字段（private static和private transient除外）
    - a. 字段的名称。
    - b. 字段的修饰符写为32位整数。
    - c. 字段的描述符。
5. 如果存在初始化器，写出如下内容：
    - a. 方法的名称，`<clinit>`。
    - b. 方法的修饰符 `java.lang.reflect.Modifier.STATIC` 写为32位整数。
    - c. 方法的描述符，`()V`。
6. 对于按方法名和签名排序的每个非私有构造函数：
    - a. 方法的名称，`<init>`。
    - b. 该方法的修饰符写为32位整数。
    - c. 该方法的描述符。
7. 对于按方法名称和签名排序的每个非私有方法：
    - a. 方法的名称。
    - b. 该方法的修饰符写为32位整数。
    - c. 该方法的描述符。
8. SHA-1算法基于DataOutputStream生成的字节流执行，并产生五个32位值sha [0..4]。
9. 哈希值是从SHA-1消息摘要的第一个和第二个32位值组合起来的。 如果消息摘要的结果，五个32位字H0 H1 H2 H3 H4位于名为sha的五个int值的数组中，则哈希值将计算如下：
```java
long hash = ((sha[0] >>> 24) & 0xFF) |
            ((sha[0] >>> 16) & 0xFF) << 8 |
            ((sha[0] >>> 8) & 0xFF) << 16 |
            ((sha[0] >>> 0) & 0xFF) << 24 |
            ((sha[1] >>> 24) & 0xFF) << 32 |
            ((sha[1] >>> 16) & 0xFF) << 40 |
            ((sha[1] >>> 8) & 0xFF) << 48 |
            ((sha[1] >>> 0) & 0xFF) << 56;
```

## 3 没有显式定义serialVersionUID的类的兼容性

|---
| 编号 | 场景 | 是否兼容
|-|-|-
| 1 | 修改类名 | 否
| 2 | 修改类的修饰符 | 否
| 3 | 修改类的接口 | 否
| 4 | 添加、修改或者删除类的字段（非private static或者private transient变量） | 否
| 5 | 添加、修改或者删除非私有构造函数 | 否
| 6 | 添加、修改或者删除非私有方法| 否

## 4 思考

1. 什么是 `class initializer`？
2. `<cinit>` VS `<init>`
3. 类名不同，但是serialVersionUID相同的两个类进行序列化和反序列化，会出现什么结果？

## 5 参考

[【MyBlog】Java序列化和反序列化](/2017/07/20/java-serialization-and-deserialization/){:target="blank"}    
[【Oracle】Java Object Serialization Specification - All][1]    
[【Oracle】Java Object Serialization Specification - Class Descriptors - Stream Unique Identifiers][2]    
[【StackOverflow】Who invokes the class initializer method `<clinit>` and when?](https://stackoverflow.com/questions/15919317/who-invokes-the-class-initializer-method-clinit-and-when)    
[【简书】Java类/对象初始化及顺序](http://www.jianshu.com/p/cc60f9c6510c)    

[1]:https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serialTOC.html
[2]:https://docs.oracle.com/javase/8/docs/platform/serialization/spec/class.html#a4100
