---
title: Kubernetes-Calico-网络原理(1) - 环境预备 & 初窥网络
date: 2019-03-17 21:48:01
tags: ["kubernetes", "network", "calico"]
categories: ["kubernetes", "network", "calico"]
---

## 安装 kubernetes

- [create cluster kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
- [fastest install kubernetes](http://mediawiki.yannxia.top/index.php/Fastest_Install_Kubernetes)

并且基于 [`Calico`](https://www.projectcalico.org/) 网络构建 `Cluster` 集群

<!-- more -->

## 部署架构

![deplyment-arch](https://ws1.sinaimg.cn/large/eddc95fcgy1g0zwugecopj20m40dkq3b.jpg)
最终我们按照上图配置好物理网络即可。

我们给机器打上不同的 `tag`

```bash
kubectl label nodes k8s-worker-1 worker=no-1
kubectl label nodes k8s-worker-2 worker=no-2
```

## 网络初窥

我们在 `k8s-master` 的机器上执行 `ip a`

```bash
# 回环地址
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
# 物理网卡
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b4:60:31 brd ff:ff:ff:ff:ff:ff
    inet 10.12.22.1/16 brd 10.12.255.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feb4:6031/64 scope link
       valid_lft forever preferred_lft forever
# Docker0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:f3:85:bf:61 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
# Tun0 网卡
4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 192.168.0.1/32 brd 192.168.0.1 scope global tunl0
       valid_lft forever preferred_lft forever
```

我们发现除了，我们经常看到的 `docker0` 网卡，还多了一个 `tunl0` 网卡，我们知道 `tunl0` 一头连接着一个`Aplication`，那我们大致上可以猜测出来，`tunl0`的出口就是`Calico`的网络处理应用。

我们先汇总下信息

| 机器     | 物理 IP    | TUN0-IP     |
| -------- | ---------- | ----------- |
| Master   | 10.12.22.1 | 192.168.0.1 |
| Worker-1 | 10.12.22.2 | 192.168.1.1 |
| Worker-2 | 10.12.22.3 | 192.168.2.1 |

## 基本原理

```bash
+--------------------+              +--------------------+
|   +------------+   |              |   +------------+   |
|   |            |   |              |   |            |   |
|   |    ConA    |   |              |   |    ConB    |   |
|   |            |   |              |   |            |   |
|   +-----+------+   |              |   +-----+------+   |
|         |veth      |              |         |veth      |
|       wl-A         |              |       wl-B         |
|         |          |              |         |          |
+-------node-A-------+              +-------node-B-------+
        |    |                               |    |
        |    | type1.  in the same lan       |    |
        |    +-------------------------------+    |
        |                                         |
        |      type2. in different network        |
        |             +-------------+             |
        |             |             |             |
        +-------------+   Routers   |-------------+
                      |             |
                      +-------------+
```

图片与文字来自<sup>[附录 2](#Calico网络的原理-组网方式与使用)</sup>

> 核心问题是，nodeA 怎样得知下一跳的地址？答案是 node 之间通过 BGP 协议交换路由信息。
> 每个 node 上运行一个软路由软件 bird，并且被设置成 BGP Speaker，与其它 node 通过 BGP 协议交换路由信息。
> 可以简单理解为，每一个 node 都会向其它 node 通知这样的信息:
> 我是 X.X.X.X，某个 IP 或者网段在我这里，它们的下一跳地址是我。
> 通过这种方式每个 node 知晓了每个 workload-endpoint 的下一跳地址。

## 参考

- [introducing-linux-network-namespaces](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
- [Calico 网络的原理-组网方式与使用](https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html#as-per-rack)
