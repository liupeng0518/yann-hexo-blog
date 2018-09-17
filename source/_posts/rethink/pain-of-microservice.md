---
title:  微服务之痛
date: 2018-09-01 18:00:48
tags:
toc: true
categories: ["rethink"]
visible: false
---

笔者在最近的2年内，一直从事微服务咨询工作，帮助两位数的企业改造项目至微服务架构，然而这一切并非如想象中的一帆风顺，写下此篇也意在反思微服务的一些伤痛。

<!-- more -->

## 系统结构之痛
微服务并非是`银弹`，在咨询的过程中，很多公司的代码就像是`意大利面`一样的缠绕在一起，而这些代码之罪源于 `过度压缩研发周期` `缺失代码评审` `缺少顶层设计` 而非是一门技术所能够解决之事。

其中最为令人窒息的是 `代码间调用混乱` 与 `数据库使用混乱`，动则就数表联查，哪怕是采用 [`database-per-service`](https://microservices.io/patterns/cn/data/database-per-service.html) 设计模式，在物理上将数据库切分开，也会导致大量的服务间的请求出现，而因为服务库的拆分，更会暴露出系统模块划分的不合理，有点像是打地鼠一样，按住一头又冒出了另外三只。

![](https://s1.ax1x.com/2018/09/01/PxVEVK.jpg)

在原本的单体系统中，不合理的函数点调用最多浪费些调用栈，至多多一些数据库的操作，而将这个问题拓展至 `微服务` 中，缺点暴露的更多，有且不限于 `调用过于频繁` `请求体和响应体过大` `无良好异常设计` 甚至于将一些内部应用的接口暴露给用户。




## 运维成本之痛
一个微服务的系统，少说得有 `业务应用 x3` `服务网关` `配置中心` 构成，应用数量的变多，使得运维变的困难起来，微服务在资源利用上的确有其独到的地方，但带来的问题是在于运维成本急速上升，所以我们看到 `CNCF` 最近又再疯狂的重新造轮子，试图使用自动化运维手段将这部分的成本降下来，但是问题不仅仅在于工具的选择，我们依然不得不去面对日益增长的应用数，不同的应用甚至于对于运维的技能都有着不一样的需求。


## 开发成本之痛
原本专注于业务开发，不得不去关注于 [`服务编排`](https://cloud.tencent.com/developer/article/1080207)，原本的系统开发在 `SOA` 时代我们需要关注的对方的 `IP` 地址，在数十年的业界的大势之下，大部分人都已经掌握了基础的网络知识，而现在开发者不得不去面对 `服务编排` 这个概念，下面有一个例子：

{% spoiler 太长了，不得不隐藏的编排 %}
{% codeblock lang:yaml %}
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
{% endcodeblock %}
{% endspoiler %}

这也就是我非常的痛恨 `kubernetes` 的原因，在现在的 `DevOps` 的飓风之下，似乎开发都得懂点运维的知识，这个不排斥，但是 `kubernetes` 又太过于难搞了一些，完全不如 `docker-compose` 来的简单快乐，当然你引入 `Docker` 的时候，你就不得不去面对那充满新特性的 `Overlay` 网络(:doge 祝你玩的开心)。


除此之外，微服务对系统的健壮性的要求是上了100层楼那个高，传统的 `中间件` 都是相对的稳定，我们在开发的过程中可以省略性的考虑比如 `Mysql` 不稳定这类的问题，而在微服务中，我们的上游或者下游都是独立的应用，而这些应用并非是经过长期的验证的，我不得不针对这些不稳定的应用进行大量的保护性代码，比如服务熔断，服务重试等等，这些成本是远超原本的单体应用的。

笔者在一些公司，一个大业务部，拆分了数十个微服务，而这些的微服务的开发者往往是一个人，这时候，一个开发者需要维护物理上切分开的软件，而在开发的过程中，对每个服务都了如指掌的时候。自然会产生一些排斥，比如为什么这个工具类我用的地方很多，我是不是可以抽到一个公共的Jar中，而抽取之后又发现每个人都需要进行一些自己的变动，最终会变成各个项目都有一些类似又不一样的代码，对于程序员来说可能是一份冗余，这也是微服务所必须付出的成本。


## 不得不去面对的新挑战
除了一些固定的开发成本以外，我们不得不去面对分布式系统中的一些常见又规避不了的问题。

**分布式事务：** 我们不得不去面对以前我们通过单体应用规避的分布式事务的问题，中间件方面我们可以假装完全的信任类如`Tidb` 这样的新时代的分布式数据库，而我们依然需要去面对业务系统的事务问题，基于`强一致性`或者`最终一致性` 的方案，我们选择 `MQ` 或者一些 `3PC` 的框架来解决这些问题，分布式事务框架本身又可能成为系统的一个单点，那`MQ`又会造成改造时候，我们不得不去改造业务的流程，又是一例技术逼迫业务改造的是惨痛案例。

**链路跟踪：** 众多的应用


可喜的是我们有一些探索者提供了一些工具，我们拿来即可满足大部分的需求，可怕的是我们需要投入更多的人力在系统之中，所以在现阶段`微服务`还是一些金主手中的玩物。


## 何时使用微服务

- 系统已经具备很大的规模，哪怕是一个单体系统，企业也为其投入了大量的支撑服务，可以考虑使用微服务改造一些高速变更的业务。
- 企业的组织架构是 `INVEST原则`
- 业务的 `投产时间` 已经跟不上业务的发展，业务驱动之下的微服务不得不用。
- 最后，清晰的明白微服务能够给你带来什么，又会让你付出什么，做好选择，切莫半路而废。
