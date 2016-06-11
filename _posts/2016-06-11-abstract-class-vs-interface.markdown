---
layout:       post
title:        "抽象类与接口"
subtitle:     ""
# date:         2016-06-10 14:05:53
author:       "K.I.S.S."
# header-img:   ""
catalog:      true
tags:
    - JAVA
    - 编程思想
---

## 一篇一句

**抽象类用来表示 is-a 的关系，是对同种多个类似实体的抽象；接口用来表示 like-a 的关系，是对多种实体的相同行为的抽象。**

## 概述

高频问题，为了尽量**【求甚解】**，特查阅资料并结合自己的理解总结如下。   
如果，只想简单的回答面试问题，你可以直接跳到[总结部分](#sum)。    
如果，想要更全面的了解，那么其他部分也很重要。


[**抽象类**](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)

- 抽象类是一个用**abstract**关键字声明的类，它不能被 *实例化* ，但是可以被子类 *继承* 。
- 抽象类可以不包含抽象方法，但是，包含了抽象方法必须声明为抽象类。

```java
public abstract class GraphicObject {
   // declare fields
   // declare nonabstract methods
   abstract void draw();
}
```

[**接口**](https://docs.oracle.com/javase/tutorial/java/IandI/createinterface.html)

接口是一种引用类型，只能包含**常量，方法签名，默认方法，静态方法和嵌套类型（如内部类）。**
方法体只能存在于默认方法和静态方法里。

```java
public interface ExampleInterface {
    String constantVariable = "Hello Interface";  //常量（成员变量默认为 public static final）
    void doSomething();                           //方法签名（成员函数默认为public）
    default void move(){                          //默认方法
        System.out.println("Move Forward");
    }
    static void print(){                          //静态方法
        System.out.println("Hello Static Method");
    }                                             
    class InnerClass{}                            //嵌套类型（内部类）
}
```

## 联系

两者都是对问题领域进行分析、设计的出的**抽象概念**（不存在于问题领域），是对一系列看上去不同，但是，**本质相同**的具体概念的抽象表示。    

> 例1：“形状”是“三角形”、“正方形”等的抽象（抽象类）    
> 例2：JDBC是Java客户端程序，**访问**MySQL、Oracle等数据库的**行为的抽象**（接口）

感性的理解，接口可以看成特殊的抽象类（抽象类可以包含非抽象方法，接口只能包含抽象方法）    
*注：从编程语言语法的角度，这个并不成立*

## 区别

#### 语法定义

本部分从Java编程语言的语法层次出发进行分析

|---
| 比较项 | 抽象类 | 接口
|-|-|-
| 继承 | 抽象类只能继承一个类，实现多个接口 | 接口可以继承多个接口
| 访问修饰 | 抽象方法不能用private，其他方法和字段可以使用所有修饰符 | 默认为public，可以加public修饰符，但是，不能加private或者protected修饰符
| 字段 | 可以不是static和final的 | 默认为static final（常量）
| 方法 | 可以有抽象和非抽象方法 | 抽象方法、default方法、static方法

#### 设计理念

本部分从更高的层次进行分析。综合了，[chenssy的博文](http://blog.csdn.net/chenssy/article/details/12858267)和[IBM developerWorks](https://www.ibm.com/developerworks/cn/java/l-javainterface-abstract/)

> 编程语言是为了用计算机解决实际问题，编程语言的不同特性（抽象类和接口）可以用来反映现实生活中各实体的不同关系。    
> 解决问题的过程中，特性的选择（抽象类或接口），是由问题领域的本质来决定的。

|---
| 比较项 | 抽象类 |接口
|-|-|-
| 抽象的“对象” | 抽象类是对**实体的抽象**（包括属性和行为） | 接口是对**行为的抽象**
| 设计方式 | 抽象类是**自下而上**的设计，先有子类并找出公共部分，然后泛化成抽象类 | 接口是**自上而下**的，先有行为的约定，再有对接口的不同实现

对于chenssy博文中，“跨域不同”的部分，我有自己的理解。（可跳过）    

- [chenssy的博文](http://blog.csdn.net/chenssy/article/details/12858267)和[IBM developerWorks](https://www.ibm.com/developerworks/cn/java/l-javainterface-abstract/)中的观点：抽象类与实现类在概念本质上应该是相同的，接口的定义与接口的实现者在概念本质上可以不一致（接口仅仅定义的是行为契约）。
- 我的观点：在chenssy的观点中，不管是抽象类还是接口的“概念本质”指的都是“某个实体”。然而，如果，对于接口来说“概念本质”指的是“某种行为契约”，那么接口的定义与接口的实现者在“概念本质”上是一致的。

**抽象类用来表示 is-a 的关系，是对同种多个类似实体的抽象；接口用来表示 like-a 的关系，是对多种实体的相同行为的抽象。（全文重点，全文重点，全文重点。重要的事情说三遍！）**

#### 例子1：报警门

本例子结合了[【IBM developerWorks】](https://www.ibm.com/developerworks/cn/java/l-javainterface-abstract/)

1. 假设问题领域只有一种门，“实木门（完全用实木加工而成）”，那么可以直接使用WoodDoor类，包含基本的开门、关门方法。
2. 问题领域又加入了一种门，“复合门（复合材料加工而成）”，即加入了CompositeDoor类。那么，需要把两种门的**公共方法**和**公共字段**提取出来，形成一个AbstractDoor抽象类。
  - 如果，不泛化出抽象类，那么公共的方法中的代码会重复，增加维护成本
  - 为什么不是把公共行为抽象成接口？因为，可能有些公共属性，比如，长宽、重量等。（接口只能有静态常量）
3. 如果，这两种门还要具有报警功能。需要加入一个AlarmInterface接口，里面包含alarm方法。
  - 为什么不在抽象类里面加入alarm方法？因为，会违反面向对象设计中的**单一责任原则**（[The Single Responsibility Principle](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design))）

```java
//********************  part1  **********************
// WoodDoor
class WoodDoor{
  double weight;
  void open(){//code of open WoodDoor}
  void close(){//code of close WoodDoor}
}

//********************  part2  **********************
// add CompositeDoor, but the menbers are duplicate in two class
class CompositeDoor{
  double weight;
  void open(){//code of open CompositeDoor}
  void close(){//code of close CompositeDoor}
}
// add a new class AbstractDoor
class AbstractDoor{
  double weight;
  abstract public open();
  abstract public close();
}

//********************  part2  **********************
// add a new function alarm
interface Alarm{
  void alarm();
}

class CompositeDoor extends AbstractDoor implements Alarm{
  void open(){//code of open CompositeDoor}
  void close(){//code of close CompositeDoor}
  void alarm(){//code of alarm}
}

```

**例子总结：**

- 在问题领域，不管是实木门还是复合门，实体本质都是“门”这个概念，所以，泛化出抽象类AbstractDoor。
- 不仅仅是门有报警功能，手机也可以有报警功能，这是不同实体的相同功能的抽象，所以使用接口Alarm。

#### 例子2：ArrayList

ArrayList源码

```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
  //code here
}
```

ArrayList继承结构图

![ArrayListExtendsStructure](/img/in-post/abstract-class-vs-interface_ArrayList_extends_structrue.png)

**例子总结：**

- ArrayList和LinkedList本质上都属于列表，所以，可以抽象出AbastractList抽象类
- List、RandomAccess、Cloneable和Serializable等是多个类都需要的功能，所以抽象成接口

## 选择

#### 根据语法特性

第一部分的内容，翻译自[【Java官方文档】](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)，主要根据语法特性，进行选择。

**使用抽象类**

- 在多个紧密相关的类之间，**共享代码**
- 你期望继承抽象类的那些子类，有公用的方法和字段，或者需要**protected**和**private**修饰（接口不支持）
- 需要**not-static**或者**not-final**的字段

**使用接口**

- 你期望不相关的类实现你的接口，比如Comparable和Cloneable
- 你想指定一种特定数据类型的**行为**，但是，不关心谁来实现它的行为
- 你想利用**多重继承**的优势

#### 根据对问题领域的分析

第二部分的内容，是综合看过的资料和自己的理解，根据对问题领域的分析，进行选择。

**使用抽象类**

- 抽象类是自底向上的抽象
- 抽象类是对多个**实体对象**的相同部分的抽象描述
- 抽象类多来自于代码重构

> 当只有一个Cat类的时候，不需要一个Animal的抽象类。但是，又加入了一个Dog类的时候，可以把Cat和Dog的公有属性和方法抽象出来。比如，sound方法和color字段等。

**使用接口**

- 接口是自顶向下的抽象
- 接口是对**行为规范**的抽象描述，是一种契约
- 接口可以用来屏蔽变更（在业务逻辑中调用接口方法，接口的实现改变，业务逻辑不需要改变）

> [策略模式](https://en.wikipedia.org/wiki/Strategy_pattern)，指对象有**某个行为**，但是在不同的场景中，该行为有不同的实现算法。其中，某个行为可以用接口来描述。

**抽象不同的实体对象的时候使用抽象类，需要约定行为规范和屏蔽变更的时候使用接口**


## 总结 {#sum}

|---
| 比较项 | 抽象类 | 接口
|-|-|-
| 继承 | 只能继承一个类，实现多个接口 | 可以继承多个接口
| 访问修饰 | 抽象方法可以为friendly（默认的包访问权限）或者protected，其他方法和所有字段可以为private | 所有字段和方法都必须为public
| 字段 | 可以有变量 | 所有的字段都是常量（static final）
| 方法 | 可以有非抽象方法 | 可以有abstract、default、static方法
| “对象” | 对实体的抽象 | 对行为的抽象
| 设计方式 | 自底向上 | 自顶向下
| 表示关系 | is-a | like-a
| **选择** | 抽象实体类 | 约定行为和屏蔽变更


## 参考

1. [【IBM developerWorks】深入理解abstract class和interface](https://www.ibm.com/developerworks/cn/java/l-javainterface-abstract/)
2. [【chenssy的博文】java提高篇（四）-----抽象类与接口](http://blog.csdn.net/chenssy/article/details/12858267)
3. [【Java官方文档】Java Documentation of Abstract](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)
4. [【Java官方文档】Java Documentation of Interface](https://docs.oracle.com/javase/tutorial/java/IandI/createinterface.html)
5. [【JavaWorld文章】JavaWorld  -----Abstract classes vs.interfaces](http://www.javaworld.com/article/2077421/learn-java/abstract-classes-vs-interfaces.html)
6. [【Bigennersbook文章】Difference Between Abstract Class and Interface in Java](http://beginnersbook.com/2013/05/abstract-class-vs-interface-in-java/)
