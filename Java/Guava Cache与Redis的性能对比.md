---
title: Guava Cache与Redis的性能对比
date: 2018/06/19 09:42:00
---

最近遇到一个需求：在程序中缓存接口的一些信息，以便尽可能快的查询到这些信息。

根据缓存应用的耦合度，可以分为local cache(本地缓存)和remote cache(分布式缓存)：

- 本地缓存：指的是在应用中的缓存组件，其最大的优点是应用和cache在同一个进程内部，请求缓存非常快速，没有过多的网络开销等，在单应用不需要集群支持或者集群情况下各节点无需互相通知的场景下使用本地缓存较为合适；同时，它的缺点也是因为缓存跟应用程序耦合，多个应用程序无法直接共享缓存，各应用或集群的各节点都需要维护自己的单独缓存，对内存是一种浪费。
- 分布式缓存：指的是与应用分离的缓存组件或服务，其最大的优点是自身就是一个独立的应用，与本地应用隔离，多个应用可直接共享缓存。
<!-- more -->
## 性能对比

本地缓存有几种实现：编程直接实现缓存、Ehcache、Guava Cache
分布式缓存有几种实现：memcached、redis

我们在本地缓存中选择Guava Cache，在分布式缓存中选择redis，来对比一下他们的性能到底相差多少。

### 读取对比

首先定义一个`User`类，用于表示需要缓存的对象类：

```java
public class User {
    private Integer id;
    private String name;
    private Integer age;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

编写一个`LoadingCache`的测试类:

```java
public class LoadingCacheTest {
    LoadingCache<Integer, User> userCache;

    @Before
    public void before() {
        userCache = CacheBuilder.newBuilder()
                .concurrencyLevel(1)
                .expireAfterWrite(60, TimeUnit.SECONDS)
                .initialCapacity(10)
                .maximumSize(100)
                .build(new CacheLoader<Integer, User>() {
                    @Override
                    public User load(Integer id) throws Exception {
                        User user = new User();
                        user.setId(id);
                        user.setName("wangqi");
                        user.setAge(20);
                        return user;
                    }
                });
    }

    @Test
    public void test01() throws ExecutionException {
        Long totalTime = 0L;
        int n = 1000;
        for (int i = 0; i < n; i++) {
            Long startTime = System.currentTimeMillis();
            User user = userCache.get(0);
            Long endTime = System.currentTimeMillis();
            totalTime += (endTime - startTime);
        }
        System.out.println((double)totalTime / n);
    }
}
```

可以看到，我们通过循序10000次来计算平均的读取时间。输出结果为**0.0025**，即每次的平均读取时间为0.0025毫秒。

再来编写一个`Redis`的测试类：

```java
public class RedisTest {
    Jedis jedis;

    @Before
    public void before() {
        jedis = new Jedis("192.168.1.229");
        User user = new User();
        user.setId(0);
        user.setName("wangqi");
        user.setAge(20);
        jedis.set("user", JSON.toJSONString(user));
    }

    @Test
    public void test01() {
        Long totalTime = 0L;
        int n = 10000;
        for (int i = 0; i < n; i++) {
            Long startTime = System.currentTimeMillis();
            String userJson = jedis.get("user");
            User user = JSON.parseObject(userJson, User.class);
            Long endTime = System.currentTimeMillis();
            totalTime += (endTime - startTime);
        }
        System.out.println((double)totalTime / n);
    }
}
```

可以看到，我们通过循序10000次来计算平均的读取时间。输出结果为**0.6887**，即每次的平均读取时间为0.6887毫秒。

我们看到，虽然redis每次读取时间的绝对值不大，但是和直接本地缓存的方式比较起来还是差距巨大的。

### 并发读取

我们再来看在并发情况下两者的性能差距如何。

```java
public class LoadingCacheConcurrentTest {
    LoadingCache<Integer, User> userCache;
    volatile Long totalTime = 0L;

    @Before
    public void before() {
        userCache = CacheBuilder.newBuilder()
                .concurrencyLevel(1)
                .expireAfterWrite(60, TimeUnit.SECONDS)
                .initialCapacity(10)
                .maximumSize(100)
                .build(new CacheLoader<Integer, User>() {
                    @Override
                    public User load(Integer id) throws Exception {
                        User user = new User();
                        user.setId(id);
                        user.setName("wangqi");
                        user.setAge(20);
                        return user;
                    }
                });
    }

    class AccessThread implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                Long startTime = System.currentTimeMillis();
                try {
                    User user = userCache.get(0);
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
                Long endTime = System.currentTimeMillis();
                totalTime += (endTime - startTime);
            }
        }
    }

    @Test
    public void test01() throws InterruptedException {
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(new AccessThread());
        }
        for (int i = 0; i < 100; i++) {
            threads[i].start();
        }
        for (int i = 0; i < 100; i++) {
            threads[i].join();
        }
        System.out.println((double)totalTime / (10000 * 100));
    }
}
```

100个线程循环读取`LoadingCache`，每个线程循环10000次，计算得到平均的读取时间为**0.006385**。

```java
public class JedisPoolTest {
    JedisPool pool;
    volatile Long totalTime = 0L;

    @Before
    public void before() {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(100);
        pool = new JedisPool(jedisPoolConfig, "192.168.1.229");
        User user = new User();
        user.setId(0);
        user.setName("wangqi");
        user.setAge(20);
        Jedis jedis = pool.getResource();
        jedis.set("user", JSON.toJSONString(user));
    }


    class AccessThread implements Runnable {
        @Override
        public void run() {
            Jedis jedis = null;
            try {
                jedis = pool.getResource();
                for (int i = 0; i < 10000; i++) {
                    Long startTime = System.currentTimeMillis();
                    String userJson = jedis.get("user");
                    User user = JSON.parseObject(userJson, User.class);
                    Long endTime = System.currentTimeMillis();
                    totalTime += (endTime - startTime);
                }
            } finally {
                jedis.close();
            }
        }
    }

    @Test
    public void test01() throws InterruptedException {
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(new AccessThread());
        }
        for (int i = 0; i < 100; i++) {
            threads[i].start();
        }
        for (int i = 0; i < 100; i++) {
            threads[i].join();
        }
        System.out.println((double)totalTime / (10000 * 100));
    }
}
```

100个线程循环读取`redis`，每个线程循环10000次，计算得到平均的读取时间为**19.878017**。

我们看到，在并发条件下`LoadingCache`的读取时间变化不大，但是redis的读取时间上升比较明显，大概比单线程下慢了30倍。

如果数据量不大，且多个服务之间没有相互同步数据的需要，则使用Guava Cache是非常理想的，它使用简单且性能很好。如果需要缓存的数据量很大或者多个服务之间需要共享缓存数据，则redis是理想的选择，虽然读取性能对比Guava Cache满了很多，但是绝对值并不大，大多数情况下满足我们的需要。

> https://tech.meituan.com/cache_about.html
> 
> http://outofmemory.cn/java/guava/cache/how-to-use-guava-cache

