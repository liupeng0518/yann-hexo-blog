---
title:  写给 Java 开发者的 Kotlin 教程 (9) -  面向对象 - 属性
date: 2018-08-07 20:00:48
tags:
toc: true
categories: ["kotlin", "easy-way-to-kotlin-for-java-programer"]
---
![banner](https://s1.ax1x.com/2018/08/07/PsMWvT.jpg)

上一章，我们聊过了面向对象的基础对象。我们继续来来对象中最为重要的 `属性` 和 `方法` 中的第一个。

<!-- more -->

## 主构造函数与属性
```kotlin
// 使用主构造函数与属性
class User(_id: Int, _name: String, _age: Int) {
    // Properties of User class
    val id: Int = _id         // val 不可变量
    var name: String = _name  // var 变量
    var age: Int = _age       // var 变量
}
```

同上文 `Class` 中的可见性修饰一样，`kotlin`中属性的默认的修饰符也是 `public` 所以我们在外部进行变量读取，我们可以使用一下的方式。

```kotlin
val user = User(1, "Jack Sparrow", 44)

// 获得一个属性
val name = user.name

// 设定一个属性
user.age = 46

user.id = 2	// Error: Val 是不能够赋值的
```

## Getters and Setters
`kotlin` 中设定 `Getters` 和 `Setters` 和 `Java` 完全不同，完整的语法规则是

```bash
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

{% tabs Getters and Setters %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
class User(_id: Int, _name: String, _age: Int) {

    val simple: Int? 
    
    val isEmpty: Boolean
        get() = this.size == 0  //函数Getter

    val id: Int = _id
        get() = field
    
    var name: String = _name
        get() = field   //普通属性Getter
        set(value) {  // 这里的 value 是默认名称可以修改为任何值
            field = value  //而这路的 field 是 也就是我们上面声明的 name
        }
    
    var age: Int = _age
        get() = field
        set(value) {
            field = value
        }
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
public class User {
    private String name;
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

肯定有同学想问，为什么看起来这么奇怪，那是因为 `Kotlin` 使用了一种 `Backing Field` 技术，我们默认并非是访问属性本身，是使用 `Getter` 和 `Setter` 去访问属性的。所以下面的代码是等价的。
{% tabs Backing Field %}
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
class DummyClass {
    var size = 0;
    var isEmpty = false
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Java -->
{% codeblock lang:java %}
public final class DummyClass {
   private int size;
   private boolean isEmpty;

   public final int getSize() {
      return this.size;
   }

   public final void setSize(int var1) {
      this.size = var1;
   }

   public final boolean isEmpty() {
      return this.isEmpty;
   }

   public final void setEmpty(boolean var1) {
      this.isEmpty = var1;
   }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

所以我们在访问的时候，其实是通过 `Get` 和 `Set` 方法进行访问的。

## 常量
已知值的属性可以使用 const 修饰符标记为 `编译期常量`。 

```kotlin
const val SUBSYSTEM_DEPRECATED: String = "This subsystem is deprecated"
```

## 延迟初始化
一般地，属性声明为非空类型必须在构造函数中初始化。 但是我们常常需要通过依赖注入来初始化， 或者在单元测试的 setup 方法中初始化。 这种情况下，我们可以使用 `lateinit` 关键字
```kotlin
public class MyTest {
    lateinit var subject: TestSubject  //注意这里并没有申明为  subject? 

    @SetUp fun setup() {
        subject = TestSubject()
    }

    @Test fun test() {
        subject.method()  // 直接解引用
    }
}
```

## 数据类
我们经常创建一些只保存数据的 POJO 类，形如：
```java
class Point {
    private double x;
    private double y;
	public double getX() { return x; }
    public double getY() { return y; }
    public void setX(double v) { x = v; }
    public void setY(double v) { y = v; }
    public boolean equals(Object other) {...}
}
```
这样的类 `Get` 和 `Set` 方式是冗余的，`Kotlin`对其进行了优化。我们可以使用 `data` 关键字进行简化。
```kotlin
data class User(val name: String, val age: Int)
```

编译器自动从主构造函数中声明的所有属性推导出函数：

- equals()/hashCode() 函数
- toString() 函数
- componentN() 解构函数（后续说明）
- copy() 拷贝函数

## 参考资料
- [属性和字段](https://www.kotlincn.net/docs/reference/properties.html)
- [Kotlin Properties, Backing Fields, Getters and Setters](https://www.callicoder.com/kotlin-properties-backing-fields-getters-setters/)