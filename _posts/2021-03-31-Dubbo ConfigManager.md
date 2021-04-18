---
layout:     post
title:      Dubbo ConfigManager
subtitle:   ConfigManager如何管理Dubbo config
date:       2021-03-31
author:     LG
header-img: img/post-bg-mma-3.jpg
catalog: true
tags:
    - java
    - Dubbo
    - 分布式
    
---



## Dubbo ConfigManager


### 1. ConfigManager概述

本文基于Dubbo 2.7.8 源码进行分析。

在Dubbo中，<dubbo: provier/>,  <dubbo:consumer/>, <dubbo:service/> 等dubbo 标签会被解析为 ProviderConfig, ConsumerConfig, ServiceBean 等。而 Dubbo 的 ConfigManager 正管理着这些config。那么下面来分析一些 ProvierConfig, ConsumerConfig等 config类是如何被加载到 ConfigManager中的呢。


### 2. 源代码分析


DubboBootstrap.exportServices 暴露所有服务。

![](https://tva1.sinaimg.cn/large/008eGmZEgy1gpnym250wkj31k40iytaz.jpg)

循环configManager.getServices()中的数据依次执行导出，那这些Service是从哪来的？
![](https://tva1.sinaimg.cn/large/008eGmZEgy1gpnynyfqvgj315w05kt96.jpg)

查看代码发现，这些ServiceConfigBase都是从ConfigManager.configsCache 获取的。
![](https://tva1.sinaimg.cn/large/008eGmZEgy1gpnyojz6j5j31oi06a0tk.jpg)


那么这些ServiceConfigBase是如何存储到 configsCache 中的呢？因为ServiceBean是ServiceConfigBase的一个子类。所以猜测在ServiceBean初始化完成以后，就会被放到缓存中。接下来看一下ServiceBean的类图关系。
![](https://tva1.sinaimg.cn/large/008eGmZEgy1gpnyovg7uaj31so0s8q3v.jpg)

再来看ServiceConfigBase的父类AbstractConfig，它里面有一个PostConstruct注解的方法:addIntoConfigManager，查看官方解释：PostConstruct 注释用于在依赖关系注入完成之后需要执行的方法上，以执行任何初始化。 此方法必须在将类放入服务之前调用。 支持依赖关系注入的所有类都必须支持此注释。 即使类没有请求注入任何资源，用PostConstruct 注释的方法也必须被调用。也就是说所有的ServiceBean在被Spring初始化之后都会调用addIntoConfigManager方法。addIntoConfigManager方法的实际作用就是向configsCache中缓存。

![](https://tva1.sinaimg.cn/large/008eGmZEgy1gpnyplth90j31mo0kojt7.jpg)


所以，Dubbo中凡是实现AbstractConfig接口的类，在被Spring初始化之后，最终都会被缓存到configManager中。



### 3. 总结

Dubbo中实现AbstractConfig接口的Bean，在被Spring初始化之后，都会被放入configManager。

### 4. 鸣谢

[从源码分析Dubbo与SpringBoot整合之Dubbo启动](https://my.oschina.net/qiantu/blog/4867893)