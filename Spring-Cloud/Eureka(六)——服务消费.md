---
title: Eureka(六)——服务消费
date: 2018/06/29 09:30:25
---

上一篇文章[Eureka(五)——高可用][1]中，我们介绍了如何构建高可用的服务注册中心和服务提供。

本文我们来看看如何去消费服务提供者的接口。

<!-- more -->

# 使用LoadBalancerClient

在Spring Cloud Commons中提供了大量的与服务治理相关的抽象接口，包括`DiscoveryClient`、这里我们即将介绍的`LoadBalancerClient`等。Spring Cloud做这一层抽象，很好的解耦了服务治理体系，使得我们可以轻易的替换不同的服务治理设施。

从`LoadBanacerClient`接口的命名中，我们就知道这是一个负载均衡客户端的抽象定义，下面我们就看看如何使用Spring Cloud提供的负载均衡器客户端接口来实现服务的消费。

首先创建一个服务消费者工程，命名为：`eureka-consumer`。并在`pom.xml`中引入依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

配置`application.yml`，指定eureka注册中心的地址：

```
spring:
  application:
    name: eureka-consumer

server:
  port: 8763

eureka:
  client:
    serviceUrl:
      defaultZone: http://peer1:8751/eureka/,http://peer2:8752/eureka/,http://peer3:8753/eureka/

```

创建应用主类。初始化`RestTemplate`，用来真正发起REST请求。

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumerApplication {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumerApplication.class, args);
    }
}
```

创建一个接口用来消费eureka-client-multi提供的接口：

```java
@RestController
public class DcController {
    @Autowired
    LoadBalancerClient loadBalancerClient;
    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/consumer")
    public String dc() {
        ServiceInstance serviceInstance = loadBalancerClient.choose("eureka-client-multi");
        String url = "http://" + serviceInstance.getHost() + ":" + serviceInstance.getPort() + "/dc";
        System.out.println(url);
        return restTemplate.getForObject(url, String.class);
    }
}
```

我们注入了`LoadBalancerClient`和`RestTemplate`，并在`/consumer`接口的实现中，先通过`loadBalancerClient`的`choose`函数来负载均衡地选出一个`eureka-client`的服务实例，这个服务实例的基本信息存储在`ServiceInstance`中，然后通过这些对象中的信息拼接出访问`/dc`接口的详细地址，最后再利用`RestTemplate`对象实现对服务提供者接口的调用。

启动eureka-consumer，这时Eureka Server的dashboard如下所示，我们看到eureka-consumer也被注册到Server中。

![eureka-consume](media/eureka-consumer.png)

访问eureka-consumer的`/consumer`接口：`http://127.0.0.1:8763/consumer`，返回`eureka-client-multi`服务的数据：

![eureka-consumer-result](media/eureka-consumer-result.png)

查看eureka-consumer输出的日志我们会发现，`loadBalancerClient`在提供服务的三个地址之间切换：

```
http://wangqideimac.lan:8741/dc
http://wangqideimac.lan:8742/dc
http://wangqideimac.lan:8743/dc
```

# LoadBalancerClient的原理

上文我们看到eureka-consumer能够成功以负载均衡的方式访问eureka-client-multi服务，下面我们来看看`LoadBalancerClient`的原理。

## RibbonLoadBalancerClient

首先我们来看看这里`LoadBalancerClient`的实现类是什么，通过加断点的方式发现它的实现类是`RibbonLoadBalancerClient`。为什么？我们明明没有引入ribbon相关的依赖。

仔细看`spring-cloud-starter-netflix-eureka-client`，发现它引入了ribbon的依赖：

![eureka-consumer-ribbon](media/eureka-consumer-ribbon.png)

进入`spring-cloud-starter-netflix-ribbon`包，其中`spring.factories`文件中定义了`EnableAutoConfiguration`：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

`RibbonAutoConfiguration`是ribbon的配置类，其中定义了`LoadBalancerClient`的实例为`RibbonLoadBalancerClient`：

```java
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
    return new RibbonLoadBalancerClient(springClientFactory());
}
```

下图是`RibbonLoadBalancerClient`类的继承关系：

![RibbonLoadBalancerClient-structure](media/RibbonLoadBalancerClient-structure.png)

`RibbonLoadBalancerClient`实现了两个接口`LoadBalancerClient`和`ServiceInstanceChooser`。

