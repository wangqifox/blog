---
title: Semaphone与AbstractQueuedSynchronizer
date: 2018/01/11 09:10:25
---

Semaphone也是一个类似锁的组件，它管理的是多个资源的分配，实现的是AbstractQueuedSynchronizer抽象类，有了前面`ReentrantLock与AbstractQueuedSynchronizer`的铺垫，Semaphone的分析变得无比轻松。

和`ReentrantLock`类似，`Semaphone`也分为公平锁和非公平锁。方便起见只分析非公平锁，两者差别不大。
<!--more-->
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

重点在于`setHeadAndPropagate`方法：

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

且node的后节点为null或者为共享模式，则调用`doRleaseShared`释放共享的节点

为什么下一个节点为null的时候也需要唤醒操作呢？仔细理解一下这句话：

> The conservatism in both of these checks may cause unnecessary wake-ups, but only when there are multiple racing acquires/releases, so most need signals now or soon anyway.

这种保守的检查方式可能会引起多次不必要的线程唤醒操作，但这些情况仅存在于多线程并发的`acquire/release`操作，所以大多数线程需要立即或者很快地一个信号。这个信号就是执行unpark方法。因为LockSupport在unpark的时候，相当于给了一个信号，即使这时候没有线程在park状态，之后有线程执行park的时候也会读到这个信号就不会被挂起。

简单点说，就是线程在执行时，如果之前没有unpark操作，在执行park时会阻塞该线程；但如果在park之前执行过一次或多次unpark(unpark调用多次和一次是一样的，结果不会累加)这时执行park时并不会阻塞该线程。

所以，如果在唤醒node的时候下一个节点刚好添加到队列中，就可能避免了一次阻塞的操作。

所以这里的propagate表示传播，传播的过程就是只要成功的获取到共享锁就唤醒下一个节点。

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

什么时候状态会是SIGNAL呢？回顾一下`shouldParkAfterFailedAcquire`方法，当状态不为`CANCELED`或者`SIGNAL`时，为了保险起见，这里把状态都设置成了`SIGNAL`，然后会再次循环进行判断是否需要阻塞。

这里为什么不直接把SIGNAL设置为PROPAGATE，而是先把SIGNAL设置为0，然后再设置为PROPAGATE呢？

原因在于`unparkSuccessor`方法，该方法会判断当前节点的状态是否小于0，如果小于0则将h的状态设置为0，如果在这里直接设置为PROPAGATE状态的话，则相当于多做了一次CAS操作。

```java
int ws = node.waitStatus;
if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
```

其实这里只判断状态为SIGNAL和0还有另一个原因，那就是当前执行doReleaseShared循环时的状态只可能为SIGNAL和0，因为如果这时没有后继节点的话，当前节点状态没有被修改，是初始的0；如果在执行setHead方法之前，这时刚好有后继节点被添加到队列中的话，因为这时后继节点判断`p == head`为false，所以会执行shouldParkAfterFailedAcquire方法，将当前节点的状态设置为SIGNAL。当状态为0时设置状态为PROPAGATE成功，则判断`h == head`结果为true，表示当前节点是队列中的唯一一个节点，所以直接就返回了；如果为false，则说明已经有后继节点的线程设置了head，这时不返回继续循环，但刚才获取的h已经用不到了，等待着被回收。

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

## 独占锁和共享锁的区别

- 独占锁在获取节点之后并且还未释放时，其他的节点会一直阻塞，直到第一个节点被释放才会唤醒
- 共享锁在获取节点之后会立即唤醒队列中的后继节点，每个节点都会唤醒自己的后继节点，这就是共享状态的传播




