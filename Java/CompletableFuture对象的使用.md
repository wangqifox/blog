---
title: CompletableFuture对象的使用
date: 2018/09/04 15:36:00
---

在[Callable、Future、FutureTask原理][1]一文中，我们介绍了`Future`的使用以及原理。`Future`虽然可以实现获取异步执行结果的需求，但是它有着显而易见的缺点：

<!-- more -->

1. `Future`没有提供通知机制，我们无法得知Future什么时候完成
2. 想要获得执行结果，要么使用阻塞，在`future.get()`的地方等待future返回的结果，这时又变成同步操作。要么使用`isDone()`轮询地判断`Future`是否完成，这样会耗费CPU的资源。

在JDK8之前，我们可以使用第三方库来解决这个问题。Netty、Guava分别扩展了Java的`Future`接口，方便做异步编程。

JDK8新增了一个`CompletableFuture`类。`CompletableFuture`类吸收了所有Guava中`ListenableFuture`和`SettableFuture`的特征，还提供了其他强大的功能，让Java拥有了完整的非阻塞编程模型：Future、Promise和Callback

`CompletableFuture`能够将回调放到与任务不同的线程中执行，也能将回调作为继续执行的同步函数，在与任务相同的线程中执行。它避免了传统回调最大的问题，那就是能够将控制流分离到不同的事件处理器中。

`CompletableFuture`弥补了Future模式的缺点。在异步的任务完成后，需要用其结果继续操作时，无需等待。可以直接通过`thenAccept`、`thenApply`、`thenCompose`等方式将前面异步处理的结果交给另外一个异步事件处理线程来处理。

# 创建CompletableFuture对象

`public static <U> CompletableFuture<U> completedFuture(U value)`，是一个静态辅助方法，用来返回一个已经计算好的`CompletableFuture`。

`completedFuture`方法的使用方式如下所示：

```java
@Test
@Test
public void test01() {
    CompletableFuture<String> cf = CompletableFuture.completedFuture("message");
    logger.info("isDone {}", cf.isDone());
    logger.info(cf.getNow(null));
}
```

其中`getNow`方法返回`CompletableFuture`当前的执行结果，如果没有执行完成则返回默认值。

执行结果：

```
17:10:52.318 [main] INFO completable_future.p3.CompletableFutureTest - isDone true
17:10:52.322 [main] INFO completable_future.p3.CompletableFutureTest - message
```

以下四个静态方法用来为一段异步执行的代码创建`CompletableFuture`对象：

```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

以`Async`结尾并且没有指定`Executor`的方法会使用`ForkJoinPool.commonPool()`作为它的线程池执行异步代码。

`runAsync`方法以`Runnable`函数式接口参数类型为参数，所以`CompletableFuture`的计算结果为空。

`supplyAsync`方法以`Supplier<U>`函数式接口类型为参数，`CompletableFuture`的计算结果类型为`U`

`runAsync`方法的使用方式如下所示：

```java
@Test
public void test02() throws InterruptedException {
    CompletableFuture<Void> cf = CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });

    logger.info("isDone 1 " + cf.isDone());
    Thread.sleep(2000);
    logger.info("isDone 2 " + cf.isDone());
    logger.info("result {}", cf.join());
}
```

执行结果：

```
17:11:43.143 [main] INFO completable_future.p3.CompletableFutureTest - isDone 1 false
17:11:45.150 [main] INFO completable_future.p3.CompletableFutureTest - isDone 2 true
17:11:45.150 [main] INFO completable_future.p3.CompletableFutureTest - result null
```

`supplyAsync`方法的使用方式如下所示：

```java
@Test
public void test02_1() throws InterruptedException {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });

    logger.info("isDone 1 " + cf.isDone());
    Thread.sleep(2000);
    logger.info("isDone 2 " + cf.isDone());
    logger.info("result {}", cf.join());
}
```

执行结果：

```
17:11:52.902 [main] INFO completable_future.p3.CompletableFutureTest - isDone 1 false
17:11:54.907 [main] INFO completable_future.p3.CompletableFutureTest - isDone 2 true
17:11:54.907 [main] INFO completable_future.p3.CompletableFutureTest - result hello
```

从结果上我们也可以看到，`runAsync`和`supplyAsync`的区别就是：`supplyAsync`能够返回执行结果，而`runAsync`不会。

# 计算结果完成时的处理

当`CompletableFuture`的计算结果完成，或者抛出异常的时候，我们可以执行特定的`Action`。主要是下面的方法：

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn)
```

