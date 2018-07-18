---
title: Hystrix(八)——断路器
date: 2018/07/11 16:13:25
---

断路器HystrixCircuitBreaker有三种状态：

- CLOSE：关闭
- OPEN：打开
- HALF_OPEN：半开

<!-- more -->

下图展示了`HystrixCommand`或`HystrixObservableCommand`如何与`HystrixCircuitBreaker`进行交互，以及`HystrixCircuitBreaker`的决策逻辑过程，包括熔断器内部计数器如何工作。

![circuit-breaker-logic-flo](media/circuit-breaker-logic-flow.png)

线路的开路闭路详细逻辑如下：

1. 假设线路内的容量（请求QPS）达到一定阈值（通过`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold()`配置）
2. 同时，假设线路内的错误率达到一定阈值（通过`HystrixCommandProperties.circuitBreakerErrorThresholdPercentage()`配置）
3. 熔断器将从`闭路`转换成`开路`
4. 若此时是`开路`状态，熔断器将短路后续所有经过该熔断器的请求，这些请求直接走`失败回退逻辑`
5. 经过一定时间（即`休眠窗口`，通过`HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds()`配置），后续第一个请求将会被允许通过熔断器（此时熔断器处于`半开`状态），若该请求失败，熔断器将又进入`开路`状态，且在休眠窗口内保持此状态；若该请求成功，熔断器将进入`闭路`状态，回到逻辑1循环往复

# HystrixCircuitBreaker

`com.netflix.hystrix.HystrixCircuitBreaker`，Hystrix断路器接口。定义接口如下代码：

```java
public interface HystrixCircuitBreaker {

    /**
     * Every {@link HystrixCommand} requests asks this if it is allowed to proceed or not.  It is idempotent and does
     * not modify any internal state, and takes into account the half-open logic which allows some requests through
     * after the circuit has been opened
     * 
     * @return boolean whether a request should be permitted
     */
    boolean allowRequest();

    /**
     * Whether the circuit is currently open (tripped).
     * 
     * @return boolean state of circuit breaker
     */
    boolean isOpen();

    /**
     * Invoked on successful executions from {@link HystrixCommand} as part of feedback mechanism when in a half-open state.
     */
    void markSuccess();

    /**
     * Invoked on unsuccessful executions from {@link HystrixCommand} as part of feedback mechanism when in a half-open state.
     */
    void markNonSuccess();

    /**
     * Invoked at start of command execution to attempt an execution.  This is non-idempotent - it may modify internal
     * state.
     */
    boolean attemptExecution();
}
```

`allowRequest()`和`attemptExecution()`方法，方法目的基本类似，差别在于当断路器满足尝试关闭条件时，前者不会修改断路器的状态(`CLOSE` => `HALF-OPEN`)，而后者会。

HystrixCircuitBreaker有两个子类实现：

- NoOpCircuitBreaker：空的断路器实现，用于不开启断路器功能的情况
- HystrixCircuitBreakerImpl：完整的断路器实现

在`AbstractCommand`创建时，初始化`HystrixCircuitBreaker`，代码如下：

```java
abstract class AbstractCommand<R> implements HystrixInvokableInfo<R>, HystrixObservable<R> {
    // 断路器
    protected final HystrixCircuitBreaker circuitBreaker;
    
    protected AbstractCommand(HystrixCommandGroupKey group, HystrixCommandKey key, HystrixThreadPoolKey threadPoolKey, HystrixCircuitBreaker circuitBreaker, HystrixThreadPool threadPool,
            HystrixCommandProperties.Setter commandPropertiesDefaults, HystrixThreadPoolProperties.Setter threadPoolPropertiesDefaults,
            HystrixCommandMetrics metrics, TryableSemaphore fallbackSemaphore, TryableSemaphore executionSemaphore,
            HystrixPropertiesStrategy propertiesStrategy, HystrixCommandExecutionHook executionHook) {
        //...省略无关代码
        this.circuitBreaker = initCircuitBreaker(this.properties.circuitBreakerEnabled().get(), circuitBreaker, this.commandGroup, this.commandKey, this.properties, this.metrics);
        //...省略无关代码
    }
}

private static HystrixCircuitBreaker initCircuitBreaker(boolean enabled, HystrixCircuitBreaker fromConstructor,
                                                        HystrixCommandGroupKey groupKey, HystrixCommandKey commandKey,
                                                        HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
    if (enabled) {
        if (fromConstructor == null) {
            // get the default implementation of HystrixCircuitBreaker
            return HystrixCircuitBreaker.Factory.getInstance(commandKey, groupKey, properties, metrics);
        } else {
            return fromConstructor;
        }
    } else {
        return new NoOpCircuitBreaker();
    }
}
```

