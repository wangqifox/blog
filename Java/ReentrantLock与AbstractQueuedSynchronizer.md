---
title: ReentrantLock与AbstractQueuedSynchronizer
date: 2018/01/10 10:41:25
---

AbstractQueuedSynchronizer(AQS)是许多Java并发控制类的基础，比如ReentrantLock、Semaphone、CountDownLatch等都是基于AQS来完成的，因此了解AQS的基本原理是了解Java并发控制类的基础。

AQS是一个抽象类，需要子类的实现其中的功能，所以我们选取ReentrantLock来分析AQS的工作原理。

在构造ReentrantLock的时候可以指定是否公平，我们先以非公平锁为例讲解。
<!--more-->
## lock

当调用ReentrantLock的lock方法时，实际调用的是NonfairSync的lock方法:

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

lock方法先通过CAS尝试将AQS中的state从0修改为1。这里的state其实表示的是当前持有锁的线程数量，如果state为0表示当前没有线程获得锁，可以直接将`exclusiveOwnerThread`设置为当前线程，这样锁就获取成功了。

如果已经有线程取得了锁，则执行`acquire(1)`：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`acquire`首先调用`tryAcquire`，`tryAcquire`有子类来实现，NonfairSync的实现方法是`nonfairTryAcquire`：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

首先获取state，如果state为0再次尝试将state从0设置为1，这个步骤在`lock`中已经尝试过，这里再一次尝试，尽量快速的获得锁。

如果state不为0，说明有线程占有了锁，于是检查当前占有锁的线程是否就是当前线程，如果是，则简单地将state增加1，然后返回true。

如果state不为0，且其他的线程占有了锁，则返回false。

回到AQS的`acquire`函数，如果`tryAcquire`返回了true，则整个if条件不成立，后面的`acquireQueued`方法不再执行，函数返回，当前线程获得了锁。

如果`tryAcquire`返回false，接着执行后面的`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`。

首先通过`addWaiter`增加一个排他类型的节点：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

首先根据当前线程和mode创建一个Node对象。

