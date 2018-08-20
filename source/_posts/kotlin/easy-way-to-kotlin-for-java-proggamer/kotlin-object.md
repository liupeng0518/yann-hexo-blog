---
title:  写给 Java 开发者的 Kotlin 教程 (11) - object 关键字
date: 2018-08-15 14:40:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
## 对象表达式
`kotlin` 里面有个关键字 `object`，用作创建一个对象。
比如我们有个例子
```kotlin
open class A(x: Int) {
    public open val y: Int = x
}

interface B {...}

val ab: A = object : A(1), B {
    override val y = 15
}
```

<!-- more -->


又比如我们可以创建一个独立的对象
```kotlin
fun foo() {
    val adHoc = object {
        var x: Int = 0
        var y: Int = 0
    }
    print(adHoc.x + adHoc.y)
}
```

我们可以申明一个匿名类
```kotlin
fun countClicks(window: JComponent) {
    var clickCount = 0
    var enterCount = 0

    window.addMouseListener(object : MouseAdapter() {
        override fun mouseClicked(e: MouseEvent) {
            clickCount++
        }

        override fun mouseEntered(e: MouseEvent) {
            enterCount++
        }
    })
    // ...
}
```

## 单例对象
在 `kotlin` 中引入了 `object` 关键字，让这个事情变的简单起来。
```kotlin
object Payroll {
    val allEmployees = arrayListOf<Person>()

    fun calculateSalary() {
        for (person in allEmployees) {
            ...
        }
    }
}
```

## 伴生对象
`kotlin` 去除了 `static` 关键字，如果我们希望声明一个 `static` 的函数，我们可以可以定一个 `pacage-level` 的函数，但是 `pacage-level` 不能够访问一个类的私有变量，解决方案是采用 `companion` 关键字进行伴生对象申明。

```kotlin
class A {
    companion object {
        fun bar() {
            println("Companion object called")
        }
    }
}
>>> A.bar()
Companion object called
```

值得注意，你如果希望能够被 `java` 调用，你需要在属性或者函数上加上一个注解 `@JvmStatic` / `@JvmField`。

## 匿名类
`object` 函数还可以构建匿名类

```kotlin
val listener = object : MouseAdapter {
    override fun mouseClicked(e: MouseEvent) { ... }
    override fun mouseEntered(e: MouseEvent) { ... }
}
```


