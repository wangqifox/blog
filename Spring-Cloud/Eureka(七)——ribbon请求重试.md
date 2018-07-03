---
title: Eureka(七)——ribbon请求重试
date: 2018/07/02 08:44:25
---

上一篇文章[Eureka(六)——服务消费][1]的例子中，实现了对服务名为`eureka-client`的`/dc`接口的调用。由于`RestTemplate`被`@LoadBalanced`修饰，所以它具备客户端负载均衡的能力，当请求真正发起的时候，url中的服务名会根据负载均衡策略从服务清单中挑选出一个实例来进行访问。
<!-- more -->

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

@Autowired
RestTemplate restTemplate;

@GetMapping("/consumer")
public String dc() {
    return restTemplate.getForObject("http://eureka-client/dc", String.class);
}
```

大多数情况下，上面的例子没有任何问题，但是总有一些意外发生，比如：有一个实例发生了故障而该情况还没有被服务治理机制及时发现和清除，这时候客户端访问该节点的时候自然会失败。所以，为了构件更为健壮的应用系统，我们希望当请求失败的时候能够有一定策略的重试机制，而不是直接返回失败。

# 重试机制实现

在`application.yml`中加入以下配置：

```
spring:
  cloud:
    loadbalancer:
      retry:
        enabled: true   #开启重试机制
        
eureka-client:
  ribbon:
    ConnectTimeout: 250             #请求连接的超时时间
    ReadTimeout: 250                #请求处理的超时时间
    OkToRetryOnAllOperations: true  #对所有操作请求都进行重试
    MaxAutoRetriesNextServer: 3     #切换实例的重试次数
    MaxAutoRetries: 1               #对当前实例的重试次数
```

pom.xml中引入spring-retry包：

```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

很多文章中都值说了上面的两步，实测发现无法实现超时重试机制，因为上面的配置不会影响到`RestTemplate`的超时时间，因此会一直等待服务返回，而不会重新尝试连接另外的服务实例。

如果要对`RestTemplate`设置超时时间，我们需要使用如下方式设置：

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

根据如上配置，当访问到故障请求的时候，它会再尝试访问一次当前实例（次数由`MaxAutoRetries`配置），如果不行，就换一个实例进行访问，如果还是不行，再换一次实例访问（更好次数由`MaxAutoRetriesNextServer`配置），如果依然不行，返回失败信息。

# 重试机制原理

支持重试机制的拦截器在`LoadBalancerAutoConfiguration`中定义：

```java
@Configuration
@ConditionalOnClass(RetryTemplate.class)
public static class RetryInterceptorAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public RetryLoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient, LoadBalancerRetryProperties properties,
            LoadBalancerRequestFactory requestFactory,
            LoadBalancedRetryFactory loadBalancedRetryFactory) {
        return new RetryLoadBalancerInterceptor(loadBalancerClient, properties,
                requestFactory, loadBalancedRetryFactory);
    }

    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(
            final RetryLoadBalancerInterceptor loadBalancerInterceptor) {
        return restTemplate -> {
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                    restTemplate.getInterceptors());
            list.add(loadBalancerInterceptor);
            restTemplate.setInterceptors(list);
        };
    }
}
```

因为我们引入了spring-retry包，存在`RetryTemplate`类，因此会初始化`RetryInterceptorAutoConfiguration`类中的`RetryLoadBalancerInterceptor`和`RestTemplateCustomizer`实例。

重试机制的拦截功能在`RetryLoadBalancerInterceptor.intercept`方法中实现。它创建`RetryTemplate`，然后调用其`execute`方法。

## RetryTemplate

`RetryTemplate`是实现重试机制的模板类。`execute`方法调用`doExecute`方法，`doExecute`方法中实现重试机制。主要代码如下：

