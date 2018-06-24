---
title: ThreadPoolExecutor
date: 2018/01/13 15:33:25
---

在web开发中，服务器需要接受并处理请求，所以会为一个请求来分配一个线程来进行处理。如果每次请求都新创建一个线程的话实现起来非常简单，但是存在一个问题：

如果并发的请求数量非常多，但每个线程执行的时间很短，这样就会频繁地创建和销毁线程，如此一来会大大降低系统的效率。可能出现服务器为每个请求创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。

那么有没有一种办法使执行完一个任务，并不被销毁，而是可以继续执行其他的任务呢？

这就是线程池的目的了。线程池为线程生命周期的开销和资源不足问题提供了解决方案。通过对多个任务重用线程，线程创建的开销被分摊到了多个任务上。

什么时候使用线程池？

- 单个任务处理时间比较短
- 需要处理的任务数量很大

使用线程池的好处：

- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗
- 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行
- 提高线程的可管理性。线程时稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

## Executor框架接口

Executor框架是一个根据一组执行策略调动，调度，执行和控制的异步任务的框架，目的是提供一种将“任务提交”与“任务如何运行”分离开来的机制。

JUC中有三个Executor接口：

- Executor：一个运行新任务的简单接口
- ExecutorService：扩展了Executor接口。添加了一些用来管理执行器生命周期和任务生命周期的方法
- ScheduledExecutorService：扩展了ExecutorService。支持Future和定期执行任务。

### Executor接口

```java
public interface Executor {
    void execute(Runnable command);
}
```

Executor接口只有一个execute方法，用来替代通常创建或启动线程的方法。

### ExecutorService接口

ExecutorService接口继承自Executor接口，提供了管理终止的方法，以及可为跟踪一个或多个异步任务执行状况而生成的Future方法。增加了`shutDown()`、`shutDownNow()`、`invokeAll()`、`invokeAny()`、`submit()`等方法。

### ScheduledExecutorService接口

ScheduledExecutorService扩展ExecutorService接口并增加了schedule方法。调用schedule方法可以在指定的延时后执行一个Runnable或者Callable任务。ScheduledExecutorService接口还定义了按照指定时间间隔定期执行任务的`scheduleAtFixedRate()`方法和`scheduleWithFixedDelay()`方法。

## 几个重要的字段

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

`ctl`是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段，它包含两部分的信息：线程池的运行状态(runState)和线程池内有效线程的数量(workerCount)，这里可以看到，使用了Integer类型来保存，高3位保存runState，低29位保存workerCount。COUNT_BITS就是29，CAPACITY就是1左移29位减1(29个1)，这个常量表示workerCount的上限值，大约是5亿。

下面再介绍下线程池的运行状态. 线程池一共有五种状态, 分别是:

1. RUNNING ：能接受新提交的任务，并且也能处理阻塞队列中的任务；
2. SHUTDOWN：关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方法进入该状态）；
3. STOP：不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
4. TIDYING：如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
5. TERMINATED：在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做。
    - 进入TERMINATED的条件如下：
    - 线程池不是RUNNING状态；
    - 线程池状态不是TIDYING状态或TERMINATED状态；
    - 如果线程池状态是SHUTDOWN并且workerQueue为空；
    - workerCount为0；
    - 设置TIDYING状态成功。
<!--more-->

下图为线程池的状态转换过程：

![threadpool-status](media/threadpool-status.png)


ctl相关方法

