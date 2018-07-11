---
title:  写给 Java 开发者的 Kotlin 教程 (2) - 基础语法
date: 2018-07-10 21:00:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/07/11/PuAzgf.png)

作为教程的第一章，我们先从 `基础语言` 开始了解一下 `Kotlin`，并了解和 `Java` 的不同之处。

<!-- more -->

## 包定义

{% tabs package %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
package my.demo
import java.util.*
// ...
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
package my.demo
import java.util.*
// 和左边是一样的
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：** `Kotlin` 和 `Java` 不同之处在于，`Kotlin` 源文件可以任意放置在文件夹内，而 `Java` 不行。
这个特性对于代码部分没有太多的影响，但是针对 `Test` 部分的代码，我们可以将测试的代码更灵活的存放于不同的 `目录` 之下。

## 函数定义
{% tabs functuin %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
fun sum(a: Int, b: Int): Int {
    return a + b
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
int sum(int a, int b) {
    return a + b;
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：** `Kotlin` 更注重代码的可读性，`变量名` 在 `变量类型` 之前，返回值在函数声明的最后。
![function syntax](https://s1.ax1x.com/2018/07/11/PumGwD.png)

## 变量定义
{% tabs vars %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val a: Int = 1  // 立即赋值
val b = 2   // `Int` 类型被推导而来
val c: Int  // 声明而无赋值
c = 3       // 延迟赋值
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
final int a = 1;
final int b;
b = 1;
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：** `Kotlin` 拥有更强的 `类型推导能力`，大部分情况下都可以通过 `右值` 推导出 `左值`的申明。另外 `Kotlin` 中区分 `val` 和 `var`， `val` 类似于 `Java` 中的 `final` 修饰符仅仅允许进行一次赋值操作。如果需要使用变量需要采用如下：

```kotlin
var x = 5 
x += 1
```

## 注释
```kotlin
// This is an end-of-line comment

/* This is a block comment
   on multiple lines. */
```

和 `Java` `Javascript` 都是一样的，唯一不同的是 `Kotlin` 的注释是允许嵌套的。比如如下是合法的
```kotlin
/*
  /*
    /*
    nest comment
     */
   */
 */
```

## String 模版

{% tabs string template %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
var a = 1
val s1 = "a is $a"  // 简单映射

a = 2
val s2 = "${s1.replace("is", "was")}, but now is $a" // 复杂表达式嵌入
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
int a = 1;
String s1 = String.format("a is %d",a);
String s2 = String.format("%s, but now is %a", s1.replace("is", "was"), a);
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

更多功能后续介绍

## 简化IF表达式

{% tabs expressions %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
fun maxOf(a: Int, b: Int) = if (a > b) a else b 
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
int maxOf(int a, int b) {
    if (a > b) {
        return a;
    } else {
        return b;
    }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：**  `Kotlin` 简化一些简单表达式。

## 可空类型
`Kotlin` 当值可能为 `null` 时，必须将引用显式标记为可为空。
{% tabs nullable %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
fun parseInt(str: String): Int? {
    // 任何Int 或者 null
}

fun parseInt(str: String): Int {
    // 任何Int 不可能为 null
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
Integer paserInt(String input){
    // 无法判断
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：**  `Kotlin` 在解决 `NPE` 问题从编译器的层面去解决，关于这个设计模式，我们将单独开辟一篇进行讲解。

```kotlin
fun printProduct(arg1: String, arg2: String) {
    val x = parseInt(arg1)
    val y = parseInt(arg2)

    // 使用 x 和 y 之前必须进行 非空判断
    if (x != null && y != null) {
        // x and y are automatically cast to non-nullable after null check
        println(x * y)
    }
    else {
        println("either '$arg1' or '$arg2' is not a number")
    }
}
```

## 类型检查与自动转换
{% tabs type checks and automatic casts %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
fun getStringLength(obj: Any): Int? {
    if (obj is String) {
        // `obj` 被自动转行成了  `String`
        return obj.length
    }
    return null
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
Integer getStringLength(Object obj) {
    if (obj instanceof String) {
        return ((String) obj).length();
    }
    return null;
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：**  `Kotlin` 必须要在判断类型之后再进行类型转换，可以节约很多代码的篇幅。


## Range

{% tabs range %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
val x = 10
val y = 9
if (x in 1..y+1) {
    println("fits in range")
}

for (x in 1..10 step 2) {
    print(x)
}
println()
for (x in 9 downTo 0 step 3) {
    print(x)
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
for(int i = 0 ;i < 9;i++){
    System.out.println(i);            
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：**  虽然只是一个语法糖，可是好吃呀。


## when
{% tabs when %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
fun describe(obj: Any): String =
when (obj) {
    1          -> "One"
    "Hello"    -> "Greeting"
    is Long    -> "Long"
    !is String -> "Not a string"
    else       -> "Unknown"
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
//switch & case 抱歉做不到
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}
**敲黑板：**  `when` 是一个增强的 `switch` 语法，尤其是在多态的情况下，尤为有用。

## 参考文献
- [kotlin reference](http://kotlinlang.org/docs/reference/basic-syntax.html)