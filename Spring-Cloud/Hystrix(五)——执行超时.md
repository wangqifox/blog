---
title: Hystrix(五)——执行超时
date: 2018/07/09 20:41:25
---

开启执行超时功能，需要配置：

- `HystrixCommandProperties.executionTimeoutEnabled`：执行命令超时功能开关

    - 值：Boolean
    - 默认值：`true`

- `HystrixCommandProperties.executionTimeoutInMilliseconds`：执行命令超时时长

    - 值：Integer
    - 单位：毫秒
    - 默认值：1000毫秒

在`AbstractCommand.executeCommandAndObserve`方法中，如果`HystrixCommandProperties`属性中开启了执行命令超时开关，则调用`lift`实现对执行命令超时的监控。代码如下：

```java
if (properties.executionTimeoutEnabled().get()) {
    execution = executeCommandWithSpecifiedIsolation(_cmd)
            .lift(new HystrixObservableTimeoutOperator<R>(_cmd));
}
```

<!-- more -->

# HystrixObservableTimeoutOperator

`HystrixObservableTimeoutOperator`为执行命令加入超时功能。代码如下：

```java
private static class HystrixObservableTimeoutOperator<R> implements Operator<R, R> {

    final AbstractCommand<R> originalCommand;

    public HystrixObservableTimeoutOperator(final AbstractCommand<R> originalCommand) {
        this.originalCommand = originalCommand;
    }

    @Override
    public Subscriber<? super R> call(final Subscriber<? super R> child) {
        // 创建订阅
        final CompositeSubscription s = new CompositeSubscription();
        // 添加订阅
        // if the child unsubscribes we unsubscribe our parent as well
        child.add(s);

        //capture the HystrixRequestContext upfront so that we can use it in the timeout thread later
        final HystrixRequestContext hystrixRequestContext = HystrixRequestContext.getContextForCurrentThread();

        TimerListener listener = new TimerListener() {

            @Override
            public void tick() {
                // if we can go from NOT_EXECUTED to TIMED_OUT then we do the timeout codepath
                // otherwise it means we lost a race and the run() execution completed or did not start
                // 如果可以将命令中的isCommandTimedOut从NOT_EXECUTED设置为TIMED_OUT，说明命令超时了，于是我们执行命令超时的任务
                // 否则说明run()方法已经执行完了，这里就不用执行了
                if (originalCommand.isCommandTimedOut.compareAndSet(TimedOutStatus.NOT_EXECUTED, TimedOutStatus.TIMED_OUT)) {
                    // report timeout failure
                    originalCommand.eventNotifier.markEvent(HystrixEventType.TIMEOUT, originalCommand.commandKey);

                    // shut down the original request
                    s.unsubscribe();

                    final HystrixContextRunnable timeoutRunnable = new HystrixContextRunnable(originalCommand.concurrencyStrategy, hystrixRequestContext, new Runnable() {

                        @Override
                        public void run() {
                            child.onError(new HystrixTimeoutException());
                        }
                    });


                    timeoutRunnable.run();
                    //if it did not start, then we need to mark a command start for concurrency metrics, and then issue the timeout
                }
            }

            @Override
            public int getIntervalTimeInMilliseconds() {
                return originalCommand.properties.executionTimeoutInMilliseconds().get();
            }
        };

        final Reference<TimerListener> tl = HystrixTimer.getInstance().addTimerListener(listener);

        // set externally so execute/queue can see this
        originalCommand.timeoutTimer.set(tl);

        /**
         * If this subscriber receives values it means the parent succeeded/completed
         */
        Subscriber<R> parent = new Subscriber<R>() {

            @Override
            public void onCompleted() {
                if (isNotTimedOut()) {
                    // stop timer and pass notification through
                    tl.clear();
                    child.onCompleted();
                }
            }

            @Override
            public void onError(Throwable e) {
                if (isNotTimedOut()) {
                    // stop timer and pass notification through
                    tl.clear();
                    child.onError(e);
                }
            }

            @Override
            public void onNext(R v) {
                if (isNotTimedOut()) {
                    child.onNext(v);
                }
            }

            // 通过CAS判断是否超时
            private boolean isNotTimedOut() {
                // if already marked COMPLETED (by onNext) or succeeds in setting to COMPLETED
                return originalCommand.isCommandTimedOut.get() == TimedOutStatus.COMPLETED ||
                        originalCommand.isCommandTimedOut.compareAndSet(TimedOutStatus.NOT_EXECUTED, TimedOutStatus.COMPLETED);
            }

        };

        // 添加订阅
        // if s is unsubscribed we want to unsubscribe the parent
        s.add(parent);

        return parent;
    }

}
```

