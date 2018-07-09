---
title: Hystrix(三)——正常执行逻辑
date: 2018/07/07 09:37:25
---

# 正常逻辑执行

`AbstractCommand.toObservable()`方法中，当缓存特性未开启或者缓存未命中时，将`applyHystrixSemantics`传入`Observable.defer`方法中，声明执行命令的`Observable`。
<!-- more -->
## applyHystrixSemantics变量

创建`applyHystrixSemantics`的代码如下：

```java
final Func0<Observable<R>> applyHystrixSemantics = new Func0<Observable<R>>() {
    @Override
    public Observable<R> call() {
        // 当commandState处于UNSUBSCRIBED时，不执行命令
        if (commandState.get().equals(CommandState.UNSUBSCRIBED)) {
            return Observable.never();
        }
        // 返回执行命令的Observable
        return applyHystrixSemantics(_cmd);
    }
};
```

## applyHystrixSemantics方法

```java
private Observable<R> applyHystrixSemantics(final AbstractCommand<R> _cmd) {
    // mark that we're starting execution on the ExecutionHook
    // if this hook throws an exception, then a fast-fail occurs with no fallback.  No state is left inconsistent
    executionHook.onStart(_cmd);

    /* determine if we're allowed to execute */
    if (circuitBreaker.attemptExecution()) {
        // 获得信号量
        final TryableSemaphore executionSemaphore = getExecutionSemaphore();
        // 信号量释放Action
        final AtomicBoolean semaphoreHasBeenReleased = new AtomicBoolean(false);
        final Action0 singleSemaphoreRelease = new Action0() {
            @Override
            public void call() {
                if (semaphoreHasBeenReleased.compareAndSet(false, true)) {
                    executionSemaphore.release();
                }
            }
        };
        // 命令执行失败Action
        final Action1<Throwable> markExceptionThrown = new Action1<Throwable>() {
            @Override
            public void call(Throwable t) {
                eventNotifier.markEvent(HystrixEventType.EXCEPTION_THROWN, commandKey);
            }
        };
        // 信号量获取
        if (executionSemaphore.tryAcquire()) {
            try {
                // 标记executionResult调用开始时间
                /* used to track userThreadExecutionTime */
                executionResult = executionResult.setInvocationStartTime(System.currentTimeMillis());
                // 获得命令执行Observable
                return executeCommandAndObserve(_cmd)
                        .doOnError(markExceptionThrown)
                        .doOnTerminate(singleSemaphoreRelease)
                        .doOnUnsubscribe(singleSemaphoreRelease);
            } catch (RuntimeException e) {
                return Observable.error(e);
            }
        } else {
            return handleSemaphoreRejectionViaFallback();
        }
    } else {
        return handleShortCircuitViaFallback();
    }
}
```

1. 调用`getExecutionSemaphore()`方法获得信号量(`TryableSemaphore`)对象
2. 创建信号量释放Action，用于下面**命令执行Observable**的`doOnTerminate`和`doOnUnsubscribe`方法
3. 创建命令执行失败Action，用于下面**命令执行Observable**的`doOnError`方法
4. 调用`TryableSemaphore.tryAcquire`方法获取信号量
5. 标记`executionResult`调用的开始时间
6. 调用`executeCommandAndObserve`方法，获取**命令执行Observable**
7. 若发生异常，调用`Observable.error`方法返回`Observable`
8. 若信号量(`TryableSemaphore`)使用失败，调用`handleSemaphoreRejectionViaFallback()`方法，处理信号量拒绝的失败回退逻辑
9. 若链路处于熔断状态，调用`handleShortCircuitViaFallback()`方法，处理链路熔断的失败回退逻辑。

## TryableSemaphore

`com.netflix.hystrix.AbstractCommand.TryableSemaphore`是Hystrix定义的信号量接口。代码如下：

```java
interface TryableSemaphore {
    public abstract boolean tryAcquire();
    public abstract void release();
    public abstract int getNumberOfPermitsUsed();
}
```

TryableSemaphore共有两个子类实现：

