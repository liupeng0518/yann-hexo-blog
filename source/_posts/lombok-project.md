---
title: Project Lombok
date: 2017-06-07 00:12:58
tags:
- Java
categories:
- Java
---


## 目录
- [Lombok简介](#Lombok简介)
- [Lombok安装](#Lombok安装)
- [Lombok使用](#Lombok使用)
- [Lombok原理](#Lombok原理)


## Lombok简介

>Project Lombok makes java a spicier language by adding 'handlers' that know >how to build and compile simple, boilerplate-free, not-quite-java code.

如Github上项目介绍所言，Lombok项目通过添加“处理程序”，使java成为一种更为简单的语言。作为一个Old Java Developer,我们都知道我们经常需要定义一系列的套路，比如定义如下的格式对象。

<!-- more -->

```java
public class DataExample {
  private final String name;
  private int age;
  private double score;
  private String[] tags;
}
```
我们往往需要定义一系列的Get和Set方法最终展示形式如：

```java
public class DataExample {
  private final String name;
  private int age;
  private double score;
  private String[] tags;

  public DataExample(String name) {
    this.name = name;
  }

  public String getName() {
    return this.name;
  }

  void setAge(int age) {
    this.age = age;
  }

  public int getAge() {
    return this.age;
  }

  public void setScore(double score) {
    this.score = score;
  }

  public double getScore() {
    return this.score;
  }

  public String[] getTags() {
    return this.tags;
  }

  public void setTags(String[] tags) {
    this.tags = tags;
  }
}
```

那我们有没有可以简化的办法呢，第一种就是使用IDEA等IDE提供的一键生成的快捷键，第二种就是我们今天介绍的 Lombok项目：

```java
@Data
public class DataExample {
  private final String name;
  @Setter(AccessLevel.PACKAGE)
  private int age;
  private double score;
  private String[] tags;
}
```

Wow...这样就可以完成我们的需求，简直是太棒了，仅仅需要几个注解，我们就拥有了完整的GetSet方法，还包含了ToString等方法的生成。

---

## Lombok安装
整个Lombok只有一个Jar包，[从此处下载](https://projectlombok.org/download)
Lombok支持多种使用安装方式，这里我们讲最常见的对两大IDE的支持

#### 1. Eclipse （含延伸版本）
双击打开 lombok.jar （前提：你得装了JDK）, 可见如下页面点击 Install/Update
![lombok-installer](https://projectlombok.org/img/lombok-installer.png)

恭喜你，已经安装成功了。我们打开 Eclipse 的 About 页面我们可以看见。
![eclipse-about](https://projectlombok.org/img/eclipse-about.png)


#### 2. IntelliJ IDEA
- 定位到 File > Settings > Plugins
- 点击 Browse repositories...
- 搜索 Lombok Plugin
- 点击 Install plugin
- 重启 IDEA

![lombok-Plugin](https://imgtn.gxnotes.com/images/2017/06/134ddf9134f65a7edfb655faac149d6b.jpg)

更多安装请参考：https://projectlombok.org/

---

## Lombok使用
Lombok 其实也不能算是一个特别新的项目从 2011 开始在中心仓库提供支持开始，现在也分为
stable 和 experimental 两个版本，本文侧重介绍 stable 功能：


#### 1. val
如果对其他的语言有研究的会发现，很多语言是使用 var 作为变量申明，val作为常量申明。这里的val也是这个作用。

```java
public String example() {
    val example = new ArrayList<String>();
    example.add("Hello, World!");
    val foo = example.get(0);
    return foo.toLowerCase();
}
```
翻译成 Java 程序是：

```java
public String example() {
    final ArrayList<String> example = new ArrayList<String>();
    example.add("Hello, World!");
    final String foo = example.get(0);
    return foo.toLowerCase();
}
```
> 作者注：也就是类型推导啦。

#### 2. @NonNull
> Null 即是罪恶

```java
public class NonNullExample extends Something {
  private String name;

  public NonNullExample(@NonNull Person person) {
    super("Hello");
    this.name = person.getName();
  }
}
```
翻译成 Java 程序是：

```java
public class NonNullExample extends Something {
  private String name;

  public NonNullExample(@NonNull Person person) {
    super("Hello");
    if (person == null) {
      throw new NullPointerException("person");
    }
    this.name = person.getName();
  }
}

```

#### 3. @Cleanup
> 自动化才是生产力

```java
public class CleanupExample {
  public static void main(String[] args) throws IOException {
    @Cleanup InputStream in = new FileInputStream(args[0]);
    @Cleanup OutputStream out = new FileOutputStream(args[1]);
    byte[] b = new byte[10000];
    while (true) {
      int r = in.read(b);
      if (r == -1) break;
      out.write(b, 0, r);
    }
  }
}
```

翻译成 Java 程序是：

```java
public class CleanupExample {
  public static void main(String[] args) throws IOException {
    InputStream in = new FileInputStream(args[0]);
    try {
      OutputStream out = new FileOutputStream(args[1]);
      try {
        byte[] b = new byte[10000];
        while (true) {
          int r = in.read(b);
          if (r == -1) break;
          out.write(b, 0, r);
        }
      } finally {
        if (out != null) {
          out.close();
        }
      }
    } finally {
      if (in != null) {
        in.close();
      }
    }
  }
}

```

> 作者注： JKD7里面就已经提供 try with resource


#### 4. @Getter/@Setter
> 再也不写 ``` public int getFoo() {return foo;} ```.

```java
public class GetterSetterExample {

  @Getter @Setter private int age = 10;

  @Setter(AccessLevel.PROTECTED) private String name;

  @Override public String toString() {
    return String.format("%s (age: %d)", name, age);
  }
}
```

翻译成 Java 程序是：


```java
public class GetterSetterExample {

  private int age = 10;

  private String name;

  @Override public String toString() {
    return String.format("%s (age: %d)", name, age);
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }

  protected void setName(String name) {
    this.name = name;
  }
}
```

#### 5. @ToString
> Debug Log 最强帮手

```java
@ToString(exclude="id")
public class ToStringExample {
  private static final int STATIC_VAR = 10;
  private String name;
  private Shape shape = new Square(5, 10);
  private String[] tags;
  private int id;

  public String getName() {
    return this.getName();
  }

  @ToString(callSuper=true, includeFieldNames=true)
  public static class Square extends Shape {
    private final int width, height;

    public Square(int width, int height) {
      this.width = width;
      this.height = height;
    }
  }
}
```
翻译后：

```java
public class ToStringExample {
  private static final int STATIC_VAR = 10;
  private String name;
  private Shape shape = new Square(5, 10);
  private String[] tags;
  private int id;

  public String getName() {
    return this.getName();
  }

  public static class Square extends Shape {
    private final int width, height;

    public Square(int width, int height) {
      this.width = width;
      this.height = height;
    }

    @Override public String toString() {
      return "Square(super=" + super.toString() + ", width=" + this.width + ", height=" + this.height + ")";
    }
  }

  @Override public String toString() {
    return "ToStringExample(" + this.getName() + ", " + this.shape + ", " + Arrays.deepToString(this.tags) + ")";
  }
}
```
> 作者注：其实和 org.apache.commons.lang3.builder.ReflectionToStringBuilder 很像。

#### 6. @NoArgsConstructor, @RequiredArgsConstructor and @AllArgsConstructor

```java
@RequiredArgsConstructor(staticName = "of")
@AllArgsConstructor(access = AccessLevel.PROTECTED)
public class ConstructorExample<T> {
  private int x, y;
  @NonNull private T description;

  @NoArgsConstructor
  public static class NoArgsExample {
    @NonNull private String field;
  }
}
```

翻译后：

```java
public class ConstructorExample<T> {
  private int x, y;
  @NonNull private T description;

  private ConstructorExample(T description) {
    if (description == null) throw new NullPointerException("description");
    this.description = description;
  }

  public static <T> ConstructorExample<T> of(T description) {
    return new ConstructorExample<T>(description);
  }

  @java.beans.ConstructorProperties({"x", "y", "description"})
  protected ConstructorExample(int x, int y, T description) {
    if (description == null) throw new NullPointerException("description");
    this.x = x;
    this.y = y;
    this.description = description;
  }

  public static class NoArgsExample {
    @NonNull private String field;

    public NoArgsExample() {
    }
  }
}
```

#### 7. @Data

这个就相当的简单啦，因为我们发现 @ToString, @EqualsAndHashCode, @Getter 都很常用，这个一个注解就相当于

@ToString, @EqualsAndHashCode, @Getter(所有字段), @Setter (所有非final字段), @RequiredArgsConstructor!


#### 8. @Value

```java
@Value public class ValueExample {
  String name;
  @Wither(AccessLevel.PACKAGE) @NonFinal int age;
  double score;
  protected String[] tags;

  @ToString(includeFieldNames=true)
  @Value(staticConstructor="of")
  public static class Exercise<T> {
    String name;
    T value;
  }
}
```

翻译后：

```java
public final class ValueExample {
  private final String name;
  private int age;
  private final double score;
  protected final String[] tags;

  @java.beans.ConstructorProperties({"name", "age", "score", "tags"})
  public ValueExample(String name, int age, double score, String[] tags) {
    this.name = name;
    this.age = age;
    this.score = score;
    this.tags = tags;
  }

  public String getName() {
    return this.name;
  }

  public int getAge() {
    return this.age;
  }

  public double getScore() {
    return this.score;
  }

  public String[] getTags() {
    return this.tags;
  }

  @java.lang.Override
  public boolean equals(Object o) {
    if (o == this) return true;
    if (!(o instanceof ValueExample)) return false;
    final ValueExample other = (ValueExample)o;
    final Object this$name = this.getName();
    final Object other$name = other.getName();
    if (this$name == null ? other$name != null : !this$name.equals(other$name)) return false;
    if (this.getAge() != other.getAge()) return false;
    if (Double.compare(this.getScore(), other.getScore()) != 0) return false;
    if (!Arrays.deepEquals(this.getTags(), other.getTags())) return false;
    return true;
  }

  @java.lang.Override
  public int hashCode() {
    final int PRIME = 59;
    int result = 1;
    final Object $name = this.getName();
    result = result * PRIME + ($name == null ? 43 : $name.hashCode());
    result = result * PRIME + this.getAge();
    final long $score = Double.doubleToLongBits(this.getScore());
    result = result * PRIME + (int)($score >>> 32 ^ $score);
    result = result * PRIME + Arrays.deepHashCode(this.getTags());
    return result;
  }

  @java.lang.Override
  public String toString() {
    return "ValueExample(name=" + getName() + ", age=" + getAge() + ", score=" + getScore() + ", tags=" + Arrays.deepToString(getTags()) + ")";
  }

  ValueExample withAge(int age) {
    return this.age == age ? this : new ValueExample(name, age, score, tags);
  }

  public static final class Exercise<T> {
    private final String name;
    private final T value;

    private Exercise(String name, T value) {
      this.name = name;
      this.value = value;
    }

    public static <T> Exercise<T> of(String name, T value) {
      return new Exercise<T>(name, value);
    }

    public String getName() {
      return this.name;
    }

    public T getValue() {
      return this.value;
    }

    @java.lang.Override
    public boolean equals(Object o) {
      if (o == this) return true;
      if (!(o instanceof ValueExample.Exercise)) return false;
      final Exercise<?> other = (Exercise<?>)o;
      final Object this$name = this.getName();
      final Object other$name = other.getName();
      if (this$name == null ? other$name != null : !this$name.equals(other$name)) return false;
      final Object this$value = this.getValue();
      final Object other$value = other.getValue();
      if (this$value == null ? other$value != null : !this$value.equals(other$value)) return false;
      return true;
    }

    @java.lang.Override
    public int hashCode() {
      final int PRIME = 59;
      int result = 1;
      final Object $name = this.getName();
      result = result * PRIME + ($name == null ? 43 : $name.hashCode());
      final Object $value = this.getValue();
      result = result * PRIME + ($value == null ? 43 : $value.hashCode());
      return result;
    }

    @java.lang.Override
    public String toString() {
      return "ValueExample.Exercise(name=" + getName() + ", value=" + getValue() + ")";
    }
  }
}
```
我们发现了 @Value 就是 @Data 的不可变版本。至于不可变有什么好处。可有参看[此篇](https://blogs.msdn.microsoft.com/ericlippert/2007/11/13/immutability-in-c-part-one-kinds-of-immutability/)

#### 9. @Builder
> 我的最爱

```java
@Builder
public class BuilderExample {
  private String name;
  private int age;
  @Singular private Set<String> occupations;
}
```

翻译后：

```java
public class BuilderExample {
  private String name;
  private int age;
  private Set<String> occupations;

  BuilderExample(String name, int age, Set<String> occupations) {
    this.name = name;
    this.age = age;
    this.occupations = occupations;
  }

  public static BuilderExampleBuilder builder() {
    return new BuilderExampleBuilder();
  }

  public static class BuilderExampleBuilder {
    private String name;
    private int age;
    private java.util.ArrayList<String> occupations;

    BuilderExampleBuilder() {
    }

    public BuilderExampleBuilder name(String name) {
      this.name = name;
      return this;
    }

    public BuilderExampleBuilder age(int age) {
      this.age = age;
      return this;
    }

    public BuilderExampleBuilder occupation(String occupation) {
      if (this.occupations == null) {
        this.occupations = new java.util.ArrayList<String>();
      }

      this.occupations.add(occupation);
      return this;
    }

    public BuilderExampleBuilder occupations(Collection<? extends String> occupations) {
      if (this.occupations == null) {
        this.occupations = new java.util.ArrayList<String>();
      }

      this.occupations.addAll(occupations);
      return this;
    }

    public BuilderExampleBuilder clearOccupations() {
      if (this.occupations != null) {
        this.occupations.clear();
      }

      return this;
    }

    public BuilderExample build() {
      // complicated switch statement to produce a compact properly sized immutable set omitted.
      // go to https://projectlombok.org/features/Singular-snippet.html to see it.
      Set<String> occupations = ...;
      return new BuilderExample(name, age, occupations);
    }

    @java.lang.Override
    public String toString() {
      return "BuilderExample.BuilderExampleBuilder(name = " + this.name + ", age = " + this.age + ", occupations = " + this.occupations + ")";
    }
  }
}
```

builder是现在比较推崇的一种构建值对象的方式。

> 作者注：[生成器模式](https://zh.wikipedia.org/wiki/%E7%94%9F%E6%88%90%E5%99%A8%E6%A8%A1%E5%BC%8F)

#### 10. @SneakyThrows

> to RuntimeException 小助手

```java
public class SneakyThrowsExample implements Runnable {
  @SneakyThrows(UnsupportedEncodingException.class)
  public String utf8ToString(byte[] bytes) {
    return new String(bytes, "UTF-8");
  }

  @SneakyThrows
  public void run() {
    throw new Throwable();
  }
}
```
翻译后

```java
public class SneakyThrowsExample implements Runnable {
  public String utf8ToString(byte[] bytes) {
    try {
      return new String(bytes, "UTF-8");
    } catch (UnsupportedEncodingException e) {
      throw Lombok.sneakyThrow(e);
    }
  }

  public void run() {
    try {
      throw new Throwable();
    } catch (Throwable t) {
      throw Lombok.sneakyThrow(t);
    }
  }
}
```

很好的隐藏了异常，有时候的确会有这样的烦恼，从某种程度上也是遵循的了 [let is crash](http://wiki.c2.com/?LetItCrash)

#### 11. @Synchronized

```java
public class SynchronizedExample {
  private final Object readLock = new Object();

  @Synchronized
  public static void hello() {
    System.out.println("world");
  }

  @Synchronized
  public int answerToLife() {
    return 42;
  }

  @Synchronized("readLock")
  public void foo() {
    System.out.println("bar");
  }
}
```
翻译后

```java
public class SynchronizedExample {
  private static final Object $LOCK = new Object[0];
  private final Object $lock = new Object[0];
  private final Object readLock = new Object();

  public static void hello() {
    synchronized($LOCK) {
      System.out.println("world");
    }
  }

  public int answerToLife() {
    synchronized($lock) {
      return 42;
    }
  }

  public void foo() {
    synchronized(readLock) {
      System.out.println("bar");
    }
  }
}
```

这个就比较简单直接添加了synchronized关键字就Ok啦。不过现在JDK也比较推荐的是 Lock 对象，这个可能用的不是特别多。

#### 12. @Getter(lazy=true)
> 节约是美德

```java
public class GetterLazyExample {
  @Getter(lazy=true) private final double[] cached = expensive();

  private double[] expensive() {
    double[] result = new double[1000000];
    for (int i = 0; i < result.length; i++) {
      result[i] = Math.asin(i);
    }
    return result;
  }
}
```
翻译后：

```java
public class GetterLazyExample {
  private final java.util.concurrent.AtomicReference<java.lang.Object> cached = new java.util.concurrent.AtomicReference<java.lang.Object>();

  public double[] getCached() {
    java.lang.Object value = this.cached.get();
    if (value == null) {
      synchronized(this.cached) {
        value = this.cached.get();
        if (value == null) {
          final double[] actualValue = expensive();
          value = actualValue == null ? this.cached : actualValue;
          this.cached.set(value);
        }
      }
    }
    return (double[])(value == this.cached ? null : value);
  }

  private double[] expensive() {
    double[] result = new double[1000000];
    for (int i = 0; i < result.length; i++) {
      result[i] = Math.asin(i);
    }
    return result;
  }
}
```

#### 13. @Log
> 再也不用写那些差不多的LOG啦

```java
@Log
public class LogExample {

  public static void main(String... args) {
    log.error("Something's wrong here");
  }
}

@Slf4j
public class LogExampleOther {

  public static void main(String... args) {
    log.error("Something else is wrong here");
  }
}

@CommonsLog(topic="CounterLog")
public class LogExampleCategory {

  public static void main(String... args) {
    log.error("Calling the 'CounterLog' with a message");
  }
}
```
翻译后：

```java
public class LogExample {
  private static final java.util.logging.Logger log = java.util.logging.Logger.getLogger(LogExample.class.getName());

  public static void main(String... args) {
    log.error("Something's wrong here");
  }
}

public class LogExampleOther {
  private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExampleOther.class);

  public static void main(String... args) {
    log.error("Something else is wrong here");
  }
}

public class LogExampleCategory {
  private static final org.apache.commons.logging.Log log = org.apache.commons.logging.LogFactory.getLog("CounterLog");

  public static void main(String... args) {
    log.error("Calling the 'CounterLog' with a message");
  }
}
```

---
## Lombok原理

说道 Lombok，我们就得去提到 [JSR 269: Pluggable Annotation Processing API](https://www.jcp.org/en/jsr/detail?id=269)  JSR 269 之前我们也有注解这样的神器，可是我们比如想要做什么必须使用反射，反射的方法局限性较大。首先，它必须定义@Retention为RetentionPolicy.RUNTIME，只能在运行时通过反射来获取注解值，使得运行时代码效率降低。其次，如果想在编译阶段利用注解来进行一些检查，对用户的某些不合理代码给出错误报告，反射的使用方法就无能为力了。而 JSR 269 之后我们可以在 Javac的编译期利用注解做这些事情。所以我们发现核心的区分是在 **运行期** 还是 **编译期**

![javac-flow](http://openjdk.java.net/groups/compiler/doc/compilation-overview/javac-flow.png)

从上图可知，Annotation Processing 是在解析和生成之间的一个步骤。

![Compile%2BProcess](http://4.bp.blogspot.com/_c3fRh9YnHfU/TSZEbT0mC2I/AAAAAAAAABk/VGcO_dLIo9k/s1600/Compile%2BProcess.gif)

上图是 Lombok 处理流程，在Javac 解析成抽象语法树之后(AST), Lombok 根据自己的注解处理器，动态的修改 AST，增加新的节点(所谓代码)，最终通过分析和生成字节码。

关于原理我们大致上的描述下，如果有兴趣可以参考 作者注。



> 作者注：   
> [jdk-compilation-overview](http://openjdk.java.net/groups/compiler/doc/compilation-overview/index.html).   
> [Project Lombok: Creating Custom Transformations](http://notatube.blogspot.jp/2010/12/project-lombok-creating-custom.html)