可以看到`Action`的类型是`BiConsumer<? super T, ? super Throwable>`，它可以处理正常的计算结果，或者异常情况。

方法不以`Async`结尾，意味着`Action`使用相同的线程执行，而`Async`可能会使用其他的线程去执行（如果使用相同的线程池，也可能会被同一个线程选中执行）。

注意这几个方法都会返回`CompletableFuture`，当`Action`执行完毕后它的结果返回原始的`CompletableFuture`的计算结果或者返回异常。

`whenComplete`方法的使用方式如下所示：

```java
@Test
public void test03() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = cf.whenComplete((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
        }
    });

    logger.info(cf.join());
    logger.info(cf1.join());
}
```

执行结果如下：

```
18:54:03.213 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:54:04.220 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
18:54:04.220 [main] INFO completable_future.p3.CompletableFutureTest - hello
18:54:04.220 [main] INFO completable_future.p3.CompletableFutureTest - hello
```

可以看到，正常情况下`whenComplete`返回`supplyAsync`执行的结果。

如果执行过程中抛出异常，`whenComplete`也可以接收到异常然后处理：

```java
@Test
public void test03_1() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (true) {
            throw new RuntimeException("exception");
        }
        return "hello";
    });
    cf.whenComplete((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
        } else {
            logger.error(throwable.getMessage());
        }
    });

    while (!cf.isDone()) {}
}
```

执行结果如下：

```
18:15:30.632 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:15:31.635 [ForkJoinPool.commonPool-worker-1] ERROR completable_future.p3.CompletableFutureTest - java.lang.RuntimeException: exception
```

`exceptionally`方法返回一个新的`CompletableFuture`，当原始的`CompletableFuture`抛出异常的时候，就会触发这个`CompletableFuture`的计算，调用`function`计算值，否则如果原始的`CompletableFuture`正常计算完后，这个新的`CompletableFuture`也计算完成，它的值和原始的`CompletableFuture`的计算的值相同。也就是这个`exceptionally`方法用来处理异常的情况。

`exceptionally`方法的使用方式如下所示：

```java
@Test
public void test05() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (true) {
            throw new RuntimeException("exception");
        }
        return "hello";
    });
    cf.whenComplete((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
        } else {
            logger.error(throwable.getMessage());
        }
    });
    CompletableFuture<String> cf1 = cf.exceptionally(throwable -> {
        logger.error(throwable.getMessage());
        return "exception happened";
    });

    logger.info(cf1.join());
}
```

执行结果如下：

```
18:38:31.461 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:38:32.467 [ForkJoinPool.commonPool-worker-1] ERROR completable_future.p3.CompletableFutureTest - java.lang.RuntimeException: exception
18:38:32.467 [ForkJoinPool.commonPool-worker-1] ERROR completable_future.p3.CompletableFutureTest - java.lang.RuntimeException: exception
18:38:32.467 [main] INFO completable_future.p3.CompletableFutureTest - exception happened
```

可以看到，当执行过程抛出异常时，会触发`exceptionally`的执行，并返回`exceptionally`的返回值。

如果执行过程中没有抛出异常：

```java
@Test
public void test05() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    cf.whenComplete((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
        } else {
            logger.error(throwable.getMessage());
        }
    });
    CompletableFuture<String> cf1 = cf.exceptionally(throwable -> {
        logger.error(throwable.getMessage());
        return "exception happened";
    });

    logger.info(cf1.join());
}
```

