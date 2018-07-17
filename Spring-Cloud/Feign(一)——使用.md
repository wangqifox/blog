---
title: Feign(一)——使用
date: 2018/07/17 10:20:25
---

Feign是一个声明式的Web Service客户端，它使得编写Web Service客户端变得更加简单。我们只需要使用Feign来创建一个接口并用注解来配置它即可完成。它具备可插拔的注解支持，包括Feign注解和JAX-RX注解。Feign也支持可插拔的编码器和解码器。Spring Cloud为Feign增加了对Spring MVC注解的支持，还整合了Ribbon和Eureka来提供均衡负载的HTTP客户端实现。
<!-- more -->
下面，通过一个例子来展现Feign如何方便地声明对服务的定义和调用。

1. 配置`pom.xml`

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
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

2. 在应用主类中通过`@EnableFeignClients`注解开启Feign功能

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumerFeignApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumerFeignApplication.class, args);
    }
}
```

3. 定义服务的接口

```java
@FeignClient("eureka-client")
public interface ComputeClient {
    @RequestMapping(value = "/add", method = RequestMethod.GET)
    Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b);
}
```

- 使用`@FeignClient("eureka-client")`注解来绑定该接口对应的服务
- 通过Spring MVC的注解来配置`eureka-client`服务下的具体实现

4. 在controller层调用上面定义的`ComputeClient`

```java
@RestController
public class ComputeController {
    @Autowired
    ComputeClient computeClient;

    @RequestMapping(value = "/add", method = RequestMethod.GET)
    public Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b) {
        return computeClient.add(a + 1, b + 1);
    }
}
```

5. 配置`application.yml`

```
spring:
  application:
    name: eureka-consumer-feign

server:
  port: 8765

eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8761/eureka/
```

访问[http://localhost:8765/add?a=1&b=5](http://localhost:8765/add?a=1&b=5)，可以得到结果`6`。





> http://blog.didispace.com/springcloud2/


