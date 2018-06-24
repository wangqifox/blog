---
title: AbstractQueuedSynchronizer
date: 2018/01/09 14:18:25
---

为依赖于先进先出(FIFO)等待队列的阻塞锁和相关同步器(信号量、事件、等等)提供一个实现的框架。此类的设计目标是为大多数依赖单个原子int值来表示状态的同步器提供一个有用的基础。子类必须定义改变状态的受保护的方法，定义何种状态对于此对象意味着被获取或被释放。有了这些条件之后，此类中的其他方法就可以实现所有排队和阻塞机制。子类可以维护其他状态字段，但只是为了获得同步而只追踪使用getState()、setState(int)和compareAndSetState(int, int)方法来操作以原子方式更新的int值。
<!--more-->
子类应该被定义为非公共的内部帮助类，用来实现其封闭类的同步属性。AbstractQueuedSynchronizer没有实现任何同步接口。而是定义了诸如acquireInterruptibly(int)之类的方法，在适当的时候可以通过具体的锁和相关同步器来调用它们，以实现其公共方法。

此类支持默认的独占模式和共享模式之一，或者二者都支持。处于独占模式下时，其他线程将无法获取该锁。在共享模式下，多个线程获取某个锁可能(但不是一定)会获得成功。此类并不了解这些不同，除了机械地意识到当在共享模式下成功获取某一锁时，下一个等待线程(如果存在)也必须确定自己是否可以成功获取该锁。处于不同模式下的等待线程可以共享相同的FIFO队列。通常，实现子类只支持其中一种模式，但两种模式都可以在ReadWriteLock中发挥作用。只支持独占模式或者只支持共享模式的子类不必定义支持未使用模式的方法。

此类通过支持独占模式的子类定义了一个嵌套的ConditionObject类，可以将这个类用作Condition实现。isHeldExclusively()方法将报告同步对于当前线程是否是独占的；使用当前getState()值调用release(int)方法则可以完全释放此对象；如果给定保存的状态值，那么acquire(int)方法可以将此对象最终恢复为它以前获取的状态。没有别的AbstractQueuedSynchronizer方法创建这样的条件，因此，如果无法满足此约束，则不要使用它。AbstractQueuedSynchronizer.ConditionObject的行为取决于其同步器实现的语义。

此类为内部队列提供了检查、检测和监视方法，还为condition对象提供了类似方法。可以根据需要使用用于其同步机制的AbstractQueuedSynchronizer将这些方法导出到类中。

此类的序列化只存储维护状态的基础原子整数，因此已序列化的对象拥有空的线程队列。需要可序列化的典型子类将定义一个readObject方法，该方法在反序列化时将此对象恢复到某个已知初始状态。

## 使用

为了将此类用作同步器的基础，需要适当地重新定义以下方法，这是通过使用getState()、setState(int)和/或compareAndSetState(int, int)方法来检查和/或修改同步状态来实现的：

- tryAcquire(int)
- tryRelease(int)
- tryAcquireShared(int)
- tryReleaseShared(int)
- isHeldExclusively()

默认情况下，每个方法都抛出UnsupportedOperationException。这些方法的实现在内部必须是线程安全的，通常应该很短并且不被阻塞。定义这些方法是使用此类的唯一受支持的方式。其他所有方法都被声明为final，因为它们是无法各不相同的。

也可以查找从AbstractOwnableSynchronizer继承的方法，用于跟踪拥有独占同步器的线程。鼓励使用这些方法，这允许监控和诊断工具来帮助用户确定哪个线程保持锁。

即使此类基于内部的某个FIFO队列，它也无法强行实施FIFO获取策略。独占同步的核心采用以下形式：

Acquire:
    while(!tryAcquire(arg)) {
        enqueue thread if it is not already queued possibly block current thread
    }
Release:
    if (tryRelease(arg)) {
        unblock the first queued thread
    }

(共享模式与此类似，但可能涉及级联信号)