执行结果如下：

```
18:42:55.469 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:42:56.476 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
18:42:56.476 [main] INFO completable_future.p3.CompletableFutureTest - hello
```

可以看到，如果执行过程中没有抛出异常`exceptionally`不会触发，它返回的值就是`supplyAsync`执行返回的原始值。

下面一组方法虽然也返回`CompletableFuture`对象，但是对象的值和原来的`CompletableFuture`计算的值不同。当原先的`CompletableFuture`的值计算完成或者抛出异常的时候，会触发这个`CompletableFuture`对象的计算，结果由`BiFunction`参数计算而得。因此这组方法兼有`whenComplete`和转换的两个功能。

```java
public <U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn)
public <U> CompletableFuture<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor)
```

同样，不以`Async`结尾的方法由原来的线程计算，以`Async`结尾的方法由默认的线程池`ForkJoinPool.commonPool()`或者指定的线程池`executor`运行。

`handle`方法的使用方式如下所示：

```java
@Test
public void test06() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = cf.handle((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
            return s + " world";
        } else {
            logger.error(throwable.getMessage());
            return "exception happened";
        }

    });

    logger.info(cf1.join());
}
```

执行结果如下：

```
18:47:03.524 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:47:04.529 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
18:47:04.530 [main] INFO completable_future.p3.CompletableFutureTest - hello world
```

如果执行过程抛出异常：

```java
@Test
public void test06() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (true) {
            throw new RuntimeException("exception");
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = cf.handle((s, throwable) -> {
        if (throwable == null) {
            logger.info(s);
            return s + " world";
        } else {
            logger.error(throwable.getMessage());
            return "exception happened";
        }

    });

    logger.info(cf1.join());
}
```

执行结果如下：

```
18:48:25.769 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
18:48:26.775 [ForkJoinPool.commonPool-worker-1] ERROR completable_future.p3.CompletableFutureTest - java.lang.RuntimeException: exception
18:48:26.776 [main] INFO completable_future.p3.CompletableFutureTest - exception happened
```

可以看到，`handle`方法接收执行结果和异常，处理之后返回新的结果。

# 转换

`CompletableFuture`可以作为monad和functor。由于回调风格的实现，我们不必因为等待一个计算完成而阻塞着调用线程，而是告诉`CompletableFuture`当计算完成的时候请执行某个`function`。而且我们还可以将这些操作串联起来，或者将`CompletableFunction`组合起来。

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

这一组函数的功能是当原来的`CompletableFuture`计算完后，将结果传递给函数`fn`，将`fn`的结果作为新的`CompletableFuture`计算结果。因此它的功能相当于将`CompletableFuture<T>`转换成`CompletableFuture<U>`。

需要注意的是，这些转换并不是马上执行的，也不会阻塞，而是在前一个stage完成后继续执行。

它们与`handle`方法的区别在于`handle`方法会处理正常计算值和异常，因此它可以屏蔽异常，避免异常继续抛出。而`thenApply`方法只是用来处理正常值，因此一旦有异常就会抛出。

`thenApply`方法的使用方式如下：

```java
@Test
public void test07() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = cf.thenApply(new Function<String, String>() {
        @Override
        public String apply(String s) {
            logger.info(s);
            return s + " world";
        }
    });

    logger.info(cf1.join());
}
```

执行结果如下：

```
20:22:24.537 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:22:25.542 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
20:22:25.543 [main] INFO completable_future.p3.CompletableFutureTest - hello world
```

# 纯消费(执行Action)