- TryableSemaphoreNoOp
- TryableSemaphoreActual


### TryableSemaphoreNoOp

`com.netflix.hystrix.AbstractCommand.TryableSemaphoreNoOp`是**无操作**的信号量。代码如下：

```java
static class TryableSemaphoreNoOp implements TryableSemaphore {
    public static final TryableSemaphore DEFAULT = new TryableSemaphoreNoOp();

    @Override
    public boolean tryAcquire() {
        return true;
    }

    @Override
    public void release() {

    }

    @Override
    public int getNumberOfPermitsUsed() {
        return 0;
    }
}
```

`tryAcquire()`方法每次都返回`true`，`release()`方法无任何操作。这是为什么？在Hystrix里提供了**两种执行隔离策略**:

- Thread。该方式不使用信号量，因此使用TryableSemaphoreNoOp，这样每次调用`tryAcquire`都能返回`true`。
- Semaphore。该方式使用信号量，因此使用TryableSemaphoreActual，这样每次调用`tryAcquire`根据情况返回`true/false`。

### TryableSemaphoreActual

`com.netflix.hystrix.AbstractCommand.TryableSemaphoreActual`是真正的信号量实现。不过实际上，`TryableSemaphoreActual`更加像一个**计数器**。代码如下：

```java
static class TryableSemaphoreActual implements TryableSemaphore {
    protected final HystrixProperty<Integer> numberOfPermits;
    private final AtomicInteger count = new AtomicInteger(0);

    public TryableSemaphoreActual(HystrixProperty<Integer> numberOfPermits) {
        this.numberOfPermits = numberOfPermits;
    }

    @Override
    public boolean tryAcquire() {
        int currentCount = count.incrementAndGet();
        if (currentCount > numberOfPermits.get()) {
            count.decrementAndGet();
            return false;
        } else {
            return true;
        }
    }

    @Override
    public void release() {
        count.decrementAndGet();
    }

    @Override
    public int getNumberOfPermitsUsed() {
        return count.get();
    }
}
```

- `numberOfPermits`属性：信号量上限。`com.netflix.hystrix.strategy.properties.HystrixProperty`是一个接口，当其使用类似`com.netflix.hystrix.strategy.properties.archaius.IntegerDynamicProperty`动态属性的实现时，可以实现动态调整信号量的上限，这就是为什么不使用`java.util.concurrent.Semaphore`的原因之一
- `count`属性：信号量使用数量。这是为什么说`TryableSemaphoreActual`更加像一个**计数器**的原因
- 另一个不使用`java.util.concurrent.Semaphore`的原因，`TryableSemaphoreActual`无阻塞获取信号量的需求，使用`AtomicInteger`可以达到更加轻量级的实现。

### getExecutionSemaphore

```java
/**
 * 命令与信号量的映射
 */
protected static final ConcurrentHashMap<String, TryableSemaphore> executionSemaphorePerCircuit = new ConcurrentHashMap<String, TryableSemaphore>();

protected TryableSemaphore getExecutionSemaphore() {
    if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.SEMAPHORE) {
        if (executionSemaphoreOverride == null) {
            TryableSemaphore _s = executionSemaphorePerCircuit.get(commandKey.name());
            if (_s == null) {
                // we didn't find one cache so setup
                executionSemaphorePerCircuit.putIfAbsent(commandKey.name(), new TryableSemaphoreActual(properties.executionIsolationSemaphoreMaxConcurrentRequests()));
                // assign whatever got set (this or another thread)
                return executionSemaphorePerCircuit.get(commandKey.name());
            } else {
                return _s;
            }
        } else {
            return executionSemaphoreOverride;
        }
    } else {
        // return NoOp implementation since we're not using SEMAPHORE isolation
        return TryableSemaphoreNoOp.DEFAULT;
    }
}
```

`getExecutionSemaphore`方法根据执行隔离策略的不同，返回不同的信号量实现：

- `Thread`。该方式不使用信号量，因此返回`TryableSemaphoreNoOp`。
- `Semaphore`。该方式使用信号量，因此返回`TryableSemaphoreActual`。不同命令的信号量都保存在`executionSemaphorePerCircuit`。

