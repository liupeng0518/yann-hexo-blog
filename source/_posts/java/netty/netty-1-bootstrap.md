---
title: Netty源码分析-(1)-Bootstrap
date: 2018-06-12 23:29:48
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

## Bootstrap
Bootstrap 是 Netty 提供的一个便利的工厂类, 我们可以通过它来完成 Netty 的客户端或服务器端的 Netty 初始化。
下面我以 Netty 源码例子中的 Discard 服务器作为例子, 从客户端和服务器端分别分析一下Netty 的程序是如何启动的。

[快速定位几个关键点](#补番-几个关键点)

<!-- more -->

## 一个标准 ServerBootStarp

```java
public static void main(String[] args) throws Exception {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        ServerBootstrap b = new ServerBootstrap();
        b.group(bossGroup, workerGroup) ➊
                .channel(NioServerSocketChannel.class) ➋
                .childHandler(new ChannelInitializer<SocketChannel>() {  ➌
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new DiscardHandler());
                    }
                })
                .option(ChannelOption.SO_BACKLOG, 128) ➍
                .childOption(ChannelOption.SO_KEEPALIVE, true);

        ChannelFuture f = b.bind(8080).sync(); ➎
        f.channel().closeFuture().sync();

    } finally {
        workerGroup.shutdownGracefully();
        bossGroup.shutdownGracefully();
    }
}
```
我们轻而易举的可以发现代码里面有几个元素
- ➊ 声明的 EventLoopGroup
- ➋ Channel的类型是 NioServerSocketChannel，从我们之前写JavaNIO编程可以猜测出来，这就是Server端的Socket
- ➌ 这里什么我们的Handler处理的逻辑，从childHandler逻辑上看起来应该是子的处理逻辑
- ➍ option() 和下一行 childOption() 配置TCP一系列的参数
- ➎ 在这里我们发现真正的函数入口

这里补充下在传统的TCP编程中，我们需要下图的一个流程
![tcp-bind](https://www.2cto.com/uploadfile/Collfiles/20180504/20180504092323746.png)

我们需要在服务器端绑定我们的监听端口，那核心的启动逻辑也可以猜测出就是在这个地方了，我们从这个`bind()`作为我们的突破口去看看这个Netty是怎么启动起来的。

## doBind函数分析
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister(); ➊
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } 
    return promise;
}

```

else代码被我省略，我们先看看核心正常逻辑，我们在 ➊ 处发现的是注册ChannelFunture，根据名称我们也可以猜测出来是一个异步的Channel注册，继续深入 `initAndRegister()` 函数。

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();  ➊
        init(channel); ➋
    } catch (Throwable t) {
        //略
    }

    ChannelFuture regFuture = config().group().register(channel); ➌
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }
    return regFuture;
}
````

依然关注核心，我们可以从这个 ➊ 处比较简单，各位可以自行看一下，NIO会采用反射的方式去获得NIOServerChannel这里刚好对应我们在Bootstarp中的 `NioServerSocketChannel.class`  
➋ 处，我们发现的是一个抽象方法，我们发现有2个实现，因为我们看的是服务器端代码，我们关注这个函数 `io.netty.bootstrap.ServerBootstrap#init(channel)` [点击转ServerBootstrap分析](#ServerBootstrap-init-函数分析)  
➌ 处是一个注册，我们大胆的预测这里就是我们的EventLoop上注册我们的NioServer，前半段` config().group()`很好理解，我们拿到的也是我们的 `bossGroup`,这块的注册看样子是一个很重要的函数，我们来到实现的地方去看一下 [点击转Register分析](#AbstractUnsafe-register-函数分析)


## ServerBootstrap.init 函数分析
```java
@Override
void init(Channel channel) throws Exception {
    //略
    ChannelPipeline p = channel.pipeline(); ➊

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() { ➋
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

我们发现在 ➊ 处，我们通过Channel打开了一个Pipeline，还记得那个Channel是什么吗？就是之前我们通过 `channelFactory.newChannel()` 创建出来的 Channel。  
➋ 处，我们看见了eventLoop这个，和我们之前看到的那个EventLoop是一个东西吗？但是这里看见一个很重要的类是 `ServerBootstrapAcceptor` 正如我们自己写的DEMO项目这势必是一个接收器。  

我们在阅读这里的代码留下了**一个疑问**，在 `initChannel(final Channel ch)`中我们获得的对象是**什么**？


## AbstractUnsafe.register 函数分析
```java
public ChannelFuture register(Channel channel) {
        return register(new DefaultChannelPromise(channel, this));
}
    
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    AbstractChannel.this.eventLoop = eventLoop; 
    if (eventLoop.inEventLoop()) {
        register0(promise); ➊
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise); ➋
                }
            });
        } catch (Throwable t) {
        }
    }
}
```

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        
    } catch (Throwable t) {  
    }
}
```


一层层的拨开，我们都发现了 ➊ 和 ➋ 都调用了 `register0(promise)` ，明显这个才是一个核心的为方法在这个方法里面有一个 `doRegister();` 勾起了我们的关注，我们在`io.netty.channel.nio.AbstractNioChannel#doRegister` 中获得这个方法的详细内容。

```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);  ➊
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```
恍然开朗，我们在 ➊ 处的得到了这个 `selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);`这才是我们真正的注册的地方。


## 小结1：注册NioServerSocketChannel的流程
我们分析到现在我们已经可以知道了注册NioServerSocketChannel的流程，我们先梳理一下

{% raw %}

