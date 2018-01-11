---
title: Semaphone与AbstractQueuedSynchronizer
---

Semaphone也是一个类似锁的组件，它管理的是多个资源的分配，实现的是AbstractQueuedSynchronizer抽象类，有了前面`ReentrantLock与AbstractQueuedSynchronizer`的铺垫，Semaphone的分析变得无比轻松。

和`ReentrantLock`类似，`Semaphone`也分为公平锁和非公平锁。方便起见只分析非公平锁，两者差别不大。

## acquire

`acquire`在`Semaphone`类中有两个方法，`acquire()`和`acquire(int permits)`。唯一的区别是`acquire()`请求一个资源，相当于`acquire(1)`。我们来看`acquire(int permits)`方法：

```java
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

首先保证permits不为0。然后调用AQS的`acquireSharedInterruptibly`方法：

```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

首先调用`tryAcquireShared`判断能否获得共享锁，这里调用的是`nonfairTryAcquireShared`：

```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

计算获取锁之后的状态(通俗的说，计算当前剩余的资源数-需要获取的资源数)，设置AQS的状态，返回状态。

回到`acquireSharedInterruptibly`方法，如果获取锁之后的状态大于等于0(资源还有剩余)，则成功获取锁。否则说明无法获得锁，进入`doAcquireSharedInterruptibly`方法：

```java
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
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
```

该方法与之前分析的`doAcquireInterruptibly`方法相差不大，区别在两处：

1. 新建的node类型为`Node.SHARED`
2. 获得锁之后调用`setHeadAndPropagate`

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

首先设置node为头结点。

接着判断如果满足以下条件中一项：

1. 指定了propagate标记
2. h(原先的头结点)为null
3. h处于`SIGNAL`、`CONDITION`、`PROPAGATE`的其中一种状态

且node的后节点不为null且为共享模式，则调用`doRleaseShared`释放共享的节点

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

这是一个死循环。

如果head(头节点)不为null且不为tail(尾节点)，进入if方法体。当head处于`SIGNAL`状态时，如果可以将其状态修改为0，调用`unparkSuccessor`唤醒后节点，否则(被其他线程修改掉了)进行下一次循环。当head处于状态0，如果可以将其状态修改为`PROPAGATE`则继续执行，否则(被其他线程修改掉了)进行下一次循环。

前面的步骤只有头节点没有发生改变才能跳出循环，如果头节点发生了改变(意味着释放锁的过程没有结束)，则继续循环。

## acquireUninterruptibly

`acquireUninterruptibly`和`acquire`的区别是前者不响应中断，后者响应中断。

我们直接看`doAcquireShared`：

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
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

与`doAcquireSharedInterruptibly`唯一的区别就是`parkAndCheckInterrupt`中断返回后仅仅设置了`interrupted`变量，而`doAcquireSharedInterruptibly`抛出了`InterruptedException`异常。

## release

release方法释放指定数量的资源：

```java
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```

可以看到，它调用了AQS中的`releaseShared`方法：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

首先调用`tryReleaseShared`方法判断是否可以释放资源，该方法在Sync中实现：

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

这方法是一个还资源的过程，就是将AQS中的state加回一定的数量，保证资源不溢出，然后设置state。

回到`releaseShared`方法，如果释放资源成功，调用`doReleaseShared`唤醒正在等待资源的线程。前文已经分析，不再赘述。


