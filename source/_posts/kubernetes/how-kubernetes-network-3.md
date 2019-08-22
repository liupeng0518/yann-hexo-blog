---
title: Kubernetes-Calico-网络原理(3) - Pod & Node
date: 2019-03-20 17:48:01
tags: ["kubernetes", "network", "calico"]
categories: ["kubernetes", "network", "calico"]
---

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

在 [Kubernetes 网络原理(2) - Container & Pod](http://blog.yannxia.top/2019/03/17/kubernetes/how-kubernetes-network-2/) 中，我们已经看到了在一个 `Node` 内部的 `Pod` 和 `Container` 的通讯，我们今天来看看跨`Node`的`Pod`通讯。

<!-- more -->

## Pod <-> Node

我们首先先部署一个环境，在 `work-2` 上部署一个新的 `echo-server`

{% spoiler Pod部署Yaml文件 %}
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
  {% endspoiler %}

我们可以查看到在 `worker-2` 上面有了新的服务。

```bash
$ kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
c2c-network-demo      2/2     Running   0          2d2h    192.168.1.10   k8s-worker-1   <none>           <none>
c2c-network-demo-w2   2/2     Running   0          87s     192.168.2.6    k8s-worker-2   <none>           <none>
echo-server           1/1     Running   0          11d     192.168.1.2    k8s-worker-1   <none>           <none>
test-tools            1/1     Running   0          7h45m   192.168.1.12   k8s-worker-1   <none>           <none>
```

我们可以看到 `c2c-network-demo-w2` 分配的 IP 是 `192.168.2.6`，我们在 `k8s-worker-1` 的 `Node` 尝试访问下，毫无疑问的可以访问通。

```bash
root@test-tools:/# ping 192.168.2.6
PING 192.168.2.6 (192.168.2.6): 56 data bytes
64 bytes from 192.168.2.6: icmp_seq=0 ttl=62 time=0.467 ms
```

在`k8s-worker-1`的路由表查一下：

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

我们可以发现命中的是

```bash
192.168.2.0     10.12.22.3      255.255.255.0   UG    0      0        0 tunl0
```

我们的网络包从 `tunl0` 网卡出去，目标是下一条的 `Gateway:10.12.22.3` 但是我们 `tunl0` 的另一头是一个单独的应用，此时我们同时抓包 `ens192` 和 `tunl0` 从黑盒的模式看看，这个`Calico`的应用做了什么事情。

- `tunl0`
  {% spoiler Tunl0 %}
  {% codeblock lang:bash %}
  ➜ ~ tcpdump -i tunl0
  01:24:59.482498 IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [S], seq 2194532999, win 28000, options [m
  ss 1400,sackOK,TS val 248040654 ecr 0,nop,wscale 7], length 0
  01:24:59.482918 IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [S.], seq 2179183161, ack 2194533000, win
  27760, options [mss 1400,sackOK,TS val 247938639 ecr 248040654,nop,wscale 7], length 0
  01:24:59.482982 IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [.], ack 1, win 219, options [nop,nop,TS v
  al 248040655 ecr 247938639], length 0
  01:24:59.483073 IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [P.], seq 1:81, ack 1, win 219, options [n
  op,nop,TS val 248040655 ecr 247938639], length 80: HTTP: GET / HTTP/1.1
  01:24:59.483209 IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [.], ack 81, win 217, options [nop,nop,TS
  val 247938639 ecr 248040655], length 0
  01:24:59.484042 IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [P.], seq 1:18, ack 81, win 217, options [
  nop,nop,TS val 247938639 ecr 248040655], length 17: HTTP: HTTP/1.0 200 OK
  01:24:59.484115 IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [.], ack 18, win 219, options [nop,nop,TS
  val 248040655 ecr 247938639], length 0
  {% endcodeblock %}
  {% endspoiler %}
- `ens192`
  {% spoiler Tunl0 %}
  {% codeblock lang:bash %}
  ➜ ~ tcpdump -i ens192 dst 10.12.22.3 and not port 6443
  01:24:59.482524 IP k8s-worker-1 > k8s-worker-2: IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [S], seq 2
  194532999, win 28000, options [mss 1400,sackOK,TS val 248040654 ecr 0,nop,wscale 7], length 0 (ipip-proto-4)
  01:24:59.482882 IP k8s-worker-2 > k8s-worker-1: IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [S.], seq
  2179183161, ack 2194533000, win 27760, options [mss 1400,sackOK,TS val 247938639 ecr 248040654,nop,wscale 7],
  length 0 (ipip-proto-4)
  01:24:59.483005 IP k8s-worker-1 > k8s-worker-2: IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [.], ack 1
  , win 219, options [nop,nop,TS val 248040655 ecr 247938639], length 0 (ipip-proto-4)
  01:24:59.483083 IP k8s-worker-1 > k8s-worker-2: IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [P.], seq
  1:81, ack 1, win 219, options [nop,nop,TS val 248040655 ecr 247938639], length 80: HTTP: GET / HTTP/1.1 (ipip-
  proto-4)
  01:24:59.483194 IP k8s-worker-2 > k8s-worker-1: IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [.], ack 8
  1, win 217, options [nop,nop,TS val 247938639 ecr 248040655], length 0 (ipip-proto-4)
  01:24:59.483992 IP k8s-worker-2 > k8s-worker-1: IP 192.168.2.6.http-alt > 192.168.1.12.46282: Flags [P.], seq
  1:18, ack 81, win 217, options [nop,nop,TS val 247938639 ecr 248040655], length 17: HTTP: HTTP/1.0 200 OK (ipi
  p-proto-4)
  01:24:59.484128 IP k8s-worker-1 > k8s-worker-2: IP 192.168.1.12.46282 > 192.168.2.6.http-alt: Flags [.], ack 1
  8, win 219, options [nop,nop,TS val 248040655 ecr 247938639], length 0 (ipip-proto-4)
  {% endcodeblock %}
  {% endspoiler %}

从上面的抓包里面可以分辨出，从 `tun0` 出来的数据包，到了 `eth0` 这里被封装成了 `IP in IP` 的数据包，也就是在本来的 `TCP/IP` 的协议包之外又封装了一层物理网络的 `IP`。

![](https://i.loli.net/2019/03/21/5c92e80a5a4da.png)

## 汇总

我们从最近的一系列分析中：

- Pod -> Node-A
  通过 veth 设备将数据从容器的 eth0 发出
- Node-A -> Node-B
  eth 通过路由表将数据发送到 tunl0, 然后 tunl0 将报文分装成 IPinIP
- Node-B -> Pod
  eth 将 IPIP 协议包解包，使用路由表将数据从 veth 输入到 Pod 的 eth0

## New Question

除了 IPIP 的模式以为，Calico 还提供了 BGP 模式，那它的工作模式又是如何呢？切看我们下章分解。
