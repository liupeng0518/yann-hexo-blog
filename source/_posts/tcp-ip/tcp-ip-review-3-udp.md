---
title: TCP/IP-(3)-UDP协议
toc: true
date: 2018-06-05 16:00:07
tags: ["TCP/IP"]
categories: ["网络编程","TCP/IP"]
---

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

UDP是基于`IP 报文`上层的一个协议，在TCP/IP协议簇中也提及到，其实UDP只是比IP多提供了进程到进程之间的通讯，除此之外并没有提供额外什么功能。

## UDP 协议格式
![udp](http://network-insight.net/wp-content/uploads/2014/10/UDP-Header.jpg)

值得注意的点：
- UDP头部多了源端口号和目标端口号
- UDP依然受限于IP的报文大小最大65535，还需要减掉20个UDP的报文头，也就是65507

<!-- more -->

## 参考资料
1. [TCP/IP协议卷一: 详解](https://book.douban.com/subject/1088054/)
2. [TCP/IP协议族](https://book.douban.com/subject/5386194/)
