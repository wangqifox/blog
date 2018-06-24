---
title: ThreadPoolExecutor分析
date: 2018/01/08 17:12:00
---

public class ThreadPoolExecutor extends AbstractExecutorService

一个ExecutorService，它使用可能的几个池线程之一执行每个提交的任务，通常使用Executors工厂方法配置

线程池可以解决两个不同问题：由于减少了每个任务调用的开销，它们通常可以在执行大量异步任务时提供增强的性能，并且还可以提供绑定和管理资源(包括执行任务集时使用的线程)的方法。每个ThreadPoolExecutor还维护着一些基本的统计数据，如完成的任务数。

为了便于跨大量上下文使用，此类提供了很多可调整的参数和扩展钩子。但是强烈建议程序员使用较为方便的`Executors`工厂方法`Executors.newCachedThreadPool()`（无界线程池，可以进行自动线程回收）、`Executors.newFixedThreadPool(int)`（固定大小线程池）和`Executors.newSingleThreadExecutor()`（单个后台线程），它们均为大多数使用场景预定义了设置。否则，在手动配置和调整此类时，使用以下指导：
<!--more-->
- 核心和最大池大小

    ThreadPoolExecutor将根据corePoolSize(参见getCorePoolSize())和maximumPoolSize(参见getMaximumPoolSize())设置的边界自动调整池大小。当新任务在方法execute(java.lang.Runnable)中提交时，如果运行的线程少于corePoolSize，则创建新线程来处理请求，即使其他辅助线程时空闲的。如果运行的线程多于corePoolSize而少于maximumPoolSize，则仅当队列慢时才创建新线程。如果设置的corePoolSize和maximumPoolSize相同，则创建了固定大小的线程池。如果将maximumPoolSize设置为基本的无界值(如Integer.MAX_VALUE)，则允许池适应任意数量的并发任务。在大多数情况下，核心和最大池大小仅基于构造来设置，不过也可以使用setCorePoolSize(int)和setMaximumPoolSize(int)进行动态更改。
    
- 按需构造

    默认情况下，即使核心线程最初只是在新任务到达时才创建和启动的，也可以使用方法prestartCoreThread()或prestartAllCoreThreads()对其进行动态重写。如果构造带有非空队列的池，则可能希望预先启动线程。
    
- 创建新线程

    使用ThreadFactory创建新线程。如果没有另外说明，则在同一个ThreadGroup中一律使用Executors.defaultThreadFactory()创建线程，并且这些线程具有相同的NORM_PRIORITY优先级和非守护进程状态。通过提供不同的ThreadFactory，可以改变线程的名称、线程组、优先级、守护进程状态，等等。如果从newThread返回null时ThreadFactory未能创建线程，则执行程序将继续运行，但不能执行任何任务。
    
- 保持活动时间

    如果池中当前有多于corePoolSize的线程，则这些多出的线程在空闲时间超过keepAliveTime时将会终止（参见getKeepAliveTime(java.util.concurrent.TimeUnit)）。这提供了当池处于非活动状态时减少资源消耗的方法。如果池后来变得更为活动，则可以创建新的线程。也可以使用方法setKeepAliveTime(long, java.util.concurrent.TimeUnit)动态地更改此参数。使用Long.MAX_VALUE,TimeUnit.NANOSECONDS的值在关闭前有效地从以前的终止状态禁用空闲线程。默认情况下，保持活动策略只在有多于corePoolSizeThreads的线程时应用。但是只要keepAliveTime值非0，allowCoreThreadTimeOut(boolean)方法也可将此超时策略应用于核心线程。
    
- 排序

    所有BlockingQueue都可用于传输和保持提交的任务。可以使用此队列与池大小进行交互：
    
        - 如果运行的线程少于corePoolSize，则Executor始终首选添加新的线程，而不进行排队
        - 如果运行的线程等于或多于corePoolSize，则Executor始终首选将请求加入队列，而不添加新的线程
        - 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出maximumPoolSize，在这种情况下，任务将被拒绝。
       
    排队有三种通用策略：
    
        1. 直接提交。工作队列的默认选项是SynchronousQueue，它将任务直接提交给线程而不保持它们。在此，如果不存在可用于立即运行任务的线程，则试图把任务加入队列将失败，因此会构造一个新的线程。此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。直接提交通常要求无界maximumPoolSize以避免拒绝新提交的任务。当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。
        2. 无界队列。使用无界队列(例如，不具有预定义容量的LinkedBlockingQueue)将导致在所有corePoolSize线程都忙时新任务在队列中等待。这样，创建的线程就不会超过corePoolSize。(因此，maximumPoolSize的值也就无效了)当每个任务完全独立于其他任务，即任务执行互不影响时，适合使用无界队列；例如，在web页服务器中。这种排队可用于处理瞬态突发请求，当命令以超过队列所能处理的平均数连续到达时，此策略允许无界线程具有增长的可能性。
        3. 有界队列。当使用有限的maximumPoolSizes时，有界队列(如ArrayBlockingQueue)有助于防止资源耗尽，但是可能较难调整和控制。队列大小和最大池大小可能需要相互折中：使用大型队列和小型池可以最大限度地降低CPU使用率、操作系统资源和上下文切换开销，但是可能导致人工降低吞吐量。如果任务频繁阻塞(例如，如果它们是I/O边界)，则系统可能为超过您许可的更多线程安排时间。使用小型队列通常要求较大的池大小，CPU使用率较高，但是可能遇到不可接受的调度开销，这样也会降低吞吐量。
        
