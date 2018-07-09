---
title: Hystrix(二)——命令执行方式
date: 2018/07/06 10:53:25
---

# 构建HystrixCommand或者HystrixObservableCommand对象

使用Hystrix的第一步是创建一个`HystrixCommand`或者`HystrixObservableCommand`对象来表示你需要发给依赖服务的请求。你可以向构造器传递任意参数。

若只期望服务每次返回单一的回应，按如下方式构造一个`HystrixCommand`即可：

```java
HystrixCommand command = new HystrixCommand(arg1, arg2);
```

若期望依赖服务返回一个`Observable`，并应用`Observable`模式监听依赖服务的回应，可按如下方式构造一个`HystrixObservableCommand`：

```java
HystrixObservableCommand command = new HystrixObservableCommand(arg1, arg2);
```
<!-- more -->
# 命令执行方式

Hystrix命令在抽象类`HystrixCommand`中，有四种调用方式

## toObservable

未做订阅，返回`Observable`对象。只有在订阅该对象时，才会发出请求，然后再依赖服务响应时，通过注册的`Subscriber`得到返回结果

## observe

调用`toObservable`方法的基础上，向`Observable`注册`rx.subjects.ReplaySubject`发起订阅。`ReplaySubject`会发射所有来自原始`Observable`的数据给观察者，无论它们是何时订阅的。

```java
public Observable<R> observe() {
    // us a ReplaySubject to buffer the eagerly subscribed-to Observable
    ReplaySubject<R> subject = ReplaySubject.create();
    // eagerly kick off subscription
    final Subscription sourceSubscription = toObservable().subscribe(subject);
    // return the subject that can be subscribed to later while the execution has already started
    return subject.doOnUnsubscribe(new Action0() {
        @Override
        public void call() {
            sourceSubscription.unsubscribe();
        }
    });
}
```

## queue

调用`toObservable()`方法的基础上，调用：

- `Observable.toBlocking`方法：将`Observable`转换成阻塞的`rx.observables.BlockingObservable`
- `BlockingObservable.toFuture`方法：返回可获得`run()`抽象方法执行结果的`Future`。`run()`方法由子类实现，执行正常的业务逻辑。

```java
toObservable().toBlocking().toFuture();
```

## execute

在调用`queue()`方法的基础上，调用`Future.get()`方法，同步返回`run()`执行结果

```java
public R execute() {
    try {
        return queue().get();
    } catch (Exception e) {
        throw Exceptions.sneakyThrow(decomposeException(e));
    }
}
```

# toObservable

```java
public Observable<R> toObservable() {
    final AbstractCommand<R> _cmd = this;

    ......

    return Observable.defer(new Func0<Observable<R>>() {
        @Override
        public Observable<R> call() {
             /* CAS保证命令只执行一次 */
            if (!commandState.compareAndSet(CommandState.NOT_STARTED, CommandState.OBSERVABLE_CHAIN_CREATED)) {
                IllegalStateException ex = new IllegalStateException("This instance can only be executed once. Please instantiate a new instance.");
                //TODO make a new error type for this
                throw new HystrixRuntimeException(FailureType.BAD_REQUEST_EXCEPTION, _cmd.getClass(), getLogMessagePrefix() + " command executed multiple times - this is not permitted.", ex, null);
            }
            // 命令开始时间戳
            commandStartTimestamp = System.currentTimeMillis();

            // 打印日志
            if (properties.requestLogEnabled().get()) {
                // log this command execution regardless of what happened
                if (currentRequestLog != null) {
                    currentRequestLog.addExecutedCommand(_cmd);
                }
            }
            // 缓存开关，缓存KEY
            final boolean requestCacheEnabled = isRequestCachingEnabled();
            final String cacheKey = getCacheKey();

            /* 首先从缓存中获取 */
            if (requestCacheEnabled) {
                HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.get(cacheKey);
                if (fromCache != null) {
                    isResponseFromCache = true;
                    return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                }
            }
            
            // 声明执行命令的Observable
            Observable<R> hystrixObservable =
                    Observable.defer(applyHystrixSemantics)
                            .map(wrapWithAllOnNextHooks);

            Observable<R> afterCache;

            // put in cache
            if (requestCacheEnabled && cacheKey != null) {
                // wrap it for caching
                HystrixCachedObservable<R> toCache = HystrixCachedObservable.from(hystrixObservable, _cmd);
                HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.putIfAbsent(cacheKey, toCache);
                if (fromCache != null) {    // 添加失败
                    // another thread beat us so we'll use the cached value instead
                    toCache.unsubscribe();
                    isResponseFromCache = true;
                    return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                } else {                    // 添加成功
                    // we just created an ObservableCommand so we cast and return it
                    afterCache = toCache.toObservable();
                }
            } else {
                afterCache = hystrixObservable;
            }

            return afterCache
                    .doOnTerminate(terminateCommandCleanup)     // perform cleanup once (either on normal terminal state (this line), or unsubscribe (next line))
                    .doOnUnsubscribe(unsubscribeCommandCleanup) // perform cleanup once
                    .doOnCompleted(fireOnCompletedHook);
        }
    });
}
```

`toObservable`通过`defer`操作符声明一个`Observable`。`Observable`的执行流程如下：

1. 调用`isRequestCachingEnabled()`方法判断请求结果缓存这个特性是否被启用。如果缓存特性被启用并且缓存命中，则缓存的回应会立即通过一个`Observable`对象的形式返回。
2. 创建执行命令的`Observable`：`hystrixObservable`。
3. 当缓存特性开启，并且缓存未命中时，创建**订阅了执行命令的Observable**：`HystrixCommandResponseFromCache`

    1. 创建存储到缓存的`Observable`：`toCache`。
    2. 将`toCache`添加到缓存中，返回获取缓存的`Observable`：`fromCache`

        1. 如果添加失败，调用`toCache.unsubscribe()`方法，取消`HystrixCachedObservable`的订阅
        2. 否则调用`toCache.toObservable()`方法，获得缓存`Observable`
        
    3. 当缓存特性未开启，使用执行命令`Observable`




> http://youdang.github.io/2016/02/05/translate-hystrix-wiki-how-it-works/
> https://github.com/YunaiV/Blog/blob/master/Hystrix/2018_10_08_Hystrix%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%20%E2%80%94%E2%80%94%20%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4%E6%96%B9%E5%BC%8F.md

