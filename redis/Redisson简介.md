---
title: Redisson简介
date: 2018/08/10 16:47:00
---

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务。其中包括`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`, `CountDownLatch`, `Publish/Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`。Redisson提供了使用Redis的最简单和最便捷的方法。
<!-- more -->
# 配置

## 程序化配置方法

Redisson程序化的配置方法是通过构建`Config`对象实例来实现的。例如：

```java
Config config = new Config();
config.setTransportMode(TransportMode.EPOLL);
config.useClusterServers()
      //可以用"rediss://"来启用SSL连接
      .addNodeAddress("redis://127.0.0.1:7181");
```

## 文件方式配置

Redisson既可以通过用户提供的JSON或YAML格式的文本文件来配置，也可以通过含有Redisson专有命名空间的，Spring框架格式的XML文本文件来配置。

### 通过JSON或YAML格式配置

Redisson的配置文件可以是JSON格式或YAML格式。可以通过调用`Config.fromJSON`方法并指定一个`File`实例来实现读取JSON格式的配置：

```java
Config config = Config.fromJSON(new File("config-file.json"));
RedissonClient redisson = Redisson.create(config);
```

调用`Config.toJSON`方法可以将一个`Config`配置实例序列化为一个含有JSON数据类型的字符串：

```java
Config config = new Config();
// ... 省略许多其他的设置
String jsonFormat = config.toJSON();
```

也通过调用`config.fromYAML`方法并指定一个`File`实例来实现读取YAML格式的配置：

```java
Config config = Config.fromYAML(new File("config-file.yaml"));
RedissonClient redisson = Redisson.create(config);
```

调用`config.toYAML`方法可以将一个`Config`配置实例序列化为一个含有YAML数据类型的字符串：

```java
Config config = new Config();
// ... 省略许多其他的设置
String jsonFormat = config.toYAML();
```

### 通过Spring XML命令空间配置

Redisson为Spring框架提供了一套通过命名空间来配置实例的方式。

一个Redisson的实例可以通过这样的方式来配置：

```
<redisson:client>
    <redisson:single-server ... />
    <!-- 或者 -->
    <redisson:master-slave-servers ... />
    <!-- 或者 -->
    <redisson:sentinel-servers ... />
    <!-- 或者 -->
    <redisson:cluster-servers ... />
    <!-- 或者 -->
    <redisson:replicated-servers ... />
</redisson:client>
```

## 集群模式

集群模式除了适用于Redis集群环境，也适用于任何云计算服务商提供的集群模式。

程序化配置集群的用法：

```java
Config config = new Config();
config.useClusterServers()
    .setScanInterval(2000) // 集群状态扫描间隔时间，单位是毫秒
    //可以用"rediss://"来启用SSL连接
    .addNodeAddress("redis://127.0.0.1:7000", "redis://127.0.0.1:7001")
    .addNodeAddress("redis://127.0.0.1:7002");

RedissonClient redisson = Redisson.create(config);
```

## 云托管模式

云托管模式适用于任何由云计算运营商提供的Redis云服务。

程序化配置云托管模式的方法如下：

```java
Config config = new Config();
config.useReplicatedServers()
    .setScanInterval(2000) // 主节点变化扫描间隔时间
    //可以用"rediss://"来启用SSL连接
    .addNodeAddress("redis://127.0.0.1:7000", "redis://127.0.0.1:7001")
    .addNodeAddress("redis://127.0.0.1:7002");

RedissonClient redisson = Redisson.create(config);
```

## 单Redis节点模式

程序化配置方法：

```java
// 默认连接地址 127.0.0.1:6379
RedissonClient redisson = Redisson.create();

Config config = new Config();
config.useSingleServer().setAddress("myredisserver:6379");
RedissonClient redisson = Redisson.create(config);
```

## 哨兵模式

程序化配置哨兵模式的方法如下：

```java
Config config = new Config();
config.useSentinelServers()
    .setMasterName("mymaster")
    //可以用"rediss://"来启用SSL连接
    .addSentinelAddress("127.0.0.1:26389", "127.0.0.1:26379")
    .addSentinelAddress("127.0.0.1:26319");

RedissonClient redisson = Redisson.create(config);
```

