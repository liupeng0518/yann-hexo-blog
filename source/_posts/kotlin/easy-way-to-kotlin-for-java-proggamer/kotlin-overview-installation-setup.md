---
title:  写给 Java 开发者的 Kotlin 教程 (1) - 概述
date: 2018-07-09 21:00:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![start](https://s1.ax1x.com/2018/07/10/PnH1S0.png)

`Kotlin` 是由 `JetBrains` 开发的一门编程语言, 也就是那个出品了一些列著名 `IDE` 比如 `IntelliJ IDEA`, `PhpStorm`, `PyCharm`, `ReSharper`的公司。

`Kotlin` 运行在 JVM 之上，并且可以可以编译成 `JavaScript` 和 `机器码` （敲黑板）。 

这是本片教程的第一章，我们先看看 `Kotlin` 的一些特性，让我们快速的了解 `Kotlin` 语言的特点。

<!-- more -->

## Kotlin 特点

### 静态类型

`Kotlin` 是一种静态类型编程语言。这意味着每个变量和表达式的类型在编译时都是已知的。静态类型的优点是编译器可以在编译时自行验证对象的方法调用和属性访问，并防止在运行时出现的许多微不足道的错误。虽然Kotlin是一种静态类型语言，但它并不要求明确指定您声明的每个变量的类型。大多数情况下，Kotlin可以从初始化表达式或周围的上下文推断变量的类型，也就是 **类型推断** 。

### 简明

`Kotlin` 很简洁。减少了在其他OOP语言（如Java）中一直编写的样板代码。（虽然java也有 `Lombok`，但是依然存在一些不太方便的问题比如 `Builder` 的继承）
例如，可以在一行中创建一个包含getter，setters，equals()，hashCode() 和 toString() 方法的POJO类

```kotlin
data class User(val name: String, val email: String, val country: String)
```

### 安全
`Kotlin` 通过支持可空性作为其类型系统的一部分来避免 `NullPointerException`
例如：
```kotlin
String str = "Hello, World"    // 非空类型
str = null // 编译错误
```

如果你想要储存 `null` 值，那必须申明的时候明确指出
```kotlin
String nullableStr? = null   
```

由于 `Kotlin` 知道哪些变量可以为空，哪些变量不可为，因此它可以在编译时检测和禁止不安全的调用，否则会在运行时导致`NullPointerException` (`JDK8 - Optional` 同样解决这个问题，不过并没有编译器上解决)

### 明确
`Kotlin` 是明确的。显性声明被认为是一件好事。
比如
- Kotlin不允许隐式类型转换，例如，int为long，或float为double。它提供了像toLong() 和toDouble() 这样的方法来显式地这样做。
- 默认情况下，Kotlin中的所有类都是 `final`（不可继承）。您需要将类显式标记为 `open` 以允许其他类继承它。同样，默认情况下，类的所有属性和成员函数都是 `final` 。您需要将函数或属性显式标记为打开，以允许子类覆盖它。
- 如果要覆盖父类函数或属性，则需要使用 `override` 修饰符显式注释它。


### 与Java完全交互
`Kotlin` 与 `Java` 可以互相操作。可以轻松地从Kotlin访问Java代码，反之亦然。

### IDEA友好
`Kotlin` 开创了 `IDEA` 先行的先例，在诞生之初就拥有良好的开发工具链。

### 全栈式
kotlin是一门真正全栈式的编程语言，可以开发web，Socket，安卓，js，NativeApp等。

### 免费&开源
天下有免费的馅饼。


## 安装 Kotlin
### 单独安装 Kotlin 编译器
1. 去 [kotlin releases](https://github.com/JetBrains/kotlin/releases/latest) 下载最近的压缩包
2. 解压缩下载的kotlin-compiler-x.x.x.zip文件
3. 将解压后的路径/bin 添加到PATH变量中
4. 验证下 kotlinc

```bash
Welcome to Kotlin version 1.2.51 (JRE 1.8.0_162-ea-b01)
Type :help for help, :quit for quit
>>>
```
### 为 IntelliJ IDEA 安装 Kotlin
新版本的  `IntelliJ IDEA` 都已经支持了 `Kotlin`，我们在创建项目的时候记得勾选 `Kotlin` 即可。

### 为 Eclipse 安装 Kotlin
1. 打开 `Eclipse Marketplace` 并搜索 `Kotlin`
![x](https://s1.ax1x.com/2018/07/10/PnXZIe.png)

2. 安装并重启 `Eclipse`
3. 创建项目选择 `Kotlin` 即可

### Hello Kotlin
```kotlin
fun main(args: Array<String>) {
    println("Hello, Kotlin!")
}
```

用 `Hello Kotlin` 结束这一个介绍的教程，希望大家在后续的课程中玩的开心, have fun。

## 参考文献
- [kotlin-overview-installation-setup](https://www.callicoder.com/kotlin-overview-installation-setup/)
- [absolute-beginners-guide-kotlin](http://blog.teamtreehouse.com/absolute-beginners-guide-kotlin)
- [Kotlin Tutorial](https://www.tutorialkart.com/kotlin-tutorial/)