- 当`HystrixCommandProperties.circuitBreakerEnabled = true`时，即断路器功能开启，使用Factory获得`HystrixCircuitBreakerImpl`对象
- 当`HystrixCommandProperties.circuitBreakerEnabled = false`时，即断路器功能关闭，创建`NoOpCircuitBreaker`对象。

# HystrixCircuitBreaker.Factory

`com.netflix.hystrix.HystrixCircuitBreaker.Factory`，HystrixCircuitBreaker工厂，主要用于：

- 创建`HystrixCircuitBreaker`对象，目前只创建`HystrixCircuitBreakerImpl`
- `HystrixCircuitBreaker`容器，基于HystrixCommandKey维护了HystrixCircuitBreaker单例对象的映射。代码如下：

```java
private static ConcurrentHashMap<String, HystrixCircuitBreaker> circuitBreakersByCommand = new ConcurrentHashMap<String, HystrixCircuitBreaker>();
```

# HystrixCircuitBreakerImpl

`com.netflix.hystrix.HystrixCircuitBreaker.HystrixCircuitBreakerImpl`，完整的断路器实现

## 构造方法

```java
class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {
    private final HystrixCommandProperties properties;
    private final HystrixCommandMetrics metrics;

    // 枚举类，断路器的三种状态
    enum Status {
        CLOSED, OPEN, HALF_OPEN;
    }
    // 断路器的状态
    private final AtomicReference<Status> status = new AtomicReference<Status>(Status.CLOSED);
    // 断路器打开，即状态变成OPEN的时间
    private final AtomicLong circuitOpened = new AtomicLong(-1);
    // 基于Hystrix Metrics对请求量统计Observable的订阅
    private final AtomicReference<Subscription> activeSubscription = new AtomicReference<Subscription>(null);

    protected HystrixCircuitBreakerImpl(HystrixCommandKey key, HystrixCommandGroupKey commandGroup, final HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
        this.properties = properties;
        this.metrics = metrics;

        //On a timer, this will set the circuit between OPEN/CLOSED as command executions occur
        Subscription s = subscribeToStream();
        activeSubscription.set(s);
    }
}
```

## subscribeToStream

`subscribeToStream`方法向Hystrix Metrics对请求量统计Observable发起订阅。代码如下：

```java
private Subscription subscribeToStream() {
    /*
     * This stream will recalculate the OPEN/CLOSED status on every onNext from the health stream
     */
    return metrics.getHealthCountsStream()
            .observe()
            .subscribe(new Subscriber<HealthCounts>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {

                }

                @Override
                public void onNext(HealthCounts hc) {
                    // check if we are past the statisticalWindowVolumeThreshold
                    if (hc.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {
                        // we are not past the minimum volume threshold for the stat window,
                        // so no change to circuit status.
                        // if it was CLOSED, it stays CLOSED
                        // if it was half-open, we need to wait for a successful command execution
                        // if it was open, we need to wait for sleep window to elapse
                    } else {
                        if (hc.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
                            //we are not past the minimum error threshold for the stat window,
                            // so no change to circuit status.
                            // if it was CLOSED, it stays CLOSED
                            // if it was half-open, we need to wait for a successful command execution
                            // if it was open, we need to wait for sleep window to elapse
                        } else {
                            // our failure rate is too high, we need to set the state to OPEN
                            if (status.compareAndSet(Status.CLOSED, Status.OPEN)) {
                                circuitOpened.set(System.currentTimeMillis());
                            }
                        }
                    }
                }
            });
}
```