## 主从模式

程序化配置主从模式的用法：

```java
Config config = new Config();
config.useMasterSlaveServers()
    //可以用"rediss://"来启用SSL连接
    .setMasterAddress("redis://127.0.0.1:6379")
    .addSlaveAddress("redis://127.0.0.1:6389", "redis://127.0.0.1:6332", "redis://127.0.0.1:6419")
    .addSlaveAddress("redis://127.0.0.1:6399");

RedissonClient redisson = Redisson.create(config);
```

# 程序接口调用方式

Redisson为每个操作都提供了自动重试策略，当某个命令执行失败时，Redisson会自动进行重试。自动重试策略可以通过修改`retryAttempts`（默认值：3）参数和`retryInterval`（默认值：1000毫秒）参数来进行优化调整。当等待时间达到`retryInternal`指定的时间间隔以后，将自动重试下一次。全部重试失败以后将抛出错误。

Redisson实例本身和Redisson框架提供的所有对象都是线程安全的。

Redisson框架提供的几乎所有对象都包含了同步和和异步相互匹配的方法。这些对象都可以通过`RedissonClient`接口获取。同时还为大部分Redisson对象提供了满足异步流处理标准的程序接口`RedissonReactiveClient`。

以下是关于使用`RAtomicLong`对象的范例：

```java
RedissonClient client = Redisson.create(config);
RAtomicLong longObject = client.getAtomicLong('myLong');
// 同步执行方式
longObject.compareAndSet(3, 401);
// 异步执行方式
longObject.compareAndSetAsync(3, 401);

RedissonReactiveClient client = Redisson.createReactive(config);
RAtomicLongReactive longObject = client.getAtomicLong('myLong');
// 异步流执行方式
longObject.compareAndSet(3, 401);
```

## 异步执行方式

几乎所有的Redisson对象都实现了一个异步接口，异步接口提供的方法名称与其同步接口的方法名称相互匹配。例如：

```java
// RAtomicLong接口继承了RAtomicLongAsync接口
RAtomicLongAsync longObject = client.getAtomicLong("myLong");
RFuture<Boolean> future = longObject.compareAndSetAsync(1, 401);
```

异步执行的方法都会返回一个实现了`RFuture`接口的对象。通过向这个对象添加监听器可以实现非阻塞的执行方式。

```java
// JDK 1.8+ 适用
future.whenComplete((res, exception) -> {
    // ...
});
// 或者
future.thenAccept(res -> {
    // 处理返回
}).exceptionally(exception -> {
    // 处理错误
});
```

```java
// JDK 1.6+ 适用
future.addListener(new FutureListener<Boolean>() {
    @Override
    public void operationComplete(Future<Boolean> future) throws Exception {
         if (future.isSuccess()) {
            // 取得结果
            Boolean result = future.getNow();
            // ...
         } else {
            // 对发生错误的处理
            Throwable cause = future.cause();
         }
    }
});
```

## 异步流执行方式

Redisson提供了满足Reactor项目的异步流处理标准的程序接口。所有Redisson异步流对象都可以通过一个单独的`RedissonReactiveClient`接口来获取。该功能要求JDK 7或以上版本。使用范例如下：

```java
RedissonReactiveClient client = Redisson.createReactive(config);
RAtomicLongReactive longObject = client.getAtomicLong("myLong");

Publisher<Boolean> csPublisher = longObject.compareAndSet(10, 91);

Publisher<Long> getPublisher = longObject.get();
```

也可以在RxJavaReactiveStreams项目的帮助下，通过使用RxJava标准来达到使用异步流处理标准的目的。例如：

```java
RMap<String, Integer> map = client.getMap("mapMap");
Observable<Integer> observable = RxReactiveStreams.toObservable(map.put("1", 324));
```








> https://github.com/redisson/redisson/wiki/1.-概述
> https://github.com/redisson/redisson/wiki/2.-配置方法
> https://github.com/redisson/redisson/wiki/3.-程序接口调用方式

