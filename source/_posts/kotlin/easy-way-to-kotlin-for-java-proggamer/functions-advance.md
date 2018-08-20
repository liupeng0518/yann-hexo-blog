---
title: 写给 Java 开发者的 Kotlin 教程 (7) - 函数高阶
date: 2018-07-25 07:58:48
? tags
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---

![banner](https://s1.ax1x.com/2018/07/25/PtmXYn.jpg)

上一章，我们看过 `kotlin` 的一些函数的基本用法，`kotlin` 作为一门年轻的语言，当然不能和 `1995`年的 `Java` 一样，当然还有一些不一样的特性，我们今天就来看看 `kotlin` 的一些函数的高阶特性。

<!-- more -->

## 操作符重载

### 一元表达式

| 表达式 | 翻译为         |
| ------ | -------------- |
| +a     | a.unaryPlus()  |
| -a     | a.unaryMinus() |
| !a     | a.not()        |


举个例子
```kotlin
data class Point(val x: Int, val y: Int)

operator fun Point.unaryMinus() = Point(-x, -y)  //这里是扩展函数

val point = Point(10, 20)
println(-point)  // 打印结果为 "(-10, -20)"
```

### 递增与递减操作符

| 表达式 | 翻译为         |
| ------ | -------------- |
| a++     | a.inc()  |
| a--    | a.dec() |

### 二元操作符
| 表达式 | 翻译为         |
| ------ | -------------- |
| a + b     | a.plus(b)  |
| a - b	   | a.minus(b)  |
| a * b	   | a.times(b)  |
| a / b	   | a.div(b)  |
| a % b	   | a.rem(b)  |
| a..b	   | a.rangeTo(b)  |

```kotlin
data class Counter(val dayIndex: Int) {
    operator fun plus(increment: Int): Counter {
        return Counter(dayIndex + increment)
    }
}
```


## 解构函数

比如我们有这么表达式

```kotlin
var collections: Collection<Pair<Int,String>>
for ((a, b) in collection) { ... }
// 注意这个 (a, b) 就是解构
```

```kotlin
data class Person(var name: String, var age: Int) {
}
var person: Person = Person("Jone", 20)
var (name, age) = person
println("name: $name, age: $age")// 打印：name: Jone, age: 20
```

这里有一个语法糖，对于解构的对象语法，kotlin 会去调用 `componentX` 函数，比如上文的 `name` 和 `age`，对应到 `kotlin` 其实是在 `var (name, age) = person` 调用了 `component1()` 和 `component2()` 函数。

```kotlin
class Test {
    private val i = 1
    private val j = 2

    operator fun component1() = i
    operator fun component2() = j
}
var (i,j) = Test()
```

## 中缀表达式

在 `kotlin` 源码中，我们发现一些函数是被 `infix` 修饰的，这种函数在 `kotlin` 是能够使用 `中缀表达式` 进行操作的，举个例子：

{% tabs Infix Notation %}

<!-- tab Kotlin infix -->

{% codeblock lang:kotlin %}
val map = mapOf(1 to "one", 2 to "two", 3 to "three")
{% endcodeblock %}

<!-- endtab -->
<!-- tab Kotlin without infix -->

{% codeblock lang:kotlin %}
val map = mapOf(1.to("one"), 2.to("two"), 3.to("three"))
{% endcodeblock %}

<!-- endtab -->

{% endtabs %}

`中缀表达式` 直观上的表现形式将 . 和 () 去掉了，带来的是代码的可读性的提高。我们看看`中缀函数的申明`

```kotlin
public infix fun <A, B> A.to(that: B): Pair<A, B> = Pair(this, that)
```

这个函数和普通的函数并没有什么区别，唯独多的也就是 `infix` 关键字。但是如果想要申明一个函数是 `infix` 必须满足一下几个条件：

- 它们必须是`成员函数`或`扩展函数`
- 它们必须只有`一个`参数
- 其参数`不得`接受`可变数量的参数`且`不能有默认值`

### 更多的例子

```kotlin
//比如我们声明
infix fun Int.shl(x: Int): Int { …… }

// 用中缀表示法调用该函数
1 shl 2

// 等同于这样
1.shl(2)
```

### 注意点

1. 中缀函数调用的优先级低于算术操作符、类型转换以及 rangeTo 操作符。
2. 中缀函数调用的优先级高于布尔操作符 && 与 ||、is- 与 in- 检测以及其他一些操作符。

## lambda 表达式 & 高阶函数

### 高阶函数

在阐述 `lambda` 之前，我们先来了解一下 `Kotlin` 函数都是头等的，这意味着它们可以存储在变量与数据结构中、作为参数传递给其他高阶函数以及从其他高阶函数返回。可以像操作任何其他非函数值一样操作函数。

举个例子

```kotlin
fun <T, R> Collection<T>.fold(➊
    initial: R,
    combine: (acc: R, nextElement: T) -> R
): R {➋
    var accumulator: R = initial
    for (element: T in this) {
        accumulator = combine(accumulator, element)
    }
    return accumulator
}
```

➊ 处为 `Collection` 类型增加一个 `fold` 函数，参数有两个 `initial` 和 `combine` 而 `combine` 的定义是一个函数类型 `(acc: R, nextElement: T) -> R` 也就说我们需要传入的第二个 参数 是一个函数。
➋ 定义了返回的 R 类型的类型

所以对于 `Kotlin` 来说，我们可以使用 `函数` 作为参数，也就是具有了 `高阶函数` 的特性。

我们在这里看到了 `Kotlin` 使用类似 `(Int) -> String` 的一系列函数类型来处理函数的声明，而在这里就是我们下面要讲到来的一个概念，Lambda 表达式。

### Lambda 表达式与匿名函数

比如我们有一个 `max` 函数，`max` 是一个高阶函数，它接受一个函数作为第二个参数。 其第二个参数是一个表达式，它本身是一个函数

```kotlin
max(strings, { a, b -> a.length < b.length })
```

而 `{ a, b -> a.length < b.length }` 等价于以下的命名函数:

```kotlin
fun compare(a: String, b: String): Boolean = a.length < b.length
```

Lambda 表达式的完整语法形式如下:

```kotlin
val sum = { x: Int, y: Int -> x + y }
```

我们把所有可选标注都留下，看起来如下:

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y }
```

---

`kotlin` 的 `Lambda` 有以下一些特性

#### lambda 表达式传给最后一个参数

在 Kotlin 中有一个约定：如果函数的最后一个参数接受函数，那么作为相应参数传入的 lambda 表达式可以放在圆括号之外：

```kotlin
val product = items.fold(1) { acc, e -> acc * e }
```

#### it：单个参数的隐式名称

一个 lambda 表达式只有一个参数是很常见的。
如果编译器自己可以识别出签名，也可以不用声明唯一的参数并忽略 ->。 该参数会隐式声明为 `it`

```kotlin
ints.filter { it > 0 } // (it: Int) -> Boolean
```

#### 从 lambda 表达式中返回一个值

我们可以使用返回语法从 lambda 显式返回一个值。 否则，将隐式返回`最后一个表达式`的值。

#### 下划线用于未使用的变量

从 1.1 版本之后，我们可以用下划线取代其名称：

```kotlin
map.forEach { _, value -> println("$value!") }
```

## 柯里化

`FP` 语言里面有个很不错的特性叫 `柯里化`

> 只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数。

比如在 `javascript` 中，我们就可以这么做

```javascript
var add = function(x) {
  return function(y) {
    return x + y;
  };
};