1. 向Hystrix Metrics对请求量统计Observable发起订阅。这里的Observable基于RxJava Window操作符。简单来说，固定间隔，`onNext()`方法将不断被调用，每次计算断路器的状态。
2. `onNext()`方法

    1. 判断周期内（可配，`HystrixCommandProperties.default_metricsRollingStatisticalWindow = 10000ms`），总请求数超过一定量（可配，`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold = 20`）
    2. 错误请求占总请求数超过一定比例（可配，`HystrixCommandProperties.circuitBreakerErrorThresholdPercentage = 50%`）
    3. 满足断路器打开条件，CAS将状态从`CLOSED`修改为`OPEN`，并设置打开时间（`circuitOpened`）

Hystrix Metrics对请求量统计Observable使用了两种RxJava Window操作符：

- `Observable.window(timespan, unit)`方法，固定周期（可配，`HystrixCommandProperties.metricsHealthSnapshotIntervalInMilliseconds = 500ms`），发射Observable窗口。
- `Observable.window(count, skip)`方法，每发射一次(`skip`)Observable忽略`count`（可配，`HystrixCommandProperties.circuitBreakerRequestVolumeThreshold = 20`）个数据项

目前该方法有两处调用：

- 构造方法，在创建`HystrixCircuitBreakerImpl`时，向Hystrix Metrics对请求量统计Observable发起订阅。固定间隔计算断路器是否要关闭
- `markSuccess`，清空Hystrix Metrics对请求量统计Observable的统计信息，取消原有订阅，并发起新的订阅

## attemptExecution

如下是`AbstractCommand.applyHystrixSemantics`方法，对`HystrixCircuitBreakerImpl.attemptExecution`方法的调用的代码：

```java
private Observable<R> applyHystrixSemantics(final AbstractCommand<R> _cmd) {
    if (circuitBreaker.attemptExecution()) {
        // 执行正常逻辑
    } else {
        // 执行回退逻辑
    }
}
```

使用`HystrixCircuitBreakerImpl.attemptExecution`方法，判断是否可以执行正常逻辑。代码如下：

```java
public boolean attemptExecution() {
    // 强制打开
    if (properties.circuitBreakerForceOpen().get()) {
        return false;
    }
    // 强制关闭
    if (properties.circuitBreakerForceClosed().get()) {
        return true;
    }
    // 打开时间为空
    if (circuitOpened.get() == -1) {
        return true;
    } else {
        // 满足间隔尝试断路器时间
        if (isAfterSleepWindow()) {
            if (status.compareAndSet(Status.OPEN, Status.HALF_OPEN)) {
                //only the first request after sleep window should execute
                return true;
            } else {
                return false;
            }
        } else {
            return false;
        }
    }
}
```

1. 当`HystrixCommandProperties.circuitBreakerForceOpen = true`（默认值：`false`），即断路器强制打开，返回`false`。当该配置接入配置中心后，可以动态实现打开熔断。为什么会有该配置？当`HystrixCircuitBreaker`创建完成后，无法动态切换`NoOpCircuitBreaker`和`HystrixCircuitBreakerImpl`，通过该配置以实现类似效果。
2. 当`HystrixCommandProperties.circuitBreakerForceClose = true`（默认值：`false`），即断路器强制关闭，返回`true`。当该配置接入配置中心后，可以动态实现关闭熔断。为什么会有该配置？当`HystrixCircuitBreaker`创建完成后，无法动态切换`NoOpCircuitBreaker`和`HystrixCircuitBreakerImpl`，通过该配置以实现类似效果。
3. 断路器打开时间（`circuitOpened`）为空，返回`true`
4. 调用`isAfterSleepWindow`方法，判断是否满足尝试调用正常逻辑的间隔时间。当满足时，使用CAS方法修改断路器状态（`OPEN` => `HALF_OPEN`），从而保证有且仅有一个线程能够尝试调用正常逻辑

