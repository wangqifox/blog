---
title: SpringCloud知识点整理
date: 2018/09/19 19:09:00
---

前段时间的项目是我第一次采用SpringCloud框架来开发，为此对SpringCloud的使用以及原理进行了一下学习，详见之间的SpringCloud系列的文章。温故而知新，这篇文章我们在之前文章的基础上将其核心的内容做一个整理。

<!-- more -->

# Eureka

`Eureka`是SpringCloud的核心组件，由3个角色`Eureka Server`、`Service Provider`、`Service Consumer`组成。下面分这三个角色来说明：

## Eureka Server

`Eureka Server`负责提供服务注册和发现。它的主要功能如下：

- 同步服务：

    `Eureka Server`启动之后从相邻的eureka节点获取注册表，将获取到的服务注册信息，注册到本地
    
- 服务注册：

    - 将注册信息保存在`ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry`结构中，第一层map的key是`app name`，也就是服务的应用名称，比如`EUREKA-CLIENT`，第二层map的key是`instance id`，也就是服务实例的id，比如`wangqideimac.lan:eureka-client:8762`。
    - 将服务注册信息复制到另外的`Eureka Server`节点上。

- 维护`Eureka Server`节点：

    `Eureka`会启动一个周期性任务（默认10min）来获取并更新`Eureka Server`节点
    
- 服务续约：

    - 如果续约的服务不存在，返回`false`让Client先为该服务进行注册。否则更新服务的`lastUpdateTimestamp`（最近续约时间）
    - 将服务续约信息复制到另外的`Eureka Server`节点上。

- 服务下线：

    - 从服务注册表中将相应的服务删除
    - 将服务下线信息复制到另外的`Eureka Server`节点上。
    - 重新计算`expectedNumberOfRenewsPerMin`和`numberOfRenewsPerMinThreshold`

- 服务获取

    - 检查本地缓存中是否存在服务，存在则从缓存中获取并返回服务信息
    - 如果缓存不存在，则根据请求的类型（所有服务、增量服务等）从注册表(`registry`)中获取相应的服务，存入缓存并返回服务信息

- 剔除失效服务

    `Eureka Server`会定时（默认60s）剔除失效的服务。失效的服务指的是超过一定时间（默认为90s）没有续约的服务。
    
    - 判断是否允许剔除失效服务：
    
        - 关闭了自我保护模式
        - 没有进入自我保护模式（`numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold`），即上一分钟的续约数量大于每分钟续约的阈值

    - 获取所有的过期服务
    - 分批随机剔除过期服务

## Service Provider

`Service Provider`是服务提供方，它将自身服务注册到`Eureka Server`，从而使服务消费方能够找到。

`Service Provider`的核心是启动时新建的几个定时任务以及服务下线的功能：

- 服务注册的定时任务

    服务注册的功能是更新本地的服务实例信息，并将本地的服务实例信息注册到`Eureka Server`中，其定时周期为30s
    
    - 刷新实例信息

        - 检查`Eureka Server`的`hostname`是否发生了修改
        - 检查`lease.duration`和`lease.renewalInterval`两个续约配置是否发生了修改
        - 获取并设置服务状态
    
    - 注册实例信息

        实例注册的过程就是向`Eureka Server`发送POST请求，将实例信息`InstanceInfo`发送给`Eureka Server`。url为`/apps/EUREKA-CLIENT`，其中`EUREKA-CLIENT`是服务实例的名称。

- 刷新服务列表缓存的定时任务

    刷新服务列表即向`Eureka Server`获取新的服务列表，更新本地的缓存。任务周期默认为30s
    
    - 重新获取`Eureka Server`的地址，因为这些配置有可能发生动态修改
    - 获取注册信息

        - 获取本地缓存的所有注册信息
        - 判断是否需要获取全量注册信息，根据结果获取全量注册信息或者增量注册信息

            如果满足：禁用了增量更新、强制全量更新，本地服务缓存为空等条件，则进行全量更新
            - 全量更新：向`Eureka Server`发送GET请求，url为`/apps`，将返回的数据生成`Applications`实例，保存在`DiscoveryClient.localRegionApps`变量中
            - 增量更新：向`Eureka Server`发送GET请求，url为`/apps/delta`，将返回的数据生成`Applications`实例 ，如果返回的增量数据为null，则获取全量数据。最后根据增量数据对更新本地缓存
        
        - 广播服务缓存刷新事件。比如`Ribbon`收到事件后会更新它保存的服务信息
        - 更新服务实例状态

- 服务续约的定时任务

    维持与`Eureka Server`的心跳，即向`Eureka Server`发送续约请求。任务周期默认为30s
    
    向`Eureka Server`发送PUT请求，请求地址为`/apps/{appName}/{id}`，完整示例为`/apps/EUREKA-CLIENT/wangqideimac.lan:eureka-client:8762?status=UP&lastDirtyTimestamp=1530096210220`，其中`EUREKA-CLIENT`表示服务名称，`wangqideimac.lan:eureka-client:8762`表示服务id，`status`表示服务的状态，`lastDirtyTimestamp`表示实例更新的时间。

- 服务下线

    服务下线一般在服务关闭的时候调用，用来把自身的服务从`Eureka Server`中删除，以防客户端调用不存在的服务。
    
    向`Eureka Server`发送DELETE请求，请求地址为`/apps/{appName}/{id}`，完整示例为`/apps/EUREKA-CLIENT/wangqideimac.lan:eureka-client:8762`，其中`EUREKA-CLIENT`表示服务名称，`wangqideimac.lan:eureka-client:8762`表示服务id。

## Service Consumer

`Service Consumer`是服务消费方，从`Eureka Server`中获取注册的服务列表，从而能够消费服务。

默认情况下`Eureka Client`会引入`Ribbon`的依赖，即服务消费默认使用的是`Ribbon`。

对于一个`Service Consumer`来说，它有两个主要的功能：

- 维护一个服务列表并能够及时更新
- 在服务调用时选择合适的服务实例

`ZoneAwareLoadBalancer`是`Ribbon`的核心，它能同时提供上面说的两个功能。

`ZoneAwareLoadBalancer`在初始化时会首先更新服务实例，然后开启一个更新服务实例的定时任务（默认周期30s）。服务实例是通过`EurekaClient`从`DiscoveryClient`中的`localRegionApps`获取，然后将它们保存在`BaseLoadBalancer`的`allServerList`。

`ZoneAwareLoadBalancer`还会开启一个更新实例状态的定时任务（默认周期10s）。遍历`allServerList`，选择状态为`UP`的服务实例保存在`upServerList`中。

从前面的原理说明中，我们看到`Service Consumer`其实也是一个`Service Provider`，但是它并不直接与服务注册中心打交道，而是通过读取并保存`Service Provider`中的实例信息来维护可以使用的示例信息。

默认情况下`ZoneAwareLoadBalancer`调用`ZoneAvoidanceRule`中的`choose`方法选择可用的服务实例，它从`allServerList`中循环选择一个服务。

# Hystrix



# Feign



# Zuul

















































