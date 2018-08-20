---
title:  写给 Java 开发者的 Kotlin 教程 (3) - 数据类型
date: 2018-07-111 21:38:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/07/11/PuwrnO.png)

`Kotlin` 的第一个特点就是一门 [`静态类型`](#参考文献) 语言，所以先从如何在 `Kotlin` 中声明变量，`Kotlin` 如何推断变量的类型，以及 `Kotlin` 支持创建变量的基本数据类型开始我们的学习之旅。

<!-- more -->

## 变量

### 数字类型
类型    | Bits
----   | ---
Double |  64
Float  |  32
Long   |  64
Int    |  32
Short  |  16
Byte   |  8

**敲黑板：** 字符类型在 `Kotlin` 不是 `Number类型`

#### 显式转换
我们先看一个例子，再来理解这个特性

{% tabs Explicit Conversions %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val b: Byte = 1 
val i: Int = b // 编译错误
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
byte a = 1;
int b = a; //OK
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

`Kotlin` 提供一系列的数据转换的函数，如下：
- toByte(): Byte
- toShort(): Short
- toInt(): Int
- toLong(): Long
- toFloat(): Float
- toDouble(): Double
- toChar(): Char

### 算术操作
`Kotlin` 支持标准的 `算术操作` 比如 `左移` `右移` 和 `Java` 不同的 `Kotlin` 认为这些也是标准的函数
举个例子

{% tabs operations %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val x = (1 shl 2) and 0x000FF000
//或者是
val x = 1.shl(2).and(0x000FF000)
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
int x = 1 << 2 & 0x000FF000;
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

- shl(bits) – 有符号左移 (Java's <<)
- shr(bits) – 有符号右移 (Java's >>)
- ushr(bits) – 无符号右移 (Java's >>>)
- and(bits) – 与操作
- or(bits) – 或操作 
- xor(bits) – 异或操作 
- inv() –  反转操作

好奇的你，肯定想问为什么可以写成 `val x = (1 shl 2) and 0x000FF000` 这样的语法，其实这个在很多语言中也有
也就是所谓的 `中缀表达式` 这里的内容会在 `函数` 中细说。所以聪明的你也会联系到是否 `+` `-` `*` `%` 也如此。
的确如此……

表达式   | 函数表示
----    | ---
a + b   |  a.plus(b)
a - b   |  a.minus(b)
a * b   |  a.times(b)
a / b   |  a.div(b)
a % b   |  a.rem(b)
a++     |  a.inc()
a−−     |  a.dec()
a > b   |  a.compareTo(b) > 0
a < b   |  a.compareTo(b) < 0
a += b  |  a.plusAssign(b)


## 字符类型
字符文字用单引号括起：如 '1'。可以使用反斜杠转义特殊字符。支持以下转义序列：`\t`，`\b`，`\n`，`\r`，`\`，`\"`，`\\`和`\$`。要对其他字符进行编码，需要使用Unicode转义序列语法：`'\uFF00'`

## 数组类型
在 `Kotlin` 将java中的数据转换为了 `Array` 类
```kotlin
class Array<T> private constructor() {
    val size: Int
    operator fun get(index: Int): T
    operator fun set(index: Int, value: T): Unit

    operator fun iterator(): Iterator<T>
}
```
`Kotlin` 创建一个数组提供一些列简单的函数来帮助我们创建数组，比如
{% tabs array %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val x = arrayOf(1, 2, 3)
// 构造函数 创建一个 Array<String> 初始化为 ["0", "1", "4", "9", "16"]
val asc = Array(5, { i -> (i * i).toString() })
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
int[] a = new int[]{1,2,3,4};
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

除此之外 `Kotlin` 为 `Array` 增加了更多的方便操作的函数。举个例子

```kotlin
var xx = arrayOf(1, 2, 3, 4)
xx = xx.plus(5) //增加一个元素在最后
xx = xx.copyOfRange(0, 2)//拷贝 0 - 2 下标的元素
```

## 字符串

### 字符串字面值
创建一个单行的字符串 `s` 这个和java一样。
```kotlin
val s = "Hello, world!\n"
```
但是 `Kotlin` 支持多行 `String`

{% tabs multie string %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val text = """
    select * from users
    where
      username = 'jack'
"""
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
String text = "select * from users\n" + 
    "where\n" +  
    "username = 'jack'"
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

这个特性在我们需要输出比较复杂的 `Sql`或者是多行文本的情况下是极为方便的。

### 字符串模板
字符串可以包含模板表达式 ，即一些小段代码，会求值并把结果合并到字符串中。 模板表达式以美元符（$）开头，由一个简单的名字构成:
```kotlin
val i = 10
println("i = $i") // 输出“i = 10”
```
或者用花括号括起来的任意表达式
```kotlin
val s = "abc"
println("$s.length is ${s.length}") // 输出“abc.length is 3”
```

### 类型别名
类型别名可以为已有的类型提供替代的名称. 如果类型名称太长, 你可以指定一个更短的名称, 然后使用新的名称。
```kotlin
typealias NodeSet = Set<Network.Node>
typealias FileTable<K> = MutableMap<K, MutableList<File>>
```

### 常量
```kotlin
const val SUBSYSTEM_DEPRECATED: String = "This subsystem is deprecated"
```
## 小结
参考官方文档中还有 `Boolean` 与 `Char` 类型，这两个类型和 Java 是极为接近就不多做阐述。

而且我们从文中，我们可以发现，`Kotlin` 不存在 `Java` 中的 原始数据类型和包装数据类型，不存在 `int Integer` `long Long` `byte Byte` 的关系，也算是去除了 `Java` 中不太 `OOP` 的一部分。

## Q&A
- Q: 执行位运算的执行顺序
  A: 中缀函数调用的优先级低于算术操作符、类型转换以及 rangeTo 操作符，但是高于布尔操作符 && 与 ||、is- 与 in- 检测以及其他一些操作符，所以 `1 shl 2 + 3  === 1 shl (2 + 3)` 

## 参考文献 & 课外阅读
- [弱类型、强类型、动态类型、静态类型语言的区别是什么](https://www.zhihu.com/question/19918532/answer/21647195)
- [Kotlin Operators with Examples](https://www.callicoder.com/kotlin-operators/)
- [Kotlin Infix Notation - Make function calls more intuitive](https://www.callicoder.com/kotlin-infix-notation/)