`isAfterSleepWindow`方法代码如下：

```java
private boolean isAfterSleepWindow() {
    final long circuitOpenTime = circuitOpened.get();
    final long currentTime = System.currentTimeMillis();
    final long sleepWindowTime = properties.circuitBreakerSleepWindowInMilliseconds().get();
    return currentTime > circuitOpenTime + sleepWindowTime;
}
```

在当前时间超过断路器打开时间`HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds`（默认值：`5000ms`），返回`true`。

## markSuccess

当尝试调用正常逻辑成功时，调用`markSuccess`方法，关闭断路器。

```java
public void markSuccess() {
    if (status.compareAndSet(Status.HALF_OPEN, Status.CLOSED)) {
        //This thread wins the race to close the circuit - it resets the stream to start it over from 0
        metrics.resetStream();
        Subscription previousSubscription = activeSubscription.get();
        if (previousSubscription != null) {
            previousSubscription.unsubscribe();
        }
        Subscription newSubscription = subscribeToStream();
        activeSubscription.set(newSubscription);
        circuitOpened.set(-1L);
    }
}
```

1. 使用CAS方式，修改断路器状态(`HALF_OPEN` => `CLOSED`)
2. 清空Hystrix Metrics对请求量统计Observable的统计信息
3. 取消原有订阅，发起新的订阅
4. 设置断路器打开时间为空

以下两处调用`markSuccess`方法：

- `markEmits`
- `markOnCompleted`

## markNonSuccess

当尝试调用正常逻辑失败时，调用`markNonSuccess`方法，重新打开断路器。

```java
public void markNonSuccess() {
   if (status.compareAndSet(Status.HALF_OPEN, Status.OPEN)) {
        //This thread wins the race to re-open the circuit - it resets the start time for the sleep window
        circuitOpened.set(System.currentTimeMillis());
    }
}
```

1. 使用CAS的方式，修改断路器的状态(`HALF_OPEN` => `OPEN`)
2. 设置断路器打开时间为当前时间。这样`attemptExecution()`过一段时间，可以再次尝试执行正常逻辑

以下两处调用了`markNonSuccess`方法：

- `handleFallback`
- `unsubscribeCommandCleanup`

## allowRequest

`allowRequest`方法和`attemptExecution`方法，目的基本类似，差别在于当断路器满足尝试关闭条件时，前者不会修改断路器的状态(`CLOSE` => `HALF-OPEN`)，而后者会。

## isOpen

`isOpen`方法比较简单：

```java
public boolean isOpen() {
    if (properties.circuitBreakerForceOpen().get()) {
        return true;
    }
    if (properties.circuitBreakerForceClosed().get()) {
        return false;
    }
    return circuitOpened.get() >= 0;
}
```

# 断路器测试

对断路器的测试，我们选择比较简单的方式，直接调用Hystrix的命令：

```java
public class CircuitBreakerCommand extends HystrixCommand<String> {

    public CircuitBreakerCommand(String name) {
        super(Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("ThreadPoolTestGroup"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("testCommandKey"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(name))
                .andCommandPropertiesDefaults(
                        HystrixCommandProperties.Setter()
                        .withCircuitBreakerEnabled(true)                    // 默认是true，本例中为了展现该参数
                        .withCircuitBreakerForceOpen(false)                 // 默认是false，本例中为了展现该参数
                        .withCircuitBreakerForceClosed(false)               // 默认是false，本例中为了展现该参数
                        .withCircuitBreakerErrorThresholdPercentage(5)      // (1)错误百分比超过5%。默认是50
                        .withCircuitBreakerRequestVolumeThreshold(10)       // (2)10s以内调用次数10次，同时满足(1)(2)熔断器打开。默认是20
                        .withCircuitBreakerSleepWindowInMilliseconds(5000)  // 隔5秒之后，熔断器会尝试半开，重新放进来请求。默认是5000
                )
                .andThreadPoolPropertiesDefaults(
                        HystrixThreadPoolProperties.Setter()
                        .withMaxQueueSize(10)                   // 配置队列大小
                        .withCoreSize(2)                        // 配置线程池里的线程数
                )
        );
    }

    @Override
    protected String run() throws Exception {
        Random rand = new Random();
        if (1 == rand.nextInt(2)) {
            throw new Exception("make exception");
        }
        return "running: ";
    }

    @Override
    protected String getFallback() {
        return "fallback: ";
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 25; i++) {
            Thread.sleep(500);
            HystrixCommand<String> command = new CircuitBreakerCommand("testCircuitBreaker");
            String result = command.execute();
            System.out.println("call times:" + (i + 1) + " result: " + result + " isCircuitBreakerOpen: " + command.isCircuitBreakerOpen());
        }
    }
}
```

