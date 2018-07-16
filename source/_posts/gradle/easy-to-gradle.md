---
title: Gradle 简易入门教程
date: 2018-07-14 08:40:48
tags:
toc: true
categories: ["buildtool", "gradle"]
---
![banner](https://s1.ax1x.com/2018/07/14/PMmeYQ.png)

`Gradle` 是一种构建工具，它抛弃了基于XML的构建脚本，取而代之的是采用一种基于 `Groovy`（现在也支持 `Kotlin`）的内部领域特定语言。

<!--more -->

## Gradle 特点
1. Gradle是很成熟的技术，可以处理大规模构建     
2. Gradle对多语言、多平台支持性更好
3. Gradle关注在构建效率上
4. Gradle发布很频繁，重要feature开发计划透明化
5. Gradle社区很活跃，并且增加迅速


## 安装Gradle
- 从 [这个页面](https://gradle.org/install/) 下载二进制文件。
- 解压Zip文件，加入环境变量（在PATH中加入GRADLE_HOME/bin目录）

如果在安装过程中遇到问题，可以进一步查看官方的 [安装指南](https://gradle.org/install/)。
最后验证一下 `Gradle` 是否工作正常

```bash
gradle -v
------------------------------------------------------------
Gradle 4.2.1
------------------------------------------------------------

Build time:   2017-10-02 15:36:21 UTC
Revision:     a88ebd6be7840c2e59ae4782eb0f27fbe3405ddf

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.6 compiled on June 29 2015
JVM:          1.8.0_162-ea (Oracle Corporation 25.162-b01)
OS:           Mac OS X 10.13.5 x86_64
```

## Gradle 快速体验
###  初始化一个项目
- 创建一个 `demo` 目录
    ```bash
    ❯ mkdir gradle-demo
    ```
- 创始化 `Gradle` 项目
    ```bash
    ❯ gradle init 
    Starting a Gradle Daemon (subsequent builds will be faster)

    BUILD SUCCESSFUL in 3s
    2 actionable tasks: 2 executed
    ```
### Gradle 目录结构
我们看看上一步我们生成了什么文件

```text
├── build.gradle  ❶
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar    ➋
│       └── gradle-wrapper.properties  ➌
├── gradlew    ➍
├── gradlew.bat  ➎
└── settings.gradle  ➏
```
❶ 当前项目的配置脚本
➋ `Gradle Wrapper` 的执行jar包（后续介绍）
➌ `Gradle Wrapper` 的配置文件
➍ `Gradle Wrapper` Unix 系执行脚本
➎ `Gradle Wrapper` Windows 系执行脚本
➏ 项目脚本设置

### 创建一个 Task
`Gradle`提供了用于通过基于`Groovy`或`Kotlin`的DSL创建和配置。项目包括一组`Task`，每个`Task`执行一些基本操作。
- 创建一个目录叫 `src`
- 在`src`目录创建一个 `myfile.txt`
- 在构建文件中定义一个名为Copy的类型 `Task` ，该任务将src目录复制到名为dest的新目录

{% tabs create task %}
<!-- tab Groovy -->
{% codeblock lang:groovy %}
task copy(type: Copy, group: "Custom", description: "Copies sources to the dest directory") {
    from "src"
    into "dest"
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Kotlin -->
{% codeblock lang:kotlin %}
tasks.create<Copy>("copy") {
    description = "Copies sources to the dest directory"
    group = "Custom"

    from("src")
    into("dest")
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

`group` 和 `description` 是自定义的任意值。现在让我们执行这个 `task`

```bash
❯ ./gradlew copy
> Task :copy

BUILD SUCCESSFUL in 0s
1 actionable task: 1 executed
```
再一次 `ls` 我们就可以看见 `gradle` 为我们创建了一个新的 `dest` 目录并且将 文件复制进去

## Gradle Task
在`Gradle`中，有两个基本概念：项目和任务。
- 项目是指我们的构建产物（比如Jar包）或实施产物（将应用程序部署到生产环境）一个项目包含一个或多个任务。
- 任务是指不可分的最小工作单元，执行构建工作（比如编译项目或执行测试）。
![](https://s1.ax1x.com/2018/07/14/PMCmKe.png)

在项目目录中的 `build.gradle` 指定了一个项目和它的任务。

### Task 执行顺序
任务可能依赖于其他任务，或者可能被安排为始终在另一个任务之后运行。

```groovy
project('projectA') {
    task taskX(dependsOn: ':projectB:taskY') {
        doLast {
            println 'taskX'
        }
    }
}

project('projectB') {
    task taskY {
        doLast {
            println 'taskY'
        }
    }
}
```

```bash
> gradle -q taskX
taskY
taskX  // taskx 在 y 之后
```

我们可以用 `dependsOn` 让我们的 `Task` 有顺序的运行起来
[Task 文档](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html)


## Gradle 插件
看到这里，如果每一件事情我们都需要写 `Task` 岂不是会累死，而且很多功能是可以被复用的，所以`Gradle` 提供一个 `插件` 功能，`Gradle` 默认就内置了大量的插件，比如在 `base` 中有一系列的功能。

{% tabs plugin %}
<!-- tab Groovy  build.gradle-->
{% codeblock lang:groovy %}
plugins {
    id "base"
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Kotlin build.gradle.kts -->
{% codeblock lang:kotlin %}
plugins {
    id("base")
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

这个时候我们就可以利用一些额外的 `Task`，举个例子，我们要把一个目录中的东西都打成一个 `ZIP` 压缩包。
{% tabs zip %}
<!-- tab Groovy  build.gradle-->
{% codeblock lang:groovy %}
task zip(type: Zip, group: "Archive", description: "Archives sources in a zip file") {
    from "src"
    setArchiveName "basic-demo-1.0.zip"
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Kotlin build.gradle.kts -->
{% codeblock lang:kotlin %}
tasks.create<Zip>("zip") {
    description = "Archives sources in a zip file"
    group = "Archive"

    from("src")
    setArchiveName("basic-demo-1.0.zip")
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

`Gradle` 的设计理念是
- 在项目中添加新任务
- 为新加入的任务提供默认配置，这个默认配置会在项目中注入新的约定（如源文件位置）。
- 加入新的属性，可以覆盖插件的默认配置属性。
- 为项目加入新的依赖。

### Gradle Java
`Gradle` 内置了 `Java` 插件，Java插件将Java编译以及测试和捆绑功能添加到项目中。它是许多其他Gradle插件的基础。
如果我们需要使用 `Java` 插件 修改 `build.gradle`

```groovy
plugins {
    id 'java'
}
```

一旦导入了 Java 插件，就会有一系列的默认的配置值，并且会导入大量的 `Task`

Task    | 含义
----   | ---
compileJava(type: JavaCompile)          |  Java 编译
processResources(type: Copy)            |  拷贝 Resources 资源
classes(type: Task)                     |  组装 Java 类
compileTestJava(type: JavaCompile)      |  Java Test 编译
processTestResources(type: Copy)        |  拷贝 Test Resources 资源
testClasses(type: Task)                 |  组装 Test 类
jar(type: Jar)                          |  合成Jar包
javadoc(type: Javadoc)                  |  生成 doc 文档 
test(type: Test)                        |  运行测试
uploadArchives(type: Upload)            |  上传 jar 到仓库
clean(type: Delete)                     |  clean

我们从这些 `Task` 名字就可以看出来他们分别作作了， 和其他的设计理念类型，在 `Task` 也会嵌入一些生命周期，其实原理也就是我们之前看的执行顺序。[Java Lifecycle](https://docs.gradle.org/current/userguide/java_plugin.html#_lifecycle_tasks)

![JavaPluginTasks](https://docs.gradle.org/current/userguide/img/javaPluginTasks.png)


### 资源
[Gradle插件仓库](https://plugins.gradle.org/)

## Gradle 依赖管理
先盗取一张官方的图 
![](https://docs.gradle.org/current/userguide/img/dependency-management-dependencies-to-modules.png
)

和 `Maven` 类似，`Gradle` 也会将依赖缓冲在本地中，方便在无网的环境使用，和依赖管理相关的有两个参数，举个例子。

```groovy
repositories {
    mavenCentral() // 定义仓库
}

dependencies {
    compile 'org.springframework:spring-web:5.0.2.RELEASE' // 定义依赖
}
```

Gradle支持以下仓库格式：
- Ivy仓库
- Maven仓库
- Flat directory仓库

### lvy仓库
我们可以通过URL地址或本地文件系统地址，将Ivy仓库加入到我们的构建中。
```groovy
repositories {
    ivy {
        url "http://ivy.petrikainulainen.net/repo"
    }
}
//或者是本地
repositories {
    ivy {       
        url "../ivy-repo"
    }
}
```

### Maven仓库
```groovy
repositories {
    maven {
        url "http://maven.petrikainulainen.net/repo"
    }
}
```
在加入Maven仓库时，Gradle提供了三种“别名”供我们使用，它们分别是
- `mavenCentral()`别名，表示依赖是从Central Maven 2 仓库中获取的。
- `jcenter()`别名，表示依赖是从Bintary’s JCenter Maven 仓库中获取的。
- `mavenLocal()`别名，表示依赖是从本地的Maven仓库中获取的。


### Flat Directory仓库
```groovy
repositories {
    flatDir {
        dirs 'lib'
    }
}

//多个仓库
repositories {
    flatDir {
        dirs 'libA', 'libB'
    }
}
```

### 依赖管理
在配置完项目仓库后，我们可以声明其中的依赖，首先 `Java` 插件指定了若干依赖配置项
- `compile` 配置项中的依赖是依赖必须的。
- `runtime` 配置项中包含的依赖在运行时是必须的。
- `testCompile` 配置项中包含的依赖在编译项目的测试代码时是必须的。
- `testRuntime` 配置项中包含的依赖在运行测试代码时是必须的。

在 `Gradle` 最新版本中更是增加 
- `implementation` 配置项中的实现类
- `api` 配置项中的暴露API

```groovy
dependencies {
    api 'commons-httpclient:commons-httpclient:3.1'
    implementation 'org.apache.commons:commons-lang3:3.5'
}
```

#### 声明项目依赖
最普遍的依赖称为外部依赖，这些依赖存放在外部仓库中。一个外部依赖可以由以下属性指定：

- `group`属性指定依赖的分组（在`Maven`中，就是`groupId`）
- `name`属性指定依赖的名称（在`Maven`中，就是`artifactId`）
- `version`属性指定外部依赖的版本（在`Maven`中，就是`version`）

```groovy
dependencies {
    compile group: 'foo', name: 'foo', version: '0.1'
}
// 我们也可以合并到一起去
dependencies {
    compile 'foo:foo:0.1'
}
```

#### 声明项目文件依赖
我们如何依赖本地的一些 `jar` 呢，正确的操作是如下
```groovy
dependencies {
    compile files('libs/commons-lang.jar', 'libs/log4j.jar')
}
```

#### 声明依赖排除项目
我们都知道在 `Maven`中我们有
```xml
<dependency>
    <groupId>sample.ProjectB</groupId>
    <artifactId>Project-B</artifactId>
    <version>1.0-SNAPSHOT</version>
    <exclusions>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
而在 `Gradle` 中，我们可以这么做。

```groovy
compile('com.example.m:m:1.0') {
    exclude group: 'org.unwanted', module: 'x 
}
```

## Gradle 与 Kotlin
我们想要在 `gradle` 中增加 `kotlin` 非常的简单，仅仅需要在 `build.gradle` 增加
```groovy
plugins {
     id "org.jetbrains.kotlin.jvm" version "x.x.xx" // 增加插件
}

dependencies {
  compile "org.jetbrains.kotlin:kotlin-stdlib:x.xx.xx" // 增加依赖
}
```
大功告成，对了，默认的 `Kotlin` 的源码路径是 `src/main/kotlin`, 测试源码是 `src/text/kotlin` 如果需要修改可以使用 

```groovy
sourceSets {
    main.kotlin.srcDirs += 'src/main/myKotlin'
    main.java.srcDirs += 'src/main/myJava'
}
```

## 其他
### Gradle Wrapper
`Gradle Wrapper` 做了三件事情
- 解析参数传入 gradlew
- 安装正确的 `Gradle` 版本
- 调用 `Gradle` 执行命令

Ops，`Gradle Wrapper` 为什么还需要安装 `Gradle`，我们在用 `Maven` 都知道，我们需要自己先安装好一个 `Maven`版本，因为 `Maven` 发展多年，现在已经稳定，已经不存在很多个版本并存的现状了，但是我们依然需要去在每个机器上去安装，那我什么我们不能在自己的 `构建脚本` 中就指定我们的构建工具呢？
所以我们在 `wrapper/gradle-wrapper.properties` 中就可以发现 `distributionUrl=https\://services.gradle.org/distributions/gradle-4.2.1-bin.zip` 这里也就是定义了我们的gradle所使用的版本。
![wrapper-workflow](https://docs.gradle.org/current/userguide/img/wrapper-workflow.png)


## Gradle In Real World
```groovy
// 定义一堆基础插件
apply plugin: 'java'
apply plugin: 'maven'
apply plugin: "jacoco"
apply plugin: 'checkstyle'
apply plugin: 'pmd'
apply plugin: 'findbugs'
apply plugin: 'eclipse'
apply plugin: 'idea'
// 定义项目属性
group = 'Common'
version = '1.0.0'
description = """Giant common library"""

// 定义依赖仓库
repositories {
    mavenCentral()
}

// 额外增加source path
sourceSets {
    main {
        resources {
            srcDir "src/main/profiles/${profile}"
        }
    }
}
// project依赖
dependencies {
    compile 'ch.qos.logback:logback-core:1.0.13'
    compile 'ch.qos.logback:logback-classic:1.0.13'
    compile 'ch.qos.logback:logback-access:1.0.13'
    compile 'commons-io:commons-io:2.0.1'
    compile 'commons-lang:commons-lang:2.6'
    compile 'joda-time:joda-time:1.6.2'
    compile 'org.testng:testng:6.8.7'
    compile 'com.googlecode.jmockit:jmockit:1.5'
    ...
}
// task配置
checkstyle {
    ignoreFailures = true
    sourceSets = [sourceSets.main]
}
findbugs {
    ignoreFailures = true
    sourceSets = [sourceSets.main]
}
pmd {
    ruleSets = ["basic", "braces", "design"]
    ignoreFailures = true
    sourceSets = [sourceSets.main]
}
jacocoTestReport {
    reports {
        xml.enabled true
        html.enabled true
        csv.enabled false
    }
    sourceSets sourceSets.main
}
tasks.withType(Compile) {
    options.encoding = "UTF-8"
}
test {
    useTestNG()
    jacoco {
        excludes = ["org.*"]
    }
}
```


## Gradle 常用指令

### 枚列所有可用任务
```bash
❯ gradle tasks

------------------------------------------------------------
All tasks runnable from root project
------------------------------------------------------------

Archive tasks
-------------
zip - Archives sources in a zip file

Build tasks
-----------
assemble - Assembles the outputs of this project.
build - Assembles and tests this project.
clean - Deletes the build directory.
```

### 构建配置属性
```bash
❯ gradle properties
```

### 显示构建详情
在 `build.gradle` 设置
```text
logging.level = LogLevel.DEBUG
```
或者在运行的
```bash
❯ gradle build --stacktrace 
```

## 参考资料 & 推荐文档

- [Gradle入门教程](http://blog.jobbole.com/71999/)
- [Gradle介绍](https://www.jianshu.com/p/00d5469e25e7)
- [官方教材](https://guides.gradle.org/creating-new-gradle-builds/)