```java
protected <T, E extends Throwable> T doExecute(RetryCallback<T, E> retryCallback,
            RecoveryCallback<T> recoveryCallback, RetryState state)
            throws E, ExhaustedRetryException {
    ...
    while (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {

        try {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Retry: count=" + context.getRetryCount());
            }
            // Reset the last exception, so if we are successful
            // the close interceptors will not think we failed...
            lastException = null;
            return retryCallback.doWithRetry(context);
        }
        catch (Throwable e) {

            lastException = e;

            try {
                registerThrowable(retryPolicy, state, context, e);
            }
            catch (Exception ex) {
                throw new TerminatedRetryException("Could not register throwable",
                        ex);
            }
            finally {
                doOnErrorInterceptors(retryCallback, context, e);
            }

            if (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
                try {
                    backOffPolicy.backOff(backOffContext);
                }
                catch (BackOffInterruptedException ex) {
                    lastException = e;
                    // back off was prevented by another thread - fail the retry
                    if (this.logger.isDebugEnabled()) {
                        this.logger
                                .debug("Abort retry because interrupted: count="
                                        + context.getRetryCount());
                    }
                    throw ex;
                }
            }

            if (this.logger.isDebugEnabled()) {
                this.logger.debug(
                        "Checking for rethrow: count=" + context.getRetryCount());
            }

            if (shouldRethrow(retryPolicy, context, state)) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Rethrow in retry for policy: count="
                            + context.getRetryCount());
                }
                throw RetryTemplate.<E>wrapIfNecessary(e);
            }

        }

        /*
         * A stateful attempt that can retry may rethrow the exception before now,
         * but if we get this far in a stateful retry there's a reason for it,
         * like a circuit breaker or a rollback classifier.
         */
        if (state != null && context.hasAttribute(GLOBAL_STATE)) {
            break;
        }
    }
    ...
}
```

`doExecute`方法主体是一个循环，循环的判断是调用`canRetry`方法，`canRetry`方法实际调用`RetryPolicy`的`canRetry`方法。

### canRetry

此处`RetryPolicy`的实现类是`InterceptorRetryPolicy`，它的`canRetry`方法如下：

```java
public boolean canRetry(RetryContext context) {
    LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext)context;
    if(lbContext.getRetryCount() == 0  && lbContext.getServiceInstance() == null) {
        //We haven't even tried to make the request yet so return true so we do
        lbContext.setServiceInstance(serviceInstanceChooser.choose(serviceName));
        return true;
    }
    return policy.canRetryNextServer(lbContext);
}
```

- 如果`if`判断为真，说明当前还没有作任何请求。于是选择一个服务实例，并返回`true`。
- 否则，说明上一次的请求发生错误，调用`LoadBalancedRetryPolicy`的`canRetryNextServer`来判断是否需要尝试下一个服务实例

`LoadBalancedRetryPolicy`的实例是`RibbonLoadBalancedRetryPolicy`，它的`canRetryNextServer`方法如下所示：

```java
public boolean canRetryNextServer(LoadBalancedRetryContext context) {
    return nextServerCount <= lbContext.getRetryHandler().getMaxRetriesOnNextServer() && canRetry(context);
}

public boolean canRetry(LoadBalancedRetryContext context) {
    HttpMethod method = context.getRequest().getMethod();
    return HttpMethod.GET == method || lbContext.isOkToRetryOnAllOperations();
}
```

`canRetryNextServer`方法判断`nextServerCount`是否小于我们配置的`MaxAutoRetriesNextServer`，并且`canRetry`方法是否返回`true`。

如果请求方法是`GET`，或者我们配置`OkToRetryOnAllOperations`为`true`，`canRetry`方法返回`true`。

### doWithRetry

回到`RetryTemplate.doExecute`方法，进入while循环体之后，调用`retryCallback.doWithRetry`方法。该方法在`RetryLoadBalancerInterceptor.intercept`方法中定义，代码如下：

```java
ServiceInstance serviceInstance = null;
if (context instanceof LoadBalancedRetryContext) {
    LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext) context;
    serviceInstance = lbContext.getServiceInstance();
}
if (serviceInstance == null) {
    serviceInstance = loadBalancer.choose(serviceName);
}
ClientHttpResponse response = RetryLoadBalancerInterceptor.this.loadBalancer.execute(
        serviceName, serviceInstance,
        requestFactory.createRequest(request, body, execution));
int statusCode = response.getRawStatusCode();
if (retryPolicy != null && retryPolicy.retryableStatusCode(statusCode)) {
    byte[] bodyCopy = StreamUtils.copyToByteArray(response.getBody());
    response.close();
    throw new ClientHttpResponseStatusCodeException(serviceName, response, bodyCopy);
}
return response;
```

