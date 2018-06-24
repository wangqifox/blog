---
title: Condition与AbstractQueuedSynchronizer
date: 2018/01/12 15:38:25
---

前两篇文章中分析了AQS的独占功能和共享功能，AQS中还实现了Condition的功能。它可以替代传统的Object中的wait()、notify()、notifyAll()方法来实现线程间的通信，使线程间协作更加安全和高效。

使用synchronized关键字时，所有没有获取锁的线程都会等待，这时相当于只有1个等待队列；而在实际应用中可能有时需要多个等待队列，比如ReadLock和WriteLock。Lock中的等待队列和Condition中的等待队列是分开的，例如在独占模式下，Lock的独占保证了在同一时刻只会有一个线程访问临界区，也就是lock()方法返回后，Condition中的等待队列保存着被阻塞的线程，也就是调用await()方法后阻塞的线程。所以使用lock比使用synchronized关键字更加灵活。
<!--more-->
Condition必须被绑定到一个独占锁上使用，在ReentrantLock中，有一个newCondition方法，该方法调用了Sync中的newCondition方法：

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

ConditionObject是在AQS中定义的，它实现了Condition接口。该类有两个重要的变量：

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

对于Condition来说，它是不与独占模式或共享模式使用相同队列的，它有自己的队列，所以这两个变量表示了队列的头结点和尾节点。

## await

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 根据当前线程创建一个Node添加到Condition队列中
    Node node = addConditionWaiter();
    // 释放当前线程的lock，从AQS的队列中移除
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 循环判断当前线程的Node是否在Sync队列中，如果不在，则park
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        // checkInterruptWhileWaiting方法根据中断发生的时机返回后续需要处理这次中断的方式，如果发生中断，退出循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // acquireQueued获取锁并返回线程是否中断
    // 如果线程被中断，并且中断的方式不是抛出异常，则设置中断后续的处理方式为REINTERRUPT
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 从头到尾遍历Condition队列，移除被cancel的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 如果线程已经被中断，则根据之前获取的interruptMode的值来判断是继续中断还是抛出异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

### addConditionWaiter

首先调用addConditionWaiter在Condition队列中增加一个等待节点：

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

根据当前线程创建一个Node并添加到Condition队列中。如果尾节点不是CONDITION状态(被取消)，调用`unlinkCancelledWaiters`方法删除Condition队列中被cancel的节点。然后将lastWaiter的nextWaiter设置为node，并将node设置为lastWaiter，即将其添加到链表的末尾。

#### unlinkCancelledWaiters

```java
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
```

`unlinkCancelledWaiters`方法从头节点开始遍历。

如果节点的状态不是`CONDITION`(被取消了)，从链表中删除该节点：如果该节点是头节点，则将firstWaiter指向该节点的后节点；否则该节点的前节点指向该节点的后节点。如果节点的后节点为null，表示它是尾节点，lastWaiter指向它的前节点。

如果节点的状态是`CONDITION`，跳过该节点。

### fullyRelease

回到`await`方法，调用`addConditionWaiter`在队列中增加一个等待节点之后，接着调用`fullyRelease`：

```java
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
```

`fullyRelease`方法的功能是将当前AQS中持有的锁全部释放。savedState表示当前线程已经加锁的次数(ReentrantLock为重入锁)。因为是可重入的原因，只有在state为0的时候才会真的释放锁，所以在fullRelease方法中，需要将之前加入的锁的次数全部释放，目的是将该线程从Sync队列中移出。

### isOnSyncQueue

接着循环调用`isOnSyncQueue`判断当前线程是否又被添加到了Sync队列中，如果已经在Sync队列中，则退出循环。什么时候会把当前线程又加入到Sync队列中呢？当然是调用signal方法的时候，因为这里需要唤醒之前调用await方法的线程。

```java
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    return findNodeFromTail(node);
}
```

`isOnSyncQueue`方法判断当前线程的node是否在Sync队列中，即被唤醒加入到Sync队列中：

1. 如果当前线程node的状态为CONDITION，或者`node.prev`为null时说明已经在Condition队列中了，所以返回false
2. 如果node.next不为null，说明在Sync队列中，返回true
3. 如果两个if都未返回时，可以断定node的prev一定不为null，next一定为null，这个时候可以认为node正处于放入Sync队列的CAS操作的执行过程中。而这个CAS操作有可能失败，所以通过`findNodeFromTail`再尝试一次判断。

```java
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
```

`findNodeFromTail`从Sync队列尾部开始判断，因为在`isOnSyncQueue`方法调用该方法时，node.prev一定不为null。但这时的node可能还没有完全添加到Sync队列中(因为node.next是null)，这时可能是在自旋中。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            // 执行findNodeFromTail方法时可能一直在此自旋
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

