---
title: Motan源码分析
date: 2018-06-30 13:25:48
tags:
toc: true
categories: ["java", "RPC", 'Motan']
---

# Motan
`Motan` 是一套基于java开发的RPC框架，除了常规的点对点调用外，`Motan` 还提供服务治理功能，包括服务节点的自动发现、摘除、高可用和负载均衡等。`Motan` 具有良好的扩展性，主要模块都提供了多种不同的实现，例如支持多种注册中心，支持多种rpc协议等。（可以看作 `Dubbo` 的复刻版）

[跳过介绍](#启动分析)
<!-- more -->

{% raw %}

<script src="https://lib.baomitu.com/webfont/1.6.28/webfontloader.js"></script>
<script src="https://lib.baomitu.com/snap.svg/0.5.1/snap.svg-min.js"></script>
<script src="https://lib.baomitu.com/underscore.js/1.9.0/underscore-min.js"></script>
<script src="https://lib.baomitu.com/raphael/2.2.7/raphael.min.js"></script>
<script src="https://lib.baomitu.com/js-sequence-diagrams/1.0.6/sequence-diagram-min.js"></script>

{% endraw %}

## Montan架构
`Motan` 中分为服务提供方(RPC Server)，服务调用方(RPC Client)和服务注册中心(Registry)三个角色。
![arch-motan)](https://s1.ax1x.com/2018/06/30/PFWib4.png)

## 模块概述
Motan框架中主要有 `register`、`transport`、`serialize`、`protocol`几个功能模块，各个功能模块都支持通过SPI进行扩展，各模块的交互如下图所示：
![motan-models](https://s1.ax1x.com/2018/06/30/PFWkVJ.png)

- `register`: 用来和注册中心进行交互，包括注册服务、订阅服务、服务变更通知、服务心跳发送等功能；Server端会在系统初始化时通过register模块注册服务，Client端在系统初始化时会通过register模块订阅到具体提供服务的Server列表，当Server 列表发生变更时也由register模块通知Client。

- `protocol`: 用来进行RPC服务的描述和RPC服务的配置管理，这一层还可以添加不同功能的filter用来完成统计、并发限制等功能。

- `serialize`: 将RPC请求中的参数、结果等对象进行序列化与反序列化，即进行对象与字节流的互相转换；默认使用对java更友好的hessian2进行序列化。

- `transport`:  用来进行远程通信，默认使用Netty nio的TCP长链接方式。

- `cluster`:  Client端使用的模块，cluster是一组可用的Server在逻辑上的封装，包含若干可以提供RPC服务的Server，实际请求时会根据不同的高可用与负载均衡策略选择一个可用的Server发起远程调用。

在进行RPC请求时，Client通过代理机制调用cluster模块，cluster根据配置的HA和LoadBalance选出一个可用的Server，通过serialize模块把RPC请求转换为字节流，然后通过transport模块发送到Server端。

## Demo without Spring
一切代码的开始总是从Demo开始的


{% tabs Sample unique name %}
<!-- tab Motan Server端 -->
这段代码可以说是最为简单的Server端了。
{% codeblock lang:java %}
public class MotanServer {
    public static void main(String[] args) {
        //1. 使用代码形式实现, 可以不使用spring 容器
        ServiceConfig<MotanDemoService> motanDemoService = new ServiceConfig<>();

        //2. 配置端口及实现类
        motanDemoService.setInterface(MotanDemoService.class);
        motanDemoService.setRef(new MotanDemoServiceImpl());

        //3. 配置服务的group及版本号
        motanDemoService.setGroup("motan-demo-rpc");
        motanDemoService.setVersion("1.0");

        //4. 配置 注册中心
        RegistryConfig directRegistry = new RegistryConfig();
        directRegistry.setRegProtocol("local");
        directRegistry.setCheck("false"); //不检查是否注册成功
        motanDemoService.setRegistry(directRegistry);

        //5. 配置RPC协议
        ProtocolConfig protocol = new ProtocolConfig();
        protocol.setId("motan");
        protocol.setName("motan");
        motanDemoService.setProtocol(protocol);

        //6. 配置协议:端口 和输出
        motanDemoService.setApplication("motan");
        motanDemoService.setExport("motan:8002");
        motanDemoService.export();

        //7. 设置心跳
        MotanSwitcherUtil.setSwitcherValue(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER, true);
        System.out.println("server start...");
    }
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab Motan Client端 -->
{% codeblock lang:java %}
public class MotanClient {
    public static void main(String[] args) {
        //1. 引用配置
        RefererConfig<MotanDemoService> motanDemoServiceReferer = new RefererConfig<>();

        //2. 设置接口及实现类
        motanDemoServiceReferer.setInterface(MotanDemoService.class);

        //3. 配置服务的group以及版本号
        motanDemoServiceReferer.setGroup("motan-demo-rpc");
        motanDemoServiceReferer.setVersion("1.0");
        motanDemoServiceReferer.setRequestTimeout(300);

        // 4. 配置注册中心直连调用
         RegistryConfig directRegistry = new RegistryConfig();
         directRegistry.setRegProtocol("local");
         directRegistry.setPort(8002);
         motanDemoServiceReferer.setRegistry(directRegistry);
         
        // 5. 配置RPC 协议
        ProtocolConfig protocol = new ProtocolConfig();
        protocol.setId("motan");
        protocol.setName("motan");

        motanDemoServiceReferer.setProtocol(protocol);
        motanDemoServiceReferer.setDirectUrl("localhost:8002");

        // 6. 使用服务调用
        MotanDemoService service = motanDemoServiceReferer.getRef();
        System.out.println(service.hello("motan"));

        System.exit(0);
    }
}
{% endcodeblock %}
<!-- endtab -->
<!-- tab 接口定义 -->
{% codeblock lang:java %}
public interface MotanDemoService {
    String hello(String name);
}

public class MotanDemoServiceImpl implements MotanDemoService {
    @Override
    public String hello(String name) {
        return "hello-" + name;
    }
}
{% endcodeblock %}
<!-- endtab -->
{% endtabs %}


执行结果如下

```bash
by-motan-motan
```

# 启动分析
## 启动流程分析
我们从上面的代码可以看出来，在服务侧的代码中，大部分都是在声明，在 `motanDemoService.export();`这部才是开始执行暴露出 `RPC` 接口的行为，我们从这里作为一个突破口。

```java
public synchronized void export() {
    checkInterfaceAndMethods(interfaceClass, methods);
    List<URL> registryUrls = loadRegistryUrls(); ➊
    Map<String, Integer> protocolPorts = getProtocolAndPort();
    for (ProtocolConfig protocolConfig : protocols) {
        Integer port = protocolPorts.get(protocolConfig.getId());
        if (port == null) {
            //Skip
        }
        doExport(protocolConfig, port, registryUrls); ➋
    }

    afterExport(); ➌
}
```
代码逻辑很清晰   
➊ 载入注册中心的地址
➋ 真实暴露接口的行为，我们从这里继续分析下去
➌ 暴露之后的操作

```java
private void doExport(ProtocolConfig protocolConfig, int port, List<URL> registryURLs) {
    String protocolName = protocolConfig.getName();    
    String hostAddress = host;

    Map<String, String> map = new HashMap<String, String>();
    collectConfigParams(map, protocolConfig, basicService, extConfig, this);
    collectMethodConfigParams(map, this.getMethods());

    URL serviceUrl = new URL(protocolName, hostAddress, port, interfaceClass.getName(), map);
    List<URL> urls = new ArrayList<URL>();

    // injvm 协议只支持注册到本地，其他协议可以注册到local、remote
    if (MotanConstants.PROTOCOL_INJVM.equals(protocolConfig.getId())) {
        URL localRegistryUrl = null;
        for (URL ru : registryURLs) {
            if (MotanConstants.REGISTRY_PROTOCOL_LOCAL.equals(ru.getProtocol())) {
                localRegistryUrl = ru.createCopy();
                break;
            }
        }
        urls.add(localRegistryUrl);
    } else {
        for (URL ru : registryURLs) {
            urls.add(ru.createCopy());
        }
    }

    for (URL u : urls) { ➊
        u.addParameter(URLParamType.embed.getName(), StringTools.urlEncode(serviceUrl.toFullStr()));
        registereUrls.add(u.createCopy());
    }

    ConfigHandler configHandler = ExtensionLoader.getExtensionLoader(ConfigHandler.class).getExtension(MotanConstants.DEFAULT_VALUE);   ➋

    exporters.add(configHandler.export(interfaceClass, ref, urls)); ➌
}
```
➊ 之前的代码都在在准备对象的URL，这个的URL是Motan自己实现的，在此处将URL收集起来
➋ 通过SPI技术载入 `ConfigHandler` 对象
➌ 暴露出接口并将其在 `exporters` 管理起来

我们来看看 `Exporter` 的定义
```java
public interface Exporter<T> extends Node {
    Provider<T> getProvider();
    void unexport();
}
```
![](https://s1.ax1x.com/2018/07/05/PZ93Y4.png)



原来 `Exporter` 是 `Provider` 的包装对象。迄今为止，我们值得启动流程为下图
<div style="width:100%; overflow-y:scroll;" id="diagram"></div>
<script>
	var data =
	['Title: Motan启动流程',
	 'ServiceConfig->ConfigHandler: 1. export(interfaceClass, ref, urls)',
	 'ConfigHandler->ServiceConfig: 2. exporters.add()',
    ].join('\n');
  	var diagram = Diagram.parse(data);
  	diagram.drawSVG("diagram", {theme: 'simple', scale: 0.5});
</script>


## 课间补习 SPI
`Java Service Provider` 主要是被框架的开发人员使用，比如java.sql.Driver接口，其他不同厂商可以针对同一接口做出不同的实现，mysql和postgresql都有不同的实现提供给用户，而Java的SPI机制可以为某个接口寻找服务实现。
> 当服务的提供者提供了一种接口的实现之后，需要在classpath下的META-INF/services/目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体的实现类。当其他的程序需要这个服务的时候，就可以通过查找这个jar包（一般都是以jar包做依赖）的META-INF/services/中的配置文件，配置文件中有接口的具体实现类名，可以根据这个类名进行加载实例化，就可以使用该服务了。JDK中查找服务的实现的工具类是：java.util.ServiceLoader。 By 《Java中SPI机制深入及源码解析》
关于详细的内容可以参考上文引用的博文 [Java中SPI机制深入及源码解析](https://cxis.me/2017/04/17/Java%E4%B8%ADSPI%E6%9C%BA%E5%88%B6%E6%B7%B1%E5%85%A5%E5%8F%8A%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)


## 继续前进
```java
public <T> Exporter<T> export(Class<T> interfaceClass, T ref, List<URL> registryUrls) {

    String serviceStr = StringTools.urlDecode(registryUrls.get(0).getParameter(URLParamType.embed.getName()));
    URL serviceUrl = URL.valueOf(serviceStr);

    // export service
    // 利用protocol decorator来增加filter特性
    String protocolName = serviceUrl.getParameter(URLParamType.protocol.getName(), URLParamType.protocol.getValue());
    Protocol orgProtocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(protocolName); ➊
    Provider<T> provider = getProvider(orgProtocol, ref, serviceUrl, interfaceClass);

    Protocol protocol = new ProtocolFilterDecorator(orgProtocol); ➋
    Exporter<T> exporter = protocol.export(provider, serviceUrl); ➌

    // register service
    register(registryUrls, serviceUrl); ➍

    return exporter;
}
```
➊ 又是熟悉的 `ExtensionLoader` 我们再次通过 `SPI` 获得我们的 `Protocal`，这里开始 `Protocal`有了多种实现，包括 `RPC`, `InJVM`, `Motan2` 等
➋ 我们将  `Protocol` 包裹为 `ProtocolFilterDecorator` [Motan Filter 机制](#Motan-Filter-机制)
➌ 将 `Protocol` 包裹为 `ProtocolFilterDecorator` 并发布出来 [Motan 注册机制](#Motan-Filter-机制)
➍ 将服务注册到服务发现上

## Motan Filter 机制
> Filter 发送/接收请求过程中增加切面逻辑，默认提供日志统计等功能
Filter 是 Motan 很重要的一个实现接口

```java
@Override
public <T> Exporter<T> export(Provider<T> provider, URL url) {
    return protocol.export(decorateWithFilter(provider, url), url); //将 provider 包装
}
private <T> Provider<T> decorateWithFilter(final Provider<T> provider, URL url) {
    List<Filter> filters = getFilters(url, MotanConstants.NODE_TYPE_SERVICE); ➊
    if (filters == null || filters.size() == 0) {
        return provider;
    }
    Provider<T> lastProvider = provider; ➋
    for (Filter filter : filters) {
        final Filter f = filter;
        if (f instanceof InitializableFilter) {
            ((InitializableFilter) f).init(lastProvider);
        }
        final Provider<T> lp = lastProvider;
        lastProvider = new Provider<T>() { ➌
            @Override
            public Response call(Request request) {
                return f.filter(lp, request);
            }
            //SKIP
        };
    }
    return lastProvider;
}
```
➊ 这里其实是老朋友了 `SPI` 去读取了 `Filter` 的实现
➋ 遍历每一个 Filter 让他们串联起来
➌ 未每个Filter 创建一个 匿名的 `Provider` 让他们变成一个 `Chain`

![Provider-Chain](https://s1.ax1x.com/2018/07/05/PVz3Yd.png)


<div style="width:100%; overflow-y:scroll;" id="diagram2"></div>
<script>
	var data2 =
	['Title: Motan启动流程',
	 'ServiceConfig->ConfigHandler: 1. export',
     'ConfigHandler->ConfigHandler: 2. 获得Protocol实现',
     'ConfigHandler->ConfigHandler: 3. Protocol 创建 Provider',
     'ConfigHandler->ProtocolFilterDecorator: 4. 构建Protocol Chain',
     'ConfigHandler->Protocol: 5. export',
	 'ConfigHandler->RegistryFactory: 6. register',
    ].join('\n');
  	var diagram2 = Diagram.parse(data2);
  	diagram2.drawSVG("diagram2", {theme: 'simple', scale: 0.5});
</script>


## Motan 注册机制
```java
private void register(List<URL> registryUrls, URL serviceUrl) {
    for (URL url : registryUrls) {
        // 根据check参数的设置，register失败可能会抛异常，上层应该知晓
        RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getExtension(url.getProtocol()); ➊
        Registry registry = registryFactory.getRegistry(url);
        registry.register(serviceUrl); ➋
    }
}
```
➊ 依然通过 `SPI` 获得 `RegistryFactory`
➋ 根据 `URL` 获得 `Registry`并注册

## export 接口

`com.weibo.api.motan.protocol.AbstractProtocol#export`
```java
public <T> Exporter<T> export(Provider<T> provider, URL url) {
    String protocolKey = MotanFrameworkUtil.getProtocolKey(url);
    synchronized (exporterMap) {
        Exporter<T> exporter = (Exporter<T>) exporterMap.get(protocolKey);
        exporter = createExporter(provider, url); ➊
        exporter.init(); ➋

        exporterMap.put(protocolKey, exporter); ➌
        return exporter;
    }
}
```
➊ 这里将必要的数据传入创建一个 `Exporter`
➋ 最终初始化的地方 `Exporter` 执行初始化行为
➌ **敲黑板** 这里这里将我们的 `protocolKey` 和 `exporter` 关联起来，也就是真实的 `Mapping` 关系

如果不慎注意，我们会忽略这里，其实在 `createExporter`中 `Motan`做了相当多的事情。

```java
public DefaultRpcExporter(Provider<T> provider, URL url, ConcurrentHashMap<String, ProviderMessageRouter> ipPort2RequestRouter,
                              ConcurrentHashMap<String, Exporter<?>> exporterMap) {
    super(provider, url);
    this.exporterMap = exporterMap;
    this.ipPort2RequestRouter = ipPort2RequestRouter;

    ProviderMessageRouter requestRouter = initRequestRouter(url); ➊
    endpointFactory =
            ExtensionLoader.getExtensionLoader(EndpointFactory.class).getExtension(
                    url.getParameter(URLParamType.endpointFactory.getName(), URLParamType.endpointFactory.getValue())); ➋
    server = endpointFactory.createServer(url, requestRouter); ➌
}
```
➊ 根据 `url` 我们初始化了 `ProviderMessageRouter` 而 `ProviderMessageRouter` 是一个 `MessageHandler` 在后续的 `NettyServer` 会被使用到
➋ 老朋友的 `SPI`
➌ 这里的 `server` 我们可以发现 只有一个的 `NettyServer` 实现，想必是为了后续可以支持多种 `Web` 框架

在  `createServer` 的时候还会 构造 `Codec`，也是一个拓展点。
```java
//com.weibo.api.motan.transport.AbstractServer#AbstractServer(com.weibo.api.motan.rpc.URL)
public AbstractServer(URL url) {
    this.url = url;
    this.codec =
            ExtensionLoader.getExtensionLoader(Codec.class).getExtension(
                    url.getParameter(URLParamType.codec.getName(), URLParamType.codec.getValue()));
}
```



这里我们发现 我们最终操作的是 
```java
public interface Node {
    void init();  // 初始化
    void destroy(); // 销毁
    boolean isAvailable(); // 是否可用
    String desc(); //描述信息
    URL getUrl(); // 访问URL
}
```
而通过Debug发现我们知道我们最终调用的地方是 
`com.weibo.api.motan.protocol.rpc.DefaultRpcExporter#doInit`
我们发现很简单只做了一件事情
```java
protected boolean doInit() {
    return server.open();
}
```
就在 `Open` 函数中已经暴露了一切。

```java
@Override
public boolean open() {
    int minWorkerThread, maxWorkerThread;
    standardThreadExecutor = (standardThreadExecutor != null && !standardThreadExecutor.isShutdown()) ? standardThreadExecutor
            : new StandardThreadExecutor(minWorkerThread, maxWorkerThread, workerQueueSize, new DefaultThreadFactory("NettyServer-" + url.getServerPortStr(), true));
    standardThreadExecutor.prestartAllCoreThreads(); ➊

    ServerBootstrap serverBootstrap = new ServerBootstrap();
    serverBootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ChannelPipeline pipeline = ch.pipeline();
                    pipeline.addLast("decoder", new NettyDecoder(codec, NettyServer.this, maxContentLength)); ➋
                    pipeline.addLast("encoder", new NettyEncoder()); ➌
                    pipeline.addLast("handler", new NettyChannelHandler(NettyServer.this, messageHandler, standardThreadExecutor)); ➍
                }
            }); 
    serverBootstrap.childOption(ChannelOption.TCP_NODELAY, true);
    serverBootstrap.childOption(ChannelOption.SO_KEEPALIVE, true);
    ChannelFuture channelFuture = serverBootstrap.bind(new InetSocketAddress(url.getPort()));
    channelFuture.syncUninterruptibly(); ➎
    serverChannel = channelFuture.channel();
    state = ChannelState.ALIVE;
    StatsUtil.registryStatisticCallback(this);
    return state.isAliveState(); 
}
```

➊ 创建一个 `Work` 线程组
➋ 构造 `Netty` 的 `Decoder` 处理器，是一个 `ByteToMessageDecoder`
➌ 构造 `Netty` 的 `Encoder` 处理器，是一个 `MessageToByteEncoder`
➍ 实际上的工作 `Handler` 在这里，而`Handler` 又是我们在 `Exporter` 中的 `ProviderMessageRouter`

## 启动顺序总结
阅读到此处，我们已经分析完成了关于 `Motan Server` 的启动流程

<div style="width:100%; overflow-y:scroll;" id="diagram3"></div>
<script>
	var data3 =
	['Title: Motan启动流程',
	 'ServiceConfig->ConfigHandler: 1. export',
     'ConfigHandler->ConfigHandler: 2. 获得Protocol实现',
     'ConfigHandler->ConfigHandler: 3. Protocol 创建 Provider',
     'ConfigHandler->ProtocolFilterDecorator: 4. 构建Protocol Chain',
     'ConfigHandler->Protocol: 5. export',
     'Protocol->Exporter: 6. create',
     'Exporter->Server: 7. create Server(Netty)',
     'Exporter->Exporter: 8. init',
     'Exporter->Exporter: 9. open Netty Server',
	 'ConfigHandler->RegistryFactory: 10. register',
    ].join('\n');
  	var diagram3 = Diagram.parse(data3);
  	diagram3.drawSVG("diagram3", {theme: 'simple', scale: 0.5});
</script>

我们回归下至今我们知道  `服务端`
- protocol
- serialize
- transport
这三个模块是如何工作的，值得注意的正如架构图所示，我们最终在 `Netty` 暴露出来的是 `ProviderMessageRouter` 所以最终工作的是 `Provider` 对象，而 `Provider` 在 `Motan` 中被 `Protocol` 管理起来了，而在架构图上没有体现这一点。

## Motan Server 接受请求
在 `NettyChannelHandler` 中接受数据请求的
```java
//com.weibo.api.motan.transport.netty4.NettyChannelHandler#processRequest
private void processRequest(final ChannelHandlerContext ctx, Request request) {
    final long processStartTime = System.currentTimeMillis();
    try {
        Object result;
        try {
            result = messageHandler.handle(channel, request); ➊
        } catch (Exception e) {
           //SKIP
        }
        DefaultResponse response; ➋
        if (result instanceof DefaultResponse) {
            response = (DefaultResponse) result;
        } else {
            response = new DefaultResponse(result);
        }
        response.setRequestId(request.getRequestId());
        response.setProcessTime(System.currentTimeMillis() - processStartTime);
        sendResponse(ctx, response); ➌
    } finally {
        RpcContext.destroy();
    }
}
```
➊ 处理请求
➋ 包装成返回
➌ 响应请求

而 `messageHandler` 即是我们传入的 `ProviderMessageRouter`

```java
public Object handle(Channel channel, Object message) {
    Request request = (Request) message;
    String serviceKey = MotanFrameworkUtil.getServiceKey(request);
    Provider<?> provider = providers.get(serviceKey);
    Method method = provider.lookupMethod(request.getMethodName(), request.getParamtersDesc());
    fillParamDesc(request, method);
    processLazyDeserialize(request, method);
    return call(request, provider);
}
protected Response call(Request request, Provider<?> provider) {
    try {
        return provider.call(request);
    } catch (Exception e) {
        DefaultResponse response = new DefaultResponse();
        response.setException(new MotanBizException("provider call process error", e));
        return response;
    }
}
```
Call 的实现在 `Provider` 中

```java
@Override
public Response invoke(Request request) {
    DefaultResponse response = new DefaultResponse();
    Method method = lookupMethod(request.getMethodName(), request.getParamtersDesc());

    try {
        Object value = method.invoke(proxyImpl, request.getArguments());
        response.setValue(value);
    } catch (Exception e) {
       //SKIP
    } catch (Throwable t) {
        //SKIP
    }

    // 传递rpc版本和attachment信息方便不同rpc版本的codec使用。
    response.setRpcProtocolVersion(request.getRpcProtocolVersion());
    response.setAttachments(request.getAttachments());
    return response;
}
```
处理消息这段代码非常的清晰也就是标准的 `Netty` 的处理流程，我们获得请求的的数据。


# 模块详解
经过上文的的分析，我们已经对 `Motan` 的整体架构有了一些了解，我们下面的篇幅将对不同的具体模块进行分析。

## Registry 模块详解
![Registry Interface](https://s1.ax1x.com/2018/07/08/PmGFN8.png)

我们从图上可以看出来，`Registry` 提供的是 `注册` 和 `发现` 两个功能，废话不多说，我们去看看源码，因为 `Registry` 有多种实现，我挑选一个 `ZooKeeper` 的分析下。

```java
// com.weibo.api.motan.registry.support.AbstractRegistry#register
@Override
public void register(URL url) {
    doRegister(removeUnnecessaryParmas(url.createCopy())); ➊
    registeredServiceUrls.add(url);
    // available if heartbeat switcher already open
    if (MotanSwitcherUtil.isOpen(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER)) {
        available(url);
    }
}
```
➊ 这里将URL注册到ZK之上，然后再下一行将 URL 缓存在本地。
而我们这里的 `URL` 是 `Motan` 自定义的格式如 
`motan://192.168.31.26:8002/top.yannxia.java.demo.motan.MotanDemoService?group=motan-demo-rpc`
我们可以看出来它由 `协议部分(Motan)` `地址部分(192.168.31.26:8002)` `接口信息(top.yannxia.java.demo.motan.MotanDemoService)` `分组(group=motan-demo-rpc)` 构成

那秘密也就是在 `doRegister` 函数了

```java
protected void doRegister(URL url) {
    try {
        serverLock.lock();
        // 防止旧节点未正常注销
        removeNode(url, ZkNodeType.AVAILABLE_SERVER);
        removeNode(url, ZkNodeType.UNAVAILABLE_SERVER);
        createNode(url, ZkNodeType.UNAVAILABLE_SERVER); ➊ 
    } catch (Throwable e) {
        throw new MotanFrameworkException(String.format("Failed to register %s to zookeeper(%s), cause: %s", url, getUrl(), e.getMessage()), e);
    } finally {
        serverLock.unlock();
    }
}
```

➊ 前面2行也就是将原有的阶段删除，比如在异常的宕机过程，导致的数据问题。然后将这个URL在 `ZK`上创建一个新的节点。
```java
private void createNode(URL url, ZkNodeType nodeType) {
    String nodeTypePath = ZkUtils.toNodeTypePath(url, nodeType);
    if (!zkClient.exists(nodeTypePath)) {
        zkClient.createPersistent(nodeTypePath, true);
    }
    zkClient.createEphemeral(ZkUtils.toNodePath(url, nodeType), url.toFullStr()); ➊
}
```
➊ 这样也很简单，只是将节点的信息在 `ZK` 创建出一个新的节点出来。

但是 **问题** 来了，我们只是将自己的 `URL`注册到 `ZK` 上，我们怎么知道其他人注册的服务呢？

这个秘密就在 `Cliet` 调用过程中的 `discoverService()` 中
```java
// com.weibo.api.motan.registry.zookeeper.ZookeeperRegistry#discoverService
protected List<URL> discoverService(URL url) {
    try {
        String parentPath = ZkUtils.toNodeTypePath(url, ZkNodeType.AVAILABLE_SERVER);
        List<String> currentChilds = new ArrayList<String>();
        if (zkClient.exists(parentPath)) {
            currentChilds = zkClient.getChildren(parentPath); ➊
        }
        return nodeChildsToUrls(url, parentPath, currentChilds); ➋
    } catch (Throwable e) {
    }
}
```
也没有什么太过于神奇的地方 ➊ 在这里通过 `ZK` 树形结构，获得父亲节点，在 ➋ 获得真正的URL地址，也就是我们注册到 `ZK` 上的那个URL值，其实拿到的是我们的 `Object` 的储存值，形如 `motan://192.168.31.26:8002/top.yannxia.java.demo.motan.MotanDemoService?protocol=motan&application=motan&refreshTimestamp=1531145421643&id=motan&nodeType=service&export=motan:8002&version=1.0&group=motan-demo-rpc&` 其中还包含了一些服务的元数据。

## 结语
本来想着把 `Motan` 都解读一遍，但是整体上来说 `Motan` 是毕竟成熟（架构很清晰） 的一个框架，大家自行阅读起来难度也不是很大，大部分的类都很少 `继承` 的过深，整体来说，`Motan`既满足了拓展性的要求，代码量也少，比较容易掌握。如果要打分，满分 100 分的话，`Motan` 也可以得 80 分以上，是一个值得初期学习的 `RPC` 框架。

## 参考资料
- [用户手册](https://github.com/weibocom/motan/wiki/zh_userguide)


