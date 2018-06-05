---
title: TCP/IP-(4)-TCP协议
toc: true
date: 2018-06-05 18:00:07
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

TCP是基于`IP 报文`上层的一个协议，和上文所言的UDP不同，TCP是为了在恶劣的网络条件下，依然可以提供`可靠`的网络连接所提供的一种协议。

## TCP 协议概述
![TCP](http://img1.51cto.com/attachment/200708/200708061186360118937.jpg)

- TCP提供的是流（可以想象成一个管道）
- TCP拥有发送和接受缓存
- TCP按报文段分发
- TCP是可靠的会进行数据包检查

总而言之和UDP不同，TCP是提供面向连接的协议，比如需要在A-B之间通讯，我们需要`使用TCP在A和B之间建立连接`。

<!-- more -->

## TCP 编号系统
TCP会将每一个字节都编号，比如我们有 6000个字节系统，我们的序号从 1002开始，那最后一个序号就是 7002，而分段机制根据我们的运行的最大长度切分开。因为我们需要检测，所以我们会有一个·`确认号`的机制，如果我们接受的报文是 1054，那接受者的返回应该是 1055（发送号+1）。


## TCP 的生命周期分述
### 1. 连接建立
TCP的建立是三次握手，这块就不展开细聊了。转一个链接 [图解TCP协议中的三次握手和四次挥手](https://www.jianshu.com/p/9968b16b607e)

不过有一个值得注意的地方：在服务端最后一个ACK的时候，下一次客户端就需要发送TCP数据包了，他的seq号是上一次的ack号，而在三次握手中的第二次的SEQ的用途仅仅是是为了第三次的ACK。见图![](https://img-blog.csdn.net/20170405171821911?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcTEwMDc3Mjk5OTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 2. 数据发送过程
这里需要注意的是
- 上面我们知道了TCP是有缓存的，如果我们想要跳过缓存服务，可以设置P的推送字段为1

### 3. 连接中止
很多网上会有中止的四次挥手之说，这里其实无论是三次还是四次都是可以的，关键在于是否存在半关闭的问题。

- case 1: 直接关闭
{% raw %}
<div id="diagram"></div>
<script>
	var data =
	['Title: 三次确认关闭流程',
	 'Client->Server: 1.seq:x  ack:y  AF',
	 'Server->Client: 2.seq:y  ack:x+1  AF',
	 'Client->Server: 3.seq:x  ack:y+1  A',
  ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple'});
</script>

{% endraw %}

- case 2: 半关闭
{% raw %}
<div id="diagram2"></div>
<script>
	var data =
	['Title: 四次确认关闭流程',
	 'Client->Server: 1.seq:x  ack:y  AF',
	 'Server->Client: 2.seq:y  ack:x+1  A',
   'Client-->Server: 3. 多次数据提交',
	 'Server->Client: 3.seq:z  ack:x+1  AF',
   'Client->Server: 1.seq:x  ack:z+1  A',
  ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram2", {theme: 'simple'});
</script>
{% endraw %}

## TCP 的窗口
TCP的流控，差错，拥堵都依赖于窗口机制。待补充

## ACK确认机制
- 累计确认: 无需每个包都确认可以累计一系列连续的包再确认
- 选择确认: 这个是高级功能

## 重传
- RTO超时重传
- 超过3次ACK之后的重传

## 计时器
- 重传计时器：TTL时间估算
- 持续计时器：这个为了防止窗口大小都为0的时候，ack重新激活，但是丢失了，之后会进入一个死锁的状态
- 保活计时器：无响应的持续
- TIME-WAIT：FIN阶段专用的


## 参考资料
1. [TCP/IP协议卷一: 详解](https://book.douban.com/subject/1088054/)
2. [TCP/IP协议族](https://book.douban.com/subject/5386194/)