可以看到，`doWithRetry`方法调用`RibbonLoadBalancerClient`的`execute`方法向服务发生请求。如果请求发生异常，`execute`方法会抛出这个异常。

### 处理请求异常

当`RetryTemplate.doWithRetry`方法抛出异常，`RetryTemplate.doExecute`方法中会捕获这个异常，然后进行一系列处理，包括调用`registerThrowable`方法注册异常、判断是否重新抛出异常等等。

其中`registerThrowable`方法最终会调用`RibbonLoadBalancedRetryPolicy`方法：

```java
public void registerThrowable(LoadBalancedRetryContext context, Throwable throwable) {
    //if this is a circuit tripping exception then notify the load balancer
    if (lbContext.getRetryHandler().isCircuitTrippingException(throwable)) {
        updateServerInstanceStats(context);
    }
    
    //Check if we need to ask the load balancer for a new server.
    //Do this before we increment the counters because the first call to this method
    //is not a retry it is just an initial failure.
    if(!canRetrySameServer(context)  && canRetryNextServer(context)) {
        context.setServiceInstance(loadBalanceChooser.choose(serviceId));
    }
    //This method is called regardless of whether we are retrying or making the first request.
    //Since we do not count the initial request in the retry count we don't reset the counter
    //until we actually equal the same server count limit.  This will allow us to make the initial
    //request plus the right number of retries.
    if(sameServerCount >= lbContext.getRetryHandler().getMaxRetriesOnSameServer() && canRetry(context)) {
        //reset same server since we are moving to a new server
        sameServerCount = 0;
        nextServerCount++;
        if(!canRetryNextServer(context)) {
            context.setExhaustedOnly();
        }
    } else {
        sameServerCount++;
    }
}
```

首先判断是否有一个引发异常的回路
 
然后调用`canRetrySameServer`方法和`canRetryNextServer`判断是否要选择下一个服务实例。如果是的话调用`RibbonLoadBalancerClient.choose`方法选择下一个服务实例。

`canRetrySameServer`方法的代码如下：

```java
public boolean canRetrySameServer(LoadBalancedRetryContext context) {
    return sameServerCount < lbContext.getRetryHandler().getMaxRetriesOnSameServer() && canRetry(context);
}
```

它判断`sameServerCount`是否小于我们配置的`MaxAutoRetries`，并且`canRetry`方法是否返回`true`。

`canRetryNextServer`方法的代码如下：

```java
public boolean canRetryNextServer(LoadBalancedRetryContext context) {
    return nextServerCount <= lbContext.getRetryHandler().getMaxRetriesOnNextServer() && canRetry(context);
}
```

它判断`nextServerCount`是否小于我们配置的`MaxAutoRetriesNextServer`，并且`canRetry`方法是否返回`true`。

最后判断`sameServerCount`是否大于等于我们配置的`MaxAutoRetries`，并且`canRetry`方法是否返回`true`：

- 如果是的话，说明要切换服务实例，于是将`sameServerCount`设为0，nextServerCount加1
- 否则，说明还是请求相同的服务实例，于是将`sameServerCount`加1

执行完异常处理后，`RetryTemplate.doExecute`方法重新执行while循环：在相同的服务实例上再次发送请求，或者切换到下一个服务实例发送请求。


# 总结

Ribbon的重试机制核心类是`RetryTemplate`，它捕获请求的异常，通过一个循环来重新请求相同的服务实例或者切换到下一个服务实例发送请求，以达到重试的效果。









[1]: /articles/Spring-Cloud/Eureka(六)——服务消费.html

> http://blog.didispace.com/spring-cloud-ribbon-failed-retry/
> http://www.itmuch.com/spring-cloud-sum/spring-cloud-timeout/

