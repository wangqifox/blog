---
title: redis分布式锁总结
date: 2018/08/02 18:42:00
---

在分布式环境中，数据一致性是一直以来需要关注并且去解决的问题，分布式锁也就成为了一种广泛使用的技术，常用的分布式实现方式为Redis，Zookeeper，其中基于Redis的分布式锁的使用更加广泛。
<!-- more -->
本文是对参考资料中redis分布式锁的进一步总结和实践。

# 版本1

首先来看第一个版本：

```
lock() {
    SETNX Key 1
    EXPIRE Key Seconds
}
unlock() {
    delete Key
}
```

给锁加一个过期时间是为了避免应用在服务重启或者异常导致锁无法释放后，不会出现一直无法释放的情况。

该方案有两个问题：

1. 如果执行完一条命令后应用异常或者重启，锁将无法过期，一种改善方案就是使用Lua脚本（包含SETNX和EXPIRE两条命令），但是如果Redis仅执行了一条命令后crash或者发生主从切换，依然会出现锁没有过期时间，最终导致无法释放。
2. 在释放锁的过程中，无论锁是否获取成功，都在finally中释放锁，这样会导致释放掉其他进程持有的锁。这个问题将在后续的3.0版本中解决。

该方案的实现详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version1/RedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version1/RedisLock.java)

# 版本1.1

这个版本基于GETSET命令，去掉了EXPIRE命令，通过value的时间戳值来判断过期。

```
lock() {
    newExpireTime = currentTimestamp + expireSeconds
    if (SETNX key newExpireTime == 0) {
        oldExpireTime = GET key
        if (oldExpireTime < currentTimestamp) {
            newExpireTime = currentTimestamp + expireSeconds
            currentExpireTime = GETSET key newExpireTime
            if (currentExpireTime == oldExpireTime) {
                return 1
            } else {
                return 0
            }
        } else {
            return 0
        }
    } else {
        return 1
    }
}
unlock() {
    delete key
}
```

流程如下：

1. SETNX(key, expireTime)获取锁

    1. 如果获取成功返回1
    2. 如果获取锁失败，通过GET(key)返回保存的时间戳

        1. 如果时间戳大于等于当前时间戳，说明保存的时间戳还没有过期，返回0
        2. 否则说明保存的时间戳已经过期了

            1. GETSET(key, newExpireTime)修改value为newExpireTime
            2. 检查GETSET返回的旧值，如果等于GET返回的值，说明获取锁成功，返回1；否则说明value被其他进行修改过了，获取锁失败，返回0

该方案有两个问题：

1. 在锁竞争较高的情况下，会出现value不断被覆盖，但是没有一个client获取到锁。
2. 在获取锁的过程中不断地修改原有锁的数据，设想一种场景C1, C2竞争锁，C1获取到了锁，C2执行了GETSET操作修改了C1锁的过期时间，如果C1没有正确释放锁，锁的过期时间被延长，其他client需要等待更久的时间

该方案详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version1_1/RedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version1_1/RedisLock.java)

# 版本2

这个版本基于SETNX命令。

```
lock() {
    SET key 1 EX timeout NX
}
unlock() {
    delete key
}
```

Redis 2.6.12后`SET`提供了一个`NX`参数，等同于`SETNX`命令（官方文档上提醒后面的版本有可能去掉`SETNX` `SETEX` `PSETEX`，并用`SET`命令代替）；提供了一个`EX`参数，设置键的过期时间。这样就解决了两条命令无法保证原子性的问题。

但是设想下面一个场景：

1. C1成功获取到了锁，之后C1因为GC进入等待或者未知原因导致任务执行过长，最后在锁失效前C1没有主动释放锁
2. C2在C1的锁超时后获取到锁，并且开始执行，这个时候C1和C2都同时执行，会因重复执行造成数据不一致等未知情况
3. C1如果先执行完毕，则会释放C2的锁，此时可能导致另外一个C3进程获取到了锁

大致的流程图：

![unsafe-lock](media/unsafe-lock.png)

存在问题：

1. 由于C1的停顿导致C1和C2同时获得了锁并且同时在执行，在业务实现中要求必须保证幂等性
2. C1释放了不属于C1的锁

该方案详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version2/RedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version2/RedisLock.java)

# 版本3

