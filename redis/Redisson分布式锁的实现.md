---
title: Redisson分布式锁的实现
date: 2018/08/11 16:48:00
---

前文[Redisson简介][1]中我们介绍了Redisson的情况，以及简单的配置与使用。Redisson在Redis的基础上实现了很多有用的功能，本文重点分析其中分布式锁的实现。

<!-- more -->

Redisson有以下几种分布式锁：

- 可重入锁(ReentrantLock)
- 公平锁(Fair Lock)
- 联锁(MultiLock)
- 红锁(RedLock)
- 读写锁(ReadWriteLock)

下文我们来看看这几种锁的实现方式。

# 可重入锁

`RedissonLock`是基于Redis的分布式可重入锁，它实现了`org.redisson.api.RLock`接口，`RLock`接口实现了`java.util.concurrent.locks.Lock`接口。

它常见的使用方法如下：

首先获取锁：

```java
RLock lock = redissonClient.getLock("anyLock");
```

然后对锁进行加锁，以下几种是常见方法：

- `lock()`：加锁，如果当前锁由另外的线程持有，则阻塞当前线程直到成功获取锁。

    在redis中加锁有两种思路。
    
    - 一种是设置key的值，并且不对key设置过期时间。这种情况下如果加锁的线程在没有解锁之前崩溃了，那么这个锁会出现死锁的状态。
    - 另外一种是设置key的值，并且对key设置过期时间。这种情况下如果加锁的线程在没有解锁之前崩溃了，那么这个锁在过期时间之后自然解锁，不会发生死锁的现象。但是这样也引入了另外一个问题，如果加锁的线程在过期时间之内没有完成操作，这时候锁就会被另外的线程获取，从而发生同时有两个线程同时在临界区运行的状况。为了避免这种情况发生，Redisson内部提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期。默认情况下，看门狗的检查锁的超时时间是30秒钟，也可以通过修改`Config.lockWatchdogTimeout`来另行指定。

- `lock(long leaseTime, TimeUnit unit)`：`leaseTime`参数表示加锁时间，超过这个时间后锁便自动解开了。
- `tryLock()`：尝试加锁，如果成功则返回`true`，如果失败则立即返回`false`。
- `tryLock(long waitTime, long leaseTime, TimeUnit unit)`：`waitTime`表示尝试加锁失败时等待锁的时间，加锁成功后等待`leaseTime`时间后释放锁。
- `tryLock(long waitTime, TimeUnit unit)`：`waitTime`表示尝试加锁失败时等待锁的时间，加锁成功后没有超时时间。

## 原理

### lock

我们以最常用的`lock()`方法为例分析Redisson的可重入锁时如何实现的。

```java
public void lock() {
    try {
        lockInterruptibly();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}

public void lockInterruptibly() throws InterruptedException {
    lockInterruptibly(-1, null);
}
```

可以看到，`lock()`方法实际调用的是`lockInterruptibly`方法，传入的过期时间参数为`-1`，表示不过期。

```java
public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
    long threadId = Thread.currentThread().getId();
    Long ttl = tryAcquire(leaseTime, unit, threadId);
    // lock acquired
    if (ttl == null) {
        return;
    }

    RFuture<RedissonLockEntry> future = subscribe(threadId);
    commandExecutor.syncSubscription(future);

    try {
        while (true) {
            ttl = tryAcquire(leaseTime, unit, threadId);
            // lock acquired
            if (ttl == null) {
                break;
            }

            // waiting for message
            if (ttl >= 0) {
                getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
            } else {
                getEntry(threadId).getLatch().acquire();
            }
        }
    } finally {
        unsubscribe(future, threadId);
    }
}
```

`lockInterruptibly`方法中尝试获取锁，获取失败时，阻塞当前线程直到获取成功。

- 首先调用`tryAcquire`尝试获取锁，如果返回的`ttl`为`null`，表示锁获取成功，方法直接返回。
- 如果锁获取失败，调用`subscribe`方法订阅解锁的消息，解锁之后会唤醒当前的阻塞线程。然后在循环中继续调用`tryAcquire`尝试获取锁。如果返回的`ttl`不为`null`，表示锁获取失败，根据返回的`ttl`数值进行不同的操作。

    - 如果返回的`ttl`大于等于0，表示当前已经获得锁的线程设置了锁的过期时间，于是调用`Semaphore`的`tryAcquire`方法获取信号量阻塞当前线程，超过`ttl`时间后自动唤醒线程，再次尝试获取锁。
    - 如果返回的`ttl`小于0，表示当前已经获得锁的线程没有设置锁的过期时间，于是调用`Semaphore`的`acquire`方法获取信号量阻塞当前线程，等待被唤醒，再次尝试获取锁。