```java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

- runStateOf：获取运行状态(高3位)
- workerCountOf：获取活动线程数(低29位)
- ctlOf：获取运行状态和活动线程数的值

## execute

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    
    int c = ctl.get();
    // 如果当前活动线程数小于corePoolSize，则新建一个线程放入线程池中。并把任务添加到该线程中
    if (workerCountOf(c) < corePoolSize) {
        /**
         * addWorker中的第二个参数表示限制添加线程的数量是根据corePoolSize来判断还是maximumPoolSize来判断
         * 如果为true，根据corePoolSize来判断
         * 如果为false，根据maximumPoolSize来判断
         */
        if (addWorker(command, true))
            return;
        // 如果添加失败，则重新获取ctl值
        c = ctl.get();
    }
    /**
     * 如果当前线程池是运行状态并且任务添加到队列成功
     * 如果能插入数据，offer返回true；否则offer返回false
     */
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        /**
         * 再次判断线程池的运行状态
         * 如果不是运行状态，由于之前已经把command添加到workQueue中了，这时需要移除该command
         * 执行过后通过handler使用使用拒绝策略对该任务进行处理，整个方法返回
         */
        if (! isRunning(recheck) && remove(command))
            reject(command);
        /**
         * 如果执行线程为0，要增加一个执行线程，保证有一个线程来执行任务
         * 这里传入的参数标识：
         * 1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动
         * 2. 第二个参数为false，将线程池的有效线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断
         * 如果执行线程数大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行
         */
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    /**
     * 如果执行到这里，有两种情况：
     * 1. 线程池已经不是RUNNING状态
     * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满
     * 这是，再次调用addWorker方法，但第二个参数传入为false，将线程池的有效线程数量的上限设置为maximumPoolSize
     * 如果失败则拒绝该任务
     */
    else if (!addWorker(command, false))
        reject(command);
}
```

1. 如果`workerCount < corePoolSize`，则创建并启动一个线程来执行新提交的任务
2. 如果`workerCount >= corePoolSize`
    1. 线程池内的阻塞队列未满，则将任务添加到阻塞队列中。
    2. 线程池内的阻塞队列已满
        1. 如果`workerCount < maximumPoolSize`，则创建并启动一个线程来执行新提交的任务
        2. 如果`workerCount >= maximumPoolSize`，则根据拒绝策略来处理该任务，默认的处理方式是直接抛出异常

这里需要注意一下`addWorker(null, false)`也就是创建一个线程，但并没有传入任务，因为任务已经被添加到workQueue中了，所以worker在执行的时候，会直接从workerQueue中获取任务。所以在`workerCountOf(recheck) == 0`时执行`addWorker(null, false)`也是为了保证线程池在RUNNING状态下必须有一个线程来执行任务。

execute方法执行流程如下：

![executo](media/executor.png)


## addWorker

