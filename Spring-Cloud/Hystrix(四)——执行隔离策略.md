---
title: Hystrix(四)——执行隔离策略
date: 2018/07/09 09:37:25
---

# 依赖隔离

Hystrix通过使用`仓壁模式`（注：将船的底部划分成一个个的舱室，这样一个舱室进水不会导致整艘船沉没。将系统所有依赖服务隔离起来，一个依赖延迟升高或者失败，不会导致整个系统失败）来隔离依赖服务，并限制访问这些依赖服务的并发度。
<!-- more -->
## 线程 & 线程池

通过将对依赖服务的访问执行放到单独的线程，将其与调用线程（例如Tomcat线程池中的线程）隔离开来，调用线程能空出来去做其他的工作而不至于被依赖服务的访问阻塞过长时间。

Hystrix使用独立的，每个依赖服务对应一个线程池的方式，来隔离这些依赖服务，这样，某个依赖服务的高延迟只会拖慢这个依赖服务对应的线程池。

当然，也可以不使用线程池来使你的系统免受依赖服务失效的影响，这需要你小心的设置网络连接/读取超时时间和重试配置，并保证这些配置能正确正常的运作，以使这些依赖服务在失效时，能快速返回错误。

### 线程池的优势

将依赖服务请求通过使用不同的线程池隔离，其优势如下：

- 系统完全与依赖服务请求隔离开来，即使依赖服务对应线程池耗尽，也不会影像系统其他请求
- 降低了系统接入新的依赖服务的风险，若新的依赖服务存在问题，也不会影响系统其他请求
- 当依赖服务失效后又恢复正常，其对应的线程池会被清理干净，相对于整个Tomcat容器的线程池被占满需要耗费更长时间以恢复可用来说，此时系统可以快速恢复
- 若依赖服务的配置有问题，线程池能迅速反应出来（通过失败次数的增加，高延迟，超时，拒绝访问等等），同时，你可以在不影响系统现有功能的情况下，处理这些问题（通常通过热配置等方式）
- 若依赖服务的实现发生变更，性能有了很大的变化（这种情况时常发生），需要进行配置调整（例如增加/减小超时阈值，调整重试策略等）时，也可以从线程池的监控信息上迅速反映出来（失败次数增加，高延迟，超时，拒绝访问等等），同时，你可以在不影响其他依赖服务，系统请求和用户的情况下，处理这些问题
- 线程池处理能起到隔离的作用以外，还能通过这种内置的并发特性，在客户端库同步网络IO上，建立一个异步的Facade（类似Netflix API建立在Hystrix命令上的Reactive、全异步化的那套Java API）

### 线程池的弊端

使用线程池的主要弊端是会增加系统CPU的负载，每个命令的执行，都包含了CPU任务的排队，调度，上下文切换。

Netflix在设计Hystrix时，认为相对于其带来的好处，其带来的负载的一点点升高对系统的影响是微乎其微的。

## 信号量

除了线程池，队列之外，你可以使用信号量（或者叫计数器）来限制单个依赖服务的并发度。Hystrix可以利用信号量，而不是线程池，来控制系统负载。如果你对客户端库有足够的信任（延迟不会过高），并且你只需要控制系统负载，那么你可以使用信号量。

`HystrixCommand`和`HystrixObervableCommand`在两个地方支持使用信号量：

- 失败回退逻辑：当Hystrix需要执行失败回退逻辑时，其在调用线程（Tomcat线程）中使用信号量
- 执行命令时：如果设置了Hystrix命令的`execution.isolation.strategy`属性为`SEMAPHORE`，则Hystrix会使用信号量而不是线程池来控制调用线程调用依赖服务的并发度

# 执行隔离原理

## HystrixThreadPool

在前文[Hystrix(三)——正常执行逻辑][1]中，我们知道`executeCommandWithSpecifiedIsolation`方法根据**执行隔离策略**的不同，创建不同的`命令执行Observable`。

如果执行隔离策略为`Thread`，调用`Observable.subscribeOn`方法指定Observable自身在哪个调度器上执行：

```java
.subscribeOn(threadPool.getScheduler(new Func0<Boolean>() {
    @Override
    public Boolean call() {
        return properties.executionIsolationThreadInterruptOnTimeout().get() && _cmd.isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT;
    }
}))
```

threadPool的类型是`com.netflix.hystrix.HystrixThreadPool`。

`HystrixThreadPool`是Hystrix线程池接口。

### HystrixThreadPoolDefault

`com.netflix.hystrix.HystrixThreadPool.HystrixThreadPoolDefault`，是Hystrix线程池的实现类。

构造方法如下：

```java
public HystrixThreadPoolDefault(HystrixThreadPoolKey threadPoolKey, HystrixThreadPoolProperties.Setter propertiesDefaults) {
    // 初始化HystrixThreadPoolProperties
    this.properties = HystrixPropertiesFactory.getThreadPoolProperties(threadPoolKey, propertiesDefaults);
    // 获得HystrixConcurrencyStrategy
    HystrixConcurrencyStrategy concurrencyStrategy = HystrixPlugins.getInstance().getConcurrencyStrategy();
    // 队列大小
    this.queueSize = properties.maxQueueSize().get();

    this.metrics = HystrixThreadPoolMetrics.getInstance(threadPoolKey,
            // 初始化ThreadPoolExecutor
            concurrencyStrategy.getThreadPool(threadPoolKey, properties),
            properties);
    // 获得ThreadPoolExecutor
    this.threadPool = this.metrics.getThreadPool();
    // 获得ThreadPoolExecutor的队列
    this.queue = this.threadPool.getQueue();

    /* strategy: HystrixMetricsPublisherThreadPool */
    HystrixMetricsPublisherFactory.createOrRetrievePublisherForThreadPool(threadPoolKey, this.metrics, this.properties);
}
```

