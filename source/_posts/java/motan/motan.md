---
title: Motan源码分析
date: 2018-06-30 13:25:48
tags:
toc: true
categories: ["java", "RPC", 'Motan']
---

# Motan
`Motan` 是一套基于java开发的RPC框架，除了常规的点对点调用外，`Motan` 还提供服务治理功能，包括服务节点的自动发现、摘除、高可用和负载均衡等。`Motan` 具有良好的扩展性，主要模块都提供了多种不同的实现，例如支持多种注册中心，支持多种rpc协议等。（可以看作 `Dubbo` 的复刻版）

[跳过介绍]()
<!-- more -->

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


## 启动流程分析
我们从上面的代码可以看出来，在服务侧的代码中，大部分都是在声明，在 `motanDemoService.export();`这部才是开始执行暴露出 `RPC` 接口的行为，我们从这里作为一个突破口。

```java
public synchronized void export() {
    checkInterfaceAndMethods(interfaceClass, methods);

    List<URL> registryUrls = loadRegistryUrls();
    if (registryUrls == null || registryUrls.size() == 0) {
        throw new IllegalStateException("Should set registry config for service:" + interfaceClass.getName());
    }

    Map<String, Integer> protocolPorts = getProtocolAndPort();
    for (ProtocolConfig protocolConfig : protocols) {
        Integer port = protocolPorts.get(protocolConfig.getId());
        if (port == null) {
            throw new MotanServiceException(String.format("Unknow port in service:%s, protocol:%s", interfaceClass.getName(),
                    protocolConfig.getId()));
        }
        doExport(protocolConfig, port, registryUrls);
    }

    afterExport();
}
```
