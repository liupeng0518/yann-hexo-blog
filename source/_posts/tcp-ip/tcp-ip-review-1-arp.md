---
title: TCP/IP-(1)-ARP
toc: true
date: 2018-05-06 22:30:07
tags: ["TCP/IP"]
categories: ["网络编程","TCP/IP"]
---

{% raw %}

<script src="https://cdn.bootcss.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://cdn.bootcss.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://cdn.bootcss.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://cdn.bootcss.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://cdn.bootcss.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

## 预备知识
![OSI](http://electricala2z.com/wp-content/uploads/2017/10/osi-model.gif)
网络是**分层**的，TCP/IP 关注的是 第三层采用 IP 协议 和 第四层 采用的是 TCP 和 UDP 协议。
对于第一层和第二层，我们需要在意的是 **MTU** （最大传输单元）,
这个值有一个**最小值**,因为载波监听的需求，并没强制性的最大值，Lan默认的是1500。(如何优化MTU可以见 **参考2**) 如何查询MTU可以通过 `ifconfig` 命令查询。


<!-- more -->

## ARP
### ARP 协议
ARP 提供的是IP转化为MAC地址的功能，提供的是在局域网内传输的能力，因为在局域网内传输数据，需要使用MAC地址（因为在数据链路层的协议需要）。

- 静态映射：直接写在路由表内
- 动态映射：采用广播的方式查询（ARP协议）

![ARP](https://networkingfolks.files.wordpress.com/2012/03/arp13.jpg)

### ARP 流程
**ARP**的核心是有一个源目标IP地址和目标IP地址，目标的物理地址是**填充0**，以广播的方式发送出去，识别出是自己IP的机器以单播的方式返回给发送者。

{% raw %}

<div id="diagram"></div>
<script>
	var data =
	['Title: ARP流程',
	 'Sender->Receiver: 1.准备广播报文',
	 'Note right of Sender: 目标地址IP，目标MAC地址置为0',
	 'Receiver->Sender: 2.单播响应',
	 'Note left of Receiver: 采用ARP报文将自己的MAC响应'].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple'});
</script>

{% endraw %}

### ARP 相关命令
- `arp -a` : 查询所有arp相关
- `tcpdump -n arp` : 监听arp请求

### ARP 相关攻击
arp本身没有验证性，可以强制回答自己的mac地址。如果有黑客接入网络会导致大批量的arp地址错误。
可以参考本文的 `参考资料3`。

## 参考资料
1. [TCP/IP协议卷一: 详解](https://book.douban.com/subject/1088054/)
2. [Ping Test to determine Optimal MTU Size on Router](https://kb.netgear.com/19863/Ping-Test-to-determine-Optimal-MTU-Size-on-Router)
3. [ARP攻击防范方法总结](http://www.freebuf.com/articles/system/10517.html)