1. 创建订阅`s`，添加订阅`s`到`child`的订阅
2. 获得`HystrixRequestContext`。因为下面`listener`的执行不再当前线程，HystrixRequestContext基于ThreadLocal实现
3. 创建执行命令的超时监听器`TimerListener`。当超过执行命令的时长`TimerListener.getIntervalTimeInMilliseconds`时，`TimerListener.tick()`方法触发调用

    1. 调用`originalCommand.isCommandTimedOut.compareAndSet(TimedOutStatus.NOT_EXECUTED, TimedOutStatus.TIMED_OUT)`尝试将命令中的`isCommandTimedOut`从`NOT_EXECUTED`设置为`TIMED_OUT`。如果设置成功说明命令超时了，于是我们执行命令超时的任务。否则说明run()方法已经执行完了，这里就不用执行了
    2. 发送命令超时事件
    3. 调用`s.unsubscribe()`方法取消订阅`s`。注意，不同执行隔离策略此处的表现不同：

        - `ExecutionIsolationStrategy.THREAD`：该策略下提供取消订阅，并且命令执行超时，强制取消命令的执行。
        - `ExecutionIsolationStrategy.SEMAPHORE`：该策略下未提供取消订阅时对超时执行命令的取消。所以，在选择执行隔离策略时要注意这块

    4. 执行`child.onError(new HystrixTimeoutException())`方法，处理`HystrixTimeoutException`异常。该异常会被`handleFallback`处理

        - `HystrixContextRunnable`设置前面获得的`HystrixRequestContext`到`Callable.run()`所在线程的`HystrixRequestContext`并继续执行。

4. 添加`TimerListener`到定时器，监听命令的超时执行
5. 设置`TimerListener`到`AbstractCommand.timeoutTimer`属性。用于执行超时等场景下的`TimerListener`的清理。以下方法有通过该属性对`TimerListener`的清理：

    - AbstractCommand.handleCommandEnd()
    - AbstractCommand.cleanUpAfterResponseFromCache()

6. 创建新的`Subscriber`(`parent`)。在传参的`child`的基础上，增加了对是否执行超时的判断(`isNotTimeOut()`)和`TimerListener`的清理
7. 添加订阅`parent`到`s`的订阅
8. 返回`parent`

## TimerListener

`com.netflix.hystrix.util.HystrixTimer.TimerListener`是Hystrix定时任务监听器接口。

```java
public static interface TimerListener {
    public void tick();
    public int getIntervalTimeInMilliseconds();
}
```

- `tick()`方法：时间达到（超时）执行的逻辑
- `getIntervalTimeInMilliseconds()`方法：返回超时时长

# HystrixTimer

`com.netflix.hystrix.util.HystrixTimer`是Hystrix的定时器。

目前有如下场景使用：

- 执行命令超时任务
- 命令批量执行

构造方法如下：

```java
public class HystrixTimer {
    private static HystrixTimer INSTANCE = new HystrixTimer();
    AtomicReference<ScheduledExecutor> executor = new AtomicReference<ScheduledExecutor>();
    
    private HystrixTimer() {
        // private to prevent public instantiation
    }

    /**
     * Retrieve the global instance.
     */
    public static HystrixTimer getInstance() {
        return INSTANCE;
    }
}
```

