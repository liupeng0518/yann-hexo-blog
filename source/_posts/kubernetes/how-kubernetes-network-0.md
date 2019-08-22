---
title: Kubernetes-网络原理(0) - 网络知识预备
date: 2019-03-07 21:48:01
tags: ["kubernetes", "network"]
categories: ["kubernetes", "network"]
---

Docker 是“新瓶装旧酒”的产物，依赖于 Linux 内核技术 chroot 、namespace 和 cgroup。本篇先来看 namespace 技术。

Docker 和虚拟机技术一样，从操作系统级上实现了资源的隔离，它本质上是宿主机上的进程（容器进程），所以资源隔离主要就是指进程资源的隔离。实现资源隔离的核心技术就是 Linux namespace。这技术和很多语言的命名空间的设计思想是一致的（如 C++ 的 namespace）。

第零篇的话，我们就来看看最基础的 `namespace`，和我们后续需要使用的 `Tcpdump` 这个抓包工具。

<!-- more -->

## NAMESPACE

我们都知道 `Docker` 的基础是 `NAMESPACE`，通过`NAMESPACE`我们做到不同的隔离，因为我们这次只聊网络，那我们来看看`NAMESPACE`的网络隔离。

1. 创建一个 `Network Namespace`
   ```bash
   ip netns add <new namespace name>
   ```
   假如我们创建一个 `test` 的 `namespace`
   ```bash
   ip netns add test
   ```
2. `namespace` 操作

   ```bash
   ip netns exec <you namespace name> <operator>
   ```

   我们打开一个 bash 在 `test` 的 `namespace` 中

   ```bash
   ip netns exec test bash
   # 查询下网络信息
   ip a
   ```

   我们获得一个有趣的结果

   ```bash
   root@debian:~# ip a
   1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   ```

   我们默认获得一个回环地址的网卡，并没有启动，那我们让其启动

   ```bash
   ip netns exec test ip link set dev lo up
   # 再次查看IP
   ip netns exec test ip a

   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
   link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   inet 127.0.0.1/8 scope host lo
      valid_lft forever preferred_lft forever
   inet6 ::1/128 scope host
      valid_lft forever preferred_lft forever
   ```

   我们就获得一个 IP 地址，这样的话，我们系统内部通讯就有了一个 `IP` 地址。

3. 在 `Network Namespace` 相互通讯
   我们首先在 `Namespace` 打开一个 `Shell`
   ```bash
   ip netns exec test zsh
   ```
   然后在里面启动一个`HTTP`服务
   ```bash
   echo "Hello world" > index.html ; python -m SimpleHTTPServer 8080
   ```
   我们再开启一个新的的 `Shell`
   ```bash
   ip netns exec test zsh
   # 尝试访问 8080，得到 hello word 的结果
   ~ curl localhost:8080
   Hello world
   ```

## 网络抓包

我们已经创建了一个虚拟的网络，内部的通讯就是通过这个 `localhost` 进行通讯。
我们使用 `tcpdump` 进行网络抓包

1. 监听 lo 的回环网卡
   `tcpdump -i lo`

2. 分析数据包

   ```bash
   listening on lo, link-type EN10MB (Ethernet)
   ## TCP 链接建立
   # TCP 第一次握手 Client -> Server，Seq = N
   15:03:42.394751 IP localhost.42514 > localhost.http-alt: Flags [S], seq 435521204, win 43690, options [mss 65495,sackOK,TS val 3638139 ecr 0,nop,wscale 7], length 0
   # TCP 第二次握手 Server -> Client，ACK = N+1， SEQ = J
   15:03:42.394774 IP localhost.http-alt > localhost.42514: Flags [S.], seq 1254554874, ack 435521205, win 43690, options [mss 65495,sackOK,TS val 3638139 ecr 3638139,nop,wscale 7], length 0
   # TCP 第三次握手 Client -> Server，Seq = J + 1，这里的ACK = 1 是相对报文的开始
   15:03:42.394792 IP localhost.42514 > localhost.http-alt: Flags [.], ack 1, win 342, options [nop,nop,TS val 3638139 ecr 3638139], length 0
   ## TCP 数据发送
   # 发送HTTP报文 1 - 79 部分
   15:03:42.396258 IP localhost.42514 > localhost.http-alt: Flags [P.], seq 1:79, ack 1, win 342, options [nop,nop,TS val 3638139 ecr 3638139], length 78: HTTP: GET / HTTP/1.1
   # 服务端应答收到
   15:03:42.396372 IP localhost.http-alt > localhost.42514: Flags [.], ack 79, win 342, options [nop,nop,TS val 3638139 ecr 3638139], length 0
   # 客户端发送额外的 18 部分
   15:03:42.396999 IP localhost.http-alt > localhost.42514: Flags [P.], seq 1:18, ack 79, win 342, options [nop,nop,TS val 3638139 ecr 3638139], length 17: HTTP: HTTP/1.0 200 OK
   # 服务端应答收到
   15:03:42.397002 IP localhost.42514 > localhost.http-alt: Flags [.], ack 18, win 342, options [nop,nop,TS val 3638139 ecr 3638139], length 0
   # 客户端发送 Fin + Ack
   15:03:42.398629 IP localhost.42514 > localhost.http-alt: Flags [F.], seq 79, ack 199, win 342, options [nop,nop,TS val 3638140 ecr 3638139], length 0
   # 服务端响应结束
   15:03:42.398639 IP localhost.http-alt > localhost.42514: Flags [.], ack 80, win 342, options [nop,nop,TS val 3638140 ecr 3638140], length 0
   ```

## 镜像准备

因为为了方便我们在容器内调试网络，我使用`ubuntu`构建一个工具箱的镜像

```bash
docker pull yannxia/ubuntu-with-tcpdump
```

里面包含了本次实验所需要的所有的网络工具。

## 参考

- [introducing-linux-network-namespaces](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
- [Tcpdump Commands](https://www.tecmint.com/12-tcpdump-commands-a-network-sniffer-tool/)
- [yannxia/ubuntu-with-tcpdump](https://cloud.docker.com/u/yannxia/repository/docker/yannxia/ubuntu-with-tcpdump)

## TCP 标志位

| 序号 | TCP 标志位 | 解释     |
| ---- | ---------- | -------- |
| 1    | SYN        | 建立连接 |
| 2    | FIN        | 关闭连接 |
| 3    | ACK        | 响应     |
| 4    | PSH        | 数据传输 |
| 5    | RST        | 链接重置 |
