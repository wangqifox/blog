---
title: Feign(三)——与Hystrix的整合
date: 2018/07/17 17:29:25
---

Feign提供了Hystrix的支持，只需要简单的几个配置，就可以实现与Hystrix的整合。

<!-- more -->

# 与Hystrix的整合

在前文示例的基础上，我们来加入Hystrix的支持。

1. 在`pom.xml`文件中加入Hystrix的依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

2. 在`application.yml`文件中打开Feign的Hystrix的开关

```
feign:
  hystrix:
    enabled: true
```

3. 在应用启动类中加入`@EnableCircuitBreaker`注释，开启hystrix

```java
@EnableCircuitBreaker
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaConsumerFeignApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaConsumerFeignApplication.class, args);
    }
}
```

4. 在`ComputeClient`中加入服务调用失败的回退

```java
@FeignClient(value = "eureka-client", fallback = ComputeClient.Fallback.class)
public interface ComputeClient {
    @RequestMapping(value = "/add", method = RequestMethod.GET)
    Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b);

    @RequestMapping(value = "/minus", method = RequestMethod.GET)
    Integer minus(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b);

    @Component
    class Fallback implements ComputeClient {

        @Override
        public Integer add(Integer a, Integer b) {
            return -1;
        }

        @Override
        public Integer minus(Integer a, Integer b) {
            return -1;
        }
    }
}
```

我们加入了一个minus的服务调用，因为这个服务不存在，因此对它的调用总是会进入回退逻辑中。我们以此来判断Hystrix是否生效。

5. 在`ComputeController`中增加对minus服务的调用

```java
@RestController
public class ComputeController {
    @Autowired
    ComputeClient computeClient;

    @RequestMapping(value = "/add", method = RequestMethod.GET)
    public Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b) {
        return computeClient.add(a, b);
    }

    @RequestMapping(value = "/minus", method = RequestMethod.GET)
    public Integer minus(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b) {
        return computeClient.minus(a, b);
    }
}
```

访问`http://localhost:8765/minus?a=5&b=1`，发现结果返回`-1`，说明Hystrix已经生效了。

# 超时设置

为了测试Hystrix的超时效果，我们将add服务延时800ms，然后配置Hystrix的超时时间：

```
hystrix:
  command:
    ComputeClient#add(Integer,Integer):
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 500
```

这里我们设置了add服务的超时时间。

Feign与Hystrix整合使用时，会自动帮我们生成CommandKey，格式为：`Feign客户端接口名#方法名(参数类型)`。例如本例中的客户端为ComputeClient，方法为add，参数为两个Integer，生成的CommandKey为`ComputeClient#add(Integer,Integer)`。

如果要针对全局做配置，则需要使用：

```
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 500
```

经过上面的配置，我们知道add()方法总是会超时，从而调用失败回退逻辑。

# 断路器状态

默认情况下，10s内请求超过20次，且失败率超过50%，断路器将被打开。我们来看看是否是这样的。

控制器方法中获取add方法的断路器，并输出其状态。

```java
@RequestMapping(value = "/add", method = RequestMethod.GET)
public Integer add(@RequestParam(value = "a") Integer a, @RequestParam(value = "b") Integer b) {
    Integer result = computeClient.add(a, b);

    HystrixCircuitBreaker breaker = HystrixCircuitBreaker.Factory
            .getInstance(HystrixCommandKey.Factory.asKey("ComputeClient#add(Integer,Integer)"));

    System.out.println("断路器状态：" + breaker.isOpen());
    return result;
}
```

接下来，编写一个测试客户端，多线程访问控制器方法：

```java
@Test
public void add() throws InterruptedException {
    final CloseableHttpClient httpClient = HttpClients.createDefault();
    for (int i = 0; i < 100; i++) {
        Thread t = new Thread() {
            @Override
            public void run() {
                try {
                    String url = "http://localhost:8765/add?a=1&b=5";
                    HttpGet httpGet = new HttpGet(url);
                    HttpResponse response = httpClient.execute(httpGet);
                    System.out.println(EntityUtils.toString(response.getEntity()));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        };
        t.start();
    }
    Thread.sleep(10000);
}
```

执行后可以看到服务消费者控制台输出如下：

```
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：false
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
断路器状态：true
...
```

可以看到，经过20次调用失败后，断路器打开。

# 重试

Feign的重试依赖于Ribbon，因此只需要加入如下Ribbon的配置就可以实现重试：

```
eureka-client:
  ribbon:
    ConnectTimeout: 250
    ReadTimeout: 250
    OkToRetryOnAllOperations: true
    MaxAutoRetriesNextServer: 3
    MaxAutoRetries: 1
```

如果要进行全局配置，加入如下配置：

```
ribbon:
  ConnectTimeout: 250
  ReadTimeout: 250
  OkToRetryOnAllOperations: true
  MaxAutoRetriesNextServer: 3
  MaxAutoRetries: 1
```






















> https://my.oschina.net/JavaLaw/blog/1575850

