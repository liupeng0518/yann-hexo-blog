---
title:  写给 Java 开发者的 Kotlin 教程 (8) -  面向对象 - 基础
date: 2018-08-03 15:00:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/08/03/PBGkQK.jpg)

话不多说，Kotlin依然是一个门 `OOP` 语言，我们从今天开始我们来踏上最后一段旅程。

<!-- more -->

## Class 类
`Kotlin` 与 `Java` 一样，依然保留 `class` 这个关键字，我们可以使用类似于 `Java` 的类声明

{% tabs class def %}
<!-- tab Kotlin 类定义 -->
{% codeblock lang:kotlin %}
class Person {
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 类定义 -->
{% codeblock lang:java %}
class Person {
    //没啥区别
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

相比较 `Java` 而言， `Kotlin` 允许省略空声明的 一对`{}` 即是
```kotlin
class Person //合法声明
```

实例化一个类的时候不再需要 `new` 这个关键字。

{% tabs instance def %}
<!-- tab Kotlin 实例化 -->
{% codeblock lang:kotlin %}
val person = Person()
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 实例化 -->
{% codeblock lang:java %}
Person person = new Person()
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

## 构造函数
`Kotlin` 具有两种 `构造函数` 分别是 `主构造函数`和 `次构造函数`，

`主构造函数`仅允许出现一次，出现在类定义上
```kotlin
class Person constructor(firstName: String) { ... }
```
如果主构造函数没有任何注解或者可见性修饰符，可以省略这个 `constructor` 关键字。

```kotlin
class Person(firstName: String) { ... }
```

`次构造函数` 声明在类定义体内
```kotlin
class Person {
    constructor(parent: Person) {
        parent.children.add(this)
    }
}
```
如果类有一个主构造函数，每个次构造函数需要委托给主构造函数， 可以直接委托或者通过别的次构造函数间接委托。委托到同一个类的另一个构造函数用 this 关键字即可：
```kotlin
class Person(val name: String) {
    constructor(name: String, parent: Person) : this(name) {
        parent.children.add(this)
    }
}
```

### Initializer 块
除构造函数之外，`Kotlin` 还有一个特殊的关键字 `init`，因为 `主构造函数` 不能申明代码，所以初始化的逻辑只能放在一个单独的代码块中，这块区域被称之为 `初始化块`

```kotlin
class Person(_firstName: String, _lastName: String) {
    var firstName: String
    var lastName: String

    // 初始化块
    init {
        this.firstName = _firstName
        this.lastName = _lastName

        println("Initialized a new Person object with firstName = $firstName and lastName = $lastName")
    }
}
```

值得注意的 `init` 段是允许出现多次的，执行的顺序是按照书写的顺序执行。
```kotlin
class InitOrderDemo(name: String) {
    val firstProperty = "First property: $name".also(::println)

    init {
        println("First initializer block that prints ${name}")
    }

    val secondProperty = "Second property: ${name.length}".also(::println)

    init {
        println("Second initializer block that prints ${name.length}")
    }
}
```

## 继承
`Kotlin` 中的所有的类都 继承于 `Any` 类，
> 注意：Any 并不是 java.lang.Object；尤其是，它除了 equals()、hashCode()和toString()外没有任何成员。

需要继承类也非常的简单，只需要使用 `:` 把类型放到类头的冒号之后即可。
{% tabs extend def %}
<!-- tab Kotlin 继承 -->
{% codeblock lang:kotlin %}
open class Base(p: Int)

class Derived(p: Int) : Base(p)
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 继承 -->
{% codeblock lang:java %}
class Derived extend Base{

}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

> 类上的 open 标注与 Java 中 final 相反，它允许其他类从这个类继承。默认情况下，在 Kotlin 中所有的类都是 final， 对应于《Effective Java》第三版书中的第 19 条：要么为继承而设计，并提供文档说明，要么就禁止继承。

## 覆盖方法 & 覆盖属性

Kotlin 力求清晰显式。与 Java 不同，Kotlin 需要显式标注可覆盖的成员。
```kotlin
open class Base {
    open fun v() { ... }
    fun nv() { ... }
}
class Derived() : Base() {
    override fun v() { ... }
}
```

Derived.v() 函数上必须加上 `override` 标注。如果没写，编译器将会报错。 如果函数没有标注 open 如 Base.nv()，则子类中不允许定义相同签名的函数， 不论加不加 override。在一个 final 类中（没有用 open 标注的类），开放成员是禁止的。

覆盖属性 与方法覆盖类似
```kotlin
open class Foo {
    open val x: Int get() { …… }
}

class Bar1 : Foo() {
    override val x: Int = ……
}
```

## 调用超类实现
派生类中的代码可以使用 `super` 关键字调用其超类的函数与属性访问器的实现：
```kotlin
open class Foo {
    open fun f() { println("Foo.f()") }
    open val x: Int get() = 1
}

class Bar : Foo() {
    override fun f() { 
        super.f()
        println("Bar.f()") 
    }

    override val x: Int get() = super.x + 1
}
```

## 抽象类
类和其中的某些成员可以声明为 `abstract`。 抽象成员在本类中可以不用实现。 需要注意的是，我们并不需要用 open 标注一个抽象类或者函数——因为这不言而喻。
我们可以用一个抽象成员覆盖一个非抽象的开放成员。
```kotlin
open class Base {
    open fun f() {}
}

abstract class Derived : Base() {
    override abstract fun f()
}
```

## 可见性修饰
`Kotlin` 与 `Java` 类型具有四种可见性修饰关键字 `public`, `private`, `protected` 和 `internal`，前三个和 `Java` 中的语义是一样的，最后`internal` 是指在同一个 模块下 可以被访问，这里的`模块`的定义是：
- 一个 IntelliJ IDEA 模块；
- 一个 Maven 项目；
- 一个 Gradle 源集（例外是 test 源集可以访问 main 的 internal 声明）；
- 一次 ＜kotlinc＞ Ant 任务执行所编译的一套文件。

## 参考文献
- [kotlin Refence](https://www.kotlincn.net/docs/reference)
- [visibility-modifiers](https://www.kotlincn.net/docs/reference/visibility-modifiers.html#%E6%A8%A1%E5%9D%97)