```java
private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
    return get(tryAcquireAsync(leaseTime, unit, threadId));
}

private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, final long threadId) {
    if (leaseTime != -1) {
        return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    }
    RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(
        commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), 
        TimeUnit.MILLISECONDS, 
        threadId, 
        RedisCommands.EVAL_LONG);
    ttlRemainingFuture.addListener(new FutureListener<Long>() {
        @Override
        public void operationComplete(Future<Long> future) throws Exception {
            if (!future.isSuccess()) {
                return;
            }

            Long ttlRemaining = future.getNow();
            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        }
    });
    return ttlRemainingFuture;
}
```

`tryAcquire`方法尝试获取锁，它首先调用`tryAcquireAsync`方法异步获取锁，然后调用`get`方法同步获取结果。

- 如果`leaseTime`不为`-1`，表示锁有一个过期时间，则调用`tryLockInnerAsync`进行加锁即可。
- 否则说明锁没有过期时间。前文我们说过Redisson通过对锁进行自动续期达到锁不过期的目的。

    - 首先加锁并设置锁的过期时间，默认为30秒
    - 如果加锁成功，调用`scheduleExpirationRenewal`创建一个定时任务刷新锁的过期时间。

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
```

`tryLockInnerAsync`是最终加锁的方法，它将加锁的命令整合在一个脚本中：

- 如果key不存在，说明还未被加锁。锁的结构是一个hash表，将锁的名称（与加锁线程的id关联）作为hash表的key，初始值设置为1，表示当前对该锁进行加锁的线程数为1。接着设置锁的过期时间。
- 如果key已经存在了，说明已经有线程获得了该锁。判断是否是同一个线程加的锁，如果是则将该锁的线程数加1。接着设置锁的过期时间。获取锁成功。
- 如果是另外的线程获得了锁，则本次尝试加锁的操作失败，返回该锁设置的过期时间。

### unlock

```java
public void unlock() {
    Boolean opStatus = get(unlockInnerAsync(Thread.currentThread().getId()));
    if (opStatus == null) {
        throw new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                + id + " thread-id: " + Thread.currentThread().getId());
    }
    if (opStatus) {
        cancelExpirationRenewal();
    }
}
```

`unlock`方法首先调用`unlockInnerAsync`异步方法来解锁。然后调用`get`方法获取解锁的结果，如果解锁失败，表示不是当前线程加的锁，抛出异常；如果解锁成功，调用`cancelExpirationRenewal()`方法去掉为锁续期的定时任务。

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                "return nil;" +
            "end; " +
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            "if (counter > 0) then " +
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "return 0; " +
            "else " +
                "redis.call('del', KEYS[1]); " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; "+
            "end; " +
            "return nil;",
            Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));
}
```

- 如果key已经不存在了，说明已经解锁了，发布解锁信息，然后返回1
- 如果锁名称在当前锁中不存在，当前线程并没有加锁，返回nil
- 将当前锁关联的线程数减1
    - 如果减完之后线程数还是大于0，说明锁还没有释放完，返回0
    - 否则锁被释放成功，删除锁，发布解锁信息，然后返回1

# 公平锁

`RedissonFairLock`是基于Redis的分布式可重入公平锁。它保证了当多个Redisson客户端线程同时请求加锁时，优先分配给先发出请求的线程。



# 联锁

`RedissonMultiLock`是基于Redis的分布式联锁。它可以将多个`RLock`对象关联为一个联锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

