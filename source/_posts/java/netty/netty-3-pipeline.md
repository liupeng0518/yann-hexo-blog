---
title: Netty源码分析-(3)-ChannelPipeline
date: 2018-06-20 12:00:48
tags:
toc: true
categories: ["java", "网络编程", 'Netty']
---

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

## ChannelPipeline
ChannelPipeline是整个Netty的流程处理的中心，整个结构如下图所示
![ChannelPipeline](https://s1.ax1x.com/2018/06/19/CzGcXn.png)
在 [ChannelPipeline.html](https://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) 有非常详细的文档。
每个channel内部都会持有一个ChannelPipeline对象pipeline。pipeline默认实现DefaultChannelPipeline内部维护了一个DefaultChannelHandlerContext链表。

<!-- more -->

## DefaultChannelPipeline
`DefaultChannelPipeline` 是Netty的实现类，我们看看他的内部实现。

```java
public class DefaultChannelPipeline implements ChannelPipeline {
    
    final AbstractChannelHandlerContext head; ➊
    final AbstractChannelHandlerContext tail;

    private final Channel channel; ➋
    private final ChannelFuture succeededFuture; ➌
    private final VoidChannelPromise voidPromise;

    private Map<EventExecutorGroup, EventExecutor> childExecutors;
    private volatile MessageSizeEstimator.Handle estimatorHandle;
    private boolean firstRegistration = true;
}
```
在 ➋ 处我们看到了支撑业务的的 `Channel`, ➌ 是一个成功的回调函数。而比较有趣的就是这个 head 和 tail了，我们都知道Pipeline就像一个管道一样将流程贯通起来，那Head和Tail看起来就是头和尾的意思。我们看看这2个变量的赋值。

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
我们在这里看到tail 和 head 分别是Netty自己实现的类，我们来看看类声明。
```java
final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler

final class HeadContext extends AbstractChannelHandlerContext
            implements ChannelOutboundHandler, ChannelInboundHandler
```
Wow，很精妙的实现，分别是一个 ChannelInboundHandler 和一个 ChannelOutboundHandler。

## Pipeline 增加 Handler
我们知道我们实现具体的业务逻辑是在 `Handler` 中，我们在Demo中有一个声明
```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ch.pipeline().addLast(new DiscardHandler());
    }
})
```
我们在这里添加一个 `DiscardHandler`,我们探索下这个 `addLast()` 行为究竟做了什么。
```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        newCtx = newContext(group, filterName(name, handler), handler);
        addLast0(newCtx); ➊ 
    }
    callHandlerAdded0(newCtx);
    return this;
}
```
在 ➊ 处我们发现最为重要的增加逻辑就是在 `addLast0()` 中
```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```
这里是一个很标准的链表操作，看到这里我们就可以大致才出来`addXXX()`一系列的行为都是类似，用一张图来表示下
![netty pipeline](https://s1.ax1x.com/2018/06/19/CzJzV0.png)

我们一直操作的都是在 `Head` 和 `Tail` 之间，我们可以任意的在之间增加或者删除具体的 `Handler`

## Pipeline的调用顺序
万物的开始我们知道从接受数据开始在 
`io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read` 中有一行
```java
allocHandle.incMessagesRead(1);
readPending = false;
pipeline.fireChannelRead(byteBuf); ➊
byteBuf = null;
```
➊ 就是万物的开始，从此开始进入Pipeline，然我又在
`io.netty.channel.DefaultChannelPipeline#fireChannelRead`中获得
```java
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg); ➋
    return this;
}
```
➋ 处我们可以看到我们第一次阅读数据是从HeadContext开始的
在 `io.netty.channel.AbstractChannelHandlerContext#fireChannelRead`
```java
@Override
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg); ➌
    return this;
}
```
➌ 处查找InBoundHandler，这个函数还蛮有意思
```java
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!ctx.inbound);
    return ctx;
}
```
是一个标准的循环，我们一直去找到下一个Inbound的Handler去处理。

## 小结1
在Head的调用中有点特殊有点复杂，我们使用时序图来理解下。
<div style="width:100%; overflow-y:scroll;" id="diagram"></div>
<script>
	var data =
	['Title: Pipeline的Head处理流程',
	 'NioByteUnsafe->DefaultChannelPipeline: 1. fireChannelRead(byteBuf)',
	 'DefaultChannelPipeline->DefaultChannelPipeline: 2.invokeChannelRead(head, msg)',
	 'DefaultChannelPipeline->DefaultChannelPipeline: 3. ((ChannelInboundHandler) handler()).channelRead(this, msg)',
     'DefaultChannelPipeline->HeadContext: 4. fireChannelRead(msg)',
     'HeadContext->ChannelHandlerContext: 5.invokeChannelRead(findContextInbound(), msg)'
    ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple', scale: 0.5});
</script>

通过 **findContextInbound()** 就获得我们装载的第一个InboundHandler。  
通过分析我们得知Pipeline的下一次的调用依赖的是 `io.netty.channel.AbstractChannelHandlerContext#fireChannelRead` 这个函数。所以在自己编写的 `ChannelInboundHandlerAdapter` 实现类中，我们在实现 `channelRead`方法的时候，如果我们需要调用下一层，我们需要手动 `ctx.fireChannelRead(msg);`

## 延伸 Pipeline 的 Write操作
从Head的Read操作中，我们可以猜测出，Write也就是Tail的Write操作，我们去验证下自己的思路。
在`io.netty.channel.DefaultChannelPipeline#writeAndFlush(java.lang.Object)`处我们发现
```java
@Override
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg); ➊
}
```
➊ 处证明处理的流程是从 Tail开始的

`io.netty.channel.AbstractChannelHandlerContext#write(java.lang.Object, boolean, io.netty.channel.ChannelPromise)` 中
```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound(); ➋
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } 
}
```
➋ 和FindInBound多么相似，也就是找个下一个OutBound
最终的写入处是在 
`io.netty.channel.DefaultChannelPipeline.HeadContext#write`
```java
@Override
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```
我们看到熟悉的unsafe，根据我们这么久以来和Netty打交道的经验，这里应该就是Java原生的Nio部分。在具体的实现部分
```java
public final void write(Object msg, ChannelPromise promise) {
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    outboundBuffer.addMessage(msg, size, promise);
}
```
我们将需要写入的数据放入了 `outboundBuffer` 中。
具体的写入逻辑在 `flush` 中，最终会调用 `io.netty.channel.socket.nio.NioSocketChannel#doWrite` 这部分就留给读者自行阅读了。这个时候我们再回顾下 ![netty pipeline](https://s1.ax1x.com/2018/06/19/CzJzV0.png) 大家会发现Netty的Pipeline实现的是如此的优雅，核心的逻辑清晰明了。



## 知识点
- ServerChannel 如何监听事件
- BossEventLoopGroup 如何将事件转让给 WorkerEventLoopGroup （ServerBootstrapAcceptor用途）

## 附录：ChannelPipeline事件
 - Inbound 事件:
    - ChannelHandlerContext.fireChannelRegistered()
    - ChannelHandlerContext.fireChannelActive()
    - ChannelHandlerContext.fireChannelRead(Object)
    - ChannelHandlerContext.fireChannelReadComplete()
    - ChannelHandlerContext.fireExceptionCaught(Throwable)
    - ChannelHandlerContext.fireUserEventTriggered(Object)
    - ChannelHandlerContext.fireChannelWritabilityChanged()
    - ChannelHandlerContext.fireChannelInactive()
    - ChannelHandlerContext.fireChannelUnregistered()
 - Outbound 事件:
    - ChannelHandlerContext.bind(SocketAddress, ChannelPromise)
    - ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromise)
    - ChannelHandlerContext.write(Object, ChannelPromise)
    - ChannelHandlerContext.flush()
    - ChannelHandlerContext.read()
    - ChannelHandlerContext.disconnect(ChannelPromise)
    - ChannelHandlerContext.close(ChannelPromise)
    - ChannelHandlerContext.deregister(ChannelPromise)