## executeCommandAndObserve

`executeCommandAndObserve`方法返回**命令执行的Observable**。代码如下：

```java
private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
    // 获取请求上下文
    final HystrixRequestContext currentRequestContext = HystrixRequestContext.getContextForCurrentThread();
    
    // doOnNext中的回调。即命令执行之前执行的操作
    final Action1<R> markEmits = new Action1<R>() {
        @Override
        public void call(R r) {
            if (shouldOutputOnNextEvents()) {
                executionResult = executionResult.addEvent(HystrixEventType.EMIT);
                eventNotifier.markEvent(HystrixEventType.EMIT, commandKey);
            }
            if (commandIsScalar()) {
                long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                circuitBreaker.markSuccess();
            }
        }
    };
    
    // doOnCompleted中的回调。命令执行完毕后执行的操作
    final Action0 markOnCompleted = new Action0() {
        @Override
        public void call() {
            if (!commandIsScalar()) {
                long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                circuitBreaker.markSuccess();
            }
        }
    };

    // onErrorResumeNext中的回调。命令执行失败后的回退逻辑
    final Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {
        @Override
        public Observable<R> call(Throwable t) {
            circuitBreaker.markNonSuccess();
            Exception e = getExceptionFromThrowable(t);
            executionResult = executionResult.setExecutionException(e);
            if (e instanceof RejectedExecutionException) {
                return handleThreadPoolRejectionViaFallback(e);
            } else if (t instanceof HystrixTimeoutException) {
                return handleTimeoutViaFallback();
            } else if (t instanceof HystrixBadRequestException) {
                return handleBadRequestByEmittingError(e);
            } else {
                /*
                 * Treat HystrixBadRequestException from ExecutionHook like a plain HystrixBadRequestException.
                 */
                if (e instanceof HystrixBadRequestException) {
                    eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
                    return Observable.error(e);
                }

                return handleFailureViaFallback(e);
            }
        }
    };
    
    // doOnEach中的回调。`Observable`每发射一个数据都会执行这个回调，设置请求上下文
    final Action1<Notification<? super R>> setRequestContext = new Action1<Notification<? super R>>() {
        @Override
        public void call(Notification<? super R> rNotification) {
            setRequestContextIfNeeded(currentRequestContext);
        }
    };

    Observable<R> execution;
    if (properties.executionTimeoutEnabled().get()) {
        execution = executeCommandWithSpecifiedIsolation(_cmd)
                .lift(new HystrixObservableTimeoutOperator<R>(_cmd));
    } else {
        execution = executeCommandWithSpecifiedIsolation(_cmd);
    }

    return execution.doOnNext(markEmits)
            .doOnCompleted(markOnCompleted)
            .onErrorResumeNext(handleFallback)
            .doOnEach(setRequestContext);
}
```

1. 获取请求上下文
2. 分别定义doOnNext中的回调、doOnCompleted中的回调、onErrorResumeNext中的回调、doOnEach中的回调
3. 调用`executeCommandWithSpecifiedIsolation`方法，获得**命令执行Observable**
4. 若执行命令超时特性开启，调用`Observable.lift`方法实现执行命令超时功能。

## executeCommandWithSpecifiedIsolation

`executeCommandWithSpecifiedIsolation`方法返回**执行命令Observable**。代码如下：

