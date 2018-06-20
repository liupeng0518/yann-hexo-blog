---
title: Netty源码分析-(2)-Acceptor与线程模型
date: 2018-06-19 12:29:48
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

## Acceptor
Acceptor 是 Netty 提供一个接收器。
![Reactor模式](https://s1.ax1x.com/2018/06/19/CziSq1.png)
还记得 `Reactor模式` 中那个的Acceptor正是连接了 `MainReactor` 和 `SubReactor`的核心所在，我们上文已经分析了Netty的 `NioServerChannel`，这篇我们就来分析下，Netty究竟是怎么去接受请求并且维护链接的。

<!-- more -->

## ServerBootstrapAcceptor

我们还得在ServerBootStrap的过程中，我们有看到一个奇怪的类 `ServerBootstrapAcceptor` 这个类，我们并没有深入去了解这个类的作用，我们现在来简单的分析下这个类。

```java
private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter { ➊

        private final EventLoopGroup childGroup;
        private final ChannelHandler childHandler;
        private final Entry<ChannelOption<?>, Object>[] childOptions;
        private final Entry<AttributeKey<?>, Object>[] childAttrs;
        private final Runnable enableAutoReadTask;

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler); ➋

            setChannelOptions(child, childOptions, logger);

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() { ➌
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
    }
```
➊ 这里指明这个 `ServerBootstrapAcceptor`类是一个 `ChannelInboundHandlerAdapter` 我们都知道 `ChannelInboundHandlerAdapter` 的作用是接受数据请求的实际处理者。  
➋ 这里我们将Child的Handler附加到`Pipeline`内
➌ 将 `ChildChannel` 注册到 `ChildEventGroup`，我们可以轻而易举的得知，这里就是实现`Reactor模式`的核心

## childGroup.register(child) 探秘
通过IDEA的强势跳转功能，我们轻松的定位到

```java
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
这个函数看上去好像有点眼熟，这不就是上一章中服务端是如何注册自己的`ServerNioChannel`的方案，原来 `NioChannel` 也是相同的味道。这里就不展开说了，具体的注册过程在上一章中已有体现。

## 小结1
我们知道 `ServerBootstrap` 在初始化的过程中创建了一个 `ServerBootstrapAcceptor`,  `ServerBootstrapAcceptor` 在读取数据的时候，会将当前的 `NioChannel`注册到  `ChildEventLoopGroup` 也就是 `workerGroup` 中

<div style="width:100%; overflow-y:scroll;" id="diagram"></div>
<script>
	var data =
	['Title: 已知的Reactor流程',
	 'Client->ServerChannel: 1. Open',
	 'ServerChannel->ServerBootstrapAcceptor: 2.read msg',
	 'ServerBootstrapAcceptor->ChildEventLoopGroup: 3.register(ChannelPromise)',
    ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple', scale: 0.5});
</script>

那对于我们来说，就是 `Client` 如何触发 `ServerChannel` 的READ事件 和  `ServerChannel` 将数据交付给 `ServerBootstrapAcceptor` 还是未知的。那我们大胆的预测第一个问题是一个Loop循环，而第二个问题是通过Server的`Pipeline` 传递过去的（因为`ServerBootstrapAcceptor` 继承自 `ChannelInboundHandlerAdapter`）,让我们来验证下吧。


## 借助Debug工具探索
这个时候就不要盲人摸象啦，我们感谢 `jetbrains` 提供了 [IntelliJ IDEA](https://www.jetbrains.com/idea/) 这个免费的IDE，我们通过断点的方式在我们的代码中设置断点，去观察下运行的`栈`。
![debug-tool-by-idea](https://s1.ax1x.com/2018/06/19/CzSfnP.jpg)

我们通过 `Debug Frames` 可以清晰的看到栈，我们一层层的去寻找，我们很快可以发现一个有趣的方案
`io.netty.channel.nio.NioEventLoop#run`

```java
protected void run() {
    for (;;) { ➊
        try {
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys(); ➋
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
        }
    }
}
```
➊ 处的Loop正对应着这个无限循环的事件
➋ 处即是最核心的处理逻辑

这个类是`NioEventLoop` 我们发现了真正处理逻辑的是在这个Thread的Run中，在这里我们就明白，整个Reactor那个循环中心就是在此处。继续往下探索。

在`io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)` 中
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    try {
        int readyOps = k.readyOps();
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { ➊
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            unsafe.finishConnect();
        }
        if ((readyOps & SelectionKey.OP_WRITE) != 0) { ➋
            ch.unsafe().forceFlush();
        }

        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) { ➌
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```
➊ 处理连接事件
➋ 处理写事件
➌ 处理读事件
我们下一部分分析下 `读事件` 的处理方式。

## Channel read 探秘
在 `io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read` 中有具体的实现。
```java
@Override
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle(); 
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            pipeline.fireChannelRead(readBuf.get(i)); ➊
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

    } finally {

    }
}

```
去除一些非主干部分，我们发现 ➊ 部分我们将数据传递给Pipeline 处理，到这里解决了我们第一个处的疑问即：就是 `Client` 如何触发 `ServerChannel` 的READ事件。那我们还不知道第二个问题，这个本来在ServerEventGroup中的事件是如何委托给 ChildEventGroup 的。那事件的核心即是 ➊ 处的 pipeline，我们通过Debug工具可以轻易的发现这里的pipeline是 `DefaultChannelPipeline{(ServerBootstrap$ServerBootstrapAcceptor#0 = io.netty.bootstrap.ServerBootstrap$ServerBootstrapAcceptor)}`，wow，我们抓住了它。

## 小结2
在[小结1](#小结1)中，我们已经发现了是如何将 `BossGroup` 的事件委托给 `WorkerGroup`，而在后续的分析中，我们知道了Netty是如何将最初的监听事件传递给 `ServerBootstrapAcceptor` 的，如下图。

<div style="width:100%; overflow-y:scroll;" id="diagram2"></div>
<script>
	var data2 =
	['Title: 完整的Reactor流程',
	 'Client->NioEventLoop: 1. Open，在NioEventLoop中循环监听',
     'NioEventLoop->DefaultChannelPipeline: 2.委托给ServerBootstrapAcceptor处理',
	 'DefaultChannelPipeline->ServerBootstrapAcceptor: 2.read msg',
	 'ServerBootstrapAcceptor->ChildEventLoopGroup: 3.register(ChannelPromise)',
    ].join('\n');
  	var diagram = Diagram.parse(data2);
  	diagram.drawSVG("diagram2", {theme: 'simple', scale: 0.5});
</script>

## 知识点
- ServerChannel 如何监听事件
- BossEventLoopGroup 如何将事件转让给 WorkerEventLoopGroup （ServerBootstrapAcceptor用途）