它的使用示例如下：

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 所有的锁都上锁成功才算成功
lock.lock();
...
lock.unlock();
```

## 原理

在之前可重入锁的基础上，联锁的加锁过程不难理解，主要的流程就是对所有的`RLock`对象实例分别进行加锁，如果所有的`RLock`对象都加锁成功了联锁才算加锁成功。

### lock方法

```java
public void lock() {
    try {
        lockInterruptibly();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}

public void lockInterruptibly() throws InterruptedException {
    lockInterruptibly(-1, null);
}

public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
    long baseWaitTime = locks.size() * 1500;
    long waitTime = -1;
    // 如果leaseTime为-1，设置一个默认的waitTime，它等于锁的大小乘以1.5秒
    if (leaseTime == -1) {
        waitTime = baseWaitTime;
        unit = TimeUnit.MILLISECONDS;
    } else {
        waitTime = unit.toMillis(leaseTime);
        // 如果watiTime小于等于2000，将其设为2000
        if (waitTime <= 2000) {
            waitTime = 2000;
        } 
        // 如果waitTime小于等于baseWaitTime，将其设为[waitTime/2, waitTime)之间的一个随机数
        else if (waitTime <= baseWaitTime) {
            waitTime = ThreadLocalRandom.current().nextLong(waitTime/2, waitTime);
        } 
        // 否则将waitTime设为[baseWaitTime, waitTime)之间的一个随机数
        else {
            waitTime = ThreadLocalRandom.current().nextLong(baseWaitTime, waitTime);
        }
        waitTime = unit.convert(waitTime, TimeUnit.MILLISECONDS);
    }
    
    while (true) {
        if (tryLock(waitTime, leaseTime, unit)) {
            return;
        }
    }
}
```

可以看到，`lock()`方法首先计算`waitTime`。然后在循环中调用`tryLock`方法，直到成功获取到锁。

### tryLock方法

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long newLeaseTime = -1;
    if (leaseTime != -1) {
        newLeaseTime = unit.toMillis(waitTime)*2;
    }
    
    long time = System.currentTimeMillis();
    // remainTime表示这一次加锁操作剩余的时长
    long remainTime = -1;
    if (waitTime != -1) {
        remainTime = unit.toMillis(waitTime);
    }
    long lockWaitTime = calcLockWaitTime(remainTime);
    
    // failedLocksLimit表示允许有几个RLock对象实例加锁失败。
    // 在联锁中failedLocksLimit为0，即必须所有的RLock对象实例都加锁成功，联锁才算加锁成功
    int failedLocksLimit = failedLocksLimit();
    List<RLock> acquiredLocks = new ArrayList<RLock>(locks.size());
    // 遍历并对所有的RLock对象实例尝试加锁
    for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
        RLock lock = iterator.next();
        // lockAcquired表示lock对象加锁是否成功
        boolean lockAcquired;
        try {
            if (waitTime == -1 && leaseTime == -1) {
                lockAcquired = lock.tryLock();
            } else {
                long awaitTime = Math.min(lockWaitTime, remainTime);
                lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
            }
        } catch (Exception e) {
            lockAcquired = false;
        }
        // 如果lock对象加锁成功，将其加入到acquiredLocks列表中
        if (lockAcquired) {
            acquiredLocks.add(lock);
        } else {
            // 如果成功加锁的lock对象数量已经达到了阈值，说明已经加锁成功了，此时跳出循环。
            // 在联锁中，因为failedLocksLimit为0，只有所有的lock对象都加锁成功才会加锁成功，这个if判断不会为真
            if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                break;
            }
            // 如果failedLocksLimit为0，只有所有的lock对象都加锁成功才会加锁成功，因此有一个lock对象加锁失败表示这个锁就失败了
            if (failedLocksLimit == 0) {
                // 将之前加锁成功的lock对象解锁
                unlockInner(acquiredLocks);
                // 如果没有设置等待时间，则立即返回false
                if (waitTime == -1 && leaseTime == -1) {
                    return false;
                }
                // 否则重置failedLocksLimit、acquiredLocks.clear，继续从头进行加锁的尝试
                failedLocksLimit = failedLocksLimit();
                acquiredLocks.clear();
                // reset iterator
                while (iterator.hasPrevious()) {
                    iterator.previous();
                }
            } 
            // 如果failedLocksLimit不为0，说明容忍加锁失败的lock对象，仅仅将其减1
            else {
                failedLocksLimit--;
            }
        }
        // 判断加锁操作是否超时，如果超时将之前加锁成功的lock对象解锁，返回false
        if (remainTime != -1) {
            remainTime -= (System.currentTimeMillis() - time);
            time = System.currentTimeMillis();
            if (remainTime <= 0) {
                unlockInner(acquiredLocks);
                return false;
            }
        }
    }

    // 如果锁指定了超时时间，则对每个加锁成功的lock对象设置过期时间
    if (leaseTime != -1) {
        List<RFuture<Boolean>> futures = new ArrayList<RFuture<Boolean>>(acquiredLocks.size());
        for (RLock rLock : acquiredLocks) {
            RFuture<Boolean> future = rLock.expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
            futures.add(future);
        }
        
        for (RFuture<Boolean> rFuture : futures) {
            rFuture.syncUninterruptibly();
        }
    }

    return true;
}
```