var increment = add(1);
var addTen = add(10);

increment(2); //结果 -> 3
addTen(2); //结果 -> 12
```

而在 `kotlin` 中我们依然可以使用这样的方式比如

```kotlin
fun add(a: Int): (Int) -> Int {
    return { b: Int -> a + b }
}

val increment2 = add(2)
increment2(10) //结果 -> 12
```

其实我们这里看到，在这里，返回了一个 `lambda`，我们利用 `lambda` 完成一个 `柯里化` 的操作。

## 协程

一些 API 启动长时间运行的操作（例如网络 IO、文件 IO、CPU 或 GPU 密集型任务等），并要求调用者阻塞直到它们完成。协程提供了一种避免阻塞线程并用更廉价、更可控的操作替代线程阻塞的方法：`协程挂起`。

### 快速体验

`kotlin` 提供一个新的关键字叫 `suspend` 被这个关键字所修饰的 函数只能被协程调用

```kotlin
suspend fun doSomething(foo: Foo): Bar{
}
```

举个例子

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val result: Deferred<String> = async { doSomethingTimeout() }
	println("I will got the result ${result.await()}")
}

suspend fun doSomethingTimeout(): String {
    delay(1000)
    return "Result"
}
```

在这里， `async` 代码块会新启动一个协程后立即执行，并且返回一个 `Deferred` 类型的值，调用它的 `await` 方法后会`暂停`当前协程，直到获取到 `async` 代码块执行结果，当前协程才会继续执行。

### 协程事件传递

如果我们需要在不同的`协程`之间传递数据，我们需要 `channel` 调用它的 `send` 与 `receive` 方法，就是最简单的使用了。不过要注意，这两个方法会互相等待，所以它们必须运行在`不同的`协程。

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val channel = Channel()
    launch {
        for (x in 1..5) channel.send(x)
        channel.close()
    }
    for (x in 1..5) println(channel.receive())
    //  or `for (x in channel) println(x)`
}
```

## 参考文献

- [kotlin-functions](https://www.callicoder.com/kotlin-infix-notation/)
- [Reference](https://hltj.gitbooks.io/kotlin-reference-chinese/content/txt/functions.html)