```
lock() {
    SET key unixTimestamp EX timeout NX
}
unlock() {
    EVAL(
        // luascript
        if redis.call("get", key) == ARGV[1] then
            return redis.call("del", key)
        else
            return 0
        end
    )
}
```

这个方案通过指定value为时间戳，并在释放锁的时候检查锁的vallue是否为获取锁的value，避免了版本2中提到的C1释放C2持有锁的问题；另外在释放锁的时候因为涉及到多个Redis操作，所以使用lua脚本来避免并发问题。

该方案存在的问题：

1. 如果在并发极高的场景下，比如抢红包场景，可能存在unixTimestamp重复问题，另外由于不能保证分布式环境下的物理时钟一致性，也可能存在unixTimestamp重复问题，只不过极少情况下会遇到。

该方案详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3/RedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3/RedisLock.java)

# 版本3.1

```
lock() {
    SET key uniqId seconds
}
unlock() {
    EVAL(
        // luascript
        if redis.call("get", key) == ARGV[1] then
            return redis.call("del", key)
        else
            return 0
        end
    )
}
```

该版本和版本3的实现基本一致，有两处优化：

1. 使用一个自增的唯一id代替时间戳来规避版本3提到的时钟问题
2. 使用`SET`代替`SETNX`命令。

这个方案是目前最优的分布式锁方案，但是仍然有以下两个问题：

1. 没有解决版本2提出的问题。当线程A执行完成前，锁就过期了，这时候线程B可以成功获得锁，此时有两个线程在同时访问代码块，仍然是不完美的。
2. 如果Redis是集群环境，由于Redis集群数据同步为异步，假设在Master节点获取到锁后未完成数据同步情况下Master节点crash，此时在新的Master节点依然可以获取锁，所以多个client同时获取到了锁。

该方案详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3_1/RedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3_1/RedisLock.java)

# 版本3.2

针对版本3.1中的第一个问题，我们可以让获得锁的线程开启一个守护线程，用来给快要过期的锁"续命"。

守护线程定期执行expire指令，为快过期的锁"续命"。当持有锁的线程执行完后，显式关掉守护线程。如果持有锁的节点突然断电，由于持有锁的线程和守护线程在同一个进程，守护线程也会停下，这把锁到了超时的时候，没人给他续命，也就自动释放了。

该方案详见：[https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3_2/AutoExpireRedisLock.java](https://github.com/wangqifox/redis-lock/blob/master/src/main/java/love/wangqi/version3_2/AutoExpireRedisLock.java)

# 分布式Redis锁：Redlock

版本3.1在单实例场景下是安全的，针对如何实现分布式Redis的锁，国外的分布式专家有过激烈的讨论，antirez提出了分布式锁算法Redlock。

有关Redlock的内容，我们在下一篇文章中详解。


# 总结

不论是基于`SETNX`版本的Redis单实例分布式锁，还是Redlock分布式锁，都是为了保证以下特性：

1. 安全性：独享（相互排斥），在任意一个时刻，只有一个客户端持有锁。
2. 活性：

    - 无死锁：即便持有锁的客户端崩溃(crashed)或者网络被分裂(gets partitioned)，锁仍然可以被获取。
    - 容错性：只要超过半数Redis节点可用，锁都能被正确获取和释放。

所以在开发或者使用分布式锁的过程中要保证安全性和活性，避免出现不可预测的结果。

另外每个版本的分布式锁都存在一些问题，在锁的使用上要针对锁的应用场景选择合适的锁，通常情况下锁的使用场景包括：

- Efficiency(效率)：只需要一个Client来完成操作，不需要重复执行，这是一个相对宽松的分布式锁，只需要保证锁的活性即可
- Correctness(正确性)：多个Client保证严格的互斥性，不允许出现同时持有锁或者同时操作统一资源，这种场景下需要在锁的选择和使用上更加严格，同时在业务代码上尽量做到幂等

















> http://tech.dianwoda.com/2018/04/11/redisfen-bu-shi-suo-jin-hua-shi/
> https://mp.weixin.qq.com/s?__biz=MzIxMjE5MTE1Nw==&mid=2653194065&idx=1&sn=1baa162e40d48ce9b44ea5c4b2c71ad7&chksm=8c99f58bbbee7c9d5b5725da5ee38fe0f89d7a816f3414806785aea0fe5ae766769600d3e982&scene=21#wechat_redirect
> http://redis.cn/topics/distlock.html


