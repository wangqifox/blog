---
title: Eureka(八)——RateLimiter
date: 2018/07/03 18:25:25
---

Eureka在实现中有一个限流器来保证Eureka Server的稳定。
<!-- more -->

常见的限流算法有漏桶算法以及令牌桶算法。这个可以参考[http://www.cnblogs.com/LBSer/p/4083131.html](http://www.cnblogs.com/LBSer/p/4083131.html)。

`Google Guava`中提供了限流工具类`RateLimiter`。一开始我以为Eureka中的限流是由`Guava`中的`RateLimiter`实现的，后来发现并不是，Eureka自己实现了限流类`com.netflix.discovery.util.RateLimiter`。

# RateLimiter

## 构造方法

RateLimiter目前支持**分钟级**和**秒级**两种速率限制。构造方法如下：

```java
public class RateLimiter {
    /**
     * 速率单位转换成毫秒
     */
    private final long rateToMsConversion;
    
    public RateLimiter(TimeUnit averageRateUnit) {
        switch (averageRateUnit) {
            case SECONDS:   // 秒级
                rateToMsConversion = 1000;
                break;
            case MINUTES:   // 分钟级
                rateToMsConversion = 60 * 1000;
                break;
            default:
                throw new IllegalArgumentException("TimeUnit of " + averageRateUnit + " is not supported");
        }
    }
}
```

`averageRateUnit`参数，速率单位。构造方法里将`averageRateUnit`转换成`rateToMsConversion`。

## acquire

`acquire`方法用于获取令牌，并返回是否获取成功

```java
public boolean acquire(int burstSize, long averageRate) {
    return acquire(burstSize, averageRate, System.currentTimeMillis());
}

public boolean acquire(int burstSize, long averageRate, long currentTimeMillis) {
    if (burstSize <= 0 || averageRate <= 0) { // Instead of throwing exception, we just let all the traffic go
        return true;
    }
    // 填充令牌
    refillToken(burstSize, averageRate, currentTimeMillis);
    // 消费令牌
    return consumeToken(burstSize);
}
```

- `burstSize`：令牌桶上限，即最大能被消耗掉`burstSize`数量的令牌
- `averageRate`：令牌填充平均速率

## refillToken

`refillToken`方法填充已消耗的令牌。填充令牌的操作并不是一个后台任务每毫秒执行填充。为什么不适合这样呢？一方面，实际项目里每个接口都会有相应的RateLimiter，导致太多执行频率极高的后台任务；另一方面，获取令牌时才计算，多次令牌填充可以合并成一次，减少冗余和无效的计算。

代码如下：

```java
/**
 * 速率单位转换成毫秒
 */
private final long rateToMsConversion;
/**
 * 消耗令牌数
 */
private final AtomicInteger consumedTokens = new AtomicInteger();
/**
 * 最后填充令牌的时间
 */
private final AtomicLong lastRefillTime = new AtomicLong(0);
    
private void refillToken(int burstSize, long averageRate, long currentTimeMillis) {
    // 获取最后填充令牌的时间
    long refillTime = lastRefillTime.get();
    // 获取距离最后一次填充令牌过去多少毫秒
    long timeDelta = currentTimeMillis - refillTime;
    // 计算可填充的最大令牌数量
    long newTokens = timeDelta * averageRate / rateToMsConversion;
    if (newTokens > 0) {
        // 计算新的令牌填充时间
        long newRefillTime = refillTime == 0
                ? currentTimeMillis
                : refillTime + newTokens * rateToMsConversion / averageRate;
        if (lastRefillTime.compareAndSet(refillTime, newRefillTime)) {
            while (true) {  // 循环直到令牌填充完成
                // 获取当前被消耗掉的令牌数
                int currentLevel = consumedTokens.get();
                // 将当前被消耗掉的令牌数和burstSize(桶大小)比较，选择较小的那个。桶大小可能会减小
                int adjustedLevel = Math.min(currentLevel, burstSize);
                // 将(adjustedLevel - newTokens)作为令牌消耗数，newTokens表示当前补充了多少个令牌。
                // 如果(adjustedLevel - newTokens < 0)表示桶中的令牌已经溢出了。这时候将令牌消耗数设为0，因为令牌消耗数是不能为负数的。
                int newLevel = (int) Math.max(0, adjustedLevel - newTokens);
                // 设置新的令牌消耗数
                if (consumedTokens.compareAndSet(currentLevel, newLevel)) {
                    return;
                }
            }
        }
    }
}
```

1. 根据距离最后一次填充令牌的时间来计算可填充的最大令牌数量`newTokens`
2. 计算新的填充令牌的时间`newRefillTime`。为什么不能用`currentTimeMillis`呢？例如，`averageRate = 500 && averageRateUnit = SECONDS`时，每2毫秒才填充一个令牌，如果设置`currentTimeMillis`，会导致不足以填充一个令牌的时长被吞了。
3. 通过CAS设置最后填充令牌的时间。并保证只有一个线程进入填充令牌的逻辑
4. 循环直到令牌填充完成

    1. 通过`consumedTokens.get()`获取消耗令牌的数量
    2. 通过与`burstSize`的比较来调整消耗令牌的数量。因为`burstSize`可能被调小，例如，系统接入分布式配置中心，可以远程调整该数值。如果此时`burstSize`更小，以它作为已消耗的令牌数量
    3. 通过`adjustedLevel - newTokens`来计算新的被消耗掉的令牌的数量。即此时补充进来`newTokens`数量的令牌，因此消耗令牌的数量减少。
    4. 通过`Math.max(0, adjustedLevel - newTokens)`的计算保证新的被消耗掉的令牌`newLevel`的数量不小于0。即此时令牌桶里的令牌是满的。
    5. 通过CAS设置消耗令牌数量。避免覆盖设置正在消费令牌的线程。

## consumeToken

`consumeToken`方法用于获取(消费)令牌。代码如下：

```java
private boolean consumeToken(int burstSize) {
    while (true) {
        int currentLevel = consumedTokens.get();
        // 当前消耗的令牌数大于等于桶的大小，说明桶里的令牌都已经消耗完了。这时获取令牌失败
        if (currentLevel >= burstSize) {
            return false;
        }
        // 消耗的令牌数量增1，获取令牌成功
        if (consumedTokens.compareAndSet(currentLevel, currentLevel + 1)) {
            return true;
        }
    }
}
```

1. 循环导致获取令牌成功或者失败
2. 获取当前消耗掉的令牌的数量`currentLevel`，如果`currentLevel >= burstSize`说明当前所有的令牌都被消耗掉了，不能再获取令牌，所以返回`false`
3. 否则，通过CAS将当前消耗掉的令牌的数量增1，获取令牌成功。

## 总结

`RateLimiter`的设计和我们的直觉不太一样：

- 首先它并不是有一个单独的线程来填充令牌，而是将填充令牌的操作放在获取令牌的方法中。
- 其次它并不是以令牌数为中心来控制令牌是否获取成功，而是以消耗掉的令牌数为中心。因此填充令牌是减少消耗令牌数，获取令牌是增加消耗令牌数。

# RateLimiter在Eureka中的应用

## RateLimitingFilter

`com.netflix.eureka.RateLimitingFilter`是Eureka Server的限流过滤器。其中使用`RateLimiting`来保证Eureka Server的稳定性。

`doFilter`方法如下：

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    // 获得Target
    Target target = getTarget(request);
    // Other Target，不做限流
    if (target == Target.Other) {
        chain.doFilter(request, response);
        return;
    }

    HttpServletRequest httpRequest = (HttpServletRequest) request;
    // 判断是否被限流
    if (isRateLimited(httpRequest, target)) {
        // 监控相关
        incrementStats(target);
        // 如果开启限流，返回503状态码
        if (serverConfig.isRateLimiterEnabled()) {
            ((HttpServletResponse) response).setStatus(HttpServletResponse.SC_SERVICE_UNAVAILABLE);
            return;
        }
    }
    chain.doFilter(request, response);
}
```

### getTarget

首先调用`getTarget`方法，获取Target。使用正则表达式`^.*/apps(/[^/]*)?$`来匹配请求的url，根据不同的url返回不同的Target类型。Target类型有以下几种：`FullFetch`, `DeltaFetch`, `Application`, `Other`

如果Target的类型为`Other`则不做限流

### isRateLimited

然后调用`isRateLimited`方法，判断是否被限流。代码如下：

```java
private boolean isRateLimited(HttpServletRequest request, Target target) {
    // 判断是否是特权应用
    if (isPrivileged(request)) {
        logger.debug("Privileged {} request", target);
        return false;
    }
    // 判断是否过载
    if (isOverloaded(target)) {
        logger.debug("Overloaded {} request; discarding it", target);
        return true;
    }
    logger.debug("{} request admitted", target);
    return false;
}
```

首先调用`isPrivileged`方法，判断是否为特权应用，对特权应用不开启限流逻辑。代码如下：

```java
private boolean isPrivileged(HttpServletRequest request) {
    // 是否对标准客户端开启限流
    if (serverConfig.isRateLimiterThrottleStandardClients()) {
        return false;
    }
    Set<String> privilegedClients = serverConfig.getRateLimiterPrivilegedClients();
    // 获取请求的DiscoveryIdentity-Name请求头
    String clientName = request.getHeader(AbstractEurekaIdentity.AUTH_NAME_HEADER_KEY);
    // 根据DiscoveryIdentity-Name请求头判断是否在特权列表中
    return privilegedClients.contains(clientName) || DEFAULT_PRIVILEGED_CLIENTS.contains(clientName);
}
```

然后调用`isOverloaded`方法，判断是否过载。代码如下：

```java
/**
 * Includes both full and delta fetches.
 */
