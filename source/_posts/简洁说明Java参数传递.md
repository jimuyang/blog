---
title: 简洁说明Java参数传递
date: 2019-11-17 16:05:36
tags: 
- Java
---

# Java程序猿经常撕的一个问题：Java到底是值传递还是引用传递？ 
* 甲说，Java都是值传递。我同意
* 乙说，Java参数传递非基本类型(引用类型)时，传递的是引用。嗯..., 我点了点头
* 丙说，对基本类型Java传递值的copy，对非基本类型(引用类型)Java传递地址的copy。解释了甲的意思，我点个赞。 

不用吵了，其实大家都懂是怎么回事，很多时候只是概念和表达上的歧义，这里我取个巧，用一套“统一”的理论来简洁解释一下 

## Java中的引用
先问是什么，一等重要的事就是搞清楚到底Java中的引用是啥？
> 留给官方说明  

<!-- 这里放一张栈堆图 -->

在“统一”理论指导下，不妨将引用类型看作是一种“特殊”的基本类型。仅仅特殊在：**引用类型的“值”是一个地址**。
现在看一下赋值：原来就是*值*的copy
```java
    // 赋零值
    int a = 0;        // 
    Object o = null;  // 可以认为null就是类似0x0000的地址

    // 赋值
    a = 1;            // a的值为1
    o = new Object(); // o的"值"是一个地址 可以认为new返回的就是堆内地址

    // 变量间赋值
    int a1 = a;       // 将a的值(1)copy给a1
    Object o1 = o;    // 将o的"值"(地址)copy给o1
```
再看看==：原来就是*值*是否相等
```java
    int a = 1;
    Object o = new Object();

    a == 2;           // a的值是否等于2
    o == new Object();// o的"值"(地址)是否和new返回的地址相等
```
final变量的语义也可以统一为变量的“值”初始化后不可更改

## Java的参数传递
我这样来描述Java的参数传递过程：**将实参赋值给形参**。赋值的本质就是**值copy**。所以回到问题本身，我认为：**Java的参数传递都是值传递**。这样的描述是比较准确也比较体现**值copy**的本质的。 
用一段简洁的代码来说明：
```java
    static void func(int a, Object o) {
        // break point
        a = 2;
        o = new Object();
    }

    public static void main(String[] args) {
        int a = 1;
        Object o = new Object();
        func(a, o);             // 帧栈图1
        func(1, new Object());  // 帧栈图2
    }
```
断点处线程帧栈一览：
![](/blog/images/javaargpass1.jpg)
![](/blog/images/javaargpass2.jpg) 

通过断点后 func方法内执行这2行代码，只会修改func方法栈内的变量a和o的**值**，而不会对main方法栈内的a和o有任何影响，这也是copy的精髓
```java
    a = 2;
    o = new Object();
```

## 其他说明
* 