<div style="width:100%; overflow-y:scroll;" id="diagram"></div>
<script>
	var data =
	['Title: NioServerSocketChannel 注册过程',
	 'Main->ServerBootstrap: 1.bind(int)',
	 'ServerBootstrap->EventLoopGroup: 2.register(Channel)',
	 'EventLoopGroup->AbstractUnsafe: 3.register0(ChannelPromise)',
     'AbstractUnsafe->AbstractNioChannel: 4.doRegister()'
    ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple', scale: 0.5});
</script>

{% endraw %}

## 继续探索 AbstractUnsafe#register0 函数
在我们的头顶上还有一个疑问，`ServerBootstrap.init` 中回调的那个 Channel 是什么，我们为了探索这个答案，我们继续往下探索。
`pipeline.invokeHandlerAddedIfNeeded();` 上面的一行注释告诉们在实际通知Promise之前我们需要增加我们的Handler，再下面反而没有什么有用的信息，我们就从这个函数入手继续分析，我们点击进去会发现。

```java
final void invokeHandlerAddedIfNeeded() {
    assert channel.eventLoop().inEventLoop();
    if (firstRegistration) {
        firstRegistration = false;
        // We are now registered to the EventLoop. It's time to call the callbacks for the ChannelHandlers,
        // that were added before the registration was done.
        callHandlerAddedForAllHandlers();
    }
}
```
这块又多了很核心的函数，我们这里需要注册所有的的Handlers，是不是感觉和我们之前留下的疑问是接近的，但是我们仍然不能断言这个就是对的。
```java
private void callHandlerAddedForAllHandlers() {
    final PendingHandlerCallback pendingHandlerCallbackHead;
    synchronized (this) {
        pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
    }

    PendingHandlerCallback task = pendingHandlerCallbackHead; ➊
    while (task != null) {
        task.execute();
        task = task.next;
    }
}
```
➊处我们发现这里就是我们的Handler的回调函数，大胆的预测也就是我们上文疑问处的回调函数调用，继续跟踪下，我们在
`io.netty.channel.DefaultChannelPipeline#callHandlerAdded0`处发现了最终的调用者。

```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        ctx.setAddComplete();
        ctx.handler().handlerAdded(ctx);
    } catch (Throwable t) {
        //略
}
```
核心的也就是在这里进行handler增加再往下跟踪一个函数
```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
            remove(ctx);
        }
        return true;
    }
    return false;
}
```
在这里➊处也就是我们自己的初始化脚本，是一个抽象类，正好对应着我们的那个回调函数。但是我在这里发现另外一个疑问 `ChannelHandlerContext`我们在使用的时候是经常会使用到，`ChannelHandlerContext`包裹了`Channel`和我们的`Pipeline`是一个有状态的对象，那他第一次初始化是在哪里？这里通过肉眼看是不明智的，这个时候借助于我们现代化的IDEA。

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                                  boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name"); ➊
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```
我们在 ➊ 处设置一个 `断点`，我们去观察下堆栈。

```java
@Override
public T newChannel() {
    try {
        return clazz.getConstructor().newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```
我们发现在初始化 channel 的时候就会初始化我们的 `AbstractChannelHandlerContext` 这就是说这个 `ServerSocket`的`HandlerContext`在整个运行的生命周期都是唯一的。那我们也去验证一下。

## 旁白: 验证 NioServerSocketChannel 的 AbstractChannelHandlerContext

```java
b.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .handler(new ChannelInitializer<NioServerSocketChannel>() {
        @Override
        protected void initChannel(NioServerSocketChannel ch) throws Exception {
            ch.pipeline().addLast(new FooHandler());
        }
    })
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new DiscardHandler());
        }
    })
    .option(ChannelOption.SO_BACKLOG, 128)
    .childOption(ChannelOption.SO_KEEPALIVE, true);


public class FooHandler extends ChannelInboundHandlerAdapter {
    private static final Logger LOG = LogManager.getLogger(FooHandler.class);
    private AttributeKey<String> FooAttributeKey = AttributeKey.newInstance("foo");

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {

        Attribute<String> attr = ctx.channel().attr(FooAttributeKey);
        if (attr.get() == null) {
            LOG.info("FooAttributeKey value setting  {}", "foo value");
            attr.setIfAbsent("foo value");
        } else {
            LOG.info("FooAttributeKey value {}", attr.get());
        }

    }
}
```

我们使用curl连接2次，会发现结果是
```bash
16:25:17.399 [nioEventLoopGroup-2-1] INFO  top.yannxia.java.understanding.netty.FooHandler - FooAttributeKey value setting  foo value
16:25:41.266 [nioEventLoopGroup-2-1] INFO  top.yannxia.java.understanding.netty.FooHandler - FooAttributeKey value foo value
```
验证了我们的想法，那我们由此推断出来，每当一个Channel建立的时候，我们都会为这个Channel创建一个Context。


## 番外: ClientServerBootStarp
我们啃完了难啃的ServerBootStarp，那ClientServerBootStarp就留给读者们自行阅读了，其实从上文我们可以大胆的猜测出来
1. ClientServerBootStarp 是 ServerBootStarp的简化，去除了 NioServerSocketChannel
2. ClientServerBootStarp 的线程模型是简化的，没有 Acceptor 的


## 补番:几个关键点
0. 阅读Netty代码的核心是了解Netty的线程模型
    参考 [Netty in action — 事件循环和线程模式](http://www.importnew.com/26718.html)

1. Netty并没有采用BossEventGroup的线程池 
    `io.netty.channel.MultithreadEventLoopGroup#register(io.netty.channel.Channel)` 中
    ```java
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }
    ```
    我们知道 ServerNioChannel 只会被注册一次，即便是我们创建了多个Thread，我们依然只能选择其中一个注册我们的 ServerNioChannel