private static final RateLimiter registryFetchRateLimiter = new RateLimiter(TimeUnit.SECONDS);

/**
 * Only full registry fetches.
 */
private static final RateLimiter registryFullFetchRateLimiter = new RateLimiter(TimeUnit.SECONDS);

private boolean isOverloaded(Target target) {
    int maxInWindow = serverConfig.getRateLimiterBurstSize();   // 10
    int fetchWindowSize = serverConfig.getRateLimiterRegistryFetchAverageRate();    // 500
    boolean overloaded = !registryFetchRateLimiter.acquire(maxInWindow, fetchWindowSize);

    if (target == Target.FullFetch) {
        int fullFetchWindowSize = serverConfig.getRateLimiterFullFetchAverageRate();    // 100
        overloaded |= !registryFullFetchRateLimiter.acquire(maxInWindow, fullFetchWindowSize);
    }
    return overloaded;
}
```

调用`registryFetchRateLimiter.acquire`获取令牌，如果target的类型是`FullFetch`则调用`registryFullFetchRateLimiter.acquire`来获取令牌，效果就是如果是FullFetch请求则限制在每秒钟100次，普通请求则限制在每秒钟500次。

## InstanceInfoReplicator

`InstanceInfoReplicator`是Eureka Client的服务实例复制器。在[Eureka(三)——client注册过程][1]中有详解。

服务实例状态发生变化时，会调用`onDemandUpdate`方法向Eureka Server发起注册，同步服务实例信息。`onDemandUpdate`方法中使用RateLimiter避免状态频繁发生变化而向Eureka Server频繁同步。代码如下：

```java
private final RateLimiter rateLimiter;
/**
 * 令牌桶上限，默认为2
 */