`tryLock`方法尝试获取联锁，只有当所有的lock对象实例都加锁成功，并且加锁操作在规定的时间内完成，联锁才算加锁成功。

# 红锁

`RedissonRedLock`是基于Redis的红锁，它实现了Redlock介绍的加锁算法。它可以将多个`RLock`对象关联为一个红锁，每个`RLock`对象实例可以来自于不同的Redisson实例。

它的使用示例如下：

```java
RLock lock1 = redissonInstance1.getLock("lock1");
RLock lock2 = redissonInstance2.getLock("lock2");
RLock lock3 = redissonInstance3.getLock("lock3");

RedissonRedlock lock = new RedissonRedlock(lock1, lock2, lock3);
// 同时加锁：lock1 lock2 lock3
// 红锁在大部分节点上加锁成功就算成功
lock.lock();
...
lock.unlock();
```

## 原理

`RedissonRedLock`继承了`RedissonMultiLock`，它的操作与`RedissonMultiLock`完全一样，不同之处如下：

### failedLocksLimit方法

```java
protected int failedLocksLimit() {
    return locks.size() - minLocksAmount(locks);
}
    
protected int minLocksAmount(final List<RLock> locks) {
    return locks.size()/2 + 1;
}
```

这是`RedissonRedLock`与`RedissonMultiLock`最大的不同。可以看到`failedLocksLimit()`方法返回的值由`RedissonMultiLock`中的`0`变成了lock对象总数的一半减一。即在红锁中只要超过一半的lock对象加锁成功就算成功。

### calcLockWaitTime方法

```java
protected long calcLockWaitTime(long remainTime) {
    return Math.max(remainTime / locks.size(), 1);
}
```

`calcLockWaitTime`用于计算对lock对象加锁的等待时间。可以看到它的返回值由`RedissonMultiLock`中的`remainTime`变成了`remainTime / locks.size()`，即每个lock对象必须在`remainTime / locks.size()`时间内完成加锁，否则即是加锁失败。

# 读写锁

`RReadWriteLock`是基于Redis的读写锁，它实现了`java.util.concurrent.locks.ReadWriteLock`接口。该对象允许同时有多个读取锁，但是最多只能有一个写入锁。

它的使用示例如下：

```java
RReadWriteLock rwlock = redisson.getLock("anyRWLock");
// 最常见的使用方法
rwlock.readLock().lock();
// 或
rwlock.writeLock().lock();
```

读写锁分为读锁和写锁两个对象，他们的实现类分别是`RedissonReadLock`和`RedissonWriteLock`。

下面针对这两个对象，分析它们是如何实现加锁解锁操作的

## RedissonReadLock

### tryLockInnerAsync

