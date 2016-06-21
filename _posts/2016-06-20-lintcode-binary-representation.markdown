---
layout:     post
title:      "[Lintcode] Binary Representation"
subtitle:   "double损失精度，带来两个问题"
# date:       2016-06-20 10:23:36
author:     "K.I.S.S."
#header-img: ""
catalog:    true
tags:
    - Lintcode
    - Bit Manipulation
---

## 0 一篇一句

**恭喜骑士获得NBA总冠军，成为历史上第一支1比3落后，成功反超的队伍。**

## 1 题目[Binary Representation](http://www.lintcode.com/en/problem/binary-representation/)

```
Given a (decimal - e.g. 3.72) number that is passed in as a string,
return the binary representation that is passed in as a string.
If the fractional part of the number can not be represented
accurately in binary with at most 32 characters, return ERROR.
```

## 2 翻译

```
给定一个数将其转换为二进制（均用字符串表示），
如果这个数的小数部分不能在 32 个字符之内来精确地表示，则返回 "ERROR"。
```

## 3 解题思路

1. 把小数部分和整数部分分开转换（[注意，先取小数部分再转成double，不能先转后取](#ID5-1)）
2. 整数部分使用Integer.toBinaryString()转换
3. 小数部分，乘以2取整数部分（循环）
4. 把两部分拼接起来，作为结果返回

## 4 最终代码

#### 4.1 能通过OJ的最终代码 {#ID4-1}

```java
public class Solution {
    public String binaryRepresentation(String n) {
        // 获取小数点的位置
        int dotIndex = n.indexOf(".");
        // 查看是否存在小数部分，存在则把小数部分转成double，不存在则把fracPart置为-1
        double fracPart = dotIndex == -1 ? -1: Double.parseDouble("0"+n.substring(dotIndex));
        // 找到整数部分
        int intPart = dotIndex == -1 ? Integer.parseInt(n): Integer.parseInt(n.substring(0,dotIndex));
        StringBuilder sb = new StringBuilder();
        // 一直乘以2，取整数部分（如果，超过32位则返回ERROR）
        while(fracPart > 0){
            if (sb.length() > 32){
                return "ERROR";
            }
            fracPart *= 2;
            if (fracPart >= 1){
                sb.append(1);
                fracPart -= 1;
            } else {
                sb.append(0);
            }
        }
        // 拼接小数部分和整数部分，并返回
        return sb.length() > 0 ? Integer.toBinaryString(intPart) + "." + sb.toString() : Integer.toBinaryString(intPart);
    }
}
```

#### 4.2 符合题意的最终代码 {#ID4-2}

```java
import java.math.BigDecimal;
public class RightSolution{
  public String binaryRepresentation(String n) {
        int dotIndex = n.indexOf(".");
        if (dotIndex == -1){
            return Integer.toBinaryString(Integer.valueOf(n));
        }
        BigDecimal bd = new BigDecimal("0" + n.substring(dotIndex));
        StringBuilder sb = new StringBuilder();
        while (bd.compareTo(BigDecimal.ZERO) > 0){
            if (sb.length() > 32){
                return "ERROR";
            }
            bd = bd.multiply(BigDecimal.valueOf(2));
            if (bd.compareTo(BigDecimal.ONE) >= 0){
                sb.append(1);
                bd = bd.subtract(BigDecimal.valueOf(1));
            } else {
                sb.append(0);
            }
        }
        return Integer.toBinaryString(Integer.valueOf(n.substring(0,dotIndex))) + "." + sb.toString();
    }
}
```

## 5 遇到的问题 {#ID5}

#### 5.1 问题1，测试用例28187281.128121212121无法通过OJ {#ID5-1}

###### 5.1.1 **问题描述**

```
Input 28187281.128121212121    
Output 1101011100001101010010001.00100000110011001000110101    
Expected ERROR

输入的数，不以5结尾。但是，没有返回ERROR。
```

###### 5.1.2 **原因分析**

分析了我的代码（part1）中的binaryRepresentation函数和九章算法的代码（part2）中的binaryRepresentation函数之后，找到原因：    
1. 我的代码中， **先把整个字符串转换成double，再取的小数部分**    
2. 九章算法中， **先取字符串的小数部分，再把小数部分转成double**    
3. 两种方式，得到的 **小数部分不一样**

**归根结底是因为：double Double.parseDouble(String str)，String转double会损失精度**

```java
//************************part1********************************
// 我的代码
public class Solution {
    public String binaryRepresentation(String n) {
        // 我的代码中，先把整个数转成了double，再取小数部分
        double d = Double.parseDouble(n);
        double fracPart = d - (int)d;
        int intPart = (int) d;
        StringBuilder sb = new StringBuilder();
        while(fracPart > 0){
            if (sb.length() > 32){
                return "ERROR";
            }
            fracPart *= 2;
            if (fracPart >= 1){
                sb.append(1);
                fracPart -= 1;
            } else {
                sb.append(0);
            }
        }
        return sb.length() > 0 ? Integer.toBinaryString(intPart) + "."
                    + sb.toString() : Integer.toBinaryString(intPart);
    }
}

//************************part2********************************
// 九章算法上可以通过OJ的代码
public class Solution {
    public String binaryRepresentation(String n) {
        if (n.indexOf('.') == -1) {
            return parseInteger(n);
        }
        String[] params = n.split("\\.");
        // 九章算法里面，先把小数部分取出来，单独转成的double
        String flt = parseFloat(params[1]);
        if (flt == "ERROR") {
            return flt;
        }
        if (flt.equals("0") || flt.equals("")) {
            return parseInteger(params[0]);
        }
        return parseInteger(params[0]) + "." + flt;
    }
    private String parseInteger(String str) {
        int n = Integer.parseInt(str);
        if (str.equals("") || str.equals("0")) {
            return "0";
        }
        String binary = "";
        while (n != 0) {
            binary = Integer.toString(n % 2) + binary;
            n = n / 2;
        }
        return binary;
    }
    private String parseFloat(String str) {
        double d = Double.parseDouble("0." + str);
        String binary = "";
        HashSet<Double> set = new HashSet<Double>();
        while (d > 0) {
            if (binary.length() > 32 || set.contains(d)) {
                return "ERROR";
            }
            set.add(d);
            d = d * 2;
            if (d >= 1) {
                binary = binary + "1";
                d = d - 1;
            } else {
                binary = binary + "0";
            }
        }
        return binary;
    }
}
```

###### 5.1.3 **分析证明**

```java
//代码证明
System.out.println(Double.parseDouble("0.128121212121"));
System.out.println(Double.parseDouble("28187281.128121212121"));
//输出
// 0.128121212121
// 2.8187281128121212E7
```

###### 5.1.4 **问题解决**

找到问题之后，修改代码。 **先取出字符串的小数部分，再处理** 。[修改后代码](#ID4-1)

#### 5.2 问题2，测试用例1.00000000023283064365386962890626不返回ERROR

###### 5.1.1 **问题描述**

```
第一组
Input 1.00000000023283064365386962890626
Output 1.00000000000000000000000000000001
Expected 1.00000000000000000000000000000001

第二组
Input 1.00000000023283064365386962890624
Output 1.00000000000000000000000000000001
Expected 1.00000000000000000000000000000001

输入的数，不以5结尾。但是，没有返回ERROR。
而且，两个输入的数不一样，返回的结果一样。
更神奇的是！！！！OJ居然过了！！！（什么鬼，兄弟，你这OJ要上天啊 = =#）
```

###### 5.1.2 **原因分析**

两个Input，1.00000000023283064365386962890626和1.00000000023283064365386962890624小数部分转换成double之后， **得到的数相同**

**归根结底是因为：double Double.parseDouble(String str)，String转double会损失精度**

###### 5.1.3 **分析证明**

```java
//代码证明
System.out.println(Double.parseDouble("1.00000000023283064365386962890626"));
System.out.println(Double.parseDouble("1.000000000232830643653869628906261"));
//输出
// 1.0000000002328306
// 1.0000000002328306
```

###### 5.1.4 **问题解决**

找到问题之后，修改代码。 **使用BigDecimal代替double进行小数部分计算** 。[修改后代码](#ID4-2)

## 6 总结 {#ID6}

1. 把String转成double会损失精度，在计算的时候需要特别注意。详情见[《double二三事》](/2016/06/20/things-about-double/)
2. 需要不损失精度的计算，使用BigDecimal代替double。但是，性能比double要低。