addWorker方法的主要工作是在线程池中创建一个新的线程并执行，firstTask参数用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程是否少于corePoolSize，false表示新增线程前需要判断当前线程数是否少于maximumPoolSize：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        // rs为运行状态
        int rs = runStateOf(c);
        
        /**
         * 这个if判断
         * 如果rs >= SHUTDOWN，则表示此时不再接受新任务
         * 接着判断以下3个条件，只要有1个不满足，则返回false
         * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
         * 2. firstTask为空
         * 3. 阻塞队列不为空
         *
         * 首先考虑rs == SHUTDOWN的情况
         * 这种情况下不会接受新提交的任务，所以在firstTask不为空的时候会返回false
         * 然后，如果firstTask为空，并且workQueue也为空，则返回false
         * 因为队列中已经没有任务了，不需要再添加线程了
         */
        if (rs >= SHUTDOWN && !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;

        for (;;) {
            // 获取线程数
            int wc = workerCountOf(c);
            /**
             * 如果wc超过CAPACITY，也就是ctl的低29位的最大值，返回false
             * 这里的core是addWorker方法的第二个参数，如果为true表示根据corePoolSize来比较，如果为false则根据maximumPoolSize来比较。
             */
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 尝试增加workerCount，如果成功，则跳出第一个for循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // 如果增加workerCount失败，则重新获取ctl的值
            c = ctl.get();
            // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 根据firstTask来创建Worker对象
        w = new Worker(firstTask);
        // 每个Worker对象都会创建一个线程
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
                /**
                 * rs < SHUTDOWN表示是RUNNING状态
                 * 如果rs是RUNNING状态或者rs是SHUTDOWN状态并且firstTask为null，向线程池中添加线程
                 * 因为在SHUTDOWN时不会再添加新的任务，但还是会执行workQueue中的任务
                 */
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 启动线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

注意一下这里的t.start()这个语句，启动时会调用Worker类中的run方法，Worker本身实现了Runnable接口，所以一个Worker类型的对象也是一个线程。

## Worker类

线程池中的每一个线程被封装成一个Worker对象，ThreadPool维护的其实就是一组Worker对象。

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    private static final long serialVersionUID = 6138294804551838833L;

    final Thread thread;
    Runnable firstTask;
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

Worker类继承了AQS，并实现了Runnable接口，注意其中的firstTask和thread属性：firstTask用它来保存传入的任务；thread是在调用构造方法时通过ThreadFactory来创建的线程，是用来处理任务的线程。

在调用构造方法时，需要把任务传入，这里通过getThreadFactory().newThread(this);来新建一个线程，newThread方法传入的参数是this，因为Worker本身继承了Runnable接口，也就是一个线程，所以一个Worker对象在启动的时候会调用Worker类中的run方法。

Worker继承了AQS，使用AQS来实现独占锁的功能。为什么不使用ReentrantLock来实现呢？可以看到tryAcquire方法，它是不允许重入的，而ReentrantLock是允许重入的：

1. lock方法一旦获取了独占锁，表示当前线程正在执行任务中；
2. 如果正在执行任务，则不应该中断线程；
3. 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断；
4. 线程池在执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲的线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；
5. 之所以设置为不可重入，是因为我们不希望任务在调用像setCorePoolSize这样的线程池控制方法时重新获取锁。如果使用ReentrantLock，它是可重入的，这样如果在任务中调用了如setCorePoolSize这类线程池控制的方法，会中断正在运行的线程。(setCorePoolSize会调用interruptIdleWorkers)

**所以，Worker继承自AQS，用于判断线程是否空闲以及是否可以被中断。**

此外，在构造方法中执行了setState(-1);，把state变量设置为-1，为什么这么做呢？是因为AQS中默认的state是0，如果刚创建了一个Worker对象，还没有执行任务时，这时就不应该被中断，看一下tryAquire方法：

```java
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

tryAcquire方法是根据state是否是0来判断的，所以，setState(-1);将state设置为-1是为了禁止在执行任务前对线程进行中断。

正因为如此，在runWorker方法中会先调用Worker对象的unlock方法将state设置为0。(调用了tryRelease将state设置为0)

## runWorker

在Worker类中的run方法调用了runWorker方法来执行任务，runWorker方法的代码如下：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 获取第一个任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 允许中断
    w.unlock();
    // 是否因为异常退出循环
    boolean completedAbruptly = true;
    try {
        // 如果task为空，则通过getTask来获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            if ((runStateAtLeast(ctl.get(), STOP) || 
                (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && 
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

这里说明一下第一个if判断，目的是：

- 如果线程池正在停止，那么要保证当前线程是中断状态
- 如果不是的话，则要保证当前线程不是中断状态

这里要考虑在执行该if语句期间可能也执行了shutdownNow方法，shutdownNow方法会把状态设置为STOP，回顾一下STOP状态：

> 不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态。

STOP状态要中断线程池中的所有线程，而这里使用Thread.interrupted()来判断是否中断是为了确保在RUNNING或者SHUTDOWN状态时线程是非中断状态的，因为Thread.interrupted()方法会复位中断的状态。

总结一下runWorker方法的执行过程：

1. while循环不断地通过getTask()方法获取任务；
2. getTask()方法从阻塞队列中取任务；
3. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态；
4. 调用task.run()执行任务；
5. 如果task为null则跳出循环，执行processWorkerExit()方法；
6. runWorker方法执行完毕，也代表着Worker中的run方法执行完毕，销毁线程。

这里的beforeExecute方法和afterExecute方法在ThreadPoolExecutor类中是空的，留给子类来实现。

completedAbruptly变量来表示在执行任务过程中是否出现了异常，在processWorkerExit方法中会对该变量的值进行判断。

## getTask

getTask方法用来从阻塞队列中获取任务

```java
private Runnable getTask() {
    // timeOut变量的值表示上次从阻塞队列中取任务时是否超时
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        /**
         * 如果线程池状态rs >= SHUTDOWN，也就是非RUNNING状态，再进行以下判断：
         * 1. rs >= STOP，线程池是否正在stop
         * 2. 阻塞队列是否为空
         * 如果以上条件满足，则将workerCount减1并返回null
         * 因为如果当前线程池状态的值是SHUTDOWN或以上时，不允许再向阻塞队列中添加任务
         */
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        /**
         * timed变量用于判断是否需要进行超时控制
         * allowCoreTheadTimeOut默认是false，也就是核心线程不允许进行超时
         * wc > corePoolSize，表示当前线程池中的线程数量大于核心线程数量
         * 对于超过核心线程数量的这些线程，需要进行超时控制
         */
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        /**
         * wc > maximumPoolSize的情况是因为可能在此方法执行阶段同时执行了setMaximumPoolSize方法
         * timed && timeOut 如果为true，表示当前操作需要进行超时控制，并且上次从阻塞队列中获取任务发生了超时
         * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1
         * 如果减1失败，则返回重试
         * 如果wc == 1时，也就说明当前线程是线程池中唯一一个线程了
         */
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            /**
             * 根据timed来判断，如果为true，则通过阻塞队列的poll方法进行超时控制，如果在keepAliveTime时间内没有获取到任务，则返回null
             * 否则通过take方法，如果这时队列为空，则take方法会阻塞直到队列不为空
             */
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            // 如果r == null，说明已经超时，timeOut设置为true
            timedOut = true;
        } catch (InterruptedException retry) {
            // 如果获取任务时当前线程发生了中断，则设置timeOut为false并返回循环重试
            timedOut = false;
        }
    }
}
```

这里重要的地方是第二个if判断，目的是控制线程池的有效线程数量。在执行execute方法时，如果当前线程池的数量超过了corePoolSize且小于maximumPoolSize，并且workQueue已满时，则可以增加工作线程，但这时如果超时没有获取到任务，也就是timeOut为true的情况，说明workQueue已经为空了，也就说明了当前线程池中不需要那么多线程来执行任务了，可以把多于corePoolSize数量的线程销毁掉，保持线程数量在corePoolSize即可。

什么时候会销毁？当然是runWorker方法执行完之后，也就是Worker中的run方法执行完，由JVM自行回收。

getTask方法返回null时，在runWorker方法中会跳出while循环，然后会执行`processWorkerExit`方法。

## processWorkerExit

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    /**
     * 如果completedAbruptly值为true，则说明线程执行时出现了异常，需要将workerCount减1
     * 如果线程执行时没有出现异常，说明在getTask()方法中已经对workerCount进行了减1操作，这里就不必再减了
     */
    if (completedAbruptly)
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 统计完成的任务数
        completedTaskCount += w.completedTasks;
        // 从workers中移除，也就表示着从线程池中移除了一个工作线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    
    // 根据线程池状态进行判断是否结束线程池
    tryTerminate();

    int c = ctl.get();
    /**
     * 当线程池是RUNNING或SHUTDOWN状态时，如果worker是异常结束，那么会直接addWorker
     * 如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker
     * 如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize
     */
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

至此，processWorkerExit执行完之后，工作线程被销毁，以上就是整个工作线程的生命周期，从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取任务，然后执行任务，如果getTask返回null，进入processWorkerExit方法，整个线程结束：

![threadpool-lifecycle](media/threadpool-lifecycle.png)


## tryTerminate

tryTerminate方法根据线程池状态进行判断是否结束线程池

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        /**
         * 当前线程池的状态为以下几种情况时，直接返回：
         * 1. RUNNING，因为还在运行中，不能停止
         * 2. TIDYING或TERMINATED，因为线程池中已经没有正在运行的线程了
         * 3. SHUTDOWN并且等待队列非空，这时要执行完workQueue中的task
         */
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        // 如果线程数量不为0，则中断一个空闲的工作线程，并返回
        // 如果线程无法响应中断，则无法强行退出线程
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 这里尝试设置状态为TIDYING，如果设置成功，则调用terminated方法
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // terminated方法默认什么都不做，留给子类实现
                    terminated();
                } finally {
                    // 设置状态为TERMINATED，此时调用isTerminated方法才返回true
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

`interruptIdleWorkers(ONLY_ONE)`的作用是因为在getTask方法中执行`workQueue.take()`时，如果不执行中断会一直阻塞。在下面介绍的shutdown方法中，会中断所有空闲的工作线程，如果在执行shutdown时工作线程没有空闲，然后又去调用了getTask方法，这时如果workQueue中没有任务了，调用`workQueue.take()`时就会一直阻塞。所以每次在工作线程结束时调用tryTerminate方法来尝试中断一个空闲工作线程，避免在队列为空时取任务一直阻塞的情况。

## shutdown

shutdown方法要将线程池切换到SHUTDOWN状态，并调用interruptIdleWorkers方法请求中断所有空闲的worker，最后调用tryTerminate尝试结束线程池。

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 安全策略判断
        checkShutdownAccess();
        // 切换状态为SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断空闲线程
        interruptIdelWorkers();
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    // 尝试结束线程池
    tryTerminate();
}
```