```java
private Observable<R> executeCommandWithSpecifiedIsolation(final AbstractCommand<R> _cmd) {
    if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.THREAD) {
        // mark that we are executing in a thread (even if we end up being rejected we still were a THREAD execution and not SEMAPHORE)
        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                executionResult = executionResult.setExecutionOccurred();
                if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                    return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                }

                metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.THREAD);

                if (isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT) {
                    // the command timed out in the wrapping thread so we will return immediately
                    // and not increment any of the counters below or other such logic
                    return Observable.error(new RuntimeException("timed out before executing run()"));
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.STARTED)) {
                    //we have not been unsubscribed, so should proceed
                    HystrixCounters.incrementGlobalConcurrentThreads();
                    threadPool.markThreadExecution();
                    // store the command that is being run
                    endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                    executionResult = executionResult.setExecutedInThread();
                    /**
                     * If any of these hooks throw an exception, then it appears as if the actual execution threw an error
                     */
                    try {
                        executionHook.onThreadStart(_cmd);
                        executionHook.onRunStart(_cmd);
                        executionHook.onExecutionStart(_cmd);
                        return getUserExecutionObservable(_cmd);
                    } catch (Throwable ex) {
                        return Observable.error(ex);
                    }
                } else {
                    //command has already been unsubscribed, so return immediately
                    return Observable.error(new RuntimeException("unsubscribed before executing run()"));
                }
            }
        }).doOnTerminate(new Action0() {
            @Override
            public void call() {
                if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.TERMINAL)) {
                    handleThreadEnd(_cmd);
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.TERMINAL)) {
                    //if it was never started and received terminal, then no need to clean up (I don't think this is possible)
                }
                //if it was unsubscribed, then other cleanup handled it
            }
        }).doOnUnsubscribe(new Action0() {
            @Override
            public void call() {
                if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.UNSUBSCRIBED)) {
                    handleThreadEnd(_cmd);
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.UNSUBSCRIBED)) {
                    //if it was never started and was cancelled, then no need to clean up
                }
                //if it was terminal, then other cleanup handled it
            }
        }).subscribeOn(threadPool.getScheduler(new Func0<Boolean>() {
            @Override
            public Boolean call() {
                return properties.executionIsolationThreadInterruptOnTimeout().get() && _cmd.isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT;
            }
        }));
    } else {
        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                executionResult = executionResult.setExecutionOccurred();
                if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                    return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                }

                metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.SEMAPHORE);
                // semaphore isolated
                // store the command that is being run
                endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                try {
                    executionHook.onRunStart(_cmd);
                    executionHook.onExecutionStart(_cmd);
                    return getUserExecutionObservable(_cmd);  //the getUserExecutionObservable method already wraps sync exceptions, so this shouldn't throw
                } catch (Throwable ex) {
                    //If the above hooks throw, then use that as the result of the run method
                    return Observable.error(ex);
                }
            }
        });
    }
}
```

根据**执行隔离策略**的不同，创建不同的`命令执行Observable`。

### 执行隔离策略为Thread:

1. 调用`executionResult.setExecutionOccurred()`，标记executionResult执行已发生
2. 将`commandState`设置为`USER_CODE_EXECUTED`。若设置失败，调用`Observable.error`方法返回Observable
3. 检查是否超时。若超时，调用`Observable.error`方法返回Observable
4. 将`threadState`设置为`ThreadState.STARTED`。若设置失败，调用`Observable.error`方法返回Observable
5. 调用`getUserExecutionObervable`方法创建**命令执行Observable**。若发生异常，调用`Observable.error`方法返回Observable
6. 调用`doOnTerminate`方法添加Action0
7. 调用`doOnUnsubscribe`方法添加Action0
8. 调用`Observable.subscribeOn`方法指定Observable自身在哪个调度器上执行。`ThreadPool.getScheduler`方法获得Hystrix自定义实现的RxJava Scheduler

### 执行隔离策略为Semaphore：

1. 调用`executionResult.setExecutionOccurred()`，标记executionResult执行已发生
2. 将`commandState`设置为`USER_CODE_EXECUTED`。若设置失败，调用`Observable.error`方法返回Observable
3. 调用`getUserExecutionObervable`方法创建**命令执行Observable**。若发生异常，调用`Observable.error`方法返回Observable

## getUserExecutionObervable

`getUserExecutionObervable`方法创建`命令执行Observable`