执行结果如下：

```
call times:1 result: running:  isCircuitBreakerOpen: false
call times:2 result: running:  isCircuitBreakerOpen: false
call times:3 result: running:  isCircuitBreakerOpen: false
call times:4 result: fallback:  isCircuitBreakerOpen: false
call times:5 result: running:  isCircuitBreakerOpen: false
call times:6 result: running:  isCircuitBreakerOpen: false
call times:7 result: running:  isCircuitBreakerOpen: false
call times:8 result: running:  isCircuitBreakerOpen: false
call times:9 result: running:  isCircuitBreakerOpen: false
call times:10 result: fallback:  isCircuitBreakerOpen: false
熔断器打开
call times:11 result: fallback:  isCircuitBreakerOpen: true
call times:12 result: fallback:  isCircuitBreakerOpen: true
call times:13 result: fallback:  isCircuitBreakerOpen: true
call times:14 result: fallback:  isCircuitBreakerOpen: true
call times:15 result: fallback:  isCircuitBreakerOpen: true
call times:16 result: fallback:  isCircuitBreakerOpen: true
call times:17 result: fallback:  isCircuitBreakerOpen: true
call times:18 result: fallback:  isCircuitBreakerOpen: true
call times:19 result: fallback:  isCircuitBreakerOpen: true
call times:20 result: fallback:  isCircuitBreakerOpen: true
5s后熔断器关闭
call times:21 result: running:  isCircuitBreakerOpen: false
call times:22 result: fallback:  isCircuitBreakerOpen: false
call times:23 result: fallback:  isCircuitBreakerOpen: false
call times:24 result: fallback:  isCircuitBreakerOpen: false
call times:25 result: fallback:  isCircuitBreakerOpen: false
```

我们看到，前10此命令执行有两次失败，于是熔断器被打开，11到20次执行全部快速失败。5s后熔断器关闭，命令可以再次尝试执行。

# 总结

总体来说，Hystrix的断路器是一个防止重复并发请求失败服务的机制，它的执行流程如下：

1. 在一定时间内（HystrixCommandProperties.metricsRollingStatisticalWindowInMilliseconds，默认为10000ms），请求次数达到一定阈值(HystrixCommandProperties.circuitBreakerRequestVolumeThreshold，默认为20)，且错误率达到一定阈值（HystrixCommandProperties.circuitBreakerErrorThresholdPercentage，默认为50%），熔断器将从闭路转换成开路。开路状态下，所有请求直接走失败回退逻辑。
2. 经过一定的时间（休眠窗口，HystrixCommandProperties.circuitBreakerSleepWindowInMilliseconds，默认为5000ms），后续第一个请求将会被允许通过熔断器（半开状态），若该请求失败，熔断器又进入开路状态，且在休眠窗口内保持此状态；若该请求成功，熔断器将进入闭路状态。






> http://youdang.github.io/2016/02/05/translate-hystrix-wiki-how-it-works/#%E8%AF%B7%E6%B1%82%E5%90%88%E5%B9%B6
> https://github.com/YunaiV/Blog/blob/master/Hystrix/2018_11_08_Hystrix%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20%E2%80%94%E2%80%94%20%E6%96%AD%E8%B7%AF%E5%99%A8%20HystrixCircuitBreaker.md
> https://www.jianshu.com/p/14958039fd15

