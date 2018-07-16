---
title:  写给 Java 开发者的 Kotlin 教程 (4) - 控制流表达式
date: 2018-07-12 08:00:15
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/07/12/Pu28Re.png)

有个先贤说过
> 掌握了规则就掌握了一切。

我认为在编程语言中掌握了 `控制语句` 就算是掌握了编程语言（在 `FP` 中效果打折）。所以我们今天开始要去探索编程语言中至为重要的 `Control Flow` 部分。

<!-- more -->

## if 表达式
{% tabs if statement %}
<!-- tab if -->
{% codeblock lang:kotlin %}
var n = 34
if(n % 2 == 0) {
	println("$n is even")
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab if else -->
{% codeblock lang:kotlin %}
var max: Int
if (a > b) {
    max = a
} else {
    max = b
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab if代码块 -->
{% codeblock lang:kotlin %}
val max = if (a > b) {
    print("Choose a")
    a
} else {
    print("Choose b")
    b
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

**敲黑板：** `Kotlin` 和 `Java` 不同之处在于 `if` 的分支可以是代码块，`最后的表达式`作为该块的值，如示例代码中的 `a` 和 `b`。

## When 表达式

```kotlin
when (x) {
    1 -> print("x == 1")
    2 -> print("x == 2")
    else -> { // 注意这个块
        print("x is neither 1 nor 2")
    }
}
```

`when` 将它的参数和所有的分支条件 **顺序** 比较，直到某个分支满足条件，如果其他分支都不满足条件将会求值 `else` 分支。
当然我们也可以把 **多个分支组合** 在一起比如如下：

```kotlin
var dayOfWeek = 6
when(dayOfWeek) {
    1, 2, 3, 4, 5 -> println("Weekday")
    6, 7 -> println("Weekend")
    else -> println("Invalid Day")
}
```

甚至可以是在一个 `Range` 内

```kotlin
var dayOfMonth = 5
when(dayOfMonth) {
    in 1..7 -> println("We're in the first Week of the Month")
    !in 15..21 -> println("We're not in the third week of the Month")
    else -> println("none of the above")
}
```

## For 循环

`for` 循环可以对任何提供 `迭代器（iterator）` 的对象进行遍历

{% tabs for statement %}
<!-- tab for range -->
{% codeblock lang:kotlin %}
for(value in 1..10) {
    print("$value ")
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab for array -->
{% codeblock lang:kotlin %}
var primeNumbers = intArrayOf(2, 3, 5, 7, 11)

for(number in primeNumbers) {
    print("$number ")
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab if代码块 -->
{% codeblock lang:kotlin %}
val max = if (a > b) {
    print("Choose a")
    a
} else {
    print("Choose b")
    b
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}


## While 循环
```kotlin
while (x > 0) {
    x--
}

do {
  val y = retrieveData()
} while (y != null) // y 在此处可见
```
与 `Java` 一致


## Break和continue
与 `Java` 一致