`tryLockInnerAsync`方法是读锁加锁的最终方法。

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                            "if (mode == false) then " +
                              "redis.call('hset', KEYS[1], 'mode', 'read'); " +
                              "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                              "redis.call('set', KEYS[2] .. ':1', 1); " +
                              "redis.call('pexpire', KEYS[2] .. ':1', ARGV[1]); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                            "end; " +
                            "if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then " +
                              "local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                              "local key = KEYS[2] .. ':' .. ind;" +
                              "redis.call('set', key, 1); " +
                              "redis.call('pexpire', key, ARGV[1]); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                            "end;" +
                            "return redis.call('pttl', KEYS[1]);",
                    Arrays.<Object>asList(getName(), getReadWriteTimeoutNamePrefix(threadId)), 
                    internalLockLeaseTime, getLockName(threadId), getWriteLockName(threadId));
}
```

首先整理一下传入lua脚本的参数：

- `KEYS[1]`：锁在redis中的key
- `KEYS[2]`：超时名称的前缀
- `ARGV[1]`：锁超时时间
- `ARGV[2]`：锁的名称
- `ARGV[3]`：写锁的名称，它是在锁名称的后面加上`:write`

下面是加锁的流程：

1. 获取锁的mode，如果锁的mode为`false`，表示之前没有设置过读写锁，此时可以获得读锁。

    1. 将锁的mode设置为`read`
    2. 将锁名称对应的线程数设置为1
    3. 设置超时名称
    4. 设置超时名称的过期时间
    5. 设置锁的过期时间

2. 如果锁的mode为`read`或者mode为`write`并且持有写锁的线程为当前线程，此时可以继续加读锁。

    换句话说，如果当前存在读锁或者持有写锁的是当前线程，都可以加读锁。
    
    1. 将锁名称对应的线程数增加1
    2. 设置超时名称
    3. 设置超时名称的过期时间
    4. 设置锁的过期时间

3. 否则返回当前锁的过期时间

### unlockInnerAsync

`unlockInnerAsync`方法是读锁解锁的最终方法。

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    String timeoutPrefix = getReadWriteTimeoutNamePrefix(threadId);
    String keyPrefix = timeoutPrefix.split(":" + getLockName(threadId))[0];

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
            "if (mode == false) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end; " +
            "local lockExists = redis.call('hexists', KEYS[1], ARGV[2]); " +
            "if (lockExists == 0) then " +
                "return nil;" +
            "end; " +
                
            "local counter = redis.call('hincrby', KEYS[1], ARGV[2], -1); " + 
            "if (counter == 0) then " +
                "redis.call('hdel', KEYS[1], ARGV[2]); " + 
            "end;" +
            "redis.call('del', KEYS[3] .. ':' .. (counter+1)); " +
            "if (redis.call('hlen', KEYS[1]) > 1) then " +
                "local maxRemainTime = -3; " + 
                "local keys = redis.call('hkeys', KEYS[1]); " + 
                "for n, key in ipairs(keys) do " + 
                    "counter = tonumber(redis.call('hget', KEYS[1], key)); " + 
                    "if type(counter) == 'number' then " + 
                        "for i=counter, 1, -1 do " + 
                            "local remainTime = redis.call('pttl', KEYS[4] .. ':' .. key .. ':rwlock_timeout:' .. i); " + 
                            "maxRemainTime = math.max(remainTime, maxRemainTime);" + 
                        "end; " + 
                    "end; " + 
                "end; " +
                        
                "if maxRemainTime > 0 then " +
                    "redis.call('pexpire', KEYS[1], maxRemainTime); " +
                    "return 0; " +
                "end;" + 
                    
                "if mode == 'write' then " + 
                    "return 0;" + 
                "end; " +
            "end; " +
                
            "redis.call('del', KEYS[1]); " +
            "redis.call('publish', KEYS[2], ARGV[1]); " +
            "return 1; ",
            Arrays.<Object>asList(getName(), getChannelName(), timeoutPrefix, keyPrefix), 
            LockPubSub.unlockMessage, getLockName(threadId));
}
```

首先整理一下传入lua脚本的参数：

- `KEYS[1]`：锁在redis中的key
- `KEYS[2]`：channel名称，用于发送解锁的消息
- `KEYS[3]`：超时名称的前缀
- `KEYS[4]`：key的前缀
- `ARGV[1]`：解锁消息
- `ARGV[2]`：锁的名称

下面是解锁的流程：

1. 获取锁的mode，如果锁的mode为`false`，表示之前没有设置过读写锁，此时可以直接解锁。发送解锁消息，返回1
2. 查看锁是否存在，如果不存在锁返回nil
3. 将锁名称对应的线程数减1。如果剩余的线程数为0，表示没有其他线程持有该锁了，于是删除该锁
4. 删除超时名称
5. 如果当前锁结构对应的hash表大小大于1，表示有其他线程持有锁。遍历其中里面所有锁的超时时间，将最大的超时时间(maxRemainTime)作为整个锁结构的超时时间。如果最大的超时时间(`maxRemainTime`)大于0，表示还有其他线程持有锁，不能完全释放锁，返回0。如果锁的mode为`write`返回0，不能释放写锁。
6. 否则，没有其他线程持有锁，此时可以彻底释放锁。删除锁结构，发送解锁消息，返回1

