---
title: TCP/IP-(2)-Internet协议
toc: true
date: 2018-05-07 19:30:07
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

## IP 协议格式
![ip](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1525701055612&di=0948d9e53d38bcc604965c72842d3074&imgtype=0&src=http%3A%2F%2F7xntdm.com1.z0.glb.clouddn.com%2FIPformat.png)

IP 报文是 `无连接` 的，IP报文可能出现丢失，重复等等，这个在IP协议中是无任何保证的，IP协议就是能够将数据发送到目标地址即可。
值得注意的点：
- IP的头部中有一个总长度字段，这个是以`字节`为单位的，这个单位只有16位，所以最大为 2^16 = 65535 长度，不过依然会被MTU限制切分
- 标识字段：这个字段为每一个报文生成一个累加的值，对于数据分片的时候，如果标识字段相同即可认为是同一个数据片
- TTL：代表最大的调数，每经过一个Route就减一，等于0的时候就自然被丢弃

<!-- more -->

## IP转发
IP包的转发逻辑是没有强制规定的，不过这个实现的话一般会有4个必要的属性
- 目的地：目标地址
- 掩码：子网掩码
- 下一跳：传递给下一个网络路由的地址
- 接口：也就是网卡

| 目的地 | 掩码 | 下一跳 | 接口 |
| - | :-: | -: | -: |
| 180.70.65.192 | /26 | - | m2 |
| 180.70.65.192 | /25 | - | m0 |

逻辑步骤是这样的
1. 将目标IP和掩码进行运算，得到正确的网段
2. 找到匹配的路由表信息
3. 如果存在下一条地址就将此报文发送给下一条地址 / 如果没有就直接交付给ARP协议通过内网转发


## 相关命令
- `route -n` : 查看所有的路由表

## 参考资料
1. [TCP/IP协议卷一: 详解](https://book.douban.com/subject/1088054/)
2. [TCP/IP协议族](https://book.douban.com/subject/5386194/)