因为要在加入队列之前检查线程的获取状况，所以新获取的线程可能闯入其他被阻塞的和已加入队列的线程之前。不过如果需要，可以内部调用一个或多个检查方法，通过定义tryAcquire和/或tryAcquireShared来禁用闯入，提供一个公平的FIFO获取顺序。特别是，大多数公平的同步器可以定义tryAcquire返回false如果hasQueuePredecessors(设计用于在公平同步器中使用)返回true。其他变体也是可能的。

对于默认闯入(也称为greedy、renouncement、convoy-avoidance)策略，吞吐量和可伸缩性通常是最高的。尽管无法保证这是公平的和无偏向的，但允许更早加入队列的线程先于更迟加入队列的再次争用资源，并且相对于传入的线程，每个参与再争用的线程都有平等的成功机会。此外，尽管从一般意义上说，获取并非"自旋"，但他们可以在阻塞之前对用其他计算所使用的tryAcquire执行多次调用。在只保持独占同步时，这为自旋提供了最大的好处，但不是这种情况时，也不会带来最大的负担。如果需要这样做，可以使用"快速路径"检查来先行调用acquire方法，如果可能不需要争用同步器，则只能通过预先检查hasContended()和/或hasQueuedThreads()来确认这一点。

通过特殊化其同步器的使用范围，此类为部分同步化提供了一个有效且可伸缩的基础，同步器可以依赖于int型的state、acquire、release参数，以及内部一个FIFO等待队列。这些还不够的时候，可以使用atomic类、自己定制的Queue类和LockSupport阻塞支持，从更低级别构建同步器。