## RedissonWriteLock

### tryLockInnerAsync

`tryLockInnerAsync`方法是写锁加锁的最终方法。

```java
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);

    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                        "local mode = redis.call('hget', KEYS[1], 'mode'); " +
                        "if (mode == false) then " +
                              "redis.call('hset', KEYS[1], 'mode', 'write'); " +
                              "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                              "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                              "return nil; " +
                          "end; " +
                          "if (mode == 'write') then " +
                              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " + 
                                  "local currentExpire = redis.call('pttl', KEYS[1]); " +
                                  "redis.call('pexpire', KEYS[1], currentExpire + ARGV[1]); " +
                                  "return nil; " +
                              "end; " +
                            "end;" +
                            "return redis.call('pttl', KEYS[1]);",
                    Arrays.<Object>asList(getName()), 
                    internalLockLeaseTime, getLockName(threadId));
}
```

首先整理一下传入lua脚本的参数：

- `KEYS[1]`：锁在redis中的key
- `ARGV[1]`：锁的超时时间
- `ARGV[2]`：锁的名称

下面是加锁的流程：

1. 获取锁的mode，如果锁的mode为`false`，表示之前没有设置过读写锁，此时可以获得写锁。

    1. 将锁的mode设置为`write`
    2. 将锁名称对应的线程数设置为1
    3. 设置锁的过期时间

2. 如果锁的mode为`write`，并且持有写锁的线程为当前线程，此时可以继续加写锁

    1. 将锁名称对应的线程数增加1
    2. 增加锁的过期时间

3. 否则返回当前锁的过期时间

### unlockInnerAsync

`unlockInnerAsync`方法是写锁解锁的最终方法。

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "local mode = redis.call('hget', KEYS[1], 'mode'); " +
            "if (mode == false) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            "if (mode == 'write') then " +
                "local lockExists = redis.call('hexists', KEYS[1], ARGV[3]); " +
                "if (lockExists == 0) then " +
                    "return nil;" +
                "else " +
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                    "if (counter > 0) then " +
                        "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                        "return 0; " +
                    "else " +
                        "redis.call('hdel', KEYS[1], ARGV[3]); " +
                        "if (redis.call('hlen', KEYS[1]) == 1) then " +
                            "redis.call('del', KEYS[1]); " +
                            "redis.call('publish', KEYS[2], ARGV[1]); " + 
                        "else " +
                            // has unlocked read-locks
                            "redis.call('hset', KEYS[1], 'mode', 'read'); " +
                        "end; " +
                        "return 1; "+
                    "end; " +
                "end; " +
            "end; "
            + "return nil;",
    Arrays.<Object>asList(getName(), getChannelName()), 
    LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));
}
```

首先整理一下传入lua脚本的参数：

- `KEYS[1]`：锁在redis中的key
- `KEYS[2]`：channel名称，用于发送解锁的消息
- `ARGV[1]`：解锁消息
- `ARGV[2]`：锁的过期时间
- `ARGV[3]`：锁的名称

下面是解锁的流程：

1. 获取锁的mode，如果锁的mode为`false`，表示之前没有设置过读写锁，此时可以直接解锁。发送解锁消息，返回1
2. 如果锁的mode为`write`

    1. 检查锁是否存在，如果不存在则返回nil
    2. 将锁名称对应的线程数减1。如果剩余的线程数大于0，表示还有其他线程持有该锁，重新设置锁结构的过期时间
    3. 如果剩余的线程数为0，表示没有其他线程持有该写锁了，于是删除该锁，返回1。如果当前锁结构对应的hash表大小等于1，表示没有其他线程持有锁，此时可以彻底释放锁。删除锁结构，发送解锁消息。否则表示还存在读锁，于是将锁的mode设置为`read`
























[1]: /articles/redis/Redisson简介.html



> http://m.php.cn/article/380297.html
> https://www.jianshu.com/p/4dfbc9b68198
> http://aperise.iteye.com/blog/2400528
> https://github.com/angryz/my-blog/issues/4
> http://dbaplus.cn/news-158-1638-1.html
> https://github.com/redisson/redisson/wiki/8.-分布式锁和同步器