上面的方法是当计算完成的时候，会生成新的计算结果（`thenApply`, `handle`），或者返回同样的计算结果`whenComplete`。`CompletableFuture`还提供了一种处理结果的方法，只对结果执行`Action`，而不返回新的计算值。因此计算值为`Void`：

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
```

看它的参数类型也就明白了，它们是函数式接口`Consumer`，这个接口只有输入，没有返回值。

`thenAccept`方法的使用方式如下：

```java
@Test
public void test08() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    cf.thenAccept(new Consumer<String>() {
        @Override
        public void accept(String s) {
            logger.info(s);
        }
    });

    cf.join();
}
```

执行结果如下：

```
20:28:30.580 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:28:31.588 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
```

## thenAcceptBoth

```java
public <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action)
public <U> CompletableFuture<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor)
```

`thenAcceptBoth`接收另一个`CompletionStage`和`action`，当两个`CompletionStage`都正常完成计算后，就会执行提供的`action`，它用来组合另外一个异步的结果。

`thenAcceptBoth`方法的使用方式如下：

```java
@Test
public void test09() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    cf.thenAcceptBoth(cf1, (s, s2) -> {
        logger.info(s + " " + s2);
    }).join();
}
```

执行结果如下：

```
20:31:40.335 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
20:31:40.335 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:31:42.344 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - hello world
```

## runAfterBoth

```java
public CompletableFuture<Void> runAfterBoth(CompletionStage<?> other, Runnable action)
public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action)
public CompletableFuture<Void> runAfterBothAsync(CompletionStage<?> other, Runnable action, Executor executor)
```

`runAfterBoth`是当两个`CompletionStage`都正常完成计算的时候，执行一个`Runnable`，这个Runnable并不使用计算的结果。

`runAfterBoth`方法的使用方式如下：

```java
@Test
public void test10() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    cf.runAfterBoth(cf1, () -> {
        logger.info("end");
    }).join();
}
```

执行结果如下：

```
20:34:07.085 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
20:34:07.085 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:34:09.094 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - end
```

## thenRun

```java
public CompletableFuture<Void> thenRun(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action)
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
```

`thenRun`当计算完成的时候会执行一个`Runnable`，与`thenAccept`不同，`Runnable`并不使用`CompletableFuture`计算的结果。

因此先前的`CompletableFuture`计算的结果被忽略，返回`Completable<Void>`类型的对象。

`thenRun`方法的使用方式如下：

```java
@Test
public void test11() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });

    cf.thenRun(() -> logger.info("end")).join();
}
```

执行结果如下：

```
20:36:05.833 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:36:06.843 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - end
```

# 组合

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn)
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn, Executor executor)
```

这一组方法接受一个`Function`作为参数，这个`Function`的输入是当前的`CompletableFuture`的计算值，返回结果将是一个新的`CompletableFuture`，这个新的`CompletableFuture`会组合原来的`CompletableFuture`和函数返回的`CompletableFuture`。因此它的功能类似于：

```
A +--> B +---> C
```

`thenCompose`返回的对象并不一定是函数`fn`返回的对象，如果原来的`CompletableFuture`还没有计算出来，它就会生成一个新的组合后的`CompletableFuture`。

`thenCompose`方法的使用方式如下：

```java
@Test
public void test12() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf2 = cf.thenCompose(new Function<String, CompletionStage<String>>() {
        @Override
        public CompletionStage<String> apply(String s) {
            return CompletableFuture.supplyAsync(() -> {
                logger.info(s);
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                return s + " world";
            });
        }
    });
    logger.info(cf2.join());
}
```

执行结果如下：

```
20:41:39.242 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:41:40.253 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
20:41:42.255 [main] INFO completable_future.p3.CompletableFutureTest - hello world
```

## thenCombine

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn)
public <U,V> CompletableFuture<V> thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor)
```

`thenCombine`用来复合另外一个`CompletionStage`的结果，它的功能类似：

```
A +
  |
  +------> C
  +------^
