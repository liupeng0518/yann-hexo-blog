---
title: Kubernetes-Calico-网络原理(2) - Container & Pod
date: 2019-03-17 22:48:01
tags: ["kubernetes", "network", "calico"]
categories: ["kubernetes", "network", "calico"]
---

在 [kubernetes 网络原理(0) - 网络知识预备]() 和 [kubernetes 网络原理(1) - 环境预备 & 初窥网络]() 中我们将基础的知识已经知悉，我们就来看看我们的网络通讯。

我们已知道 `Kubernetes` 的逻辑架构如下：
![arc](https://i.loli.net/2019/03/17/5c8e1371f3af1.png)

<!-- more -->

## Container <-> Container

在`Kubernetes`中我们知道一个 `Pod` 由多个 `Container` 构成，而在 `Pod` 内的`Container`和`Container` 的通讯是通过 `localhost` 这个虚拟独立出来的 `namespace` 。

![](https://i.loli.net/2019/03/18/5c8ef761d3bae.png)

我们来做个实验确认下。
我们使用如下的 `yaml` 部署一个 `Pod`

{% spoiler Pod部署Yaml文件 %}
{% codeblock lang:yaml %}
apiVersion: v1
kind: Pod
metadata:
name: c2c-network-demo
labels:
app: c2c-network-demo
spec:
containers:

- name: hello-world-server
  image: python:2.7
  imagePullPolicy: Always
  command: ["/bin/bash"]
  args: ["-c", "echo \"<p>Hi from $(hostname)</p>\" > index.html; python -m SimpleHTTPServer 8080"]
  ports:
  - name: http
    containerPort: 8080
- name: curl-client
  image: yannxia/ubuntu-with-tcpdump
  imagePullPolicy: Always
  command: ["/bin/sh"]
  args: ["-c", "while true; do echo 'GET / HTTP/1.1\r\n\r\n' | nc localhost 8080; sleep 10; done;"]
  {% endcodeblock %}
  {% endspoiler %}

我们使用如下脚本进入系统内部

```bash
kubectl exec c2c-network-demo -c curl-client -it bash
```

我们先看看我们的网卡信息

```bash
root@c2c-network-demo:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether aa:43:a3:65:96:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.10/32 scope global eth0
       valid_lft forever preferred_lft forever
```

我们发现了我们有 `3` 张网卡，分别是 `lo`,`tunl0`,`eth0`，我们监听下我们的 `lo`

```bash
root@c2c-network-demo:/# tcpdump -i lo
```

{% spoiler TcpDump信息 %}
{% codeblock lang:bash %}
08:08:42.491989 IP localhost.51366 > localhost.8080: Flags [S], seq 350716622, win 43690, options [mss 65495,sackOK,TS val 196496407 ecr 0,nop,wscale 7], length 0
08:08:42.492014 IP localhost.8080 > localhost.51366: Flags [S.], seq 1782151785, ack 350716623, win 43690, options [mss 65495,sackOK,TS val 196496407 ecr 196496407,nop,wscale 7], length 0
08:08:42.492035 IP localhost.51366 > localhost.8080: Flags [.], ack 1, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 0
08:08:42.492084 IP localhost.51366 > localhost.8080: Flags [P.], seq 1:20, ack 1, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 19: HTTP: GET / HTTP/1.1
08:08:42.492886 IP localhost.8080 > localhost.51366: Flags [P.], seq 1:18, ack 20, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 17: HTTP: HTTP/1.0 200 OK
08:08:42.492908 IP localhost.51366 > localhost.8080: Flags [.], ack 18, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 0
08:08:42.492995 IP localhost.8080 > localhost.51366: Flags [P.], seq 18:56, ack 20, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 38: HTTP
08:08:42.493006 IP localhost.51366 > localhost.8080: Flags [.], ack 56, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 0
08:08:42.493768 IP localhost.8080 > localhost.51366: Flags [P.], seq 56:93, ack 20, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 37: HTTP
08:08:42.493786 IP localhost.51366 > localhost.8080: Flags [.], ack 93, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 0
08:08:42.493883 IP localhost.8080 > localhost.51366: Flags [P.], seq 93:118, ack 20, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 25: HTTP
08:08:42.494702 IP localhost.8080 > localhost.51366: Flags [P.], seq 138:184, ack 20, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 46: HTTP
08:08:42.494717 IP localhost.51366 > localhost.8080: Flags [.], ack 184, win 342, options [nop,nop,TS val 196496407 ecr 196496407], length 0
08:08:42.494980 IP localhost.8080 > localhost.51366: Flags [P.], seq 184:186, ack 20, win 342, options [nop,nop,TS val 196496408 ecr 196496407], length 2: HTTP
08:08:42.494999 IP localhost.51366 > localhost.8080: Flags [.], ack 186, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 0
08:08:42.495134 IP localhost.8080 > localhost.51366: Flags [P.], seq 186:218, ack 20, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 32: HTTP
08:08:42.495151 IP localhost.51366 > localhost.8080: Flags [.], ack 218, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 0
08:08:42.495397 IP localhost.8080 > localhost.51366: Flags [F.], seq 218, ack 20, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 0
08:08:42.495464 IP localhost.51366 > localhost.8080: Flags [F.], seq 20, ack 219, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 0
08:08:42.495499 IP localhost.8080 > localhost.51366: Flags [.], ack 21, win 342, options [nop,nop,TS val 196496408 ecr 196496408], length 0
{% endcodeblock %}
{% endspoiler %}

切换监听另外一个 `eth0`

```bash
root@c2c-network-demo:/# tcpdump -i eth0
```

我们发现并没有任何的输出，（虽然如同一个废话）

## Pod <-> Pod

### 嗅探蛛丝马迹

因为在一个 Pod 内的通讯是简单的，我们来看看我们我们下面需要去进阶的一个网络，`Pod` 和 `Pod` 之间的通讯。

![](https://i.loli.net/2019/03/18/5c8fb5f3aa88e.png)

现在我们就想知道 `Pod` 和 `Pod` 之间他们是怎么通讯的。
在容器内进行 `ip a` 的时候，我们发现除了 `lo` 的回环地址之外，就还有一个我们所熟悉的 `eth0` 网卡，而这个网卡的`IP` 也显得颇为特别。`192.168.1.10/32`，我们在定位下这个 POD 所在的机器

```bash
$ kubectl get pods -o wide
NAME               READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
c2c-network-demo   2/2     Running   0          7h34m   192.168.1.10   k8s-worker-1   <none>           <none>
```

还记得 `k8s-worker-1` 这个机器的 `Tun0` 的 `IP` 是 `192.168.1.1`，Wow，我们发现 `Pod`的 IP 和`Tun0` 的 `IP` 在同一个网段中。我们在 `c2c-network-demo` 检查下网络状态。

```bash
root@c2c-network-demo:/# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         169.254.1.1     0.0.0.0         UG    0      0        0 eth0
169.254.1.1     0.0.0.0         255.255.255.255 UH    0      0        0 eth0
```

我们可以发现我们网络包的网关是一个 `169.254.1.1` 的私有地址，实际上应该也不对应任何的设备，从路由表中可以得到的信息是，我们所有的数据包都会从 `eth0` 的网卡发出去。

我们新起一个 `pod` 尝试访问下我们的 `echo-server`

{% spoiler 部署一个Tools-Pod %}
{% codeblock lang:yaml %}
apiVersion: v1
kind: Pod
metadata:
name: test-tools
labels:
app: test-tools
spec:
containers:

- name: curl-client
  image: yannxia/ubuntu-with-tcpdump
  imagePullPolicy: Always
  command: ["/bin/sh"]
  args: ["-c", "while true; do sleep 10; done;"]
  nodeSelector:
  worker: no-1
  {% endcodeblock %}
  {% endspoiler %}

我们在新启动的 `test-tools` 中检查下网络状态

```bash
root@test-tools:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether ee:c9:6c:15:ea:ca brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.12/32 scope global eth0
       valid_lft forever preferred_lft forever
```

不出意外的话，我们可以看的出来 `192.168.1.12/32` 和上述的 `192.168.1.10/32` 看起来就像是在同一个网段内，子网掩码是`32`又在无处的命中我们的内心，他们的确是隔离的网络，我们来看看这**两个网络怎么相通**吧。

- Ping 一下，是成功的网络是相通

  ```bash
  root@test-tools:/# ping 192.168.1.10
  PING 192.168.1.10 (192.168.1.10): 56 data bytes
  64 bytes from 192.168.1.10: icmp_seq=0 ttl=63 time=0.270 ms
  64 bytes from 192.168.1.10: icmp_seq=1 ttl=63 time=0.083 ms
  ```

- 抓个包看看（上文已经知道出口是 eth0）

  ```bash
  # 请求一个HTTP
  root@test-tools:/# curl 192.168.1.10:8080
  <p>Hi from c2c-network-demo</p>

  # 同时抓包网卡，没有获得什么有用的信息，网络是直接相通的
  root@test-tools:/# tcpdump -i eth0
  tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
  listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  03:21:20.695727 IP test-tools.33968 > 192.168.1.10.http-alt: Flags [S], seq 1375634196, win 28000, options [mss 1400,sackOK,TS val 235385958 ecr 0,nop,wscale 7], length 0
  03:21:20.695831 IP 192.168.1.10.http-alt > test-tools.33968: Flags [S.], seq 2004093135, ack 1375634197, win 27760, options [mss 1400,sackOK,TS val 235385958 ecr 235385958,nop,wscale 7], length 0
  03:21:20.695847 IP test-tools.33968 > 192.168.1.10.http-alt: Flags [.], ack 1, win 219, options [nop,nop,TS val 235385958 ecr 235385958], length 0
  03:21:20.695904 IP test-tools.33968 > 192.168.1.10.http-alt: Flags [P.], seq 1:82, ack 1, win 219, options [nop,nop,TS val 235385958 ecr 235385958], length 81: HTTP: GET / HTTP/1.1
  03:21:20.695916 IP 192.168.1.10.http-alt > test-tools.33968: Flags [.], ack 82, win 217, options [nop,nop,TS val 235385958 ecr 235385958], length 0
  03:21:20.696182 IP test-tools.55118 > kube-dns.kube-system.svc.cluster.local.domain: 42450+ PTR? 10.1.168.192.in-addr.arpa. (43)
  03:21:20.696341 IP 192.168.1.10.http-alt > test-tools.33968: Flags [P.], seq 1:18, ack 82, win 217, options [nop,nop,TS val 235385958 ecr 235385958], length 17: HTTP: HTTP/1.0 200 OK
  03:21:20.714519 IP test-tools.35007 > kube-dns.kube-system.svc.cluster.local.domain: 29106+ PTR? 10.0.96.10.in-addr.arpa. (41)
  03:21:20.715149 IP kube-dns.kube-system.svc.cluster.local.domain > test-tools.35007: 29106* 1/0/0 PTR kube-dns.kube-system.svc.cluster.local. (116)
  ```

  既然中间没有什么 TCP 的中继，那显然是通过路由表实现这样的功能，我们再来看看路由是怎么跳的。

没有发现什么内容，显然如果就这么放弃显然也为之过早，我们回到宿主机上检查下。

{% spoiler 网卡信息 %}
{% codeblock lang:bash %}
➜ ~ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
inet 127.0.0.1/8 scope host lo
valid_lft forever preferred_lft forever
inet6 ::1/128 scope host
valid_lft forever preferred_lft forever

2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
link/ether 00:50:56:b4:07:5f brd ff:ff:ff:ff:ff:ff
inet 10.12.22.2/16 brd 10.12.255.255 scope global ens192
valid_lft forever preferred_lft forever
inet6 fe80::250:56ff:feb4:75f/64 scope link
valid_lft forever preferred_lft forever

3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
link/ether 02:42:fb:53:a2:33 brd ff:ff:ff:ff:ff:ff
inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
valid_lft forever preferred_lft forever

4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1
link/ipip 0.0.0.0 brd 0.0.0.0
inet 192.168.1.1/32 brd 192.168.1.1 scope global tunl0
valid_lft forever preferred_lft forever

5: cali0005aea454a@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
inet6 fe80::ecee:eeff:feee:eeee/64 scope link
valid_lft forever preferred_lft forever

13: calid9d486e54a8@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
inet6 fe80::ecee:eeff:feee:eeee/64 scope link
valid_lft forever preferred_lft forever

15: calied9d42cb137@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
inet6 fe80::ecee:eeff:feee:eeee/64 scope link
valid_lft forever preferred_lft forever
{% endcodeblock %}
{% endspoiler %}

再次执行 `ip link type veth`

```bash
5: cali0005aea454a@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
13: calid9d486e54a8@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
15: calied9d42cb137@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
```

我们发现了我们其实多了一些 `veth`，让我们嗅到了一些蛛丝马迹的感觉，我们知道`veth`设备总是成对出现的。那我们`eth0`的关联的又是哪一个虚拟网卡呢？我们借助`ifindex` 这个全局唯一的信息来查看。

在容器内执行如下操作：

```bash
root@test-tools:/# cat /sys/class/net/eth0/iflink
15 #iflink
```

我们再回到宿主机上：

```bash
➜  ~ grep -l 13 /sys/class/net/cali*/ifindex
/sys/class/net/calid9d486e54a8/ifindex
```

我们发现了容器内的 `eth0` 绑定到了 `calid9d486e54a8` 这个虚网卡上了。
![](https://i.loli.net/2019/03/20/5c91f0d27c90e.png)

现在我们知道所有的网卡设备的网络请求都通过 容器的`eth0` <-> 宿主机 `calid9d486e54a8` 上。

### Pod 的通讯

我们现在对 `calid9d486e54a8` 抓个包，可以轻易获得数据流量，也验证了我们上述的证明事情。

{% spoiler 网卡信息 %}
{% codeblock lang:bash %}
➜ ~ tcpdump -i calid9d486e54a8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on calid9d486e54a8, link-type EN10MB (Ethernet), capture size 262144 bytes
15:53:54.987899 IP 192.168.1.12.50360 > 192.168.1.10.http-alt: Flags [S], seq 1365715020, win 28000, options [
mss 1400,sackOK,TS val 239474531 ecr 0,nop,wscale 7], length 0
15:53:54.987928 IP 192.168.1.10.http-alt > 192.168.1.12.50360: Flags [S.], seq 3189753753, ack 1365715021, win
27760, options [mss 1400,sackOK,TS val 239474531 ecr 239474531,nop,wscale 7], length 0
15:53:54.987965 IP 192.168.1.12.50360 > 192.168.1.10.http-alt: Flags [.], ack 1, win 219, options [nop,nop,TS
val 239474531 ecr 239474531], length 0
15:53:54.988232 IP 192.168.1.12.50360 > 192.168.1.10.http-alt: Flags [P.], seq 1:82, ack 1, win 219, options [
nop,nop,TS val 239474531 ecr 239474531], length 81: HTTP: GET / HTTP/1.1
15:53:54.988277 IP 192.168.1.10.http-alt > 192.168.1.12.50360: Flags [.], ack 82, win 217, options [nop,nop,TS
val 239474531 ecr 239474531], length 0
15:53:54.988802 IP 192.168.1.10.http-alt > 192.168.1.12.50360: Flags [P.], seq 1:18, ack 82, win 217, options
[nop,nop,TS val 239474531 ecr 239474531], length 17: HTTP: HTTP/1.0 200 OK
15:53:54.988852 IP 192.168.1.12.50360 > 192.168.1.10.http-alt: Flags [.], ack 18, win 219, options [nop,nop,TS
val 239474531 ecr 239474531], length 0
15:54:00.222860 ARP, Request who-has 192.168.1.10 tell k8s-worker-1, length 28
15:54:00.222974 ARP, Request who-has 169.254.1.1 tell 192.168.1.10, length 28
15:54:00.222981 ARP, Reply 169.254.1.1 is-at ee:ee:ee:ee:ee:ee (oui Unknown), length 28
15:54:00.222994 ARP, Reply 192.168.1.10 is-at aa:43:a3:65:96:87 (oui Unknown), length 28
{% endcodeblock %}
{% endspoiler %}

那这个数据到了 `calid9d486e54a8` 之后又是怎么扭转的呢？既然`IP`没有任何改变，应该是通过路由表实现的。
我们查询下路由表

```bash
➜  ~ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.12.0.1       0.0.0.0         UG    0      0        0 ens192
10.12.0.0       0.0.0.0         255.255.0.0     U     0      0        0 ens192
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.0.0     10.12.22.1      255.255.255.0   UG    0      0        0 tunl0
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 *
192.168.1.2     0.0.0.0         255.255.255.255 UH    0      0        0 cali0005aea454a
192.168.1.10    0.0.0.0         255.255.255.255 UH    0      0        0 calid9d486e54a8
192.168.1.12    0.0.0.0         255.255.255.255 UH    0      0        0 calied9d42cb137
192.168.2.0     10.12.22.3      255.255.255.0   UG    0      0        0 tunl0
```

因为我们的 `dest ip` 是 `192.168.1.12`，显而易见的我们命中了

```bash
192.168.1.12    0.0.0.0         255.255.255.255 UH    0      0        0 calied9d42cb137
```

我们现在的网络显然就是这样的：
![](https://i.loli.net/2019/03/20/5c91f7f31eea6.png)

轻而易举的可以补全这个图：
![](https://i.loli.net/2019/03/20/5c9262addd093.png)

## 附录

- [finding-out-the-veth-interface-of-a-docker-container](https://superuser.com/questions/1183454/finding-out-the-veth-interface-of-a-docker-container)
- [Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/)