- 被拒绝的任务

    当Executor已经关闭，并且Executor将有限边界用于最大线程和工作队列容量，且已经饱和时，在方法execute(java.lang.Runnable)中提交的新任务将被拒绝。在以上两种情况下，execute方法都将调用其RejectedExecutionHandler的RejectExecutionHandler.rejectedExecution(java.lang.Runnable, java.util.concurrent.ThreadPoolExecutor)方法。下面提供了四种预定义的处理程序策略：
    
        1. 在默认的ThreadPoolExecutor.AbortPolicy中，处理程序遭到拒绝将抛出运行时RejectedExecutionException
        2. 在ThreadPoolExecutor.CallerRunsPolicy中，线程调用运行该任务的execute本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度
        3. 在ThreadPoolExecutor.DiscardPolicy中，不能执行的任务将被删除
        4. 在ThreadPoolExecutor.DiscardOldestPolicy中，如果执行程序尚未关闭，则位于工作队列头部的任务被删除，然后重试执行程序(如果再次失败，则重复此过程)
        
     定义和使用其他种类的RejectedExecutionHandler类也是可能的，但这样做需要非常小心，尤其是策略仅用于特定容量或排队策略时。

- 钩子(hook)方法

    此类提供protected可重写的beforeExecute(java.lang.Thread, java.lang.Runnable)和afterExecute(java.lang.Runnable, java.lang.Throwable)方法，这两种方法分别在执行每个任务之前和之后调用。它们可用于操纵执行环境；例如，重新初始化ThreadLocal、搜集统计信息或添加日志条目。此外，还可以重新方法terminated()来执行Executor完全终止后需要完成的所有特殊处理。
    
    如果钩子(hook)或回调方法抛出异常，则内部辅助线程将依次失败并突然终止。
    
- 队列维护

    方法getQueue()允许出于监控和调试目的而访问工作队列。强烈反对出于其他任何目的而使用此方法。remove(java.lang.Runnable)和purge()这两种方法可用于在取消大量已排队任务时帮助进行存储回收。
    
- 终止

    程序中不再引用池，且没有剩余线程自动shutdown。如果希望确保回收取消引用的池(即使用户忘记调用shutdown())，则必须安排未使用的线程最终终止：设置适当保持活动时间，使用0核心线程的下边界和/或设置allowCoreThreadTimeOut(boolean)


运行状态：

    - RUNNING: 接收新的任务，且执行队列中的任务
    - SHUTDOWN: 不接收新的任务，但是仍然执行队列中的任务
    - STOP: 不接收新的任务，不执行队列中的任务，且中断正在执行的任务
    - TIDYING: 所有的任务都已经终止了，workerCount为0，线程状态迁移到TIDYING执行terminated()钩子函数
    - TERMINATED: terminated()已经完成
    
运行状态随着时间推移单调递增，但是不需要经过每个状态。状态变迁如下：

    - RUNNING -> SHUTDOWN : 调用shutdown()，也许隐含在finalize()中
    - (RUNNING or SHUTDOWN) -> STOP : 调用shutdownNow()
    - SHUTDOWN -> TIDYING : 队列和池都空了
    - STOP -> TIDYING : 当池空了
    - TIDYING -> TERMINATED : 当terminated()钩子函数调用完成

在awaitTermination()中等待的线程当状态变为TERMINATED的时候返回

## 代码分析:


```java

public class ThreadPoolExecutor extends AbstractExecutorService {

	// ctl是一个原子整数类型，包装了两个概念上的字段：workerCount(有效的线程数量), runState(运行状态)
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	private static final int COUNT_BITS = Integer.SIZE - 3;         // 29
	private static final int CAPACITY   = (1 << COUNT_BITS) - 1;    // 线程数量被限制在(2^29)-1, 11111111111111111111111111111

	// 运行状态保存在高位
	private static final int RUNNING    = -1 << COUNT_BITS;         // 11100000000000000000000000000000
	private static final int SHUTDOWN   =  0 << COUNT_BITS;         // 0
	private static final int STOP       =  1 << COUNT_BITS;         // 100000000000000000000000000000
	private static final int TIDYING    =  2 << COUNT_BITS;         // 1000000000000000000000000000000
	private static final int TERMINATED =  3 << COUNT_BITS;         // 1100000000000000000000000000000

	// Packing and unpacking ctl
	private static int runStateOf(int c)     { return c & ~CAPACITY; }  // 获取高3位
	private static int workerCountOf(int c)  { return c & CAPACITY; }   // 获取低29位
	private static int ctlOf(int rs, int wc) { return rs | wc; }

    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
    
    // CAS增加workerCount的值
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    // CAS减少workerCount的值
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }
    
    // 减去ctl的workerCount域。这仅在线程的突然终止时才被调用(请参阅processWorkerExit)。在getTask中执行其他递减
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }
    
    
    private final BlockingQueue<Runnable> workQueue;
	
	private final ReentrantLock mainLock = new ReentrantLock();
	
	private final HashSet<Worker> workers = new HashSet<Worker>();
	
	private final Condition termination = mainLock.newCondition();
	
	// 曾经达到的最大池大小
	private int largestPoolSize;
	
	// 完成的任务数，只在worker线程结束时更新。只在mainLock中做访问
	private long completedTaskCount;
	
	// 创建新线程的工厂
	private volatile ThreadFactory threadFactory;
	
	// 当线程池满或者关闭时调用的处理程序
	private volatile RejectedExecutionHandler handler;
	
	// 空闲线程等待的超时时间(纳秒)，当线程数超过corePoolSize或者allowCoreThreadTimeOut为true时。否则它们永远等待新的worker
	private volatile long keepAliveTime;
	
	// 如果为false，核心线程即使空闲也保持活跃。如果为true，核心线程等待keepAliveTime的时间，之后结束空闲的核心线程
	private volatile boolean allowCoreThreadTimeOut;
	
	// 核心线程的数量
	private volatile int corePoolSize;
	
	// 最大允许的池数量，注意真实的最大大小由CAPACITY约束
	private volatile int maximumPoolSize;
	
	// 默认的拒绝异常处理程序
	private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();
	
	// 调用者调用shutdown和shutdownNow需要的权限
	private static final RuntimePermission shutdownPerm = new RuntimePermission("modifyThread");
	
	/**
	 * 维护现场运行任务的中断控制状态，以及其他较小的簿记。扩展了AbstractQueuedSynchronizer，以简化获取和释放围绕每个任务执行的锁。这样可以防止意图唤醒等待任务的工作线程的中断，而不是中断正在运行的任务。我们实现一个简单的非可重入互斥锁，而不是使用ReentrantLock，因为我们不希望工作任务在调用诸如setCorePoolSize之类的池控制方法时重新获取锁。另外，为了抑制中断，直到线程实际开始运行任务，我们将锁状态初始化为负值，并在启动时(在runWorker中)清除它。
	 */
	private final class Worker
            extends AbstractQueuedSynchronizer
            implements Runnable
    {
        /**
         * 该类不会被序列化，但是我们提供serialVersionUID来抑制警告
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** 在该worker中运行的线程，如果线程工厂失败了则为null*/
        final Thread thread;
        /** 初始化运行的任务，可能为null*/
        Runnable firstTask;
        /** 每个线程的任务计数器 */
        volatile long completedTasks;

        /**
         * 使用firstTask通过ThreadFactory来创建任务
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** 将主运行循环委托给外部的runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

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
            // 初始化时state == -1
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
    
    /**
     * 将runState转移到指定的状态
     */
    private void advanceRunState(int targetState) {
        for (;;) {
            int c = ctl.get();
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                break;
        }
    }
    
    
    /**
     * 如果当前(状态为关闭关闭且池和队列为空)或者(状态为停止且池为空)将状态转变为终止状态。如果当前可以终止但是workerCount不为0，将中断空闲的worker以确保关闭信号传播。必须在任何可能终止的操作之后调用此方法——减少worker或在关闭期间从队列中删除任务。该方法是非私有的，允许从ScheduledThreadExecutor访问
     *
     * processWorkerExit方法中会尝试调用tryTerminate来终止线程池。这个方法在任何可能导致线程池终止的动作后执行：比如减少workerCount或SHUTDOWN状态下从队列中移除任务
     */
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();

            /**
             * 以下状态直接返回：
             * 1. 线程池还处于RUNNING状态
             * 2. SHUTDOWN状态但是任务队列非空
             * 3. runState >= TIDYING线程池已经停止了或正在停止
             */
            if (isRunning(c) || runStateAtLeast(c, TIDYING) || (runStateOf(c) == SHUTDOWN && !workQueue.isEmpty()))
                return;
            /**
             * 只有以下情形会继续下面的逻辑：结束线程池
             * 1. SHUTDOWN状态，这时不再接受新任务而且任务队列也空了
             * 2. STOP状态，当调用了shutdownNow方法
             */

            // workerCount不为0则还不能停止线程池，而且这时线程都处于空闲等待的状态
            // 需要中断让线程醒过来，醒过来的线程才能继续处理shutdown的信号
            if (workerCountOf(c) != 0) {
                // runWorker方法中w.unlock就是为了可以被中断，getTask方法也处理了中断
                // ONLY_ONE：这里只需要中断1个线程去处理shutdown信号就可以了
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            
            final ReetrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 进入TIDYING状态
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 子类重载：一些资源清理工作
                        terminated();
                    } finally{
                        // TERMINATED状态
                        ctl.set(ctlOf(TERMINATED, 0));
                        // 继续awaitTermination
                        termination.signalAll();
                    }
                    return;
                }
            } finally{
                mainLock.unlock();
            }
        }
    }
    
    /**
     * 如果有安全控制器，确保调用者有关闭线程的权限(详见shutdownPerm)。如果拥有该权限，还要确保调用者有中断每个工作线程的权限。如果SecurityManager将线程单独对待，即使第一次检查通过了，后面的也可能不通过
     */
    private void checkShutdownAccess() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkPermission(shutdownPerm);
            final ReetrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                for (Worker w : workers)
                    security.checkAccess(w.thread);
            } finally {
                mainLock.unlock();
            }
        }
    }
    
    /**
     * 中断所有的线程，即使是活动的。忽略SecurityExceptions(这种情况下，某些线程没有被中断)
     */
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
    
    /**
     * 中断可能正在等待任务的线程，以便它们可以检查终止或配置更新。忽略SecurityExceptions(这种情况下，某些线程没有被中断)
     * 
     * onlyOne: 如果为true，最多中断一个worker。这只在termination开启时在tryTerminate中调用，其他workers不受影响。这种情况下，最多一个等待的worker被中断来传播关闭信号，如果当前所有的线程都在等待。中断任意的线程将确保从shutdown以来所有新来的worker最终都会终止。为了保证最终终止，只需要中断一个空闲线程就够了，但是shutdown()会中断所有空闲的workers，这样冗余workers会立即退出，而不是等待其他任务完成
     */
    private void interruptedIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                // w.tryLock能获取到锁，说明该线程没有在运行，因为runWorker中执行任务会先lock，因此保证了中断的肯定是空闲的线程
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
        } finally{
            mainLock.unlock();
        }
    }
    
    /**
     * interruptedIdleWorkers的正常形式。不用记忆boolean参数的含义
     */
    private void interruptIdleWorkers() {
        interruptedIdleWorkers(false);
    }
    
    private static final boolean ONLY_ONE = true;
    
    /**
     * 调用任务拒绝Exception的函数
     */
    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
    
    /**
     * 执行关闭调用后，运行状态转换后进一步清理。这里没有操作，但ScheduledThreadPoolExecutor用于取消延迟的任务
     */
    void onShutdown() {
    }
    
    /**
     * ScheduledThreadPoolExecutor需要的状态检查，用于启动关闭的任务
     */
    final boolean isRunningOrShutdown(boolean shutdownOK) {
        int rs = runStateOf(ctl.get());
        return rs == RUNNING || (rs == SHUTDOWN && shutdownOK);
    }
    
    /**
     * 将任务队列转移到新的列表中，通常使用drainTo方法。但是如果队列是DelayQueue或者任务poll、drainTo执行删除元素会失败的队列，则一个一个地删除
     */
    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }
    
    /**
     * 检查是否可以针对当前池状态和核心值、最大值添加新worker。如果是这样，则相应地调整worker数量，并且如果可能的话，创建并启动新的worker，运行firstTask作为第一个任务。如果池已经停止了或者即将关闭，该方法返回false。如果线程工厂无法创建线程，也返回false。如果创建失败，不管是线程工厂返回null还是其他异常(通常为Thread.start()中抛出OutOfMemoryError)，我们进行回滚。
     * 
     * firstTask：新线程应该首先运行的任务(如果没有，则返回null)。Workers使用初始的第一个任务来创建(在方法execute()中)，以便在少于corePoolSize线程(这种情况下始终启动一个)或者队列已满时绕过队列(这种情况下，必须绕过队列)。初始的空闲线程通常通过prestartCoreThread来创建或替换其他将死的workers。
     * 
     * core：如果为true则使用corePoolSize，否则使用maximumPoolSize
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c); // 当前线程池状态

            /**
             * 这条语句等价：rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty())
             * 满足下列条件则直接返回false，线程创建失败：
             * rs > SHUTDOWN: STOP || TIDYING || TERMINATED 此时不再接受新的任务，且所有任务执行结束
             * rs == SHUTDOWN, firstTask != null: 此时不再接受任务，但是仍然会执行队列中的任务
             * rs == SHUTDOWN, firstTask == null: 见execute方法的addWorker(null, false)，任务为null&&队列为空
             * 最后一种情况也就是说SHUTDOWN状态下，如果队列不为空还得接着往下执行，为什么？add一个null任务目的到底是什么？
             * 看execute方法只有workCount==0的时候firstTask才会为null，结合这里的条件就是线程池SHUTDOWN了不再接受新任务，但是此时队列不为空，那么还得创建线程把任务给执行完才行
             */
            if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
                return false;

            /**
             * 走到这里的情形：
             * 1. 线程池状态为RUNNING
             * 2. SHUTDOWN状态，但队列中还有任务需要执行
             */
            for (;;) {
                int wc = workerCountOf(c);
                // 超出最大数量
                if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 原子操作递增workCount，操作成功跳出重试的循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();
                if (runStateOf(c) != rs)    // 如果线程池的状态发生变化则重试
                    continue retry;
            }
        }

        // workerCount递增成功

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 并发地访问线程池workers对象必须加锁
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());
                    // RUNNING状态 || SHUTDOWN状态下清理队列中剩余的任务
                    if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive())
                            throw new IllegalThreadStateException();
                        // 将新启动的线程添加到线程池中
                        workers.add(w);
                        // 更新largestPoolSize
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 启动新添加的线程，这个线程首先执行firstTask，然后不停的从队列中取任务执行
                // 当等待keepAliveTime还没有任务执行则该线程结束。见runWorker和getTask方法的代码
                if (workerAdded) {
                    t.start();  // 最终执行的是ThreadPoolExecutor的runWorker方法
                    workerStarted = true;
                }
            }
        } finally {
            // 线程启动失败，则从workers中移除w并递减workerCount
            if (!workerStarted)
                // 递减workerCount会触发tryTerminate方法
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    
    /**
     * 工作线程创建之后回滚
     * - 从workers中删除worker，如果有的话
     * - 减少worker的数量
     * - 重新检查终止，防止该worker阻止进入终止状态
     */
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 对将死的worker进行清理以及簿记。只在worker线程中被调用。除非设置了completedAbruptly，否则假定已经调整了workerCount以退出。该方法从worker集中删除线程，如果由于用户任务异常退出，或者正在运行的workers少于corePoolSize，或者队列不为空但是没有workers，则可能会终止该池或替换worker。
     * 
     * completedAbruptly: worker由于用户异常而退出
     */
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        /**
         * 如果是由于用户异常而退出的，workerCount是没有经过调整的
         * 正常的话在runWorker的getTask方法中workerCount已经被减一了
         */
        if (completedAbruptly)
            decrementWorkerCount();
        
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 累加线程的completedTasks
            completedTaskCount += w.completedTasks;
            // 从线程池中移除超时或者出现异常的线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        // 尝试停止线程池
        tryTerminate();
        
        int c = ctl.get();
        // runState为RUNNING或SHUTDOWN
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                // 线程池最小空闲数，允许core thread超时就是0，否则就是corePoolSize
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // 如果min == 0，但是队列不为空要保证有1个线程来执行队列中的任务
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 线程池还不为空，那就不用担心了
                // 如果worker数量大于等于min，则直接返回，因为前面有workers.remove(w)的代码，该worker已经从workers集合中删除掉了。否则在后面通过addWorker(null, false)添加一个空的worker，以保证worker的数量保持恒定
                if (workerCountOf(c) >= min)
                    return;
            }
            /**
             * 1. 线程异常退出
             * 2. 线程池为空，但是队列中还有任务没有执行，看addWorker方法对这种情况的处理
             */
            addWorker(null, false);
        }
    }
    
    /**
     * 根据当前的配置执行阻塞或者定时等待任务。如果worker因为以下的原因而必须退出则返回null：
     * 1. 有多于maximumPoolSize的workers(因为调用了setMaximumPoolSize)
     * 2. 该池停止了
     * 3. 该池关闭了且队列为空
     * 4. worker等待任务超时了，且超时的workers将被终止
     */
    private Runnable getTask() {
        boolean timedOut = false;
        
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            /**
             * 1. rs > SHUTDOWN 所以rs至少等于STOP，这时不再处理队列中的任务
             * 2. rs = SHUTDOWN 所以rs >= STOP肯定不成立，这时还需要处理队列中的任务，除非队列为空
             * 这两种情况都会返回null，让runWorker退出while循环也就是当前线程结束了，所以必须要decrementWorkerCount
             */
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount(); // 递减workerCount的值
                return null;
            }

            /**
             * 1. RUNNING状态
             * 2. SHUTDOWN状态，但队列中还有任务需要执行
             */
            int wc = workerCountOf(c);

            /**
             * 1. core thread允许被超时，那么超过corePoolSize的线程必定有超时
             * 2. allowCoreThreadTimeOut == false && wc > corePoolSize，一般都是这种情况，core thread即使空闲也不会被回收，只有超过的线程才会
             */
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            /**
             * 线程超时，且wc大于1或者队列为空
             */
            if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))  // workerCount递减，结束当前thread。如果失败则重新检查线程池状态
                    return null;
                continue;
            }
            
            try {
                /**
                 * 1. 以指定的超时时间从列队中取任务
                 * 2. core thread没有超时
                 */
                // 如果没有后续的任务添加到workQueue队列中，则线程池阻塞在workQueue.take()处，等待新的任务到来。于是这个worker就处于等待状态。
                Runnable r = timed ? 
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timeOut = true; // 超时
            } catch (InterruptedException retry) {
                timedOut = false;   // 线程被中断，重试
            }
        }
    }
    
    /**
     * 主要的工作循环。从队列中循环获取任务然后执行，同时对应若干问题：
     * 
     * 1. 开始时可能会有一个初始任务，这种情况下我们不需要获取第一个任务，否则，只要线程在运行，我们都从getTask中获取任务。如果返回了null则worker退出，根据改变了的线程池状态或者配置参数。外部代码抛出的异常也会导致退出，这种情况下，保持completedAbruptly，这通常会导致processWorkerExit替换该线程。
     * 
     * 2. 运行任意任务之前，需要获得锁以防止任务执行时其他池中断，我们确保除非线程池停止了，该线程不会拥有它的中断集。
     * 
     * 3. 每个任务运行之前都会调用beforeExecute，它有可能抛出异常。这种情况下我们退出线程(将completedAbruptly设置为true来打断循环)，不在执行该任务。
     * 
     * 4. 假设beforeExecute正常完成了，我们执行该任务，收集它抛出的任意异常并发送给afterExecute，我们区分RuntimeException, Error(这两个我们保证能捕捉到), 其他的Throwables。因为我们不能在Runnable.run内重新抛出Throwables，所以我们把它们包裹在Errors中(对线程的UncaughtExceptionHandler)。任何抛出的异常都可能导致线程退出。
     * 
     * 5. task.run完成后，我们调用afterExecute，它也会抛出异常，并导致线程退出。根据JLS Sec 14.20，该异常即使task.run抛出也会生效。
     * 
     * 异常机制的净效果是afterExecute和线程的UncaughtExceptionHandler具有与用户代码遇到任何问题一样准确的信息
     * 
     */
     /**
      * 第一次启动会执行初始化传进来的任务firstTask
      * 然后会从workQueue中取任务执行，如果队列为空则等待keepAliveTime这么长时间
      */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        // Worker的构造函数中抑制了线程中断setState(-1)，所以这里需要unlock从而允许中断
        w.unlock();
        // 用于标识是否异常终止，finally中processWorkerExit的方法会有不同逻辑
        // 为true的情况：1.执行任务抛出异常，2.被中断
        boolean completedAbruptly = true;
        try {
            // 如果getTask返回null那么getTask中会将workerCount递减，如果异常了这个递减操作会在processWorkerExit中处理
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 如果线程池停止了，确保线程中断，
                // 否则，保证线程没有被中断。这需要在第二种情况下重新检查，在清除中断时处理shutdownNow
                if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 任务执行前可以插入一些处理，子类重载该方法
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run(); // 执行用户任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 和beforeExecute一样，留给子类去重载
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
            // 结束线程的一些清理工作
            processWorkerExit(w, completedAbruptly);
        }
    }    
    
    public ThreadPoolExecutor(int corePoolSize,
                                int maximumPoolSize,
                                long keepAliveTime,
                                TimeUnit unit,
                                BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), defaultHandler);
    }
    
    public ThreadPoolExecutor(int corePoolSize,
                                int maximumPoolSize,
                                long keepAliveTime,
                                TimeUnit unit,
                                BlockingQueue<Runnable> workQueue,
                                ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            threadFactory, defaultHandler);
    }
    
    public ThreadPoolExecutor(int corePoolSize,
                                int maximumPoolSize,
                                long keepAliveTime,
                                TimeUnit unit,
                                BlockingQueue<Runnable> workQueue,
                                RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), handler);
    }
    
    public ThreadPoolExecutor(int corePoolSize,
                                int maximumPoolSize,
                                long keepAliveTime,
                                TimeUnit unit,
                                BlockingQueue<Runnable> workQueue,
                                ThreadFactory threadFactory,
                                RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    
    /**
     * 在将来的某个时间执行给定的任务。任务可能在新线程或者现有线程池中执行。
     * 
     * 如果无法将任务提交执行，或者此执行程序已关闭，或者因为已达到其容量，则该任务由当前RejectedExecutionHandler处理。
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        /**
         * 执行分3个步骤：
         * 
         * 1. 如果正在运行的线程小于corePoolSize，使用给定的命令启动一个线程作为它第一个任务。addWorker函数自动检查runState、workerCount，避免了无法添加线程时的失败警告。
         * 2. 如果任务成功插入队列，我们仍然需要二次检查线程是否添加成功(已存在的线程从最后的检查之后退出)或者线程池关闭了。所以我们需要重新检查状态，需要的话，如果线程停止了则回滚入队操作，或者如果没有线程的话则开启一个新线程
         * 3. 如果我们不能将任务加入队列，尝试添加一个新线程。如果失败了，则说明线程池关闭了或者队列满了，于是拒绝该任务
         */
        
        int c = ctl.get();
        // 活动线程数 < corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            // 直接启动新的线程。第二个参数true：addWorker中会重新检查workerCount是否小于corePoolSize
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 活动线程数 >= corePoolSize
        // runState为RUNNING && 队列未满
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // double check
            // 非RUNNING状态，则从workQueue中移除任务并拒绝
            if (! isRunning(recheck) && remove(command))
                reject(command);    // 采用线程池指定的策略拒绝任务
            // 线程池处于RUNNING状态 || 线程池处于非RUNNING状态但是任务移除失败
            else if (workerCountOf(recheck) == 0)
                // 这行代码是为了SHUTDOWN状态下没有活动线程，但是队列里还有任务没有执行这种情况
                // 添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
                addWorker(null, false);
        }
        // 两种情况：
        // 非RUNNING状态拒绝信的任务
        // 队列满了启动新的线程失败(workCount > maximumPoolSize)
        else if (!addWorker(command, false))
            reject(command);
    }
    
    /**
     * 将之前提交的正在运行的任务按顺序关闭，不再接受新的任务。如果该线程已经关闭了，该调用也没有其他的影响。
     * 
     * 该方法不会等待之前提交的任务完成执行
     *
     * shutdown方法可能会在finalize被隐式调用
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 线程池状态设为SHUTDOWN，如果已经至少是这个状态那么则直接返回
            advanceRunState(SHUTDOWN);
            /**
             * 注意这里是中断所有空闲线程：
             * runWorker中等待的线程被中断->进入processWorkerExit->tryTerminate方法中会保证队列中剩余的任务得到执行
             */
            interruptIdleWorkers();
            onShutdown();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
    
    /**
     * 尝试停止所有正在执行的任务，停止等待队列的运行，返回正在等待执行的任务列表。这些任务返回前从任务队列中移除。
     * 
     * 该方法不等待正在运行的任务终止。如果要等待，应该使用awaitTermination
     * 
     * 尽量停止正在执行的任务，但是不能保证任务能够被停止。通过Thread.interrupt来取消任务，如果任务不响应中断则该任务永远不会终止。
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock();
        mainLock.lock();
        try {
            checkShutdownAccess();
            // STOP状态：不再接受新任务且不再执行队列中的任务
            advanceRunState(STOP);
            // 中断所有线程
            interruptWorkers();
            // 返回队列中还没有被执行的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
    
    public boolean isShutdown() {
        return ! isRunning(ctl.get());
    }
    
    /**
     * 调用shutdown或者shutdownNow之后，如果还有未完成的任务在返回true。该方法在调试的时候很有用。如果在调用shutdown的很长时间后仍然返回true，表示提交的任务忽略或者禁止了中断，导致没有正常终止
     */
    public boolean isTerminatin() {
        int c = ctl.get();
        return ! isRunning(c) && runStateLessThan(c, TERMINATED);
    } 

    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedExeception {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
    
    /**
     * 当executor不再被引用且没有线程时，调用shutdown
     */
    protected void finalize() {
        shutdown();
    }

    /**
     * 设置用来创建新线程的线程工厂
     */
    public void setThreadFactory(ThreadFactory threadFactory) {
        if (threadFactory == null)
            throw new NullPointerException();
        this.threadFactory = threadFactory;
    }

    /**
     * 返回用于创建新线程的线程工厂
     */
    public ThreadFactory getThreadFactory() {
        return threadFactory;
    }

    /**
     * 设置不可执行任务的处理器
     */
    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        if (handler == null)
            throw new NullPointerException();
        this.handler = handler;
    }

    /**
     * 返回不可执行任务的处理器
     */
    public RejectedExecutionHandler getRejectedExecutionHandler() {
        return handler;
    }

    /**
     * 设置线程的核心数量。覆盖构造器中设置的值。如果新的值小于当前的值，超出的线程会在变为空闲之后终止。如果新的值大于当前的值，则启动在队列中的任务
     */
    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // 我们并不真正知道多少线程是需要的。
            // 作为一个启发式的算法，先启动足够的新workers(到新的core值)，如果队列已经空了则停止
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }

    /**
     * 返回线程的核心数量
     */
    public int getCorePoolSize() {
        return corePoolSize;
    }

    /**
     * 启动核心线程，使其处于等待工作的空闲状态。仅当执行新任务时，此操作才重写默认的启动核心线程策略。如果已启动所有核心线程，此方法将返回false。
     */
    public boolean prestartCoreThread() {
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }

    /**
     * 和prestartCoreThread一样，除了至少启动一个线程即使corePoolSize为0
     */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }

    /**
     * 启动所有的核心线程，使其处于等待工作的空闲状态。仅当执行新任务时，此操作才重写默认的启动核心线程策略。
     */
    public int prestartAllCoreThreads() {
        int n = 0;
        while (addWorker(null, true))
            ++n;
        return n;
    }

    /**
     * 如果线程池允许核心线程当在keepAlive时间内没有任务到达超时并退出，新任务到达时被替换，则返回true。当返回true时，适用于非核心线程的相同的保持策略也同样适用于核心线程。当返回false(默认值)时，由于没有传入任务，核心线程不会终止。
     */
    public boolean allowsCoreThreadTimeOut() {
        return allowCoreThreadTimeOut;
    }

    /**
     * 如果在keep-alive时间内没有任务到达，新任务到达时被替换，设置控制核心线程时超时还是终止的策略。当为false(默认值)时，由于没有传入任务，核心线程将永远不会终止。当为true时，适用于非核心线程的keep-alive策略也同样适用于核心线程。为了避免连续线程替换，keep-alive时间在设置为true时必须大于0。通常应该在使用该池前主动调用此方法。
     */
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptedIdleWorkers();
        }
    }

    /**
     * 设置最大允许的线程数量。覆盖了构造函数中设置的任何值。如果新值比当前值要小，超出的线程会在变为空闲之后终止。
     */
    public void setMaximumPoolSize(int maximumPoolSize) {
        if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
        this.maximumPoolSize = maximumPoolSize;
        if (workerCountOf(ctl.get()) > maximumPoolSize)
            interruptedIdleWorkers();
    }

    /**
     * 获取最大允许的线程数量
     */
    public int getMaximumPoolSize() {
        return maximumPoolSize;
    }

    /**
     * 设置线程在终止前可以保持空闲的时间限制。如果池中的当前线程数多于核心线程数，在不处理任务的情况下等待一段时间之后，多于的线程将被终止。此操作将重写构造方法中设置的任何值。
     */
    public void setKeepAliveTime(long time, TimeUnit unit) {
        if (time < 0)
            throw new IllegalArgumentException();
        if (time == 0 && allowsCoreThreadTimeOut())
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        long keepAliveTime = unit.toNanos(time);
        long delta = keepAliveTime - this.keepAliveTime;
        this.keepAliveTime = keepAliveTime;
        if (delta < 0)
            interruptedIdleWorkers();
    }

    /**
     * 返回线程保持活动的时间，该时间就是超过核心池大小的线程可以在终止前保持空闲的时间值
     */
    public long getKeepAliveTime(TimeUnit unit) {
        return unit.convert(keepAliveTime, TimeUnit.NANOSECONDS);
    }

    /**
     * 获取任务队列。对任务队列的访问主要用于调试和监控。此队列可能正处于活动使用状态中。获取任务队列不妨碍已加入队列的任务的执行。
     */
    public BlockingQueue<Runnable> getQueue() {
        return workQueue;
    }

    /**
     * 从执行程序的内部队列中移除此任务(如果存在)，如果尚未开始，则其不再运行。
     *
     * 此方法可用作取消方案的一部分。它可能无法移除在放置到内部队列之前已经转换为其他形式的任务。例如，使用submit提交的任务可能被转换为维护Future状态的形式。但是，在此情况下，purge()方法可用于移除那些已被取消的Future
     */
    public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate();
        return removed;
    }

    /**
     * 尝试从工作队列中移除所有已取消的Future任务。此方法可用作存储回收操作，它对功能没有任何影响。取消的任务不会再次执行，但是它们可能在工作队列中累积，直到worker线程主动将其移除。调用此方法将试图立即移除它们。但是，如果出现其他线程的干预，那么此方法移除任务将失败。
     */
    public void purge() {
        final BlockingQueue<Runnable> q = workQueue;
        try {
            Iterator<Runnable> it = q.iterator();
            while (it.hasNext()) {
                Runnable r = it.next();
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    it.remove();
            }
        } catch (ConcurrentModificationException fallThrough) {
            for (Object r : q.toArray())
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    q.remove(r);
        }
        tryTerminate();
    }

    /**
     * 返回线程池中当前线程数
     */
    public int getPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 排除isTerminated() && getPoolSize() > 0的情况
            return runStateAtLeast(ctl.get(), TIDYING) ? 0
                : workers.size();
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回正在执行任务的近似线程数
     */
    public int getActiveCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int n = 0;
            for (Worker w : workers)
                if (w.isLocked())
                    ++n;
            return n;
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回曾经同时位于池中的最大线程数
     */
    public int getLargestPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            return largestPoolSize;
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回曾计划执行的近似任务总数。因为在计算期间任务和线程的状态可能动态改变，所以返回值只是一个近似值。
     */
    public long getTaskCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            long n = completedTaskCount;
            for (Worker w : workers) {
                n += w.completedTasks;
                if (w.isLocked())
                    ++n;
            }
            return n + workQueue.size();
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回已完成执行的近似任务总数。因为在计算期间任务和线程的状态可能动态改变，所以返回值只是一个近似值，但是该值在整个连续调用过程中不会减少
     */
    public long getCompletedTaskCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            long n = completedTaskCount;
            for (Worker w : workers)
                n += w.completedTasks;
            return n;
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回该线程池的字符串表示，包括状态、预估的worker和task数量
     */
    public String toString() {
        long ncompleted;
        int nworkers, nactive;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            ncompleted = completedTaskCount;
            nactive = 0;
            nworkers = workers.size();
            for (Worker w : workers) {
                ncompleted += w.completedTasks;
                if (w.isLocked())
                    ++nactive;
            }
        } finally {
            mainLock.unlock();
        }
        int c = ctl.get();
        String rs = (runStateLessThan(c, SHUTDOWN) ? "Running" :
                    (runStateAtLeast(c, TERMINATED) ? "Terminated" :
                    "Shutting down"));
        return super.toString() + 
                "[" + rs +
                ", pool size = " + nworkers +
                ", active threads = " + nactive +
                ", queued tasks = " + workQueue.size() +
                ", completed tasks = " + ncompleted +
                "]";
    }

    /**
     * 在执行给定线程中的给定Runnable之前调用的方法。此方法由将执行任务r的线程t调用，并且可用于重新初始化ThreadLocals或者执行日志记录
     *
     * 此实现不执行任何操作，但可在子类中定制。注：为了正确嵌套多个重写操作，此方法结束时，子类通常应该调用super.beforeExecute。
     */
    protected void beforeExecute(Thread t, Runnable r) {}

    /**
     * 基于完成执行给定Runnable所调用的方法。此方法由执行任务的线程调用。如果非null，则Throwable是导致执行突然终止的未捕获的RuntimeException或Error。
     * 当操作显示地或者通过submit之类的方法包含在任务内时(如FutureTask)，这些任务对象捕获和维护计算异常，因此它们不会导致突然终止，内部异常不会传递给此方法。
     * 
     * 此实现不执行任何操作，但可在子类中定制。注：为了正确嵌套多个重写操作，此方法结束时，子类通常应该调用super.afterExecute。
     */
    protected void afterExecute(Runnable r, Throwable t) {}

    /**
     * 当 Executor 已经终止时调用的方法。默认实现不执行任何操作。注：为了正确嵌套多个重写操作，子类通常应该在此方法中调用 super.afterExecute。
     */
    protected void terminated() {}

    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        public CallerRunsPolicy() {}

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() {}

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                " rejected from " +
                                                e.toString());
        }
    }

    public static class DiscardPolicy implements RejectedExecutionHandler {
        public DiscardPolicy() {}

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public DiscardOldestPolicy() {}

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
}

```




