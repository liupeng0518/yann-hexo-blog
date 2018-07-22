---
title:  写给 Java 开发者的 Kotlin 教程 (5) - Null对象与类型安全
date: 2018-07-17 21:00:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/07/17/PlWoad.png)

Kotlin 的类型系统旨在消除来自代码空引用的危险,许多编程语言（包括 `Java`）中最常见的陷阱之一，就是访问空引用的成员会导致空引用异常。在 `Java` 中，这等同于 `NullPointerException` 或简称 `NPE` 。

<!-- more -->

## 一件趣事
> “我把 Null 引用称为自己的十亿美元错误。它的发明是在1965 年，那时我用一个面向对象语言( ALGOL W )设计了第一个全面的引用类型系统。我的目的是确保所有引用的使用都是绝对安全的，编译器会自动进行检查。但是我未能抵御住诱惑，加入了Null引用，仅仅是因为实现起来非常容易。它导致了数不清的错误、漏洞和系统崩溃，可能在之后 40 年中造成了十亿美元的损失。近年来，大家开始使用各种程序分析程序，比如微软的 PREfix 和 PREfast 来检查引用，如果存在为非 Null 的风险时就提出警告。更新的程序设计语言比如 Spec# 已经引入了非 Null 引用的声明。这正是我在1965年拒绝的解决方案。” 
—— 《Null References: The Billion Dollar Mistake》托尼·霍尔（Tony Hoare），图灵奖得主

![](https://s1.ax1x.com/2018/07/17/PlWqRP.md.png)

## Null的错误
简单来说：NULL 是一个不是值的值，而它现在有很多名字：NULL、nil、null、None、Nothing、Nil 和 nullptr。每种语言都有自己的细微差别。`Null` 有很多缺点，我们细数下罪状。

### 类型检测失败
```java
int length(String input){
    return input.Length();
}
```
在 Java 中，如果我编写 x.Length()，编译器会检查 x 的类型。如果 x 是一个 String，那么类型检查成功；

如果 x 是一个 Socket，那么类型检查失败，编译时检查存在一个致命缺陷：任何引用都可以是 null，而调用一个 null 对象的方法会产生一个 NullPointerException。

所以 `null.Length()` 会抛出一个 `NullPointerException` 而和原本的类型系统冲突，因为 `NULL` 超出了类型检查的范围。越过类型检查，运行时会给你一个巨大的惊喜。解决这种问题我们一般会选择 `入参检测` 来让错误尽早的暴露出来。

```java
int length(String input){
    Validate.nonNull(input, "input must be not null");
    return input.Length();
}
```

### NULL 是一个特例
```java
UserVo getUser(Long id){
    //略
}
```

在这个函数定义中，返回值会有三种可能 `UserVo` `null` `Expection`， 而我们在 获取 `null` 值的时候，我们不知道是因为 `ID` 不对，还是其他其他问题，甚至于在实践阶段，我们可能赋予 `null`返回是一个特殊的情况，在外部应该给予处理。

除此之外 `Null`还有
- `NULL` 难以调试
- `NULL` 使我们需要进行大量的 `IF` 判断
- `NULL` 存储在数据结构中，导致数据意义失效


## Kotlin 解决之道
在 `Java 1.8` 之前我们使用使用 `Guava` 的 `Optional` 对，`1.8` 之后我们使用内置数据结构。
在 `Kotlin` 中，我们使用 `类型系统` 来帮助我们。

{% tabs null %}
<!-- tab nonNull -->
{% codeblock lang:kotlin %}
var a: String = "abc"
a = null // 编译错误
{% endcodeblock %}
<!-- endtab -->
<!-- tab nullable -->
{% codeblock lang:kotlin %}
var b: String? = "abc"
b = null // ok
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

看出来了吗？我们使用 `类型系统` 区分一个引用可以为 null 还是不能容纳。这个时候我们如果访问其中的函数或变量
```kotlin
var b: String? = "abc"
val l = b.length // 错误：变量“b”可能为空
```
那我们如何去访问这个对象的函数呢？有以下几种方式

### 在条件中检查 null
```kotlin
var b: String? = "abc"
val l = if (b != null) b.length else -1
```

### 安全调用

```kotlin
var b: String? = "abc"
b?.length
```
这里需要注意 如果 `b` 非空，就返回 `b.length`，否则返回 `null`，这个表达式的返回值是 `Int?` 类型。
而且我们可以采用 `链式调用`
```kotlin
bob?.department?.head?.name
```

### Elvis 操作符
我们都在 `Optional` 对象往往有这种的操作
```java
Optional<String> someOpt;
String unwrap = someOpt.orElse("Default");
```
而在 `Kotlin` 我们可以如下操作 
```kotlin
val l = b?.length ?: -1
```

## !! 操作符
非空断言运算符（!!）将任何值转换为非空类型
```kotlin
val l = b!!.length  //如果 b 为空，就会抛出一个 NPE 异常
```

### 题外话
在大量的第三类库中 
- `Guava` 宣称在 `95%` 的代码处没有 `null`
- `Lombok` 有一个注解 `@Nonull` 可以在生成代码中检测是否为空
- `Spring JPA` 在 `Reposity` 定义中全面支持 `Optional` 

`Nonull` 设计已经算是在 `Java` 领域非常成功的设计了，希望大家早日上船。

## 参考文献
- [计算机科学中的最严重错误，造成十亿美元损失](http://blog.jobbole.com/93667/)
- [null-safety](https://hltj.gitbooks.io/kotlin-reference-chinese/content/txt/null-safety.html)
- [java-optional](http://www.baeldung.com/java-optional)