`HystrixThreadPool`的`getScheduler`方法代码如下：

```java
public Scheduler getScheduler(Func0<Boolean> shouldInterruptThread) {
    touchConfig();
    return new HystrixContextScheduler(HystrixPlugins.getInstance().getConcurrencyStrategy(), this, shouldInterruptThread);
}
```

1. 调用`touchConfig()`方法，动态调整threadPool的`coreSize`、`maximumSize`、`keepAliveTime`参数
2. 新建`HystrixContextScheduler`并返回

### HystrixContextScheduler

`HystrixContextScheduler`的构造方法如下：

```java
private final HystrixConcurrencyStrategy concurrencyStrategy;
private final Scheduler actualScheduler;
private final HystrixThreadPool threadPool;

public HystrixContextScheduler(HystrixConcurrencyStrategy concurrencyStrategy, HystrixThreadPool threadPool, Func0<Boolean> shouldInterruptThread) {
    this.concurrencyStrategy = concurrencyStrategy;
    this.threadPool = threadPool;
    this.actualScheduler = new ThreadPoolScheduler(threadPool, shouldInterruptThread);
}
```

actualScheduler属性的类型为ThreadPoolScheduler

`createWorker()`方法的代码如下：

```java
public Worker createWorker() {
    return new HystrixContextSchedulerWorker(actualScheduler.createWorker());
}
```

使用`actualScheduler`创建ThreadPoolWorker，传参给`HystrixContextSchedulerWorker`。

### HystrixContextSchedulerWorker

`HystrixContextSchedulerWorker`的构造方法如下：

```java
private class HystrixContextSchedulerWorker extends Worker {
    private final Worker worker;
    private HystrixContextSchedulerWorker(Worker actualWorker) {
        this.worker = actualWorker;
    }
}
```

`worker`属性为`actualScheduler`创建的`ThreadPoolWorker`。

`schedule(Action0)`方法代码如下：

```java
public Subscription schedule(Action0 action) {
    if (threadPool != null) {
        if (!threadPool.isQueueSpaceAvailable()) {
            throw new RejectedExecutionException("Rejected command because thread-pool queueSize is at rejection threshold.");
        }
    }
    return worker.schedule(new HystrixContexSchedulerAction(concurrencyStrategy, action));
}
```

1. 调用`ThreadPool.isQueueSpaceAvailable()`方法，判断线程池队列是否有空余。
2. 调用`ThreadPoolWorker.schedule`方法

### ThreadPoolWorker

构造方法代码如下：

```java
private final HystrixThreadPool threadPool;
private final CompositeSubscription subscription = new CompositeSubscription();
private final Func0<Boolean> shouldInterruptThread;

public ThreadPoolWorker(HystrixThreadPool threadPool, Func0<Boolean> shouldInterruptThread) {
    this.threadPool = threadPool;
    this.shouldInterruptThread = shouldInterruptThread;
}
```

`subscription`属性表示订阅信息。

`schedule(Action0)`方法代码如下：

```java
public Subscription schedule(final Action0 action) {
    // 未订阅，返回
    if (subscription.isUnsubscribed()) {
        // don't schedule, we are unsubscribed
        return Subscriptions.unsubscribed();
    }

    // 创建ScheduledAction
    // This is internal RxJava API but it is too useful.
    ScheduledAction sa = new ScheduledAction(action);

    // 添加到订阅
    subscription.add(sa);
    sa.addParent(subscription);

    // 提交任务
    ThreadPoolExecutor executor = (ThreadPoolExecutor) threadPool.getExecutor();
    FutureTask<?> f = (FutureTask<?>) executor.submit(sa);
    sa.add(new FutureCompleterWithConfigurableInterrupt(f, shouldInterruptThread, executor));

    return sa;
}
```

1. 调用`subscription.isUnsubscribed()`是否未订阅，如果未订阅则返回。
2. 创建`ScheduledAction`
3. 将`ScheduledAction`添加到订阅(`subscription`)
4. 使用`threadPool`提交任务，并创建`FutureCompleterWithConfigurableInterrupt`添加到订阅(`sa`)
5. 返回订阅(`sa`)



























[1]: /articles/Spring-Cloud/Hystrix(三)——正常执行逻辑.html

> http://youdang.github.io/2016/02/05/translate-hystrix-wiki-how-it-works/#%E4%BE%9D%E8%B5%96%E9%9A%94%E7%A6%BB
> https://github.com/YunaiV/Blog/blob/master/Hystrix/2018_10_25_Hystrix%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20%E2%80%94%E2%80%94%20%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%EF%BC%88%E4%BA%8C%EF%BC%89%E4%B9%8B%E6%89%A7%E8%A1%8C%E9%9A%94%E7%A6%BB%E7%AD%96%E7%95%A5.md