如果此时CAS还未成功，那只好返回false了。

### checkInterruptWhileWaiting

回到`await`方法，如果节点不在Sync队列中，则挂起线程。线程唤醒之后调用`checkInterruptWhileWaiting`方法检查是否被中断，如果中断再调用`transferAfterCancelledWait`方法判断后续的处理应该是抛出`InterruptedException`还是重新中断。

```java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

```java
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

该方法判断，在线程中断的时候，是否有signal方法的调用。

1. 如果`compareAndSetWaitStatus(node, Node.CONDITION, 0)`执行成功，则说明中断发生时，没有signal的调用，因为signal方法会将状态设置为0
2. 如果第1步执行成功，则将node添加到Sync队列中，并返回true，表示中断再signal之前
3. 如果第1步失败，则检查当前线程的node是否已经在Sync队列中了，如果不在Sync队列中，则让步给其他线程执行，直到当前的node已经被signal方法添加到Sync队列中。
4. 返回false

这里需要注意的地方是，如果第一次CAS失败了，则不能判断当前线程是先进行了中断还是先进行了signal方法的调用，可能是先执行了signal然后中断，也可能是先执行了中断，后执行了signal。当然，这两个操作肯定是发生在CAS之前。这时需要做的就是等待当前线程的node被添加到Sync队列后，也就是enq方法返回后，返回false告诉`checkInterruptWhileWaiting`方法返回`REINTERRUPT`，后续进行重新中断。

简单来说，该方法的返回值代表当前线程是否在park的时候被中断唤醒，如果为true表示中断在signal调用之前，signal还未执行，否则表示signal已经执行过了。

回到`await`函数，`checkInterruptWhileWaiting`返回不为0(THROW_IE或者REINTERRUPT)说明遭遇中断，跳出循环。

接着获取独占锁，如果node还有后节点则调用`unlinkCancelledWaiters`从头到尾遍历Condition队列，移除状态为非CONDITION的节点。

因为在执行该方法时已经获取了独占锁，所以不需要考虑多线程问题。

### reportInterruptAfterWait

如果当前线程被中断，则在`await`方法中调用`reportInterruptAfterWait`方法：

```java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

该方法根据interruptMode来确定是应该抛出InterruptedException还是继续中断。

## signal

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

首先判断当前线程是否占有独占锁，然后通过firstWaiter依次唤醒Condition队列中的node，并把node添加到Sync队列中。

在await方法中可以知道添加到Condition队列中的node每次都是添加到队列的尾部，在signal方法中是从头开始唤醒的，所以Condition是公平的，signal是按顺序来进行唤醒的。

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```

doSignal方法先将队列前面节点依次从队列中取出，然后调用transferForSignal方法去唤醒节点，这个方法有可能失败，因为等待线程可能已经到时或者被中断，因此while循环这个操作直到成功唤醒或队列为空。

```java
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

该方法首先尝试设置node的状态为0：

- 如果设置失败，说明已经被取消，没必要再进入Sync队列了，doSignal中的循环会找到下一个node再次执行
- 如果设置成功，但之后又被取消了呢？无所谓，虽然会进入到Sync队列，但在获取锁的时候会调用`shouldParkAfterFailedAcquire`方法，该方法会移除此节点。

如果执行成功，则将node加入到Sync队列中，enq会返回node的前继节点p。这里的if判断只有在p节点是取消状态或者设置p节点的状态为SIGNAL失败的时候才会执行unpark。

什么时候`compareAndSetWaitStatus(p, ws, Node.SIGNAL)`会执行失败呢？如果p节点的线程在这时执行了unlock方法，就会调用`unparkSuccessor`方法，`unparkSuccessor`方法可能就将p的状态改为了0，那么执行就会失败。

到这里，signal方法已经完成了所有的工作，唤醒的线程已经成功加入Sync队列并已经参与锁的竞争了，返回true。

## signalAll

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
```

判断当前线程是否占有独占锁，然后执行doSignalAll方法。

```java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

与doSignal方法比较一下，doSignal方法只是唤醒了一个node并加入到Sync队列中，而doSignalAll方法方法唤醒了所有的Condition节点，并加入到Sync队列中。

Condition必须与一个独占锁绑定使用，在await或signal之前必须先持有独占锁。Condition队列时一个单向链表，它是公平的，按照先进先出的顺序从队列中被唤醒并添加到Sync队列中，这时便恢复了参与竞争锁的资格。

> http://www.ideabuffer.cn/2017/03/20/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AbstractQueuedSynchronizer%EF%BC%88%E4%B8%89%EF%BC%89/

