---
title: 为什么JetBrains需要Kotlin [译]
date: 2017-01-05 22:28:38
tags:
---

[原文链接](https://blog.jetbrains.com/kotlin/2011/08/why-jetbrains-needs-kotlin/)

当人们需要学习使用一门新的工作语言的时候，往往第一个问题就是为什么我们需要它。
Kotlin的文档中详细的阐述了为何Kotlin会存在。尽管如此，我们仍想清晰的知道JetBrains希望从中得到什么。显然我们在其中投入了长期的经历，花费多年的时间希望达成我的期望。
在此我将解释为何我们乐意投入。


第一并且最重要的原因是，这关乎到我们的生产力。尽管我们已经有了数个基于JVM的编程语言，可至今我们所有基于IntelliJ的IDEs都几乎全部还是使用Java来编写的。
IntelliJ的构建系统基于Groovy 和 Gant，Groovy适用于一些测试代码，有些JRuby代码在RubyMine中，仅仅如此。但是我们想要通过切换一门编程语言从而获得更高的生产力。
于此同时，有两件需求是我们必须要满足的，Java代码的兼容系（这个语言必须是渐进的，能够良好的Java代码兼容）和编译速度的提高（我们现在的代码已经非常的慢，不能给再慢了）。

第二点原因也非常的明确：我们期望Kotlin能够提高IntelliJ IDEA的销量，我们完全在一个新语言上工作，但是我们不准备把整个JVM的生态给替换掉，所以你可以继续使用Spring，Hibernate，或者其他的一切。
如果你的项目使用Kotlin构建，Kotlin的开发工具免费，并且即将开源(译者注：此项目已经开源 [kotlin](https://github.com/JetBrains/kotlin))，剩下的对于企业级开发的部分，仍会
保留在IntelliJ IDEA Ultimate版本中，而社区版本的IDE依然会支持与Kotlin的集成。

最后的原因比重较小，但是也是极为重要的：一个新的语言总是人们所热衷讨论的话题，在我们推出Kotlin的第一天就得到了证明。我们看见人们已经非常熟悉JetBrains并且相信我们可以将此项目做的越来越好，
因此，我们相信这份用户对我们的信任与JetBrains的社区品牌意识，不仅仅可以驱动公司的业务，甚至与吸引更多的人加入我们的开发，让我们都享受开发的乐趣。

最重特别重申，我们对Kotlin的投入并不会影响任何关于我们去其他开发工具的开发，特别是Scala的插件，如果你已经选择Scala而不需要其他语言，我们仍会竭尽全力的为你提供一流的Scala开发工具。


**译者** [yannxia](https://github.com/yannxia)