`LoadBalancerClient`接口有三个方法，其中`execute()`为执行请求，`reconstructURI()`用来重构url：

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
    <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
    URI reconstructURI(ServiceInstance instance, URI original);
}
```

`ServiceInstanceChooser`接口，主要有一个方法，用来根据serviceId来获取`ServiceInstance`，代码如下：

```java
public interface ServiceInstanceChooser {
    ServiceInstance choose(String serviceId);
}
```

## RibbonLoadBalancerClient.choose

上文我们知道了`LoadBalancerClient`的实现类是`RibbonLoadBalancerClient`，这个类是非常重要的一个类，最终的负载均衡由它来执行。

现在我们来看看它是如何选择具体服务实例并获取到服务信息的。

```java
public ServiceInstance choose(String serviceId) {
	Server server = getServer(serviceId);
	if (server == null) {
		return null;
	}
	return new RibbonServer(serviceId, server, isSecure(server, serviceId),
			serverIntrospector(serviceId).getMetadata(server));
}
```

`choose`方法调用`getServer`去获取实例：

```java
protected Server getServer(String serviceId) {
	return getServer(getLoadBalancer(serviceId));
}
```

1. 调用`getLoadBalancer()`方法从Spring Context中获取`ILoadBalancer`的实例
2. 然后在`getServer()`方法中调用这个`ILoadBalancer`实例的`chooseServer`方法选择服务实例。传入的key固定为"default"。

### ILoadBalancer

`ILoadBalancer`在ribbon-loadbalancer包下，它是定义了实现软件负载均衡的一个接口，它需要一组可供选择的服务注册列表信息，以及根据特定方法区选择服务：

```java
public interface ILoadBalancer {
    // 添加一个Server集合
    public void addServers(List<Server> newServers);
    // 根据key去获取Server
    public Server chooseServer(Object key);
    // 标记某个服务下线
    public void markServerDown(Server server);
    // 获取可用的Server集合
    public List<Server> getReachableServers();
    // 获取所有的Server集合
    public List<Server> getAllServers();
}
```

`RibbonLoadBalancerClient.getLoadBalancer()`方法获取的`ILoadBalancer`实例类为`ZoneAwareLoadBalancer`。其继承关系如下：

![ZoneAwareLoadBalancer-structure](media/ZoneAwareLoadBalancer-structure.png)

#### chooseServer

`ZoneAwareLoadBalancer`的`chooseServer`方法代码如下：

```java
public Server chooseServer(Object key) {
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        }

        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        }
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                server = zoneLoadBalancer.chooseServer(key);
            }
        }
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    }
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    }
}
```

调用`getLoadBalancerStats()`方法获取`LoadBalancerStats`，`LoadBalancerStats`类保存了LoadBalancer每个节点的操作特征和统计信息。然后调用`getAvailableZones`方法获取可用的Zone，

根据可用Zone的不同，`chooseServer`方法选择不同的执行流程。本地中我们配置了`defaultZone: http://peer1:8751/eureka/,http://peer2:8752/eureka/,http://peer3:8753/eureka/`，因此可用的Zone只有一个。我们先看看可用Zone只有一个的情况。

##### 可用Zone只有一个

如果可用Zone只有一个，调用父类`BaseLoadBalancer`的`chooseServer`方法。

`chooseServer`方法中调用`IRule`的`choose`方法选择一个Server。

IRule用于复杂均衡的策略，它有三个方法：

```java
public interface IRule{
    public Server choose(Object key);
    public void setLoadBalancer(ILoadBalancer lb);
    public ILoadBalancer getLoadBalancer();    
}
```

其中`choose()`是根据key来获取server，`setLoadBalancer()`和`getLoadBalancer()`是用来设置和获取ILoadBalancer的。

IRule有很多默认的实现类，这些实现类根据不同的算法和逻辑来处理负载均衡。Ribbon实现的IRule如下图所示：

![IRule-structure](media/IRule-structure.png)

1. BeastAvailableRule
2. ClientConfigEnableRoundRobinRule
2. AvailabilityFilteringRule
3. ZoneAvoidanceRule
4. RoundRobinRule
4. WeightedResponseTimeRule
5. RetryRule
6. RandomRule

###### ZoneAvoidanceRule

这里默认选择的`ZoneAvoidanceRule`，其`choose`方法如下：

```java
public Server choose(Object key) {
    ILoadBalancer lb = getLoadBalancer();
    Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    if (server.isPresent()) {
        return server.get();
    } else {
        return null;
    }       
}
```

1. 首先调用`getPredicate()`方法获取`CompositePredicate`实例
2. 然后调用`CompositePredicate`的父类`AbstractServerPredicate`的`chooseRoundRobinAfterFiltering`方法

`chooseRoundRobinAfterFiltering`方法如下：

```java
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
```

1. 调用`getEligibleServers`方法获取适合的Server列表，判断是否合适的方法为`ZoneAvoidanceRule.apply`方法
2. 然后调用`incrementAndGetModulo`方法获取Server的index，该方法就是在modulo范围内轮询
3. 最后通过index获取Server

##### 可用Zone大于一个

修改eureka-consumer的`application.yml`：

```java
spring:
  application:
    name: eureka-consumer
  profiles:
      active: consumer1

server:
  port: 8761

eureka:
  instance:
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${server.port}
  client:
    availability-zones:
      region-1: zone1,zone2,zone3
    service-url:
      zone1: http://peer1:8751/eureka/
      zone2: http://peer2:8752/eureka/
      zone3: http://peer3:8753/eureka/
    region: region-1

---
spring:
  profiles: consumer1

server:
  port: 8761

eureka:
  instance:
    metadata-map.zone: zone1,zone2,zone3

---
spring:
  profiles: consumer2

server:
  port: 8762

eureka:
  instance:
    metadata-map.zone: zone1,zone2,zone3

---
spring:
  profiles: consumer3

server:
  port: 8763

eureka:
  instance:
    metadata-map.zone: zone1,zone2,zone3
```

如果可用Zone大于一个，流程如下：

1. 调用`getLoadBalancerStats()`方法获取`LoadBalancerStats`
2. 调用`ZoneAvoidanceRule.createSnapsot`保存关于zone的快照
3. 调用`ZoneAvoidance.getAvailableZones`返回可用的Zones
4. 如果可用的Zones数量小于刚才zone快照里保存的数量

    1. 调用`ZoneAvoidanceRule.randomChooseZone`从可用的Zones中随机选择一个zone
    2. 根据选择的zone，调用`getLoadBalancer`方法获取`BaseLoadBalancer`
    3. 调用`BaseLoadBalancer.chooseServer`方法选择服务实例
  
5. 否则，和可用Zone只有一个的情况一样，调用父类`BaseLoadBalancer`的`chooseServer`方法。




















[1]: /articles/Spring-Cloud/Eureka(五)——高可用.html

> http://blog.didispace.com/spring-cloud-starter-dalston-2-1/
> https://blog.csdn.net/forezp/article/details/74820899

