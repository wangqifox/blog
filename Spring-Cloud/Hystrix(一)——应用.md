---
title: Hystrix(一)——应用
date: 2018/07/05 14:25:25
---

在微服务架构中，我们将系统拆分成了一个个的服务单元，各单元应用间通过服务注册与订阅的方式互相依赖。由于每个单元都在不同的进程中运行，依赖通过远程调用的方式执行，这样就有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成任务积压，线程资源无法释放，最终导致自身服务的瘫痪，进一步甚至出现故障的蔓延最终导致整个系统的瘫痪。如果这样的架构存在如此严重的隐患，那么相较于传统架构就更加的不稳定。为了解决这样的问题，产生了断路器等一系列的服务保护机制。

针对上述问题，在Spring Cloud Hystrix中实现了线程隔离、断路器等一系列的服务保护功能。它也是基于Netflix的开源框架Hystrix实现的，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大功能。
<!-- more -->
# 应用

在`pom.xml`中引入`spring-cloud-starter-netflix-hystrix`依赖：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

在应用主类中使用`@EnableCircuitBreaker`或`@EnableHystrix`注解开启Hystrix的使用：

```java
@EnableCircuitBreaker
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumerRibbonHystrixApplication {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumerRibbonHystrixApplication.class, args);
    }
}
```

注意：这里我们还可以使用Spring Cloud应用中的`@SpringCloudApplication`注解来修饰应用主类，该注解的具体定义如下所示。我们可以看到该注解中包含了我们所引用的三个注解，这也意味着一个Spring Cloud标准应用应包含服务发现以及断路器。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```

新建服务消费类。新增`ComsumerService`类，然后将在`Controller`中的逻辑迁移过去。最后，在为具体执行逻辑的函数上增加`@HystrixCommand`注解来指定服务降级方法。

```java
@RestController
public class DcController {
    @Autowired
    ConsumerService consumerService;

    @GetMapping("/consumer")
    public String dc() {
        return consumerService.consumer();
    }

    @Service
    class ConsumerService {
        @Autowired
        RestTemplate restTemplate;

        @HystrixCommand(fallbackMethod = "fallback")
        public String consumer() {
            return restTemplate.getForObject("http://eureka-client/dc", String.class);
        }

        public String fallback() {
            return "fallback";
        }
    }
}
```

启动消费服务，访问`http://localhost:8765/consumer`，可以获取正常的返回。

为了触发服务降级逻辑，我们可以将服务提供者`eureka-client`的逻辑增加一些延迟，比如：

```java
@GetMapping("/dc")
public String dc() throws InterruptedException {
    Thread.sleep(5000);
    String services = "Services: " + discoveryClient.getServices();
    System.out.println(services);
    return services;
}
```

重启`eureka-client`之后，再尝试访问`http://localhost:8765/consumer`，此时我们将获得的返回结果为：`fallback`。我们从`eureka-client`的控制台中，可以看到服务提供方输出了原本要返回的结果，但由于返回前延迟了5秒，而服务消费方触发了服务请求超时异常，服务消费者就通过`HystrixCommand`注解中指定的降级逻辑进行执行，因此该请求的结果返回了`fallback`。这样的机制，对自身服务起到了基础的保护，同时还为异常情况提供了自动的服务降级切换机制。

# 整合ribbon的请求重试

为了防止请求到失效的服务导致触发Hystrix的fallback，我们希望请求服务时能够有加入重试机制，关于ribbon的请求重试详见[Eureka(七)——ribbon请求重试][1]。

首先在`pom.xml`中引入`spring-retry`依赖

```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

`application.yml`中加入服务重试的配置：

```
eureka-client-multi:
  ribbon:
    OkToRetryOnAllOperations: true
    MaxAutoRetriesNextServer: 3
    MaxAutoRetries: 1
```

`RestTemplate`新建时加入超时配置：

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    SimpleClientHttpRequestFactory simpleClientHttpRequestFactory = new SimpleClientHttpRequestFactory();
    simpleClientHttpRequestFactory.setConnectTimeout(1000);
    simpleClientHttpRequestFactory.setReadTimeout(1000);
    return new RestTemplate(simpleClientHttpRequestFactory);
}
```

为了确保Ribbon重试的时候不被熔断，我们需要让Hystrix的超时时间大于Ribbon的超时时间，否则Hystrix命令超时后，该命令直接熔断，重试机制就没有意义了。在`application.yml`中加入Hystrix的超时配置：

```
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            timeoutInMilliseconds: 6000
```

ribbon的超时时间为1000，请求超时后，该实例会重试一次，更新实例会重试3次。

Hystrix的超时时间要大于`(1 + MaxAutoRetries + MaxAutoRetriesNextServer) * ReadTimeout`。具体要看需求进行配置。




[1]: /articles/Spring-Cloud/Eureka(七)——ribbon请求重试.html


> http://blog.didispace.com/spring-cloud-starter-dalston-4-1/
> https://blog.csdn.net/akaks0/article/details/80040181

