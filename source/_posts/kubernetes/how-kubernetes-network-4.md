---
title: Kubernetes-Calico-网络原理(4) - BGP
date: 2019-03-21 17:48:01
tags: ["kubernetes", "network", "calico"]
categories: ["kubernetes", "network", "calico"]
---

我们按照 [kubernetes 网络原理(1) - 环境预备 & 初窥网络](http://blog.yannxia.top/2019/03/17/kubernetes/how-kubernetes-network-1/) 重新部署一个 `BGP` 集群的 `Calico`。

<!-- more -->

![arch](https://i.loli.net/2019/03/21/5c934dd658d89.png)

## 部署 Demo

按照下文的 2 个 yaml 文件部署我们的测试镜像。

{% tabs 部署 Yaml %}

<!-- tab c2c-demo.yaml -->

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
nodeSelector:
  worker: no-1
{% endcodeblock %}

<!-- endtab -->
<!-- tab c2c-demo-w2.yaml -->

{% codeblock lang:yaml %}
apiVersion: v1
kind: Pod
metadata:
name: c2c-network-demo-w2
labels:
app: c2c-network-demo-w2
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
  args: ["-c", "while true; do sleep 10; done;"]
nodeSelector:
  worker: no-2
{% endcodeblock %}

<!-- endtab -->
{% endtabs %}

执行部署

```bash
$ kubectl apply -f c2c-demo.yaml
$ kubectl apply -f c2c-demo-w2.yaml
```

## 状态检测
我们先看看我们的 `Pod` 运行状态:
```bash
$ kubectl get pods -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
c2c-network-demo      2/2     Running   0          12m     192.168.1.2   k8s-worker-1   <none>           <none>
c2c-network-demo-w2   2/2     Running   0          3h57m   192.168.2.3   k8s-worker-2   <none>           <none>
```

我们从 `192.168.1.2` 访问 `192.168.2.3` 访问是通畅的。检查下网卡状态



{% tabs 网卡状态 %}
<!-- tab 容器内 -->
{% codeblock lang:bash %}
root@c2c-network-demo-w2:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether de:bf:dc:6d:a0:ed brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.2.3/32 scope global eth0
       valid_lft forever preferred_lft forever
{% endcodeblock %}
<!-- endtab -->
<!-- tab 宿主机 -->
{% codeblock lang:bash %}
➜  ~ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b4:bf:02 brd ff:ff:ff:ff:ff:ff
    inet 10.12.22.151/16 brd 10.12.255.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feb4:bf02/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:40:ac:69:a4 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: calid9d486e54a8@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}

按照上文中的检查`veth`的方式，我们还是可以发现这里的 `eth0` 和 `calid9d486e54a8` 是一对`veth`，那我们的数据包从此处流出。

## 抓包大作战
尝试在物理网卡上抓取 `10.12.22.152`  的包，并没有获得什么有用的信息。
```bash
tcpdump -i ens192 dst host 10.12.22.152
```

尝试使用 `tcpdump -i ens192 src 192.168.1.2` 抓包，我们却获得真实的数据，这又是为什么？

```bash
➜  ~ tcpdump -i ens192 src 192.168.1.2

21:10:19.551022 IP 192.168.1.2.44728 > 192.168.2.3.http-alt: Flags [S], seq 1948023235, win 28000, options [ms
s 1400,sackOK,TS val 6225662 ecr 0,nop,wscale 7], length 0
21:10:19.551462 IP 192.168.1.2.44728 > 192.168.2.3.http-alt: Flags [.], ack 1874405356, win 219, options [nop,
nop,TS val 6225662 ecr 6117673], length 0
21:10:19.551492 IP 192.168.1.2.44728 > 192.168.2.3.http-alt: Flags [P.], seq 0:80, ack 1, win 219, options [no
p,nop,TS val 6225662 ecr 6117673], length 80: HTTP: GET / HTTP/1.1
21:10:19.552159 IP 192.168.1.2.44728 > 192.168.2.3.http-alt: Flags [.], ack 18, win 219, options [nop,nop,TS v
al 6225663 ecr 6117674], length 0
21:10:19.552328 IP 192.168.1.2.44728 > 192.168.2.3.http-alt: Flags [F.], seq 80, ack 222, win 228, options [no
p,nop,TS val 6225663 ecr 6117674], length 0
```
我们检查下`路由表`，我们发现其中的玄妙：

```bash
➜  ~ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.12.0.1       0.0.0.0         UG    0      0        0 ens192
10.12.0.0       0.0.0.0         255.255.0.0     U     0      0        0 ens192
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.0.0     10.12.22.150    255.255.255.0   UG    0      0        0 ens192
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 *
192.168.1.2     0.0.0.0         255.255.255.255 UH    0      0        0 calid9d486e54a8
192.168.2.0     10.12.22.152    255.255.255.0   UG    0      0        0 ens192
```
我们发现，系统在处理的时候，将数据包报文直接从 `ens192` 的网卡，将数据交付给下一跳的 `10.12.22.152` 机器。因为这两台机器隶属于同一个网络，不需要交换机作为外部桥接，直接采用 `IP转发-直接交付` 的方式，将数据包转发到 `10.12.22.152`

所以我们自然在 `10.12.22.152` 的机器上去监听网卡就获得了如下的数据
```bash
➜  ~ tcpdump -i ens192 src host 192.168.1.2

21:58:11.027773 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [S], seq 957159585, win 28000, options [mss
 1400,sackOK,TS val 6943526 ecr 0,nop,wscale 7], length 0
21:58:11.028019 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [.], ack 702012394, win 219, options [nop,n
op,TS val 6943526 ecr 6835553], length 0
21:58:11.028058 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [P.], seq 0:80, ack 1, win 219, options [no
p,nop,TS val 6943526 ecr 6835553], length 80: HTTP: GET / HTTP/1.1
21:58:11.029092 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [.], ack 18, win 219, options [nop,nop,TS v
al 6943527 ecr 6835553], length 0
21:58:11.029258 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [.], ack 56, win 219, options [nop,nop,TS v
al 6943527 ecr 6835553], length 0
21:58:11.029438 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [.], ack 184, win 228, options [nop,nop,TS 
val 6943527 ecr 6835553], length 0
21:58:11.029538 IP 192.168.1.2.47600 > 192.168.2.3.http-alt: Flags [F.], seq 80, ack 222, win 228, options [no
p,nop,TS val 6943527 ecr 6835553], length 0
```

到现在，整个系统的扭转如下：
![](https://i.loli.net/2019/03/21/5c939cae25908.png)

摆在我们面前，也就是`10.12.22.152`的机器又如何处理这个 `IP` 报文呢？

## 抵达目的地
当我们抵达`10.12.22.152`的机器的时候，我们在此机器上查看下路由表
```bash
➜  ~ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.12.0.1       0.0.0.0         UG    0      0        0 ens192
10.12.0.0       0.0.0.0         255.255.0.0     U     0      0        0 ens192
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.0.0     10.12.22.150    255.255.255.0   UG    0      0        0 ens192
192.168.1.0     10.12.22.151    255.255.255.0   UG    0      0        0 ens192
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 *
192.168.2.3     0.0.0.0         255.255.255.255 UH    0      0        0 calia4d5046802b
```
熟悉的配方，我们又从 192.168.2.3 的路由表将数据转发到真正的容器内。

## 汇总

我们从最近的一系列分析中：

- Pod -> Node-A
  通过 veth 设备将数据从容器的 eth0 发出
- Node-A -> Node-B
  通过路由表将数据发送到 eth0，直接转发IP包
- Node-B -> Pod
  eth0 使用路由表将数据从 veth 输入到 Pod 的 eth0

![](https://i.loli.net/2019/03/22/5c9454b024160.png)

## New Question
我们整个过程，我们有一些疑问，
- 这些路由表的维护是如何维护的？
- iptable 好像一直没有发现用途？
带着这些问题我们进入下一步的。


## 附录

- [What stores Kubernetes in Etcd?](https://jakubbujny.com/2018/09/02/what-stores-kubernetes-in-etcd/)

### ETCD 操作指南
```bash
ETCDCTL_API=3 etcdctl --endpoints=<etcd_ip>:2379 get / --prefix --keys-only
ETCDCTL_API=3 etcdctl --endpoints <etcd_ip>:2379 --cacert <ca_cert_path> --cert <cert_path> --key <cert_key_path> get / --prefix --keys-only
```