- `INSTANCE`是单例的静态属性
- `executor`属性是定时任务执行器(`ScheduledExecutor`)

`addTimerListener(TimerListener)`方法提交定时监听器，生成定时任务。代码如下：

```java
public Reference<TimerListener> addTimerListener(final TimerListener listener) {
    startThreadIfNeeded();
    // add the listener

    Runnable r = new Runnable() {

        @Override
        public void run() {
            try {
                listener.tick();
            } catch (Exception e) {
                logger.error("Failed while ticking TimerListener", e);
            }
        }
    };

    ScheduledFuture<?> f = executor.get().getThreadPool().scheduleAtFixedRate(r, listener.getIntervalTimeInMilliseconds(), listener.getIntervalTimeInMilliseconds(), TimeUnit.MILLISECONDS);
    return new TimerReference(listener, f);
}
```

1. 调用`startThreadIfNeeded()`方法保证`executor`延迟初始化已完成
2. 创建定时任务Runnable。在`Runnable.run()`方法里，调用`TimerListener.tick()`方法
3. 调用`scheduleAtFixedRate`方法生成定时任务。延迟`IntervalTimeInMilliseconds`时间后执行`listener.tick()`方法
4. 返回`listener` + `f`创建TimerReference返回

## TimerReference

`com.netflix.hystrix.util.HystrixTimer.TimerReference`是Hystrix的定时任务引用。代码如下：

```java
private static class TimerReference extends SoftReference<TimerListener> {
    private final ScheduledFuture<?> f;

    TimerReference(TimerListener referent, ScheduledFuture<?> f) {
        super(referent);
        this.f = f;
    }

    @Override
    public void clear() {
        super.clear();
        // stop this ScheduledFuture from any further executions
        f.cancel(false);
    }
}
```

- 通过`clear()`方法，可以取消定时任务的执行

# ScheduledExecutor

`com.netflix.hystrix.util.HystrixTimer.ScheduledExecutor`是Hystrix定时任务执行器。代码如下：

```java
static class ScheduledExecutor {
    // 定时任务线程池执行器
    volatile ScheduledThreadPoolExecutor executor;
    // 是否初始化
    private volatile boolean initialized;

    /**
     * We want this only done once when created in compareAndSet so use an initialize method
     */
    public void initialize() {
        // 从HystrixTimerThreadPoolProperties.corePoolSize配置中获取线程池的大小coreSize
        HystrixPropertiesStrategy propertiesStrategy = HystrixPlugins.getInstance().getPropertiesStrategy();
        int coreSize = propertiesStrategy.getTimerThreadPoolProperties().getCorePoolSize().get();
        // 创建ThreadFactory
        ThreadFactory threadFactory = null;
        if (!PlatformSpecific.isAppEngineStandardEnvironment()) {
            threadFactory = new ThreadFactory() {
                final AtomicInteger counter = new AtomicInteger();

                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "HystrixTimer-" + counter.incrementAndGet());
                    thread.setDaemon(true);
                    return thread;
                }

            };
        } else {
            threadFactory = PlatformSpecific.getAppEngineThreadFactory();
        }
        // 创建ScheduledThreadPoolExecutor
        executor = new ScheduledThreadPoolExecutor(coreSize, threadFactory);
        // 已初始化
        initialized = true;
    }

    public ScheduledThreadPoolExecutor getThreadPool() {
        return executor;
    }

    public boolean isInitialized() {
        return initialized;
    }
}
```


































> https://github.com/YunaiV/Blog/blob/master/Hystrix/2018_10_28_Hystrix%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20%E2%80%94%E2%80%94%20%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%EF%BC%88%E4%B8%89%EF%BC%89%E4%B9%8B%E6%89%A7%E8%A1%8C%E8%B6%85%E6%97%B6.md