这里思考一个问题：在runWorker方法中，执行任务时对Worker对象w进行了lock操作，为什么要在执行任务的时候对每个工作线程都加锁呢？

下面仔细分析一下：

- 在getTask方法中，如果这时线程池的状态是SHUTDOWN并且workQueue为空，那么就应该返回null来结束这个工作线程，而使线程池进入SHUTDOWN状态需要调用shutdown方法
- shutdown方法会调用interruptIdleWorkers来中断空闲的线程，interruptIdleWorkers持有mainLock，会遍历workers来逐个判断工作线程是否空闲。但getTask方法中没有mainLock
- 在getTask中，如果判断当前线程池状态是RUNNING，并且阻塞队列为空，那么会调用`workQueue.take()`进行阻塞
- 如果在判断当前线程池状态是RUNNING后，这时调用了shutdown方法把状态改为了SHUTDOWN，这时如果不进行中断，那么当前的工作线程在调用了`workQueue.take()`后会一直阻塞而不会被销毁，因为在SHUTDOWN状态下不允许再有新的任务添加到workQueue中，这样一来线程池永远都关闭不了了
- 由上可知，shutdown方法与getTask方法(从队列中获取任务时)存在竞争条件
- 解决这一问题就需要用到线程的中断，也就是为什么要用interruptIdleWorkers方法。在调用`workQueue.take()`时，如果发现当前线程在执行之前或者执行期间是中断状态，则会抛出InterruptedException，解除阻塞的状态
- 但是要中断工作线程，还要判断工作线程是否是空闲的，如果工作线程正在处理任务，就不应该发生中断
- 所以Worker继承自AQS，在工作线程处理任务时会进行lock，`interruptIdleWorkers`在进行中断时会使用tryLock来判断该工作线程是否正在处理任务，如果tryLock返回true，说明该工作线程当前未执行任务，这时才可以被中断(执行线程阻塞在workQueue.take()中)。