因为一开始的tail肯定为null，之间进入`enq(node)`

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
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
```

`enq`函数的主要作用是将node节点添加到AQS管理的链表中。这是一个死循环，根据CAS操作来保证关键步骤的原子性，从而可以保证多线程访问时的线程安全。

当第一个线程进入`enq`时，head和tail都是null。进入`if(t == null)`的代码块，执行`compareAndSetHead(new Node)`新建一个空的head。如果成功，将tail也指向head所指向的节点，然后继续循环。如果失败(被其他线程抢先新建了head)，继续循环。

当链表完成初始化之后，循环进入`else`代码块。首先将`node.prev`指向tail，再将tail指向node，接着原先tail的后继指向node。即尝试将node挂到tail后面，tail指向node，这是一个在双向链表尾部增加节点的操作。因为有`compareAndSetTail(t, node)`的存在，每次只有一个线程能在尾部成功添加节点，失败的线程继续循环。

回到前面的`addWaiter`，这个函数的目的就是要在双向链表的尾部添加node节点，当链表初始化完成后，进入`if (pred != null)`代码块，尝试一次添加节点的动作，成功则返回，如果失败进入`enq`的死循环，直到node添加成功为止。

在回到前面的`acquireQueued`：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

`acquireQueued`也是一个死循环，查看`if (p == head && tryAcquire(arg))`，p是node节点的前一个节点，我们发现只有当前节点的前一个节点是head，且`tryAcquire`成功获取锁(state为0且通过CAS设置state成功，或者锁的占有者是当前线程)时，进行进一步处理，然后跳出死循环。删除原来的头节点，重新设置头节点。

如果无法获取锁，执行后面的代码。

首先调用`shouldParkAfterFailedAcquire(p, node)`判断是否应该挂起线程：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

首先判断前一个节点的状态是否为`Node.SIGNAL`，如果是则返回true，如果不是则返回false。

如果前一个节点的状态大于0（被`CANCELLED`掉了），通过while循环向前寻找一个没有`CANCELLED`的节点，删除那些`CANCELLED`的节点。

如果前一个节点的值不大于0，则通过CAS将前一个节点的状态修改为`Node.SIGNAL`。因为`acquireQueued`方法是一个死循环，所以一定会有一个节点的状态为`Node.SIGNAL`，然后返回true。

`shouldParkAfterFailedAcquire`返回true，则执行`parkAndCheckInterrupt()`方法，它通过`LockSupport.park(this)`将当前线程挂起，它需要等待一个中断、`unpark()`方法来唤醒它。

## unlock

当调用ReentrantLock的lock方法时，实际调用的是AQS的release方法:

```java
public void unlock() {
    sync.release(1);
}
```

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

`release`方法首先调用`tryRelease`判定可以释放锁，以ReentrantLock为例：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

首先计算锁释放之后AQS的state，如果state变为0，则表示锁已经完全释放，再将`exclusiveOwnerThread`设置为null，返回true，这样其他线程就可以获得锁；否则仅仅是设置释放之后的state，返回false。

回到`release`方法，如果`tryRelease`函数返回true，也就是锁完全释放，则唤醒头节点的后继节点。

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

首先获取head节点的next节点。

如果这个后继节点不为空，则通过`LockSupport.unpark`方法来唤醒对应被挂起的线程，于是该节点从`acquireQueued`方法的`parkAndCheckInterrupt`处唤醒，继续循环。判断`tryAcquire`是否能获得锁，能获得锁的话返回执行任务，如果获取锁又失败了，继续挂起等待。

如果这个后继节点为空或者它的状态为`CANCELLED`，则从双向链表的尾部遍历寻找一个状态不为`CANCELLED`的节点，唤醒它。

至于为什么从尾部开始向前遍历，需要看后文的`lockInterruptibly`方法，其中的`cancelAcquire`方法中只是设置了next的变化，没有设置prev的变化，在最后由这样一行代码`node.next = node`，如果这时执行了`unparkSuccessor`方法，并且向后遍历的话，就成了死循环了，所以这时只有prev是稳定的。

## lockInterruptibly

该方法和`lock`方法的区别是`lockInterruptibly`会响应中断。也就是说，如果线程在`lockInterruptibly`处等待，可以使用interrupt来使线程继续执行(抛出`InterruptedException`)，而如果线程在`lock`处等待，线程不会响应中断。

在ReentrantLock中，`lockInterruptibly`方法调用的是AQS中的`acquireInterruptibly`：

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

首先仍然调用`tryAcquire`方法判断是否可以获得锁，如果无法获得锁，执行`doAcquireInterruptibly`方法：

```java
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
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
```

该方法和前面分析过的`acquireQueued`方法差不多，真正的区别在于`parkAndCheckInterrupt`被中断唤醒之后的操作，`parkAndCheckInterrupt`被中断唤醒之后返回值为true，因此进入if代码块中：

`acquireQueued`方法仅仅将变量`interrupted`设为true，进入下一次循环，下一次循环中如果仍然获取不到锁，线程再一次进入等待状态，因此给程序的现象就是线程无法响应中断。

而`doAcquireInterruptibly`方法抛出了`InterruptedException`，线程得以退出等待状态。因为此时的`failed`变量为true，所以最后还需执行`cancelAcquire`方法，用于将该节点标记为取消状态：

```java
private void cancelAcquire(Node node) {
    if (node == null)
        return;

    node.thread = null;

    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    Node predNext = pred.next;

    node.waitStatus = Node.CANCELLED;

    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

首先设置该节点不再关联任何线程，跳过node前面被`CANCELLED`的节点。

如果node是tail(尾节点)，将tail指向的位置向前移动一个节点，即将node节点从链表的末尾中删除。

如果node不是tail。

    1. 如果pred(node的前节点)不是head，且它的状态为SIGNAL(等待被唤醒)，且pred的线程不为null，则将node的后节点赋值给pred的后节点，即将node节点从链表的中间删除。

    2. 否则(pred是头结点，或者状态不为SIGNAL，或者pred的线程为null)唤醒node的后节点。

然后将next指向了自己。

这里可能会有疑问，既然要删除节点，为什么都没有对prev进行操作，而仅仅是修改了next？

要明确的一点是，这里修改指针的操作都是CAS操作，在AQS中所有以`compareAndSet`开头的方法都是尝试更新，并不保证成功。

那么在执行cancelAcquire方法时，当前节点的前继节点有可能已经执行完并移除队列了(参见`setHead`方法)，所以在这里只能用CAS来尝试更新，而就算是尝试更新，也只能更新next，不能更新prev，因为prev是不确定的，否则有可能会导致整个队列的不完整，例如把prev指向一个已经移除队列的node。

什么时候修改prev呢？其实prev是由其他线程来修改的。回到`shouldParkAfterFailedAcquire`方法，该方法有这样一段代码：

```java
do {
    node.prev = pred = pred.prev;
} while (pred.waitStatus > 0);
pred.next = node;
```

这段代码的作用就是通过prev遍历到第一个不是取消状态的node，并修改prev。

这里为什么可以更新prev？因为`shouldParkAfterFailedAcquire`方法是在获取锁失败的情况下才能执行，因此进入该方法时，说明已经有线程获得锁了，并且在执行该方法时，当前节点之前的节点不会变化(因为只有当下一个节点获得锁的时候才会设置head)，所以这里可以更新prev，而且不必用CAS来更新。



> Java特种兵
> http://www.ideabuffer.cn/2017/03/15/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AbstractQueuedSynchronizer%EF%BC%88%E4%B8%80%EF%BC%89/

