---
title: Kubernetes-网络原理(5) - ClusterIP
date: 2019-03-22 21:48:01
tags: ["kubernetes", "network"]
categories: ["kubernetes", "network"]
---

我们看完的了简单的网络通讯，在`Kubernetes`中还有一个很重要的`ClusterIP`的概念。如下图所示，这个是`K8s`实现服务负载均衡的核心基础。
![](https://i.loli.net/2019/03/22/5c946f9b14785.png)

<!-- more -->

## 部署环境

{% tabs deployment environment %}

<!-- tab hello-world-server-deployment.yaml -->

{% codeblock lang:yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
name: hello-world-server
labels:
app: hello-world-server
spec:
replicas: 3
selector:
matchLabels:
app: hello-world-server
template:
metadata:
labels:
app: hello-world-server
spec:
containers: - name: hello-world-server
image: python:2.7
imagePullPolicy: Always
command: ["/bin/bash"]
args: ["-c", "echo \"<p>Hi from $(hostname)</p>\" > index.html; python -m SimpleHTTPServer 8080"]
ports: - name: http
containerPort: 8080 - name: curl-client
image: yannxia/ubuntu-with-tcpdump
imagePullPolicy: Always
command: ["/bin/sh"]
args: ["-c", "while true; do sleep 10; done;"]
{% endcodeblock %}

<!-- endtab -->
<!-- tab hello-world-server-service.yaml -->

{% codeblock lang:yaml %}
kind: Service
apiVersion: v1
metadata:
name: hello-world-service
spec:
selector:
app: hello-world-server
ports:

- protocol: TCP
  port: 80
  targetPort: 8080
  {% endcodeblock %}
  <!-- endtab -->
  {% endtabs %}

敲个小命令

```bash
$ kubectl apply -f hello-world-server-deployment.yaml
$ kubectl apply -f hello-world-server-service.yaml
```

检查下环境

```bash
$ kubectl get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
c2c-network-demo                      2/2     Running   0          18h   192.168.1.2   k8s-worker-1   <none>           <none>
c2c-network-demo-w2                   2/2     Running   0          22h   192.168.2.3   k8s-worker-2   <none>           <none>
hello-world-server-6469546c67-bbtjp   2/2     Running   0          71s   192.168.2.4   k8s-worker-2   <none>           <none>
hello-world-server-6469546c67-d6j9n   2/2     Running   0          71s   192.168.1.3   k8s-worker-1   <none>           <none>
hello-world-server-6469546c67-fbsvv   2/2     Running   0          72s   192.168.1.4   k8s-worker-1   <none>           <none>

$ kubectl get services -o wide
NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE   SELECTOR
hello-world-service   ClusterIP   10.100.3.59   <none>        80/TCP    16s   app=hello-world-server
kubernetes            ClusterIP   10.96.0.1     <none>        443/TCP   24h   <none>
```

此时我们有了一个 `CLUSTER-IP:10.100.3.59`

![](https://i.loli.net/2019/03/22/5c9494bf2ea2f.png)
<p class="image-footer">cluster ip</p>


## 探索

在我们的服务器上执行 `curl`

```bash
➜  ~ curl 10.100.3.59
<p>Hi from hello-world-server-6469546c67-bbtjp</p>
➜  ~ curl 10.100.3.59
<p>Hi from hello-world-server-6469546c67-d6j9n</p>
➜  ~ curl 10.100.3.59
<p>Hi from hello-world-server-6469546c67-bbtjp</p>
➜  ~ curl 10.100.3.59
<p>Hi from hello-world-server-6469546c67-d6j9n</p>
➜  ~ curl 10.100.3.59
<p>Hi from hello-world-server-6469546c67-fbsvv</p>
```

我们发现我们的请求会均衡的转发到不同的`Pod`上去了，显然也达成了一个`load balance`的功能。
在我们的 `worker-2` 的节点上，我们检查我们的路由表:

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
192.168.1.3     0.0.0.0         255.255.255.255 UH    0      0        0 cali11d6478e38f
192.168.1.4     0.0.0.0         255.255.255.255 UH    0      0        0 calief5eb3245cc
192.168.2.0     10.12.22.152    255.255.255.0   UG    0      0        0 ens192
```

没有任意匹配的路由表（除了 0.0.0.0）。

## ClusterIP 模式

按照官网的说明，`ClusterIP` 有三种工作模式

- `userspace`(since v1.0)
- `iptables` (since v1.1)
- `ipvs` (since v1.9)

```bash
➜  ~ iptables -L | grep 10.100.3.59
```

我们可以从 `kube-proxy` 的启动日志中发现：

```bash
W0321 06:39:55.760705       1 server_others.go:295] Flag proxy-mode="" unknown, assuming iptables proxy
I0321 06:39:55.778302       1 server_others.go:148] Using iptables Proxier.
I0321 06:39:55.778589       1 server_others.go:178] Tearing down inactive rules.
I0321 06:39:56.188482       1 server.go:483] Version: v1.13.4
```

如果没有特意的指定，我们所使用的还是`iptables` 的方式

## Debug Iptable

我们从本机访问出去的连接会经历 `POSTROUTING` 和 `OUTPUT` 这两个`Chain`，我们给`OUTPUT` 增加一个 `DEBUG` 的 `Rule`

```bash
iptables  -A OUTPUT -t raw -p tcp --destination 10.100.3.59 -j TRACE
```

从 `/var/log/kern.log` 中我们看到

{% spoiler kern.log %}
{% codeblock lang:log %}
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158750] TRACE: raw:OUTPUT:policy:3 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158769] TRACE: mangle:OUTPUT:policy:1 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158779] TRACE: nat:OUTPUT:rule:1 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158794] TRACE: nat:cali-OUTPUT:rule:1 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158806] TRACE: nat:cali-fip-dnat:return:1 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158816] TRACE: nat:cali-OUTPUT:return:2 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158823] TRACE: nat:OUTPUT:rule:2 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158832] TRACE: nat:KUBE-SERVICES:rule:3 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158840] TRACE: nat:KUBE-MARK-MASQ:rule:1 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158848] TRACE: nat:KUBE-MARK-MASQ:return:2 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0 MARK=0x4000
Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158857] TRACE: nat:KUBE-SERVICES:rule:4 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0 MARK=0x4000

# ➊

Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158870] TRACE: nat:KUBE-SVC-5MRENC7Q6ZQR6GKR:rule:3 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0 MARK=0x4000

# ➋ 这里进行了 IP 地址的

Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158879] TRACE: nat:KUBE-SEP-M7XDXP4L3HZCSYD6:rule:2 IN= OUT=ens192 SRC=10.12.22.151 DST=10.100.3.59 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=80 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0 MARK=0x4000

# ➌ 这里首次将 ClusterIP 转换为了 容器的 IP

Mar 23 23:51:35 k8s-worker-1 kernel: [207676.158896] TRACE: filter:OUTPUT:rule:1 IN= OUT=ens192 SRC=10.12.22.151 DST=192.168.2.4 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=31766 DF PROTO=TCP SPT=47644 DPT=8080 SEQ=1655794081 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A031715B70000000001030307) UID=0 GID=0 MARK=0x4000

{% endcodeblock %}
{% endspoiler %}

1. 我们再去查询下 `iptables` 查询 ➌ 处的

   ```bash
   ➜  ~ iptables -L KUBE-SEP-M7XDXP4L3HZCSYD6 -t nat --line-number
   Chain KUBE-SEP-M7XDXP4L3HZCSYD6 (1 references)
   num  target     prot opt source               destination
   1    KUBE-MARK-MASQ  all  --  192.168.2.4          anywhere
   2    DNAT       tcp  --  anywhere             anywhere             tcp to:192.168.2.4:8080
   ```

   在这里，我们使用了使用第二条的 `DNAT` 方式将数据包转换了 `IP目标地址`

2. 我们沿着记录往上追溯，查询 ➋ 处
   ```bash
    ➜  ~ iptables -t nat -L KUBE-SVC-5MRENC7Q6ZQR6GKR
   Chain KUBE-SVC-5MRENC7Q6ZQR6GKR (1 references)
   target     prot opt source               destination
   # 192.168.1.3
   KUBE-SEP-PG5HHQCRLP6OHBWJ  all  --  anywhere             anywhere             statistic mode random probability 0.33332999982
   # 192.168.1.4 
   KUBE-SEP-Z7HABNZOJAQBPZC7  all  --  anywhere             anywhere             statistic mode random probability 0.50000000000
   # 192.168.2.4
   KUBE-SEP-M7XDXP4L3HZCSYD6  all  --  anywhere             anywhere
   ```
   我们发现了我们是这里直接将数据进行随机的选择，（为什么不是均等的：黑脸问号），我们现在得到 `Kubernats` 针对 `ClusterIP` 的处理方式如下

![all-in-one](https://i.loli.net/2019/03/24/5c97319c45b83.png)
<p class="image-footer">cluster ip: all in one</p>

## 附录

- [creating a deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)
- [service](https://kubernetes.io/docs/concepts/services-networking/service/)
