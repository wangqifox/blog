---
title: 基于redis的分布式限流方案
date: 2018/08/01 14:14:00
---

上一篇文章[限流技术总结][1]中，我们说到传统的限流算法只能实现单机的限流，如果要实现分布式限流，可以考虑借助redis来实现限流算法。

这篇文章我们就来实现这个基于redis的分布式限流方案。
<!-- more -->
为什么要选用redis：

1. redis效率高，易扩展
2. redis对语言无关，可以更好的接入不同语言开发的系统（异构）
3. redis单进程单线程的特点可以更好的解决最终一致性，多进程间协同控制更为容易

# 计数器算法实现

先来看一个比较简单的计数器算法的实现。

计数器算法前文[限流技术总结][1]有过详细说明，现在我们用lua代码来实现这个算法：

```lua
-- 资源唯一标识
local key = KEYS[1]
-- 时间窗口内最大并发数
local max_permits = tonumber(KEYS[2])
-- 窗口的间隔时间
local interval_milliseconds = tonumber(KEYS[3])
-- 获取的并发数
local permits = tonumber(ARGV[1])

local current_permits = tonumber(redis.call("get", key) or 0)

-- 如果超过了最大并发数，返回false
if (current_permits + permits > max_permits) then
    return false
else
    -- 增加并发计数
    redis.call("incrby", key, permits)
    -- 如果key中保存的并发计数为0，说明当前是一个新的时间窗口，它的过期时间设置为窗口的过期时间
    if (current_permits == 0) then
        redis.call("pexpire", key, interval_milliseconds)
    end
    return true
end
```

# 令牌桶算法

计数器算法容易出现不平滑的情况，瞬间的qps有可能超过系统的承载。因此在实际场景中我们一般很少使用。

令牌桶算法是一个比较常用的限流算法，也是Guava中使用的算法。我们使用lua来实现令牌桶算法：

```lua
-- key
local key = KEYS[1]
-- 最大存储的令牌数
local max_permits = tonumber(KEYS[2])
-- 每秒钟产生的令牌数
local permits_per_second = tonumber(KEYS[3])
-- 请求的令牌数
local required_permits = tonumber(ARGV[1])

-- 下次请求可以获取令牌的起始时间
local next_free_ticket_micros = tonumber(redis.call('hget', key, 'next_free_ticket_micros') or 0)

-- 当前时间
local time = redis.call('time')
local now_micros = tonumber(time[1]) * 1000000 + tonumber(time[2])

-- 查询获取令牌是否超时
if (ARGV[2] ~= nil) then
    -- 获取令牌的超时时间
    local timeout_micros = tonumber(ARGV[2])
    local micros_to_wait = next_free_ticket_micros - now_micros
    if (micros_to_wait > timeout_micros) then
        return micros_to_wait
    end
end

-- 当前存储的令牌数
local stored_permits = tonumber(redis.call('hget', key, 'stored_permits') or 0)
-- 添加令牌的时间间隔
local stable_interval_micros = 1000000 / permits_per_second

-- 补充令牌
if (now_micros > next_free_ticket_micros) then
    local new_permits = (now_micros - next_free_ticket_micros) / stable_interval_micros
    stored_permits = math.min(max_permits, stored_permits + new_permits)
    next_free_ticket_micros = now_micros
end

-- 消耗令牌
local moment_available = next_free_ticket_micros
local stored_permits_to_spend = math.min(required_permits, stored_permits)
local fresh_permits = required_permits - stored_permits_to_spend;
local wait_micros = fresh_permits * stable_interval_micros

redis.replicate_commands()
redis.call('hset', key, 'stored_permits', stored_permits - stored_permits_to_spend)
redis.call('hset', key, 'next_free_ticket_micros', next_free_ticket_micros + wait_micros)
redis.call('expire', key, 10)

-- 返回需要等待的时间长度
return moment_available - now_micros
```

- 如果是`acquire`方法，则执行：

```
eval 'lua脚本' 3 '自定义的key' '最大存储的令牌数' '每秒钟产生的令牌数' '请求的令牌数'
```

redis将返回获取请求成功后，线程需要等待的微秒数

- 如果是`tryAcquire`方法，则执行：

```
eval 'lua脚本' 3 '自定义的key' '最大存储的令牌数' '每秒钟产生的令牌数' '请求的令牌数' '最大等待的微秒数'
```

redis同样返回需要等待的微秒数，将该返回值与最大等待微秒数做比较，如果redis返回的值较大，则说明失败；反之则是成功，并根据返回值让线程等待。



































完整代码请参考：[https://github.com/wangqifox/redis-limiter](https://github.com/wangqifox/redis-limiter)


[1]: /articles/Java/限流技术总结.html

> http://tech.dianwoda.com/2017/09/11/talk-about-rate-limit/
> https://www.zybuluo.com/kay2/note/949160
> https://segmentfault.com/a/1190000012947169


