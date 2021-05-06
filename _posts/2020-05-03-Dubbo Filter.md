---
layout:     post
title:      Dubbo Filter
subtitle:   Dubbo Filter
date:       2020-05-03
author:     LG
header-img: img/post-bg-mma-3.jpg
catalog: true
tags:
    - java
    - Dubbo
    - 分布式
    
---


## Dubbo Filter

Dubbo Filter 的入口是ProtocolFilterWrapper。ProtocolFilterWrapper 实现了 Protocol 接口。当调用ExtensionLoader的getExtension方法时，会做拦截处理，如果存在封装器，则返回封装器实现，而将真实实现通过构造方法注入到封装器中。举个简单的例子，当获取DubboProtocol的时候，会
先经过ProtocolFilterWrapper进行包装。


### 1. 源码分析

#### ProtocolFilterWrapper 实现了 Protocol 接口，是一个包装类。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gq90j3xn1nj315a0gqq74.jpg)



#### 生产者和消费者都会创建过滤器链

无论是生产者还是消费者都会创建过滤器链，看一下buildInvokerChain的源码。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gq90jh2xdzj30u01azkjl.jpg)

注意，在创建装饰者模式的时，Filter是倒序的。看一下debug的结果图。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gq90k0p7irj31db0e1417.jpg)


最终，消费者的链路为:

	ConsumerContextFilter -> FutureFilter -> MonitorFilter -> GenericImplFilter -> DubboInvoker

生产者的链路为:

	EchoFilter -> ClassloaderFilter -> GenericFilter -> ContextFilter -> TraceFilter -> TimeoutFilter -> MonitorFilter -> ExceptionFilter -> AbstractProxyInvoker


### 2. 总结

Dubbo可以通过装饰器模式增强Invoker功能, Dubbo Filter是Dubbo重要的调用拦截扩展。


### 3. 鸣谢

[Dubbo Filter](https://jishuin.proginn.com/p/763bfbd54b0b)