## interruptIdleWorkers

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
```

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 判断该工作线程是否正在处理任务，如果tryLock返回true，说明该工作线程当前未执行任务，这时才可以被中断
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

interruptIdleWorkers遍历workers中所有的工作线程，若线程没有被中断且tryLock获取锁成功，就中断该线程。

为什么需要持有mainLock？因为workers是HashSet类型的，不能保证线程安全

## shutdownNow

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        // 中断所有工作线程，无论是否空闲
        interruptWorkers();
        // 取出队列中没有执行的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

shutdownNow方法与shutdown方法类似，不同的地方在于：

1. 设置状态为STOP
2. 中断所有工作线程，无论是否是空闲的
3. 取出阻塞队列中没有被执行的任务并返回

shutdownNow方法执行完之后调用tryTerminate方法，目的就是使线程池的状态设置为TERMINATED

## interruptWorkers

```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
```

`interruptWorkers`不检测工作线程是否空闲，调用`interruptIfStarted`强制中断线程。

```java
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```


## 线程池的监控

通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

- getTaskCount：线程池已经执行的和未执行的任务总数
- getCompletedTaskCount：线程池已完成的任务数量，该值小于等于taskCount
- getLargestPoolSize：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了maximumPoolSize
- getPoolSize：线程池当前的线程数量
- getActiveCount：当前线程池中正在执行任务的线程数量

通过这些方法，可以对线程池进行监控，在ThreadPoolExecutor类中提供的几个空方法，如beforeExecute方法，afterExecute方法和terminated方法，可以扩展这些方法在执行前或执行后增加一些新的操作，例如统计线程池的执行任务的时间等，可以继承自ThreadPoolExecutor来进行扩展。


> http://www.ideabuffer.cn/2017/04/04/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%9AThreadPoolExecutor/