B +
```

两个`CompletionStage`是并行执行的，他们之间没有先后依赖顺序，`other`并不会等待先前的`CompletableFuture`执行完毕后再执行。

从功能上来讲，它们的功能更类似`thenAcceptBoth`，只不过`thenAcceptBoth`是纯消费，它的函数参数没有返回值，而`thenCombine`的函数参数`fn`有返回值。

`thenCombine`方法的使用方式如下：

```java
@Test
public void test13() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    CompletableFuture<String> cf2 = cf.thenCombine(cf1, (s, s2) -> s + " " + s2);
    logger.info(cf2.join());
}
```

执行结果如下：

```
20:45:01.018 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
20:45:01.018 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
20:45:03.028 [main] INFO completable_future.p3.CompletableFutureTest - hello world
```

# Either

`thenAcceptBoth`和`runAfterBoth`是当两个`CompletableFuture`都计算完成，而下面的方法是当任意一个`CompletableFuture`计算完成的时候就会执行。

```java
public CompletableFuture<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action)
public CompletableFuture<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor)

public <U> CompletableFuture<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn)
public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn)
public <U> CompletableFuture<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn, Executor executor)
```

`acceptEither`方法是当任意一个`CompletionStage`完成的时候，`action`这个消费者就会被执行。这个方法返回`CompletableFuture<Void>`

`applyToEither`方法是当任意一个`CompletionStage`完成的时候，`fn`会被执行，它的返回值会当做新的`CompletableFuture<U>`的计算结果

`acceptEither`方法的使用方式如下：

```java
@Test
public void test14() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    cf.acceptEither(cf1, s -> logger.info(s)).join();
}
```

执行结果如下：

```
21:21:36.031 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
21:21:36.031 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
21:21:37.039 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
```

可以看到，当`cf`执行完毕后，`acceptEither`方法就被触发执行了

`applyToEither`方法的使用方式如下：

```java
@Test
public void test15() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    CompletableFuture<String> cf2 = cf.applyToEither(cf1, s -> {
        logger.info(s);
        return s + " end";
    });
    logger.info(cf2.join());
}
```

执行结果如下：

```
21:25:43.441 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
21:25:43.441 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
21:25:44.447 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - hello
21:25:44.448 [main] INFO completable_future.p3.CompletableFutureTest - hello end
```

# allOf 和 anyOf

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```

`allOf`方法是当所有的`CompletableFuture`都执行完后执行计算

`anyOf`方法是当任意一个`CompletableFuture`执行完后就会执行计算，计算的结果相同

`anyOf`和`applyToEither`不同，`anyOf`接受任意多的`CompletableFuture`但是`applyToEither`只是判断两个`CompletableFuture`。`anyOf`返回值的计算结果是参数中其中一个`CompletableFuture`的计算结果，`applyToEither`返回值的计算结果却是要经过`fn`处理的。当然还有静态方法的区别，线程池的选择等。

`allOf`方法的使用方式如下：

```java
@Test
public void test16() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    CompletableFuture.allOf(cf, cf1).whenComplete((v, throwable) -> {
        List<String> list = new ArrayList<>();
        list.add(cf.join());
        list.add(cf1.join());
        logger.info("result {}", list);
    }).join();
}
```

执行结果如下：

```
21:36:36.938 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
21:36:36.938 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
21:36:38.951 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - result [hello, world]
```

`anyOf`方法的使用方式如下：

```java
@Test
public void test17() {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello";
    });
    CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
        logger.info("start");
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "world";
    });

    CompletableFuture<Object> cf2 = CompletableFuture.anyOf(cf, cf1).whenComplete((o, throwable) -> logger.info("result {}", o));
    logger.info("result {}", cf2.join());
}
```

执行结果如下：

```
21:43:12.562 [ForkJoinPool.commonPool-worker-2] INFO completable_future.p3.CompletableFutureTest - start
21:43:12.562 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - start
21:43:13.573 [ForkJoinPool.commonPool-worker-1] INFO completable_future.p3.CompletableFutureTest - result hello
21:43:13.576 [main] INFO completable_future.p3.CompletableFutureTest - result hello
```

























[1]: /articles/Java/Callable、Future、FutureTask原理.html

> https://colobu.com/2016/02/29/Java-CompletableFuture/
> https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html
> https://www.jianshu.com/p/dff9063e1ab6