private final int burstSize;
/**
 * 令牌再装平均速率，默认为:60 * 2 / 30 = 4
 */
private final int allowedRatePerMinute;

InstanceInfoReplicator(DiscoveryClient discoveryClient, InstanceInfo instanceInfo, int replicationIntervalSeconds, int burstSize) {
    ...
    this.rateLimiter = new RateLimiter(TimeUnit.MINUTES);
    this.replicationIntervalSeconds = replicationIntervalSeconds;
    this.burstSize = burstSize;

    this.allowedRatePerMinute = 60 * this.burstSize / this.replicationIntervalSeconds;
    logger.info("InstanceInfoReplicator onDemand update allowed rate per min is {}", allowedRatePerMinute);
}

public boolean onDemandUpdate() {
    if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
        if (!scheduler.isShutdown()) {
            scheduler.submit(new Runnable() {
                @Override
                public void run() {
                    logger.debug("Executing on-demand update of local InstanceInfo");
    
                    Future latestPeriodic = scheduledPeriodicRef.get();
                    if (latestPeriodic != null && !latestPeriodic.isDone()) {
                        logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                        latestPeriodic.cancel(false);
                    }
    
                    InstanceInfoReplicator.this.run();
                }
            });
            return true;
        } else {
            logger.warn("Ignoring onDemand update due to stopped scheduler");
            return false;
        }
    } else {
        logger.warn("Ignoring onDemand update due to rate limiter");
        return false;
    }
}
```

`onDemandUpdate`方法调用`RateLimiter.acquire`方法获取令牌：

- 若获取成功，向Eureka Server发起注册，同步服务实例信息
- 若获取失败，不向Eureka Server发起注册，同步服务实例信息。这样会不会有问题？答案是不会。因为`InstanceInfoReplicator`会固定周期检查本地服务实例，如果发生了改变就会向Eureka Server同步实例信息。













[1]: /articles/Spring-Cloud/Eureka(三)——client注册过程.html



> http://www.iocoder.cn/Eureka/rate-limiter/
> http://www.itmuch.com/spring-cloud-sum/spring-cloud-ratelimit/