```java
private Observable<R> getUserExecutionObservable(final AbstractCommand<R> _cmd) {
    Observable<R> userObservable;

    try {
        userObservable = getExecutionObservable();
    } catch (Throwable ex) {
        // the run() method is a user provided implementation so can throw instead of using Observable.onError
        // so we catch it here and turn it into Observable.error
        userObservable = Observable.error(ex);
    }

    return userObservable
            .lift(new ExecutionHookApplication(_cmd))
            .lift(new DeprecatedOnRunHookApplication(_cmd));
}
```

调用`getExecutionObservable`方法创建`命令执行Observable`。`getExecutionObservable`方法是个抽象方法，`HystrixCommand`实现了该方法。

若发生异常，调用`Observable.error`方法返回Observable


## HystrixCommand.getExecutionObservable

调用`HystrixCommand.getExecutionObservable`方法创建**命令执行Observable**

```java
final protected Observable<R> getExecutionObservable() {
    return Observable.defer(new Func0<Observable<R>>() {
        @Override
        public Observable<R> call() {
            try {
                return Observable.just(run());
            } catch (Throwable ex) {
                return Observable.error(ex);
            }
        }
    }).doOnSubscribe(new Action0() {
        @Override
        public void call() {
            // Save thread on which we get subscribed so that we can interrupt it later if needed
            executionThread.set(Thread.currentThread());
        }
    });
}

protected abstract R run() throws Exception;
```

1. 调用Observable.defer方法创建**命令执行Observable**

    调用`run`方法，运行正常执行逻辑。通过`Observable.just`方法返回创建的Observable
    
2. 调用`doOnSubscribe`方法，添加Action。该操作记录执行线程（`executionThread`）。`executionThread`用于`HystrixCommand.queue()`方法，返回的Future结果，可以调用`Future.cancel`方法
3. `run()`抽象方法，实现该方法，运行正常逻辑。

`run()`抽象方法的实现为`GenericCommand.run()`

## GenericCommand.run()

`GenericCommand.run()`方法的代码如下：

```java
protected Object run() throws Exception {
    LOGGER.debug("execute command: {}", getCommandKey().name());
    return process(new Action() {
        @Override
        Object execute() {
            return getCommandAction().execute(getExecutionType());
        }
    });
}
```

`AbstractHystrixCommand.process`方法中调用`Action`的`execute()`方法。

`execute()`方法：

- 首先调用`getCommandAction()`方法获取`CommandAction`，我们的示例中获取到的是`MethodExecutionAction`。
- 然后调用`MethodExecutionAction.execute`方法，传入`ExecutionType`参数，我们的示例中传入的是`ExecutionType.SYNCHRONOUS`。

代码如下：

```java
public Object execute(ExecutionType executionType) throws CommandActionExecutionException {
    return executeWithArgs(executionType, _args);
}

public Object executeWithArgs(ExecutionType executionType, Object[] args) throws CommandActionExecutionException {
    if(ExecutionType.ASYNCHRONOUS == executionType){
        Closure closure = AsyncClosureFactory.getInstance().createClosure(metaHolder, method, object, args);
        return executeClj(closure.getClosureObj(), closure.getClosureMethod());
    }

    return execute(object, method, args);
}

private Object execute(Object o, Method m, Object... args) throws CommandActionExecutionException {
    Object result = null;
    try {
        m.setAccessible(true); // suppress Java language access
        if (isCompileWeaving() && metaHolder.getAjcMethod() != null) {
            result = invokeAjcMethod(metaHolder.getAjcMethod(), o, metaHolder, args);
        } else {
            result = m.invoke(o, args);
        }
    } catch (IllegalAccessException e) {
        propagateCause(e);
    } catch (InvocationTargetException e) {
        propagateCause(e);
    }
    return result;
}
```

我们看到最终在`MethodExecutionAction.execute`方法中通过反射调用其中的`Method`，返回执行结果。



> https://github.com/YunaiV/Blog/blob/master/Hystrix/2018_10_22_Hystrix%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20%E2%80%94%E2%80%94%20%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%EF%BC%88%E4%B8%80%EF%BC%89%E4%B9%8B%E6%AD%A3%E5%B8%B8%E6%89%A7%E8%A1%8C%E9%80%BB%E8%BE%91.md

