---
title: Zuul2源码分析
date: 2018-06-22 10:49:48
tags:
toc: true
categories: ["java", "微服务", '网关']
---

# Zuul2 
Zuul2的难产，终于在 2018.4.13 上架了中心仓库，也代表着Zuul正式加入Netty全家桶的怀抱，关于 Zuul2 有一篇宏观性的博文有兴趣的可以阅读 [Zuul 2 : The Netflix Journey to Asynchronous, Non-Blocking Systems](https://medium.com/netflix-techblog/zuul-2-the-netflix-journey-to-asynchronous-non-blocking-systems-45947377fb5c) 此篇博客。

<!-- more -->

简而言之，Zuul2也就是从传统的 BIO 切换到了 NIO 模式。
![bio-thread](https://s1.ax1x.com/2018/06/22/PSzIQs.png)
传统的BIO模型基于Thread的方式

![nio-thread](https://s1.ax1x.com/2018/06/22/PSzHe0.png)
NIO模型基于Reactor模型


## Zuul2 架构
从上帝视角来开，Zuul2是一个在 `Netty` 上运行一系列Filter的服务，执行完成PreFilter (inbound filters)之后将请求通过 `Netty Client` 转发出去，然后将请求的结果通过一系列PostFilter (outbound filters) 返回，如下图所示。
![Architectural-over](https://s1.ax1x.com/2018/06/22/PpSZSH.png)

正如之前的 `ZuulFilter` 分为了 `Pre`,`Post`,`Route`,`Error`，Zuul2的Filter分为三种类型
- Inbound Filters: 在路由之前执行
- Endpoint Filters: 路由操作
- Outbound Filters: 得到相应数据之后执行

我们用官方的Demo进行分析 [zuul-sample](https://github.com/Netflix/zuul/tree/2.1/zuul-sample)，诸位看官自行下载导入。

## ServerStartup
在Demo的启动中，我们发现启动的入口
```java
{
    ConfigurationManager.loadCascadedPropertiesFromResources("application"); ➊
    Injector injector = InjectorBuilder.fromModule(new ZuulSampleModule()).createInjector(); ➋
    BaseServerStartup serverStartup = injector.getInstance(BaseServerStartup.class);
    server = serverStartup.server(); 

    long startupDuration = System.currentTimeMillis() - startTime;
    System.out.println("Zuul Sample: finished startup. Duration = " + startupDuration + " ms");

    server.start(true); ➌
}
```
➊ 获得系统的一些配置参数
➋ Demo中使用的是Google的Guice进行依赖注入的，这个就展开了有兴趣的可以自行去搜索
➌ 启动一个Zuul2服务

我们从这个 `start()` 作为我们的突破口，

```java
public void start(boolean sync){
    serverGroup = new ServerGroup("Salamander", eventLoopConfig.acceptorCount(),   eventLoopConfig.eventLoopCount(), eventLoopGroupMetrics); ➊
    serverGroup.initializeTransport();
    try {
        List<ChannelFuture> allBindFutures = new ArrayList<>();
        // Setup each of the channel initializers on requested ports.
        for (Map.Entry<Integer, ChannelInitializer> entry : portsToChannelInitializers.entrySet())
        {
            allBindFutures.add(setupServerBootstrap(entry.getKey(), entry.getValue())); ➋
        }

        // Once all server bootstraps are successfully initialized, then bind to each port.
    }
    catch (InterruptedException e) {
    }
}
```
➊ 构建一个新的 `ServerGroup`
➋ `setupServerBootstrap()` 初始化一个 `ServerBootstrap`，根据之前Netty的分析，我们知道Netty需要使用 `ServerBootstrap` 进行 端口绑定，那这里是不是就是那个东西。
我们继续深入

```java
private ChannelFuture setupServerBootstrap(int port, ChannelInitializer channelInitializer)
            throws InterruptedException{
    ServerBootstrap serverBootstrap = new ServerBootstrap().group(
            serverGroup.clientToProxyBossPool,
            serverGroup.clientToProxyWorkerPool); ➊

    serverBootstrap.childHandler(channelInitializer); ➋
    serverBootstrap.validate();

    LOG.info("Binding to port: " + port);

    // Flag status as UP just before binding to the port.
    serverStatusManager.localStatus(InstanceInfo.InstanceStatus.UP); ➌

    // Bind and start to accept incoming connections.
    return serverBootstrap.bind(port).sync();  ➍
}
```
在 ➊ 构建了一个 `ServerBootstrap` 这个正是Netty的启动类
➋ 正如Netty的启动中的处理数据的 `Handler` 那这里应该也就是Zuul处理的核心所在
➌ 一个我们的老朋友，和Eureka集成时改变服务器状态
➍ 绑定 `ServerNioSockertChannel` 不再多做分析

## 1/3 休息
我们已经发现了Zuul2是如何启动一个Netty服务的，我们解决了图中红框部分的原理，那我们接下来去了解最为重要的这些Filter是如何工作的，我们上启动中已经发现一个很重要的对象 `channelInitializer` 我们知道在Netty中，是将一系列的 `Handler` 聚合在一起并使用 `Pipeline` 执行（参考[Netty源码分析-(3)-ChannelPipeline](http://blog.yannxia.top/2018/06/20/java/netty/netty-3-pipeline/)）我们可以猜测 zuul 的做法是类型，我们从这个 `channelInitializer` 入手去研究。

## ZuulServerChannelInitializer
我们轻而易举的可以发现 `ChannelInitializer` 其实是  `ZuulServerChannelInitializer` 对象。在initChannel中我们发现了
```java
@Override
protected void initChannel(Channel ch) throws Exception{
    ChannelPipeline pipeline = ch.pipeline();
    storeChannel(ch);
    addTimeoutHandlers(pipeline);
    addPassportHandler(pipeline);
    addTcpRelatedHandlers(pipeline);
    addHttp1Handlers(pipeline);
    addHttpRelatedHandlers(pipeline);
    addZuulHandlers(pipeline); ➊
}
```

在➊前面的都比较简单都是一些标准的 `Handler` 大家可以自己阅读，最为重要是 `addZuulHandlers(pipeline);` 这个函数，我们继续深入。

```java
protected void addZuulHandlers(final ChannelPipeline pipeline){
    pipeline.addLast("logger", nettyLogger);
    pipeline.addLast(new ClientRequestReceiver(sessionContextDecorator));
    pipeline.addLast(passportLoggingHandler);
    addZuulFilterChainHandler(pipeline); ➋
    pipeline.addLast(new ClientResponseWriter(requestCompleteHandler, registry));
}
```
上面的都很容易看出来，是日志，Session之类的Handler，最为重要的是 ➋ 处增加 ZuulFilter。

```java
protected void addZuulFilterChainHandler(final ChannelPipeline pipeline) {
    final ZuulFilter<HttpResponseMessage, HttpResponseMessage>[] responseFilters = getFilters(
            new OutboundPassportStampingFilter(FILTERS_OUTBOUND_START),
            new OutboundPassportStampingFilter(FILTERS_OUTBOUND_END)); ➊

    // response filter chain
    final ZuulFilterChainRunner<HttpResponseMessage> responseFilterChain = getFilterChainRunner(responseFilters,
            filterUsageNotifier); ➋

    // endpoint | response filter chain
    final FilterRunner<HttpRequestMessage, HttpResponseMessage> endPoint = getEndpointRunner(responseFilterChain,
            filterUsageNotifier, filterLoader); ➌

    final ZuulFilter<HttpRequestMessage, HttpRequestMessage>[] requestFilters = getFilters(
            new InboundPassportStampingFilter(FILTERS_INBOUND_START),
            new InboundPassportStampingFilter(FILTERS_INBOUND_END));

    // request filter chain | end point | response filter chain
    final ZuulFilterChainRunner<HttpRequestMessage> requestFilterChain = getFilterChainRunner(requestFilters,
            filterUsageNotifier, endPoint);

    pipeline.addLast(new ZuulFilterChainHandler(requestFilterChain, responseFilterChain));
}
```
从 ➊ 深入可以看到 

```java
public <T extends ZuulMessage> ZuulFilter<T, T> [] getFilters(final ZuulFilter start, final ZuulFilter stop) {
    final List<ZuulFilter> zuulFilters = filterLoader.getFiltersByType(start.filterType());
    final ZuulFilter[] filters = new ZuulFilter[zuulFilters.size() + 2];
    filters[0] = start;
    for (int i=1, j=0; i < filters.length && j < zuulFilters.size(); i++,j++) {
        filters[i] = zuulFilters.get(j);
    }
    filters[filters.length -1] = stop;
    return filters;
}
```
这里返回了一个 `ZuulFilter` 的数组，开始分别是 `start` 和 `stop` 对应的刚好是 `OutboundPassportStampingFilter`。

然我们继续回到 `addZuulFilterChainHandler()` 函数上来，我们发现有三段相似的代码正好对应着获得了 `InBound`
 `OutBond` `EndPoint` 这三种Filter，在代码我们可以看出顺序是

 1. `requestFilters` 和 `endPointFilters` 合并成 `requestFilterChain`
 2. `responseFilters` 构建成 `responseFilterChain`
 3. `requestFilterChain` 和 `responseFilterChain` 组合成 `ZuulFilterChainHandler`
 4. 将 `ZuulFilterChainHandler` 添加至 `pipeline` 中

那这里我们还有一个疑问，这些Filter是从何而来的？这个答案隐藏在 
`com.netflix.zuul.FilterLoader#getFiltersByType` 中，通过简单的跟踪我们可以得到
```java
public Collection<ZuulFilter> getAllFilters() {
    return this.filters.values(); ➊
}
```
在这里 ➊ 获得所有的Fiter，而这里的Filter看起来是通过 `Put`进来的，通过一个简单的断点，我们就可以发现
```java
//com.netflix.zuul.FilterFileManager#init
@PostConstruct
public void init() throws Exception{    
    filterLoader.putFiltersForClasses(config.getClassNames()); ➊
    manageFiles();
    startPoller();
}
```
➊ 我们通过类的全称限定类名获得的这个Fitler，这个配置是在我们的配置文件中配置的。

## 2/3 休息
文至中场，我们已经明白了Zuul2如何将自己的 `ZuulFilter` 变换成 `Netty Handler` 并添加到 `Netty Pipeline` 之中的，那我们还剩下一个问题，这个 `ZuulFilter` 是如何运作的。但是我们在上段中，我们已经发现了最后是一个 `ZuulFilterChainHandler` 通过名称我们可以推测出，这是一个 `Chain 链`，我们继续往下探索吧。

## ZuulFilterChainHandler
我们知道，最终注册到 `Netty Pipeline` 上的最终肯定是 `Handler`, 我们只需要从 Netty 的 `channelRead()` 函数作为突破口去阅读。

```java
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof HttpRequestMessage) { ➊
        zuulRequest = (HttpRequestMessage)msg;

        //Replace NETTY_SERVER_CHANNEL_HANDLER_CONTEXT in SessionContext
        final SessionContext zuulCtx = zuulRequest.getContext();
        zuulCtx.put(NETTY_SERVER_CHANNEL_HANDLER_CONTEXT, ctx);
        zuulCtx.put(ZUUL_FILTER_CHAIN, requestFilterChain);

        requestFilterChain.filter(zuulRequest); ➋
    }
    else if ((msg instanceof HttpContent)&&(zuulRequest != null)) {  ➌
        requestFilterChain.filter(zuulRequest, (HttpContent) msg);
    }
    else {
        LOG.debug("Received unrecognized message type. " + msg.getClass().getName());
        ReferenceCountUtil.release(msg); ➍
    }
}
```
➊ 这段逻辑处理 已经被转化为 `HttpRequestMessage` 类型的消息
➋ 实际上的 filter 处理逻辑
➌ 处理还没被转化为 `HttpRequestMessage` 类型的消息
➍ 无法处理抛出异常，释放MSG

而这里的 `requestFilterChain` 就是之前我们传入进去的 `ZuulFilterChainRunner` 我们看看这 `filter()` 函数做了什么？

```java
// com.netflix.zuul.netty.filter.ZuulFilterChainRunner#runFilters 
private final void runFilters(final T mesg, final AtomicInteger runningFilterIdx) {
    T inMesg = mesg;
    String filterName = "-";
    try {
        Preconditions.checkNotNull(mesg, "Input message");
        int i = runningFilterIdx.get(); ➊

        while (i < filters.length) {
            final ZuulFilter<T, T> filter = filters[i]; ➋
            filterName = filter.filterName();
            final T outMesg = filter(filter, inMesg); ➌
            if (outMesg == null) {
                return; //either async filter or waiting for the message body to be buffered
            }
            inMesg = outMesg;
            i = runningFilterIdx.incrementAndGet(); ➍
        }
        invokeNextStage(inMesg); ➎
    }
    catch (Exception ex) {
    }
}
```
➊ 获得当前运行的Filter的下标值
➋ 获得对应的 `ZuulFilter`
➌ 调用 `ZuulFilter` 进行处理
➍ 将下标志值 +1，继续循环体
➎ 执行下个阶段，这里对应着我们自己再构建 `new InboundPassportStampingFilter(FILTERS_INBOUND_END)`

通过这段代码，我们知道了 `Zuul2` 的Chain是由 `ChainRunner`运行，和Netty的Head tail的链方式大相径庭。
在 `BaseZuulFilterRunner` 中的 `filter()` 函数也相当有趣。
```java
//com.netflix.zuul.netty.filter.BaseZuulFilterRunner#filter 
protected final O filter(final ZuulFilter<I, O> filter, final I inMesg) {
  filter.incrementConcurrency();
    resumer = new FilterChainResumer(inMesg, filter, snapshot, startTime); ➊
    filter.applyAsync(inMesg) ➋
        .observeOn(Schedulers.from(getChannelHandlerContext(inMesg).executor()))  ➌
        .doOnUnsubscribe(resumer::decrementConcurrency)
        .subscribe(resumer);  ➍

    return null;  //wait for the async filter to finish
}
```

➊ 临时存储起来这个Filter待异步完成回调
➋ 具体的执行处
➌ 获取当前的所在的 `EventExecutor` 并在这个线程上观察
➍ 将数据在 resumer 中消费

```java
@Override
public void onNext(O outMesg) {  ➊
    try {
        recordFilterCompletion(SUCCESS, filter, startTime, inMesg, snapshot);
        if (outMesg == null) {
            outMesg = filter.getDefaultOutput(inMesg);
        }
        resumeInBindingContext(outMesg, filter.filterName());
    }
}
```
➊ 处获得我们的结果并将其恢复到我们保管的 `filter`。

剩余关于 `ReponseFilter` 的运行机制是类似的，诸位看官自行分析下。

## 终场休息
通过上面的一系列分析，我们已经知道的，Zuul的 `调用链模型`，`PreFilters`的运行机制，整个Zuul2的运行机制在我们的面前一览无遗。整体的Zuul代码是相当的明了的，代码的分层也很好，但是还有一朵乌云在我们的头顶之上，那就是在那张途中的 `Netty Client` 我们并没有发现其踪迹，但是我们知道只有在 `End` 阶段，我们才会对外进行访问，在官网的Wiki中，我们也可以获得

> Zuul does not use Ribbon for making outgoing connections and instead uses its own connection pool, using a Netty client. Zuul creates a connection pool per host, per event loop. It does this in order to reduce context switching between threads and to ensure sanity for both the inbound event loops and outbound event loops. The result is that the entire request is run on the same thread, regardless of which event loop is running it.

我们从 `Wiki` 中可以得知，Netty不再默认使用 Ribbon 而是默认使用 `Netty` 作为一个 `Client`.

## ProxyEndpoint
这个类的对象及其的复杂，我们从filter的核心逻辑 `apply`看起来。
```java
@Override
public HttpResponseMessage apply(final HttpRequestMessage input) {
    try {
        origin.getProxyTiming(zuulRequest).start();
        origin.onRequestExecutionStart(zuulRequest, 1);
        proxyRequestToOrigin(); ➊
        return null;
    } catch (Exception ex) {
    }
}
```
➊ 将请求转发至远端
```java
private void proxyRequestToOrigin() {
    Promise<PooledConnection> promise = null;
    try {
        promise = origin.connectToOrigin(zuulRequest, channelCtx.channel().eventLoop(), attemptNum, passport, chosenServer); ➊

        currentRequestAttempt = origin.newRequestAttempt(chosenServer.get(), context, attemptNum); 
        if (promise.isDone()) {
            operationComplete(promise); ➋
        } else {
            promise.addListener(this); 
        }
    }
    catch (Exception ex) {
    }
}
```
➊ 处将请求包装，连接到远端地址，获得 Promise
➋ 结束的 Promise 处理，在 `operationComplete()` 中包含了成功的执行代码，至于 `connectToOrigin` Zuul 包装了 Netty的Client。

```java
private void writeClientRequestToOrigin(final PooledConnection conn) {
    final Channel ch = conn.getChannel(); ➊

    ch.write(zuulRequest);   ➋
    writeBufferedBodyContent(zuulRequest, ch);
    ch.flush(); ➌
    //Get ready to read origin's response
    ch.read(); ➍

    originConn = conn;
    channelCtx.read(); ➎
}
```
➊ 获得建立的连接
➋ 写入Zuul的请求，也就是用户的请求
➌ 将消息Flush出去
➍ 在这里读取响应的数据，也就是触发 `OutBoundHandler` 的处理时间

## 总结
Zuul整体逻辑，我们通过博文可以分析而出。
1. ZuulFilter 分为 `Inbound`, `Outbound`, `EndPoint`
2. `Inbound`, `Outbound`, `EndPoint` 包裹成 `ChainRunner`
3. `ChainRunner` 组合成一个 `ZuulFilterChainHandler`，而 `ZuulFilterChainHandler` 是Netty的 一个Handler
4. `ZuulFilterChainHandler` 会组装到 Netty 的 `Pipeline` 中，剩下来就是Netty的流程了


## 参考知识
- [给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)
- [谜之RxJava （三）update 2 —— subscribeOn 和 observeOn 的区别](https://segmentfault.com/a/1190000004856071)
- [Zuul-Wiki](https://github.com/Netflix/zuul/wiki)