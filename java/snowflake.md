---
title: SnowFlake
date: 2017-11-22 18:37:44
---

# SnowFlake

分布式ID生成一个很小但是很重要的基础应用，比如想要手动给数据库中的一条数据设置一个id。

最先能想到是UUID。UUID保证对在同一时空中的所有机器都是唯一的。UUID的缺点是太长(32位)，并且既有数字又有字母。

如果想要生成纯数字的id，则Twitter的SnowFlake是一个非常优秀的id生成方案。

实现也非常简单，8Byte是一个Long，8Byte等于64bit，SnowFlake就是由毫秒级的时间41位 + 机器ID 10位 + 毫秒内序列12位组成。当然也可以根据需要调整机器位数和毫秒内序列位数比例。
<!--more-->
SnowFlake的优点是：

- 比UUID短，一般9-17位左右
- 性能非常出色，每秒几十万

Java实现：

```java
public class SnowFlake {
    /**
     * 起始时间戳
     */
    private final long startStamp = 1480166465631L;
    /**
     * 机器id所占的位数
     */
    private final long workerIdBits = 10L;
    /**
     * 序列号所占的位数
     */
    private final long sequenceBits = 12L;
    /**
     * 机器id的最大值
     */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    /**
     * 序列号的最大值
     */
    private final long maxSequence = -1L ^ (-1L << sequenceBits);
    /**
     * 机器id左移的位数
     */
    private final long workerIdShift = sequenceBits;
    /**
     * 时间戳左移的位数
     */
    private final long timeStampShift = workerIdShift + workerIdBits;

    private long workerId;
    private long sequence = 0L;
    private long lastTimeStamp = -1L;

    public SnowFlake(long workerId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker id can't be greater than %d or less than 0", maxWorkerId));
        }
        this.workerId = workerId;
    }

    /**
     * 获得下一个ID
     * @return
     */
    public synchronized long nextId() {
        long currentTimeStamp = timeGen();

        // 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过，这个时间应该抛出异常
        if (currentTimeStamp < lastTimeStamp) {
            throw new RuntimeException(String.format("Clock moved backwards. Refusing to generate id for %d milliseconds", lastTimeStamp - currentTimeStamp));
        }

        // 如果是同一毫秒生成的，则进行序列自增
        if (lastTimeStamp == currentTimeStamp) {
            sequence = (sequence + 1) & maxSequence;
            // 同一毫秒内序列溢出
            if (sequence == 0) {
                // 阻塞到下一个毫秒，获得新的时间戳
                currentTimeStamp = tilNextMillis(lastTimeStamp);
            }
        } else {
            sequence = 0L;
        }

        lastTimeStamp = currentTimeStamp;
        return (currentTimeStamp - startStamp) << timeStampShift    // 时间戳部分
                | (workerId << workerIdShift)                       // 机器id部分
                | sequence;                                         // 序列号部分
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     * @param lastTimeStamp
     * @return
     */
    private long tilNextMillis(long lastTimeStamp) {
        long timeStamp = timeGen();
        while (timeStamp < lastTimeStamp) {
            timeStamp = timeGen();
        }
        return timeStamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     * @return
     */
    private long timeGen() {
        return System.currentTimeMillis();
    }

    public static void main(String[] args) {
        SnowFlake snowFlake = new SnowFlake(1);
        int n = 100;
        Set<String> set = new HashSet<>();
        for (int i = 0; i < n; i++) {
            long id = snowFlake.nextId();
            String s = String.valueOf(id);
            set.add(s);
            System.out.println(s);
        }
        System.out.println(set.size());
    }
}
```

参考资料：
> https://leokongwq.github.io/2016/11/02/distributed-id-generation.html
> 
> http://www.lanindex.com/twitter-snowflake%EF%BC%8C64%E4%BD%8D%E8%87%AA%E5%A2%9Eid%E7%AE%97%E6%B3%95%E8%AF%A6%E8%A7%A3/
> 
> https://www.cnblogs.com/relucent/p/4955340.html
> 
> https://github.com/beyondfengyu/SnowFlake/blob/master/SnowFlake.java


