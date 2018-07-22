---
title:  写给 Java 开发者的 Kotlin 教程 (6) - 函数基础
date: 2018-07-22 15:52:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![kotlin functions](https://s1.ax1x.com/2018/07/22/PGJYwT.png)
函数是构成软件的基础块。我们今天就开始 `Kotlin` 旅程的第二站 - `函数`

<!-- more -->

## 函数定义
我们都知道函数的本质也就是接受一些东西然后吐出一些东西，那最为重要的是 `入参`，`出惨`，然后又知道 `Kotlin` 是一门静态语言，那类型又很重要，那我们来看看 `Kotlin` 是如何申明函数的。

{% tabs for function def %}
<!-- tab Kotlin 函数定义 -->
{% codeblock lang:kotlin %}
fun avg(a: Double, b: Double): Double {
    return  (a + b)/2
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 函数定义 -->
{% codeblock lang:java %}
Double avg(Double a, Double b) {
    return (a + b) / 2;
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

和 `Java` 略有不同 
- `Kotlin` 申明函数使用关键字 `fun`
- `Kotlin` 的`入参`类型是`后置`的
- `Kotlin` 的`出参`类型是`后置`的

所以我们通常定义一个函数都会使用以下的格式
```kotlin
fun functionName(param1: Type1, param2: Type2,..., paramN: TypeN): Type {
	// 函数体
}
```
### 单行函数定义
和 `java` 略有不同，`kotlin` 有一些特殊的函数格式比如 `单行函数`
```kotlin
fun avg(a: Double, b: Double) = (a + b)/2
```
### Unit 类型
如果希望函数无返回值，可以采用 `Unit` 作为返回类型，这个和 `java` 的 `void` 一个含义
```kotlin
fun printAverage(a: Double, b: Double): Unit {
    println("Avg of ($a, $b) = ${(a + b)/2}")
}
```
当然，默认如果这里不填写任何类型，默认即是 `Unit`，下面的代码和上面等价
```kotlin
fun printAverage(a: Double, b: Double) {
    println("Avg of ($a, $b) = ${(a + b)/2}")
}
```
## 函数特性
### 函数默认参数
`Kotlin` 借鉴其他语言的实现，也支持函数的默认参数值。如下
```kotlin
fun displayGreeting(message: String, name: String = "Guest") {
    println("Hello $name, $message")
}
```
这样我们就可以这样调用这个`函数`
```kotlin
displayGreeting("Welcome to the Yann Blog", "John") 
// Hello John, Welcome to the Yann Blog
displayGreeting("Welcome to the Yann Blog") 
// Hello Guest, Welcome to the Yann Blog
```
考虑以下的定义
```kotlin
fun arithmeticSeriesSum(a: Int = 1, n: Int, d: Int = 1): Int {
    return n/2 * (2*a + (n-1)*d)
}
```
那我们尝试用下面的方式调用，我们会发现失败
```kotlin
arithmeticSeriesSum(10) // error: 缺少参数
```
因为我们的参数只能自左向右的传递入函数，所以我们可以这样调用
```kotlin
arithmeticSeriesSum(1, 10)  // 结果 = 55
```
所以我们在申明函数的时候，尽可能的将带默认值的参数放在函数的右侧。

### 命名参数
我们经常在 `Python` 的代码中看见，如下的定义
```python
def named_function(a, b=20, c=10):
    return a + b + c
named_function(10, c=30)
```
这种称之为 `Named Arguments`，`Kotlin` 也可以做到如何
```kotlin
fun arithmeticSeriesSum(a: Int = 1, n: Int, d: Int = 1): Int {
    return n/2 * (2*a + (n-1)*d)
}

arithmeticSeriesSum(n=10)  // 结果 = 55
arithmeticSeriesSum(n=10, d=2, a=3) //OK
arithmeticSeriesSum(n=10, 2) //EROOR
```
注意最后一种方式不被允许了，原因也很简单，因为一旦命名了，我们就无法得知后续的参数应该从哪里开始。

### 可变参数
和 `Java` 不同`Kotlin`采用一种特殊的关键字 `vararg` 来申明可变参数

{% tabs for Varargs %}
<!-- tab Kotlin 可变参数 -->
{% codeblock lang:kotlin %}
fun sumOfNumbers(vararg numbers: Double): Double {
    var sum: Double = 0.0
    for(number in numbers) {
        sum += number
    }
    return sum
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 可变参数 -->
{% codeblock lang:java %}
Double sumOfNumbers(Double... numbers) {
    Double rs = 0D;
    for (Double d : numbers) {
        rs += d;
    }
    return rs / numbers.length;
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}
`值得注意` 在一个函数中只允许出现一个 `vararg`。

## 函数作用域
### 包级别作用域
比如下面这个函数的作用域就是这个包级别，也就是在同一个`包`里都可以访问这个函数。
```kotlin
package maths

fun findNthFibonacciNo(n: Int): Int {
    var a = 0
    var b = 1
    var c: Int

    if(n == 0) {
        return a
    }

    for(i in 2..n) {
        c = a+b
        a = b
        b = c
    }
    return b
}
```
比如我们可以这么调用
```kotlin
package maths

fun main(args: Array<String>) {
    println("10th fibonacci number is - ${findNthFibonacciNo(10)}")
}
```

### 成员函数
和大部分的 `Java` 函数相同，我们可以在 `类` 声明函数
```kotlin
class User(val firstName: String, val lastName: String) {
	// 成员函数
    fun getFullName(): String {
        return firstName + " " + lastName
    }
}
```

### 嵌套函数
在 `java` 程序中，我们经常需要为了代码的可读性将一些列的行为抽象成一个独特的函数，这些函数往往会被定义为 
 `private` 但是往往被复用一次而已。比如下面这个 `calculateBMI`，对比之下，我个人更喜欢的是`嵌套函数`
 
{% tabs Nested Functions %}
<!-- tab Java 嵌套函数 -->
{% codeblock lang:kotlin %}
private Double calculateBMI(Double weightInKg, Double heightInCm) {
    return weightInKg / (heightInCm / 100);
}

Double findBodyMassIndex(Double weightInKg, Double heightInCm) {
    if (weightInKg <= 0) {
        throw new IllegalArgumentException("Weight must be greater than zero")
    }
    if (heightInCm <= 0) {
        throw new IllegalArgumentException("Height must be greater than zero")
    }

    return calculateBMI(weightInKg, heightInCm);
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 嵌套匿名表达式 -->
{% codeblock lang:java %}
Double findBodyMassIndex(Double weightInKg, Double heightInCm) {
    if (weightInKg <= 0) {
        throw new IllegalArgumentException("Weight must be greater than zero");
    }
    if (heightInCm <= 0) {
        throw new IllegalArgumentException("Height must be greater than zero");
    }

    BiFunction<Double, Double, Double> calculateBMI = (w, h) -> w / (h / 100);

    return calculateBMI.apply(weightInKg, heightInCm);
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Kotlin 嵌套函数 -->
{% codeblock lang:java %}
fun findBodyMassIndex(weightInKg: Double, heightInCm: Double): Double {
    
    if(weightInKg <= 0) {
        throw IllegalArgumentException("Weight must be greater than zero")
    }
    if(heightInCm <= 0) {
        throw IllegalArgumentException("Height must be greater than zero")
    }

    fun calculateBMI(weightInKg: Double, heightInCm: Double): Double {
        val heightInMeter = heightInCm / 100
        return weightInKg / (heightInMeter * heightInMeter)
    }

    // 计算 BMI
    return calculateBMI(weightInKg, heightInCm)
}

{% endcodeblock %}
<!-- endtab -->
{% endtabs %}


## 内联函数
内联函数可以在编译器展开，减少 `Function Stack` 的使用，对于提高性能有 `迷之作用`
我们想要将一个函数设置为内联函数只需要如下声明即可：
```kotlin
inline fun <T> lock(lock: Lock, body: () -> T): T {
    // 函数体
}
```
只需要在函数的前面增加一个 `inline` 关键字即可。


## 扩展函数
想想一个场景，我们需要给某个类型的对象增加一个函数，而这个类型的源码可能是在某个第三方的`Jar`内，我们仅仅能通过继承来实现我们的需求，而现在 `kotlin` 可以通过拓展函数来实现。
```kotlin
fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1] // “this”对应该列表
    this[index1] = this[index2]
    this[index2] = tmp
}
```
这样我们就为了 `MutableList<Int>` 增加一个新的函数 `swap(index1: Int, index2: Int)` 

**值得注意：** 扩展是静态解析的，也就是说并不会因为运行时的状态而采用多态的函数进行调用。举个例子
```kotlin
open class C

class D: C()

fun C.foo() = "c"

fun D.foo() = "d"

fun printFoo(c: C) {
    println(c.foo())
}

printFoo(D())
// "c" not "d"
```


## 函数泛型
既然有函数，怎么可能没有泛型呢？`Kotlin` 的泛型系统比 `java` 更加复杂，关于这块，我们将在一个独立的章节进行讲解。
```kotlin
fun <T> singletonList(item: T): List<T> {
    // 函数体
}
```

## 参考文献
- [扩展特性](https://hltj.gitbooks.io/kotlin-reference-chinese/content/txt/extensions.html)
- [kotlin-functions](https://www.callicoder.com/kotlin-functions/)
- [Reference](https://hltj.gitbooks.io/kotlin-reference-chinese/content/txt/functions.html)




