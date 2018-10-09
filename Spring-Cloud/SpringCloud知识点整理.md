---
title: SpringCloud知识点整理
date: 2018/09/19 19:10:00
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

    刷新服务列表即向`Eureka Server`获取新的服务列表，更新本地的缓存。任务周期默认为30s
    
    - 重新获取`Eureka Server`的地址，因为这些配置有可能发生动态修改
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

我们依赖的服务是通过远程调用方式执行的，因为网络原因或是依赖服务自身问题出现调用故障或延迟会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成任务积压，线程资源无法释放，最终导致自身服务的瘫痪，进一步甚至出现故障的蔓延最终导致整个系统的瘫痪。

`Hystrix`正是针对以上问题而设计的一种服务保护机制，它包含以下一系列功能：

- 服务降级。服务降级指的是命令执行失败之后转而执行回退逻辑的过程。命令执行失败包括以下几种情况：

    - 处理链路处于熔断状态
    - 以信号量作为隔离方式时，信号量获取失败
    - 以线程池作为隔离方式时，提交任务失败
    - 命令执行超时
    - 命令执行异常

- 服务熔断。熔断器是为了避免重复调用失效服务而设计的。默认情况下，当10s中内的请求数超过20次，且错误率超过50%，熔断器将从`闭路`状态转换成`开路`，这时后续所有经过该熔断器的请求直接走`失败回退逻辑`。经过一定时间（`休眠窗口`，默认为5s），后续第一个请求将会被允许通过熔断器（此时熔断器处于`半开`状态），若该请求失败，熔断器将又进入`开路`状态，且在休眠窗口内保持此状态；若该请求成功，熔断器将进入`闭路`状态，回到最开始的逻辑往复。
- 依赖隔离。`Hystrix`有两种隔离策略：线程池和信号量。通过使用线程池对不同依赖服务的隔离，某个高延迟的服务只会拖慢这个服务对应的线程池，而不会影响所有的服务。也可以使用信号量来限制单个依赖服务的并发度。
- 请求合并。请求合并可以减少线程和网络连接的数量，通过在`HystrixCommand`之前放置一个`请求合并器`，可以将多个发往同一个后端依赖服务的请求合并成一个。

# Feign

`Feign`是一个伪客户端，即它不做任务的请求处理。`Feign`通过处理注解生成request，从而实现简化HTTP API开发的目的，即开发人员可以使用注解的方式定制request api模板，在发送http request请求之前，`Feign`通过处理注解的方式替换掉request模板中的参数，这种实现方式显得更为直接、可理解。

`Feign`在启动过程中会扫描被`@FeignClient`注释的类，为这些类动态创建代理类。代理类的执行流程如下：

1. 根据请求参数生成`RequestTemplate`对象，该对象就是http请求的模板
2. 调用`LoadBalancerFeignClient`的`execute`方法发送请求，接收`Response`
3. 最后根据`Feign`方法的返回值来解码`Response`，即将请求返回的响应转换成需要的类型返回

`LoadBalancerFeignClient`有以下几个需要注意的点：

- 首先是`Client`组件，它用于发送request请求以及接收response响应。默认使用的网络框架是`HttpURLConnection`，也可以引入另外的依赖替换成`HttpClient`或者`OkHttp`。
- `execute`方法首先选择服务实例（其中的流程与Ribbon选择服务实例的步骤基本一致），然后调用`Client`的`execute`方法执行请求并返回响应。

# Zuul

`Zuul`是一个服务网关，通过它统一向外部提供REST API。服务网关具备服务路由、均衡负载、权限控制等功能。

`Zuul`的主体是一个`ZuulServlet`。在它执行`service`方法时，调用各个过滤器对请求进行处理，再将结果设置到response中返回。

过滤器分为4中类型：`Pre`、`Route`、`Post`、`Error`。其中`Pre`类型的过滤器负载前期处理，包括包装对象、添加调试信息、根据请求路径查询路由。`Route`类型的过滤器负责发送请求返回请求结果。`Post`类型的过滤器负责发送响应内容。`Error`类型的过滤器负责处理异常并产生错误响应。