```java

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer 
    implements java.io.Serializable {

    private static final long serialVersionUID = 7373984972572414691L;
    
    protected AbstractQueuedSynchronizer() {}
    
    
    /**
     * 等待队列节点类
     * 
     * 等待队列是"CLH"锁队列的一个变种。CLH锁通常用于自旋锁。我们将它们用于阻塞同步器，但是使用相同的基础策略，在节点的前继保存一些线程的控制信息。每个节点的"status"字段保持跟踪线程是否应该被阻塞。当前继释放，节点会接收到信号。队列中的每个节点都作为一个"特殊通知方式"监视器，保存单个的等待线程。状态字段不控制线程是否保证上锁。线程可能尝试acquire如果它是队列中的第一个。但是第一个并不保证能获得执行，仅仅是给与竞争的权力。所以当前释放竞争的线程可能需要重新等待。
     *
     * CLH lock queue其实就是一个FIFO的队列，队列中的每个节点(线程)只要等待其前继释放锁就可以了
     *
     * 进入队列时，你需要插入一个新的尾部节点。出队列时，你只需要设置头字段。
     * 
     * 插入CLH队列只需要在尾部进行一个简单的原子操作，因此存在从未排队到排队的简单原子点。类似的，出队列只涉及更新"头"。然而，节点需要更多的工作来确定他们的后继者是什么，部分原因是由于超时和中断而可能的取消。
     * 
     * 处理取消需要"prev"链接(不用于原始CLH锁)。如果一个节点被取消，其后继者(通常)将重新链接到未被取消的前继节点。关于自旋锁的类似机制的解释，请参阅Scott和Scherer的论文，网址为：http://www.cs.rochester.edu/u/scott/synchronization/
     * 
     * 我们还使用"下一个"链接来实现阻塞机制。每个节点的线程id保存在其自己的节点中，因此前导通过遍历下一个链接来通知唤醒下一个节点，来确定它是哪个线程。后继的确定必须避免与新排队的节点进行比赛，以设置其前继节点的"next"字段。当需要时，当节点的后继看起来为空时，通过从原子更新的尾部向后检查来解决这个问题。(或者说，下一个链接是一个优化，所以我们通常不需要反向扫描)
     * 
     * 取消操作为基本算法引入了一些保守性。由于我们必须轮询取消其他节点，我们可能会注意到一个被取消的节点是否在我们之前或之后。这取决于取消后继节点，允许他们稳定一个新的前继节点，除非我们可以识别一个未取消的前继节点，它将对此负责。
     * 
     * CLH队列需要一个虚拟头节点才能开始。但我们不会再构造函数中创建他们，因为如果没有争用就会被浪费。相反，在第一次争用时，节点被构造，头和尾指针被设置。
     * 
     * 等待条件的线程使用相同的节点，但是使用额外的链接。条件只需要在简单(非并发)链接队列中链接节点，因为它们仅在独占时被访问。等待时，将节点插入到条件队列中。一旦发出信号，节点就被传送到主队列。状态字段的特殊值用于标记节点所在的队列。
     */
    static final class Node {
        // 表示节点是共享模式
        static final Node SHARED = new Node();
        // 表示节点是独占模式
        static final Node EXCLUSIVE = null;
        
        // 等待状态，表示线程被取消
        static final int CANCELLED = 1;
        // 等待状态，表示线程需要唤醒
        static final int SIGNAL = -1;
        // 等待状态，表示线程等待条件
        static final int CONDITION = -2;
        // 等待状态，表示下一个acquireShared应该无条件传播
        static final int PROPAGATE = -3;
        
        /**
         * 状态字段，只能是以下的值：
         * SIGNAL: 当前节点的后继已经(或即将)被阻塞，所以当前节点在释放或取消时，要唤醒它的后继。为了避免竞争，acquire方法必须先将它们标记为需要signal，然后重试原子acquire，然后失败或者阻塞。
         * CANCELLED: 节点由于超时或者中断被取消，节点再不会离开该状态。被取消节点中的线程不会再阻塞。
         * CONDITION: 该节点目前在条件队列中。它不会被用于同步队列节点，直到被转移，转移之后状态会被设置为0。(使用该值不会影响该字段的其他使用，但是简化了机制)
         * PROPAGATE: releaseShared应该被传播到其他节点。这在doReleaseShared中被设置(只在头结点)来保证传播继续，即使有其他操作干涉。
         * 0: 非上面的值
         * 
         * 这些值以数字排列以简化使用。非负值意味着节点不需要信号。所以，大多数代码不需要检查特定的值，只是为了符号。
         * 
         * 该字段初始化为0作为普通的同步节点，初始化为CONDITION作为条件节点。使用CAS来修改。
         * 
         */
        volatile int waitStatus;
        
        volatile Node prev;
        
        volatile Node next;
        
        volatile Thread thread;
        
        // 链接到下一个等待条件的节点，或者特殊值SHARD。因为条件队列只在独占模式下访问，我们只需要一个简单的链接队列来保存节点，当他们在等待条件。然后它们被转移到队列中用于re-acquire。因为条件是独占的，我们使用特殊值表示共享模式来节省一个字段。
        Node nextWaiter;
        
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        
        Node() {
        }
        
        Node(Thread thread, Node mode) {
            this.nextWaiter = mode;
            this.thread = thread;
        }
        
        Node(Thread thread, int waitStatus) {
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
    
    private transient volatile Node head;
    
    private transient volatile Node tail;

    /**
     * state的值表示其状态，如果是0，那么当前还没有线程独占此变量，
     * 否则就是已经有线程独占了这个变量，也就是代表已经有线程获得了锁。
     */
    private volatile int state;
    
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
    static final long spinForTimeoutThreshold = 1000L;
    
    // 将节点插入队列，如果队列为空，则先初始化头节点
    private Node enq(final Node node) {
        for (;;) {  // cas无锁算法的标准for循环，不停地尝试
            Node t = tail;
            if (t == null) {    // 初始化
                if (compareAndSetHead(new Node()))  // 需要注意的是head是一个哨兵的作用,并不代表某个要获取锁的线程节点
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    
    // 创建节点，插入队列
    private Node addWaiter(Node mode) {
        // 用当前线程去构造一个Node对象，mode是一个表示Node类型的字段，仅仅表示这个节点是独占的还是共享的
        Node node = new Node(Thread.currentThread(), mode);

        /**
         * 创建好节点后，将节点加入到队列尾部，此处，在队列不为空的时候，先尝试通过CAS方式修改尾节点为最新的节点，
         * 如果修改失败，意味着有并发，这个时候才会进入enq中死循环，"自旋"方式修改
         */
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;   // 该节点的前驱指针指向tail
            if (compareAndSetTail(pred, node)) {    // cas将尾指针指向该节点
                pred.next = node;   // 如果成功，让旧尾节点的next指针指向该节点
                return node;
            }
        }
        enq(node);
        return node;
    }
    
    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
    
    // 唤醒节点的后继
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            // 把标记设置为0，表示唤醒操作已经开始进行，提高并发环境下性能
            compareAndSetWaitStatus(node, ws, 0);
        
        /*
         * 需要唤醒的线程保存在后继节点中，通常就在下一个节点。
         * 但是如果获取锁的操作被取消了或者节点为null时，则从尾部向前遍历，一直找到离当前节点最近的一个等待唤醒的节点
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            // 调用了unpark方法后，进行lock操作被阻塞的线程就恢复到运行状态，
            // 就会再次执行acquireQueued中的无限for循环中的操作，再次尝试获取锁
            LockSupport.unpark(s.thread);
    }
    
    /**
     * 共享模式下的释放操作
     */
    private void doReleaseShared() {
        /*
         * 确保释放传播，即使还有其他正在进行的acquires/releases。如果需要信号，通常尝试唤醒头结点的后继。如果没有，则将状态设置为PROPAGATE以确保在释放时继续传播。此外，我们必须循环，以防在添加新节点时执行此操作。此外，与其他用途的unparkSuccessor不同，我们需要知道CAS复位状态是否失败。
         */
        for (;;) {
            // 唤醒操作由头结点开始，注意这里的头结点已经是上面新设置的头结点了
            // 其实就是唤醒上面新获取到共享锁的节点的后继节点
            Node h = head;
            // 队列不为空且有后继节点
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 不管是共享还是独占，只有节点状态为SIGNAL才尝试唤醒后继节点
                if (ws == Node.SIGNAL) {
                    // 将waitStatus设置为0
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;
                    unparkSuccessor(h); // 唤醒后继节点
                }
                // 如果状态为0(暂时不需要唤醒)，则更新状态为PROPAGATE确保以后可以传递下去，更新失败则重试
                else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;
            }
            // 如果头节点没有发生变化，表示设置完成，退出循环
            // 如果头节点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试
            if (h == head)
                break;
        }
    }
    
    /**
     * 设置队列的头结点，检查后继是否以共享模式等待，如果是共享模式或者propagate>0或者设置了PROPAGATE状态，则执行传播
     * node是当前成功获取共享锁的节点
     * propagate是tryAcquireShared方法的返回值，它可能大于0也可能等于0
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head;
        // 设置新的头节点，即把当前获取到锁的节点设置为头结点
        // 注：这里是获取到锁之后的操作，不需要并发控制
        setHead(node);

        /**
         * 尝试唤醒后继的节点：
         * propagate > 0说明许可还有能够继续被线程acquire
         * 或者之前的head被设置为PROPAGATE(PROPAGATE可以被转换为SIGNAL)说明需要往后传递
         * 或者为null，我们不确定什么情况
         * 并且后继节点是共享模式或者如上为null
         *
         * 上面的检查有点保守，在很多个线程竞争获取/释放的时候可能会导致不必要的唤醒
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            /**
             * 后继节点是共享模式或者s == null(没有后继节点)，则进行唤醒
             * 如果后继是独占模式，那么即使剩下的许可大于0也不会继续往后传递唤醒操作，即使后面有节点是共享模式。这可以理解为除非明确不需要唤醒(后继等待节点是独占模式)，否则都要唤醒
             */
            if (s == null || s.isShared())
                // 唤醒后继节点
                doReleaseShared();
        }
    }
    
    /**
     * 取消持续的acquire的尝试
     * node是当前获取锁资源失败的节点
     */
    private void cancelAcquire(Node node) {
        if (node == null)
            return;
        node.thread = null;
        
        // 跳过已经取消的前继节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;
        
        Node predNext = pred.next;
        // 把当前节点waitStatus置为取消，这样别的节点在处理时就会跳过该节点
        node.waitStatus = Node.CANCELLED;

        // 如果是尾节点，直接删除
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && pred.thread != null) {
                /**
                 * 1. 当前节点的前置节点不是头结点
                 * 2. 前置节点的waitStatus为signal，或者如果waitStatus小于0则设置为signal
                 * 3. 前置节点的线程不为null
                 */
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    // 如果后继节点没有被取消，就把前置节点跟后置节点进行连接，相当于删除了当前节点
                    compareAndSetNext(pred, predNext, next);
            } else {
                // 前置节点是头结点或者waitStatus是PROPAGATE，直接唤醒当前节点的后继节点
                unparkSuccessor(node);
            }
            node.next = node;
        }
    }

    /**
     * 检查且更新acquire失败节点的状态
     * node是当前线程的节点，pred是它的前置节点
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)  // 前一个节点在等待独占性变量释放的通知，所以，当前节点可以阻塞
            return true;
        if (ws > 0) {   // 前一个节点处于取消获取独占性变量的状态，所以，可以跳过去
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /**
             * waitStatus等于0(初始化)或PROPAGATE(CONDITION用在ConditionObject，这个不会是这个值)。
             * 这说明线程还没有park，会先重试确定无法acquire到再park
             */
            // 将上一个节点的状态设置为signal，返回false，在park前需要确定无法acquire。
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 该方法如果返回false，即挂起条件没有完备，那就会重新执行acquireQueued方法的循环体，进行重新判断，如果返回true，那就表示万事俱备，可以刮起了，就会进入parkAndCheckInterrupt。
        return false;
    }

    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }

    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this); // 将AQS对象自己传入
        return Thread.interrupted();    // 返回中断标记的同时会清除中断标记
        /**
         * 线程被唤醒只可能是：被unpark，被中断或伪唤醒。
         * 被中断会设置interrupted，acquire方法返回前会selfInterrupted重置线程的中断状态，
         * 如果是伪唤醒的话会for循环re-check
         */
    }

    /**
     * 以独占不中断模式acquire队列中的线程
     *
     * 由于进入阻塞状态的操作会降低执行效率，所以AQS会尽力避免视图获取独占变量的线程进入阻塞状态。
     * 所以，当线程加入等待队列之后，acquireQueued会执行一个for循环，每次都判断当前节点是否应该获取这个变量(在队首了)，
     * 如果不应该获取或者再次尝试获取失败，那么就调用shouldParkAfterFailedAcquire判断是否应该进入阻塞状态，
     * 如果当前节点之前的节点已经进入阻塞状态了，那么就可以判定当前节点不可能获取到锁，
     * 为了防止CPU不停地执行for循环，消耗CPU资源，调用parkAndCheckInterrupt函数来进入阻塞状态
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;  // 锁资源获取失败标记
        try {
            boolean interrupted = false;    // 等待线程被中断标记位
            // 等待前继节点释放锁，自旋re-check
            for (;;) {
                final Node p = node.predecessor();
                // node的前驱是head，就说明，node是将要获取锁的下一个节点，所以再次尝试获取独占性变量
                if (p == head && tryAcquire(arg)) {
                    setHead(node);  // 成功后，将头结点删除，node变成头节点。头结点就表示当前正占有锁资源的节点
                    p.next = null;  // help GC
                    failed = false;
                    return interrupted; // 此时，还没有进入阻塞状态，所以直接返回false，表示不需要中断
                }
                // 如果获取锁失败，则进入挂起逻辑
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 判断是否要进入阻塞状态。如果`shouldParkAfterFailedAcquire`返回true，表示需要进入阻塞调用`parkAndCheckInterrupt`，
                    // 否则表示还可以再次尝试获取锁，继续进行for循环
                    parkAndCheckInterrupt())
                    // 如果需要，借助JUC包下的LockSupport类的静态方法park挂起当前线程，直到被唤醒
                    interrupted = true;
            }
        } finally {
            if (failed)
                // 如果有异常，则取消请求，对应到队列操作，就是将当前节点从队列中移除
                cancelAcquire(node);
        }
    }

    /**
     * 以独占中断模式acquire
     */
    private void doAcquireInterruptibly(int arg) throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null;
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
    /**
     * 以独占计时模式acquire
     */
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L) // 超时
                    return false;
                // 如果超时时间很短的话，自旋效率会更高
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 以非中断、共享模式acquire
     */
    private void doAcquireShared(int arg) {
        // 添加等待节点的方法跟独占锁一样，唯一区别就是节点类型变为了共享型
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 等待前继释放并传递
            for (;;) {
                final Node p = node.predecessor();
                // 表示前面的节点已经获取到锁，自己会尝试获取锁
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 获取成功则前继出队，跟独占不同的是，
                        // 会往后面节点传播唤醒的操作，保证剩下等待的线程能够尽快获取到剩下的许可。
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        // 如果是因为中断醒来则设置中断标记位
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                // 挂起逻辑跟独占锁一样
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 获取失败的取消逻辑跟独占锁一样
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 以共享中断模式acquire
     */
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 以共享计时模式acquire
     */
    private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return true;
                    }
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    /**
     * 视图在独占模式下获取对象状态
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * 试图设置状态来反映独占模式下的释放
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * 试图在共享模式下获取对象状态
     * 
     * 返回值：
     *  - 小于0：表示获取锁失败，需要进入等待队列
     *  - 等于0：表示当前线程获取共享锁成功，但它后续的线程无法继续获取，也就是不需要把它后面等待的节点唤醒
     *  - 大于0：表示当前线程获取共享锁成功，且它后续等待的节点也有可能继续获取共享锁成功，也就是说此时需要把后续节点唤醒让它们去尝试获取共享锁
     */
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * 试图设置状态来反映共享模式下的释放
     */
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * 如果对于当前(正调用的)线程，同步是以独占方式进行的，则返回true。
     */
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }

    /**
     * 以独占模式获取对象，忽略中断
     */
    public final void acquire(int arg) {
        // 尝试获取锁，获取不到则创建一个waiter后加入阻塞队列，并等待前继节点释放锁
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            // acquireQueued返回true，说明当前线程被中断唤醒后获取到锁，
            // 重置其interrupt status为true
            selfInterrupt();
    }

    /**
     * 以独占模式获取对象，如果被中断则终止。
     */
    public final void acquireInterruptibly(int arg) throw InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly();
    }

    /**
     * 试图以独占模式获取对象，如果被中断则终止，如果到了给定超时时间，则会失败。
     */
    public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
    }

    /**
     * 释放独占模式的对象
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {  // 释放独占性变量，其实就是将status的值减1，因为acquire时是加1
            Node h = head;
            // waitStatus为0说明没有需要被唤醒的节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h); // 唤醒head的后继节点，后继节点会从parkAndCheckInterrupt方法中返回
            return true;
        }
        return false;
    }

    /**
     * 以共享模式获取对象，忽略中断
     */
    public final void acquireShared(int arg) {
        // 如果没有许可则入队等待
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    /**
     * 以共享模式获取对象， 如果被中断则终止
     */
    public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throws new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
    
    /**
     * 试图以共享模式获取对象，如果被中断则终止，如果给了给定的超时时间，则会失败
     */
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 |
            doAcquireSharedNanos(arg, nanosTimeout);
    }

    /**
     * 以共享模式释放对象
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    /**
     * 查询是否有正在等待获取的任何线程
     */
    public final boolean hasQueuedThreads() {
        return head != tail;
    }

    /**
     * 查询是否其他线程曾争着获取此同步器；也就是说，是否某个acquire方法已经阻塞
     */
    public final boolean hasContended() {
        return head != null;
    }

    /**
     * 返回队列中第一个(等待时间最长的)线程，如果目前没有将任何线程加入队列，则返回null
     */
    public final Thread getFirstQueuedThread() {
        return (head == tail) ? null : fullGetFirstQueuedThread();
    }

    private Thread fullGetFirstQueuedThread() {
        Node h, s;
        Thread st;
        if (((h = head) != null && (s = h.next) != null &&
            s.prev == head && (st = s.thread) != null) ||
            ((h = head) != null && (s = h.next) != null &&
            s.prev == head && (st = s.thread) != null))
            return st;

        Node t = tail;
        Thread firstThread = null;
        while (t != null && t != head) {
            Thread tt = t.thread;
            if (tt != null)
                firstThread = tt;
            t = t.prev;
        }
        return firstThread;
    }

    /**
     * 如果给定线程已加入队列，则返回true
     */
    public final boolean isQueued(Thread thread) {
        if (thread == null)
            throw new NullPointerException();
        for (Node p = tail; p != null; p = p.prev)
            if (p.thread == thread)
                return true;
        return false;
    }

    /**
     * 如果第一个队列中的线程存在且以独占模式等待，返回true
     */
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next) != null &&
            !s.isShared() &&
            s.thread != null;
    }

    /**
     * 查询是否还有任意线程等待acquire的时间比当前线程长
     */
    public final boolean hasQueuedPredecessors() {
        Node t = tail;
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

    /**
     * 返回等待acquire的线程估计数
     */
    public final int getQueueLength() {
        int n = 0;
        for (Node p = tail; p != null; p = p.prev) {
            if (p.thread != null)
                ++n;
        }
        return n;
    }

    /**
     * 返回包含可能正在等待获取的线程collection
     */
    public final Collection<Thread> getQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<>();
        for (Node p = tail; p != null; p = p.prev) {
            Thread t = p.thread;
            if (t != null)
                list.add(t);
        }
        return list;
    }

    /**
     * 返回包含可能正以独占模式等待acquire的线程collection
     */
    public final Collection<Thread> getExclusiveQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (!p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }

    /**
     * 返回包含可能正以共享模式等待获取的线程collection
     */
    public final Collection<Thread> getSharedQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }

    /**
     * 返回标识此同步器及其状态的字符串
     */
    public String toString() {
        int s = getState();
        String q  = hasQueuedThreads() ? "non" : "";
        return super.toString() +
            "[State = " + s + ", " + q + "empty queue]";
    }

    /**
     * 如果节点初始在条件队列，现在在同步队列中等待重新acquire，返回true
     */
    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null)
            return true;
        return findNodeFromTail(node);
    }

    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }

    /**
     * 将节点从条件队列转移到同步队列
     */
    final boolean transferForSignal(Node node) {
        /**
         * 如果无法修改waitStatus，表示节点已经被取消了
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        /**
         * 将node插入到等待队列中，并且唤醒node
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

    /**
     * 取消等待后，如果需要，将节点转移到同步队列
     */
    final boolean transferAfterCancelledWait(Node node) {
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }

    /**
     * 以当前状态值调用release，返回保存的状态
     */
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }

    /**
     * 查询给定的ConditionObject是否使用了此同步器作为其锁
     */
    public final boolean owns(ConditionObject condition) {
        return condition.isOwnedBy(this);
    }

    /**
     * 查询是否有现车正在等待给定的、与此同步器相关的条件
     */
    public final boolean hasWaiters(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.hasWaiters();
    }

    /**
     * 返回正在等待与此同步器有关的给定条件的线程数估计值
     */
    public final int getWaitQueueLength(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.getWaitQueueLength();
    }

    /**
     * 返回一个collectio，其中包含可能正在等待与此同步器有关的给定条件的那些线程。
     */
    public final Collection<Thread> getWaitingThreads(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.getWaitingThreads();
    }

    /**
     * AbstractQueuedSynchronizer的Condition实现是Lock实现的基础
     */
    public class ConditionObject implements Condition, Serializable {
        private static final long serialVersionUID = 1173984872572414699L;

        /** 条件队列中第一个节点 */
        private transient Node firstWaiter;
        /** 条件队列中最后一个节点 */
        private transient Node lastWaiter;

        public ConditionObject() {}

        /**
         * 增加新节点到等待队列中
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 如果lastWaiter被取消了，则将其清除
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currrentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

        /**
         * 移除并转移节点，直到遇见未取消的节点
         */
        private void doSignal(Node first) {
            /**
             * 将firstWaiter往Condition队列的后面移一位，并且唤醒first
             */
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

        /**
         * 移除并转移所有的节点
         */
        private void doSignalAll(Node first) {
            /**
             * 把Condition队列中的所有node全部取出插入到等待队列中
             */
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }

        /**
         * 从条件队列中移除已取消的等待节点
         */
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }

        /**
         * 将等待时间最长的线程（如果存在）从此条件的等待队列中移动到拥有锁的等待队列。
         */
        public final void signal() {
            /**
             * 先判断当前线程是否持有锁，如果没有持有，则抛出异常
             * 然后判断整个condition队列是否为空，不为空则调用doSignal方法来唤醒线程
             */
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        /**
         * 将所有线程从此条件的等待队列移动到拥有锁的等待队列中。
         */
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }

        /**
         * 实现不可中断的条件等待。
         */
        public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();
        }

        /** 从等待中退出时重新中断 */
        private static final int REINTERRUPT =  1;
        /** 从等待中退出时抛出InterruptedException */
        private static final int THROW_IE    = -1;

        /**
         * 检查中断，如果中断发生在信号发生之前则返回THROW_IE，如果中断发生在信号发生之后则返回REINTERRUPT，如果没有中断则返回0
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

        /**
         * 抛出InterruptedException，根据模式重新中断当前线程或者什么都不做
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

        /**
         * 实现不可中断的条件等待。
         */
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();   // 将节点加入到等待队列中
            int savedState = fullyRelease(node);    // 释放当前线程的锁
            int interruptMode = 0;
            /**
             * 判断节点是否在等待队列中(signal操作会将Node从Condition队列中拿出并且放入到等待队列中去)，
             * 如果不在等待队列中了，就park当前线程，如果在，就退出循环，这个时候如果被中断，那么就退出循环
             */
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 尝试再次获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        /**
         * 实现定时的条件等待。
         */
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }

        /**
         * 实现绝对定时条件等待。
         */
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        /**
         * 实现定时的条件等待。
         */
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        /**
         * 如果该条件由给定的同步对象创建则返回true
         */
        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
            return sync == AbstractQueuedSynchronizer.this;
        }

        /**
         * 查询是否有正在等待此条件的任何线程
         */
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

        /**
         * 返回正在等待此条件的线程数估计值
         */
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }

        /**
         * 返回包含那些可能正在等待此条件的线程collection
         */
        protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }

    }

    // 支持compareAndSet
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;
    
    static {
        try {
            stateOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset(AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset(Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset(Node.class.getDeclaredField("next"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
    private final boolean compareAndSetHead(Node update) {
        return unsafe.comareAndSwapObject(this, headOffset, null, update);
    }
    
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
    
    private static final boolean compareAndSetWaitStatus(Node node,
                                                         int expect,
                                                         int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset,
                                        expect, update);    
    }
    
    private static final boolean compareAndSetNext(Node node,
                                                   Node expect,
                                                   Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }
}

```


