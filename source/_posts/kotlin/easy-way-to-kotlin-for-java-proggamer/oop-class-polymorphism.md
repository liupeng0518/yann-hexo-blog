---
title:  写给 Java 开发者的 Kotlin 教程 (10) -  面向对象 - 继承与多态
date: 2018-08-14 21:40:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/08/14/P2gOAK.jpg)
继承是面向对象的最重要的特性之一，我们今天就来先看看继承这个特性，我们都知道 `kotlin` 的任何一个类都是继承自 `Any` 类。

```kotlin
class Person // 隐形的 Person 继承自 Any
```

<!-- more -->

## 继承
如果我们需要继承一个类，我们可以使用如下

### 类继承
{% tabs Inheritance %}
<!-- tab Kotlin 继承 -->
{% codeblock lang:kotlin %}
open class Computer {
}

class Laptop: Computer() {
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java 继承 -->
{% codeblock lang:java %}
class Computer {
}

class Laptop: Computer() {
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

这里有些不一样的地方就是，`kotlin` 默认的认为所有的类都是不能够被继承的，我们希望自己的类能够被继承，我们需要在`class` 前面增加 `open` 这个关键字。
我们也知道 `kotlin` 有主构造器，那它的继承语法是这样的。
```kotlin
open class Computer(val name: String,
                    val brand: String) {
}

class Laptop(name: String, 
             brand: String, 
             val batteryLife: Double) : Computer(name, brand) {
   
}
```
### 方法覆写
和类相似，`kotlin` 默认的函数都是 `final` 不能够被重载的。我们也需要在能够被重载的函数上进行 `open` 修饰。
```kotlin
open class Teacher {
    // 必须定义为 open
    open fun teach() {
        println("Teaching...")
    }
}

class MathsTeacher : Teacher() {
    // override 关键字也是必须的
    override fun teach() {
        println("Teaching Maths...")
    }
}
```

### 属性覆写
与函数类似
```kotlin
open class Employee {
    // 必须定义为 open
    open val baseSalary: Double = 30000.0
}

class Programmer : Employee() {
    // override 关键字也是必须的
    override val baseSalary: Double = 50000.0
}
```

### 调用父类方法
与 `Java` 一样，采用的是 `super` 关键字
```kotlin
open class Employee {
    open val baseSalary: Double = 10000.0

    open fun displayDetails() {
        println("I am an Employee")
    }
}

class Developer: Employee() {
    override var baseSalary: Double = super.baseSalary + 10000.0

    override fun displayDetails() {
        super.displayDetails()
        println("I am a Developer")
    }
}
```

## 抽象类与属性函数
我们依然可以使用 `abstract` 定义一个抽象类
```kotlin
abstract class Vehicle
```

`kotlin` 除了支持抽象类，还支持抽象函数和抽象属性，抽象函数和传统的 `java` 类似，但是抽象属性在java中并没有类型的东西，我们来看一下

```kotlin
abstract class Vehicle(val name: String,
                       val color: String,
                       val weight: Double) {  

    // 抽象属性必须被子类覆写
    abstract var maxSpeed: Double

    // 抽象属性必须被子类实现
    abstract fun start()
    abstract fun stop()
}
```

## 类型检查
因为多态的存在，我们需要进行类型检查。
{% tabs Type Checks %}
<!-- tab Kotlin Type Check -->
{% codeblock lang:kotlin %}
val mixedTypeList: List<Any> = listOf("I", "am", 5, "feet", 9.5, "inches", "tall")

for(value in mixedTypeList) {
    if (value is String) {
        println("String: '$value' of length ${value.length} ")
    } else if (value is Int) {
        println("Integer: '$value'")
    } else if (value is Double) {
        println("Double: '$value' with Ceil value ${Math.ceil(value)}")
    } else {
        println("Unknown Type")
    }
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java Type Check -->
{% codeblock lang:java %}
List<Object> objects = Arrays.asList("123", 123, 1D);
for (Object v : objects) {
    if (v instanceof String) {
        ((String) v).toLowerCase();
    }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

看起来好像差不多，`kotlin` 在进行完 `is` 比较之后，在后续的代码块中会自行的将对象转化为 `is` 比较的对象，减少一个强制类型转换。

{% tabs Type Smart Casts %}
<!-- tab Kotlin Cast -->
{% codeblock lang:kotlin %}
val obj: Any = "The quick brown fox jumped over a lazy dog"
if(obj is String) {
    // The variable obj is automatically cast to a String in this scope.
    // No Explicit Casting needed. 
    println("Found a String of length ${obj.length}")
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java Smart Cast -->
{% codeblock lang:java %}
Object obj = "The quick brown fox jumped over a lazy dog";
if(obj instanceof String) {
    // Explicit Casting to `String`
    String str = (String) obj;
    System.out.println("Found a String of length " + str.length());
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

除了这么用之外，`kotlin` 还可以和 `when` 结合

```kotlin
for(value in mixedTypeList) {
    when(value) {
        is String -> println("String: '$value' of length ${value.length} ")
        is Int -> println("Integer: $value")
        is Double -> println("Double: $value with Ceil value ${Math.ceil(value)}")
        else -> println("Unknown Type")
    }
}
```

### 强制类型转换
在 java 里我们可以将对象强制的转换，但是这个行为不一定是安全的，在 `kotlin` 里也提供了这样的行为关键字 `as` 我们可以使用如下代码段。

```kotlin
val obj: Any = "The quick brown fox jumped over a lazy dog"
val str: String = obj as String
println(str.length)

val str: String = obj as String //会抛出运行时 ClassCastException
```

但是`kotlin` 提供了 **安全的转换尝试** 可以使用 `as?` 进行转换

```kotlin
val obj: Any = 123
val str: String? = obj as? String 
println(str)  // 这里会是 null 对象
```

## 参考资料
- [smart casts](https://www.callicoder.com/kotlin-type-checks-smart-casts/)
- [abstract class](https://www.callicoder.com/kotlin-abstract-classes/)