---
title:  Gateway版本迭代反思
date: 2018-08-15 17:00:48
tags:
toc: true
categories: ["java", "gateway"]
---

笔者在 2017.12 - 至今（2018.8）都沉浸在 `微服务治理平台` 的 `网关` 这个组件的构造之上，恰逢系统更新到 `1.2` 著此作为反思。
> 此处的网关指的是七层的 API 网关

![banner](https://s1.ax1x.com/2018/08/15/PRero4.png)

<!--more-->

## 0.99 时期
2017年底的时候，随着公司的咨询业务开展以及`微服务`的流行起来，我们隐约的摸到了大多数的公司仅仅使用`SpringCloud`的`Zuul`作为唯一的外置的`API网关`是不够的，其中又涉及到大量的相同类似的功能，我们因此萌生了开发一个开箱即用的网关的想法，这个项目在12月的时候正式开始启动。

当时作为流行的是 `Spring Cloud Netflix Zuul`，直至今日都是作为流行的选择之一。`Zuul` 的编程模型也足够简单，我们也在之上为不同的企业做了多套的方案，是成熟可靠的选择。
![Zuul](https://s1.ax1x.com/2018/08/15/PReO6P.png)

不过当时，我们已经成功的使用 `Netty` 给 `东风汽车` 新能源项目做了一套接入的网关，`Netty`的稳定性和高性能非常的吸引我们，当时也有基于`Netty`的高性能的  `Web框架` [`vertx`](https://vertx.io/) ，这里涉及到最为重要的就是 `NIO` 与 `BIO` 之间的比较。如果能够基于 `Netty` 的 `Reactor` 的模型，可以提高大量的吞吐量。

![Netty Reactor](https://s1.ax1x.com/2018/08/15/PRuAdU.png)

所以我们决定不基于 `Zuul`的编程模型，做一套可以同时在 `Zuul` 与 `Vertx` 运行的框架。在开发的过程中，我们借鉴了 `Netty` 的 `Pipeline` 机制，做了一套流水线。
![Pipeline](https://s1.ax1x.com/2018/08/15/PRnWVO.png)

在物理架构上，我们将 `管理端` 与 `运行端` 进行了分离。这样的架构可以保证，`运行端` 的性能足够好，并且不会因为暴露出一些非必要的请求地址给外部用户。
![架构图](http://i1.bvimg.com/596754/6a0e6f920fa9177b.png)

然后将系统划分为了这么几个模块：
- `端点`：提供对外暴露接口的能力
- `路由`：路由转发能力
- `流控`：控制对外接口的转发速度
- `安全`：鉴权相关的能力
- `缓存`：提供系统的缓存服务

### 不足之处
1. 因为为了兼容 `Zuul` 我们整个编程模型，在 `Redis` 读写 和 `Request Body` 读写方都是同步阻塞的。所以整体线程模型会是这样的:
    ![0.99](http://i2.bvimg.com/596754/0a69ee94fd3bc281.png)
    我们不得不在网关处持有大量的线程，这样对于网关的性能来说是一个致命的伤。
2. Redis的读取我们的用了 [Redisson](https://github.com/redisson/redisson/)，当时 Redisson 不支持 Reactive的编程模型，所以的行为是阻塞的，除此之外还有一点很可惜，`Redisson` 存储到Redis的数据往往是非精简的，导致读取数据大小太大这是一个优化点

### 总结
在 0.99 版本完成的时候，因为有两个运行端版本共存，一个是 `Vertx` 和 `Zuul`，当时为了兼容 `Vertx` 做了一系列的  `Facede`项目，后来被证明都是徒劳的，不过当时已经完成了网关的动态化的需求，整体的性能在测试的过程中持平`Zuul`，性能损耗不大。

## 1.0 - 1.1 时期
这个时期，随着 `Spring 5` GA，我们也作为第一批吃螃蟹的，我们将网关的底层基座直接切换到了 `SpringWebFlux`之上，依然可以借力于 `Spring Cloud` 的生态组件，我们将`Vertx`的支持放弃，既然`Spring` 都有了 `Netty` 的运行时，我们整个业务体系都在`Spring`上，就没必要投入更多的人力去维护`Vertx`。
这个阶段，我们尝试使用 `WebFlux`作为网关的基座，这是非常痛苦的阵痛期，有几个问题：
- `Pipeline` 设计不满足 `reactive` 编程的需要
- `webflux` 本身的缺陷

`webflux` 5.0.0 版本，不仅仅在 `ServerExchange` 中缺少 `RemoteIP`，而且无法强制的关闭 `CORS` 的处理，这些细小的问题却要花上不少的时间去做一些`魔改`。

最后在 1.1 完成后，整个系统的架构变成了 `Input Reactive` 和 `Output Reactive`，但是由于pipeline的处理流程还是阻塞的，导致系统依然存在较大的IO瓶颈。

![](http://i2.bvimg.com/596754/df1991fdcf2a17e1.png)


## 1.2 时期
在 `1.2` 的重构目标就是去除复杂的 `Pipeline` 机制，我们可以利用更为完善的 `projectreactor` 进行流程的控制，这个 `Reactive` 的编程库足够好也更为的通用。所以在这个阶段，我们将之前的 `Pipeline` 中的 `Stage` 概念替换为了 `Handler` 在系统中保留了之前经过验证的简单又可靠的概念。
```java
@FunctionalInterface
public interface ReactiveHandler<T> extends Function<T, Mono<T>> {

    @Nonnull
    Mono<T> handle(T httpExchange);

    @Override
    default Mono<T> apply(T t) {
        return handle(t);
    }
}
```
我们使用单一的处理抽象，结合 `Projectreactor` 的 `Mono` 可以达到以前的 `Pipeline` 的效果。

在这个阶段，我们完成了项目的全`响应式`:
![reactive](http://i2.bvimg.com/596754/a69f8b261cd3c627.png)

### 启迪
过度设计是万恶之源，一点都不为过，`Pipeline` 的设计太过于超前也并不实用，导致在 `1.1` 时期增加一个功能非常的困难，`Pipeline` 的编程模型并非是`响应式`的，所以在我们底层的编程模型变化之后，直接就判处了死刑。

### 展望
现在的流控方式还是基于集中化的`Redis`这个在极大规模的请求情况下，依然会存在性能的瓶颈，我们在后续的开发增加独立的高性能的内存模式，将限流器可以进行配置化。