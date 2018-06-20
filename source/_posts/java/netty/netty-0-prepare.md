---
title: Netty源码分析-(0)-预备知识
date: 2018-06-12 21:29:48
tags:
toc: true
categories: ["java", "网络编程", 'Netty']
---

## 前言
关于Netty我们都知道（现在还不知道的Netty的怕是凤毛麟角）

>Netty is an asynchronous event-driven network application framework
for rapid development of maintainable high performance protocol servers & clients.

Netty是一个异步事件驱动的应用框架，能够让我们更快的做一些网络开发。

阅读此系列博文，**我对你的要求是** 使用过原生的Java Nio进行过网络编程。
此篇博客算是我阅读Netty的笔记，会有自己探索的过程，望斧正。

<!-- more -->

## 预备知识
- Java NIO
   因为Netty也是建立在Java的Nio基础之上，所以需要有Java NIO的基础，可以参考 [Java Nio 基础](http://blog.yannxia.top/2018/05/07/java/nio/java-nio-1-basic/)

- Reactor线程模式
  推荐2篇文章
  - [Understanding Reactor Pattern: Thread-Based and Event-Driven](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve)
  - [Netty 源码分析之 三 我就是大名鼎鼎的 ](https://segmentfault.com/a/1190000007403873)
  简而言之Netty的线程模式是标准的 `多线程Reactor模型`
  ![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1528824537102&di=b9b5678aee772b881bb9a76fb1d4c8a2&imgtype=jpg&src=http%3A%2F%2Fimg2.imgtn.bdimg.com%2Fit%2Fu%3D3111796757%2C2564753992%26fm%3D214%26gp%3D0.jpg)

## Netty模块
![components](http://netty.io/images/components.png)
正如图上所示，我们核心关注的应该是 core 部分
- Zero Copy Capable Byte Buf: 零拷贝数组
- Exstensible Event Model: 可扩展的事件模型


## 源码解读顺序
1. Netty 服务端启动流程
   - 如何初始化NioServer
   - NioEventLoop模型和Channel


2. Netty 服务端接受请求分析
  - Netty 线程模式
  - 如何调用Pipeline
  - Pipeline工作机制


3. Netty 的ByteBuf
  - Netty的Bytebuf实现
