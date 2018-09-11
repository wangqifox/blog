---
title: ScheduledThreadPoolExecutor
date: 2018/06/27 10:18:00
---

`ScheduledThreadPoolExecutor`继承`ThreadPoolExecutor`类，实现`ScheduledExecutorService`接口。它是一个支持定时执行或者延时执行的线程池。
<!-- more -->

# ScheduledThreadPoolExecutor的使用

`ScheduledExecutorService`的定义如下：

```java
public interface ScheduledExecutorService extends ExecutorService {
    /**
     * 在给定的delay延时之后，执行command任务。返回的ScheduledFuture表示任务执行完成，其get()方法返回null
     */
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
                                       
    /**
     * 在给定的delay延时之后，执行callable任务。返回的ScheduledFuture可以用于获取结果，或者取消任务的执行
     */
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
    
    /**
     * 在给定的delay延时之后，执行command任务，然后以period为周期循环执行任务
     */
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);
    /**
     * 在给定的initialDelay延时之后，执行command任务，之后的任务总是延时delay时间之后执行下一次的任务
     */
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
}
```

`scheduleAtFixedRate()`方法和`scheduleWithFixedDelay()`方法看起来十分相似，我们来看看他们之间的区别。

## scheduleAtFixedRate

编写一个测试方法：

```java
private static void scheduleAtFixedRate(ScheduledExecutorService service, final int sleepTime) {
    service.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            System.out.println("scheduleAtFixedRate 开始执行时间：" + DateFormat.getTimeInstance().format(new Date()));
            try {
                Thread.sleep(sleepTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            long end = System.currentTimeMillis();
            System.out.println("scheduleAtFixedRate 执行花费时间：" + (end - start));
            System.out.println("scheduleAtFixedRate 执行完成时间：" + DateFormat.getTimeInstance().format(new Date()));
            System.out.println("===============");
        }
    }, 1000, 5000, TimeUnit.MILLISECONDS);
}
```

### 执行时间小于间隔时间

执行`scheduleAtFixedRate`方法：

```java
scheduleAtFixedRate(service, 1000);
```

执行结果：

```
scheduleAtFixedRate 开始执行时间：10:47:55
scheduleAtFixedRate 执行花费时间：1026
scheduleAtFixedRate 执行完成时间：10:47:56
===============
scheduleAtFixedRate 开始执行时间：10:48:00
scheduleAtFixedRate 执行花费时间：1001
scheduleAtFixedRate 执行完成时间：10:48:01
===============
scheduleAtFixedRate 开始执行时间：10:48:05
scheduleAtFixedRate 执行花费时间：1003
scheduleAtFixedRate 执行完成时间：10:48:06
===============
```

可以看到：在任务执行时间小于间隔时间的情况下，程序以起始时间为准则，每隔指定时间执行一次，不受任务执行时间影响。

### 执行时间大于间隔时间

执行`scheduleAtFixedRate`方法：

```java
scheduleAtFixedRate(service, 6000);
```

执行结果：

```
scheduleAtFixedRate 开始执行时间：10:50:20
scheduleAtFixedRate 执行花费时间：6029
scheduleAtFixedRate 执行完成时间：10:50:26
===============
scheduleAtFixedRate 开始执行时间：10:50:26
scheduleAtFixedRate 执行花费时间：6003
scheduleAtFixedRate 执行完成时间：10:50:32
===============
scheduleAtFixedRate 开始执行时间：10:50:32
scheduleAtFixedRate 执行花费时间：6002
scheduleAtFixedRate 执行完成时间：10:50:38
===============
```

可以看到，在任务执行时间大于间隔时间的情况下，此方法不会重新开启一个新的任务进行执行，而是等待原有任务执行完成，马上开启下一个任务进行执行。此时，执行间隔时间已经被打乱。

## scheduleWithFixedRate

编写一个测试方法：

```java
private static void scheduleWithFixedDelay(ScheduledExecutorService service, final int sleepTime) {
    service.scheduleWithFixedDelay(new Runnable() {
        @Override
        public void run() {
            long start = System.currentTimeMillis();
            System.out.println("scheduleWithFixedDelay 开始执行时间：" + DateFormat.getTimeInstance().format(new Date()));
            try {
                Thread.sleep(sleepTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            long end = System.currentTimeMillis();
            System.out.println("scheduleWithFixedDelay 执行花费时间：" + (end - start));
            System.out.println("scheduleWithFixedDelay 执行完成时间：" + DateFormat.getTimeInstance().format(new Date()));
            System.out.println("===============");
        }
    }, 1000, 5000, TimeUnit.MILLISECONDS);
}
```

### 执行时间小于间隔时间

执行`scheduleWithFixedDelay`方法：

```java
scheduleWithFixedDelay(service, 1000);
```

执行结果：

```
scheduleWithFixedDelay 开始执行时间：10:54:26
scheduleWithFixedDelay 执行花费时间：1029
scheduleWithFixedDelay 执行完成时间：10:54:27
===============
scheduleWithFixedDelay 开始执行时间：10:54:32
scheduleWithFixedDelay 执行花费时间：1005
scheduleWithFixedDelay 执行完成时间：10:54:33
===============
scheduleWithFixedDelay 开始执行时间：10:54:38
scheduleWithFixedDelay 执行花费时间：1004
scheduleWithFixedDelay 执行完成时间：10:54:39
===============
```

可以看到：当执行任务小于延迟时间时，第一个任务执行之后，延迟指定时间，然后开始执行第二个任务。

### 执行时间大于间隔时间

执行`scheduleWithFixexDelay`方法：

```java
scheduleWithFixedDelay(service, 6000);
```

执行结果：

```
scheduleWithFixedDelay 开始执行时间：11:28:18
scheduleWithFixedDelay 执行花费时间：6026
scheduleWithFixedDelay 执行完成时间：11:28:24
===============
scheduleWithFixedDelay 开始执行时间：11:28:29
scheduleWithFixedDelay 执行花费时间：6002
scheduleWithFixedDelay 执行完成时间：11:28:35
===============
scheduleWithFixedDelay 开始执行时间：11:28:40
scheduleWithFixedDelay 执行花费时间：6106
scheduleWithFixedDelay 执行完成时间：11:28:46
===============
```

可以看到：当执行任务大于延迟时间时，第一个任务执行之后，延迟指定时间，然后开始执行第二个任务。

无论任务的执行时间长短，`scheduleWithFixedDelay`方法都是当第一个任务执行完成之后，延迟指定时间再开始执行第二个任务。

# ScheduledThreadPoolExecutor原理



> https://blog.csdn.net/wo541075754/article/details/51556198

