---
title: 从Paxos到ZooKeeper读书笔记
date: 2018/04/02 21:35:00
---

# 第一章 分布式架构

## ACID:

- 原子性：事务包含的各项操作在一次执行过程中，只允许出现以下两种状态之一：全部成功执行、全部不执行
- 一致性：事务执行的结果必须是使数据库从一个一致性状态转变到另一个一致性状态，当数据库只包含成功事务提交的结果时，就能说数据库处于一致性状态
- 隔离性：不同的事务并发操纵相同的数据时，每个事务都有各自完整的数据空间，即一个事务内部的操作及使用的数据对其他并发事务是隔离的，并发执行的各个事务之间不能互相干扰
- 持久性：一个事务一旦提交，它对数据库中对应数据的状态变更就应该是永久性的。

## CAP理论：

一个分布式系统不可能同时满足一致性(C: Consistency)、可用性(A: Availability)和分区容错性(P: Partition tolerance)这三个基本需求，最多只能同时满足其中的两项

- 一致性：在分布式环境中，一致性是指数据在多个副本之间是否能够保持一致的特性。
- 可用性：系统提供的服务必须一直处于可用的状态，对于用户的每个操作请求总是能够在有限的时间内返回结果。
- 分区容错性：分布式系统在遇到任何网络分区故障的时候，仍然需要能够保证对外提供一致性和可用性的服务，除非是整个网络环境都发生了故障

|放弃CAP定理|说明|
|----------|---|
|放弃P|如果希望能够避免系统出现分区容错性问题，一种较为简单的做法是将所有的数据都放在一个分布式节点上|
|放弃A|一旦系统遇到网络分区或其他故障时，那么受到影响的服务需要等待一定的时间，因此在等待期间系统无法对外提供正常的服务，即不可用|
|放弃C|放弃一致性指的是放弃数据的强一致性，而保留数据的最终一致性。这样的系统无法保证数据保持实时的一致性，但是能够承诺的是，数据最终会达到一个一致的状态。|

对于一个分布式系统而言，分区容错性是一个最基本的要求。因此系统架构设计师往往需要把精力花在如何根据业务特点在C（一致性）和A（可用性）之间寻求平衡。

## BASE理论：

BASE是Basically Available（基本可用）、Soft state（软状态）和Eventually consistent（最终一致性）三个短语的简写。BASE是对CAP中一致性和可用性权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，其核心思想是即使无法做到强一致性（Strong consistency），但每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性（Eventual consistency）。

- 基本可用

    基本可用是指分布式系统在出现不可预知故障的时候，允许损失部分可用性。以下两个就是“基本可用”的典型例子。
    
    - 响应时间上的损失
    - 功能上的损失

- 弱状态

    弱状态也称为软状态，和硬状态相对，是指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时。
    
- 最终一致性

    最终一致性强调的是系统中所有的数据副本，在经过一段时间的同步后，最终能够达到一个一致的状态。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性。
    
    最终一致性存在以下五类主要变种：
    
    - 因果一致性（Causal consistency）

        因果一致性是指，如果进程A在更新完某个数据项后通知了进行B，那么进程B之后对该数据项的访问都应该能够获取到进程A更新后的最新值，并且如果进程B要对该数据项进行更新操作的话，务必基于进行A更新后的最新值，即不能发生丢失更新情况。与此同时，与进行A无因果关系的进程C的数据访问则没有这样的限制。
        
    - 读己之所写（Read your writes）

        读已之所写是指，进程A更新一个数据项之后，它自己总是能够访问到更新过的最新值，而不会看到旧值。也就是说，对于单个数据获取者来说，其读取到的数据，一定不会比自己上次写入的值旧。因此，读己之所写也可以看作是一种特殊的因果一致性。
        
    - 会话一致性（Session consistency）

        会话一致性将对系统数据的访问过程框定在了一个会话当中：系统能保证在同一个有效的会话中实现“读己之所写”的一致性，也就是说，执行更新操作之后，客户端能够在同一个会话中始终读取到该数据项的最新值。
        
    - 单调读一致性（Monotonic read consistency）

        单调读一致性是指如果一个进程从系统中读取一个数据项的某个值后，那么系统对于该进程后续的任何数据访问都不应该返回更旧的值。
        
    - 单调写一致性（Monotonic write consistency）

        单调写一致性是指，一个系统需要能够保证来自同一个进程的写操作被顺序地执行。
        
        
# 第二章 一致性协议

## 2PC与3PC

当一个事务操作需要跨越多个分布式节点的时候，为了保持事务处理的ACID特性，就需要引入一个称谓“协调者(Coordinator)”的组件来统一调度所有分布式节点的执行逻辑，这些被调度的分布式节点则被称为“参与者(Participant)”。

### 2PC

#### 阶段一：提交事务请求

1. 事务询问

    协调者向所有的参与者发送事务内容，询问是否可以执行事务提交操作，并开始等待各参与者的响应

2. 执行事务

    各参与者节点执行事务操作，并将undo和redo信息记入事务日志中

3. 各参与者向协调者反馈事务询问的响应

    如果参与者成功执行了事务操作，那么就反馈给协调者Yes响应，表示事务可以执行；如果参与者没有成功执行事务，那么就反馈给协调者No响应，表示事务不可以执行

#### 阶段二：执行事务提交

协调者根据各参与者的反馈情况来决定最终是否可以进行事务提交操作，正常情况下，包含以下两种可能：

- 执行事务提交

    假如协调者从所有的参与者获得的反馈都是Yes响应，那么就会执行事务提交。

    1. 发送提交请求：协调者向所有参与者节点发出Commit请求
    2. 事务提交：参与者接收到Commit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源
    3. 反馈事务提交结果：参与者在完成事务提交之后，向协调者发送Ack消息
    4. 完成事务：协调者接收到所有参与者反馈的Ack消息后，完成事务

- 中断事务

    假如任何一个参与者向协调者反馈了No响应，或者在等待超时之后，协调者尚无法接收到所有参与者的反馈响应，那么就会中断事务。
    
    1. 发送回滚请求：协调者向所有参与者节点发出Rollback请求
    2. 事务回滚：参与者接收到Rollback请求后，会利用其在阶段一中记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源
    3. 反馈事务回滚结果：参与者在完成事务回滚之后，向协调者发送Ack消息
    4. 中断事务：协调者接收到所有参与者反馈的Ack消息后，完成事务中断

优点：原理简单，实现方便
缺点：同步阻塞、单点问题、数据不一致、太过保守
    
### 3PC

三阶段提交，是2PC的改进版，其将二阶段提交协议的“提交事务请求”过程一分为二，形成了由CanCommit、PreCommit和doCommit三个阶段组成的事务处理协议。

#### 阶段一：CanCommit

1. 事务询问：协调者向所有的参与者发送一个包含事务内容的CanCommit请求，询问是否可以执行事务提交操作，并开始等待各参与者的响应。
2. 各参与者向协调者反馈事务询问的响应：参与者在接收到来自协调者的CanCommit请求后，正常情况下，如果其自身认为可以顺利执行事务，那么会反馈Yes响应，并进入预备状态，否则反馈No响应。

#### 阶段二：PreCommit

协调者根据各参与者的反馈情况来决定是否可以进行事务的PreCommit操作，正常情况下，包含两种可能。

- 执行事务预提交

    假如协调者从所有的参与者获得的反馈都是Yes响应，那么就会执行事务预提交

    1. 发送预提交请求：协调者向所有参与者节点发出preCommit的请求，并进入Prepared阶段
    2. 事务预提交：参与者接收到preCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中
    3. 各参与者向协调者反馈事务执行的响应：如果参与者成功执行了事务操作，那么就会反馈给协调者Ack响应，同时等待最终的指令：提交(commit)或中止(abort)

- 中断事务

    假如任何一个参与者向协调者反馈了No响应，或者在等待超时之后，协调者尚无法接收到所有参与者的反馈响应，那么就会中断事务。
    
    1. 发送中断请求：协调者向所有参与者节点发出abort请求
    2. 中断事务：无论是收到来自协调者的abort请求，或者是在等待协调者请求过程中出现超时，参与者都会中断事务

#### 阶段三：doCommit

- 执行提交

    1. 发送提交请求：进入这一阶段，假设协调者处于正常工作状态，并且它接收到了来自所有参与者的Ack响应，那么它将从“预提交”状态转换到“提交”状态，并向所有的参与者发送doCommit请求
    2. 事务提交：参与者接收到doCommit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源
    3. 反馈事务提交结果：参与者在完成事务提交之后，向协调者发送Ack消息
    4. 完成事务：协调者接收到所有参与者反馈的Ack消息后，完成事务

- 中断事务

    假设协调者处于正常工作状态，并且有任意一个参与者向协调者反馈了No响应，或者在等待超时之后，协调者尚无法接收到所有参与者的反馈响应，那么就会中断事务。
    
    1. 发送中断请求：协调者向所有的参与者节点发送abort请求
    2. 事务回滚：参与者接收到abort请求后，会利用其在阶段二中记录的undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源
    3. 反馈事务回滚结果：参与者在完成事务回滚之后，向协调者发送Ack消息
    4. 中断事务：协调者接收到所有参与者反馈的Ack消息后，中断事务

一旦进入阶段三，可能会存在以下两种故障：协调者出现问题、协调者和参与者之间的网络出现故障。无论出现哪种情况，最终都会导致参与者无法及时接收到来自协调者的doCommit或是abort请求，针对这样的异常情况，参与者都会在等待超时之后，继续进行事务提交。

优点：降低了参与者的阻塞范围，并且能够在出现单点故障后继续达成一致。
缺点：参与者接收到preCommit消息后，如果网络出现分区，此时协调者所在的节点和参与者无法进行正常的网络通信，在这种情况下，该参与者依然会进行事务的提交，这必然出现数据的不一致性。

# 第4章 ZooKeeper与Paxos

Zookeeper为分布式应用提供了高效且可靠的分布式协调服务，提供了诸如统一命名服务、配置管理和分布式锁等分布式的基础服务。在解决分布式数据一致性方面，Zookeeper并没有直接采用Paxos算法，而是采用了一种被称为ZAB的一致性协议。

Zookeeper是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁、分布式队列等功能。ZooKeeper可以保证如下分布式一致性特性。

- 顺序一致性
    
    从同一个客户端发起的事务请求，最终将会严格地按照其发起顺序被应用到Zookeeper中去
    
- 原子性

    所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说，要么整个集群所有机器都成功应用了某一个事务，要么都没有应用，一定不会出现集群中部分机器应用了该事务，而另外一部分没有应用的情况。
    
- 单一视图

    无论客户端连接的是哪个ZooKeeper服务器，其看到的服务器数据模型都是一致的。
    
- 可靠性

    一旦服务端成功地应用了一个事务，并完成对客户端的响应，那么该事务所引起的服务端状态变更将会被一直保留下来，除非有另一个事务又対其进行了变更。
    
- 实时性

    通常人们看到实时性的第一个反应是，一旦一个事务被成功应用，那么客户端能够立刻从服务端上读取到这个事务变更后的最新数据状态。这里需要注意的是，ZooKeeper仅仅保证在一定的时间段内，客户端最终一定能够从服务端上读取到最新的数据状态。

# 第5章 使用ZooKeeper

## Java客户端API使用

### 创建会话

```java
ZooKeeper(String connectString, int sessionTimeout, Watcher watcher);
ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly);
ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd);
ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly);
```

### 创建节点

```java
String create(final String path, byte data[], List<ACL> acl, CreateMode createMode);
String create(final String path, byte data[], List<ACL> acl, CreateMode createMode, StringCallback cb, Object ctx);
```

### 删除节点

```java
public void delete(final String path, int version);
public void delete(final String path, int version, VoidCallback cb, Object ctx);
```

### 读取数据

```java
List<String> getChildren(final String path, Watcher watcher);
List<String> getChildren(String path, boolean watch);
void getChildren(final String path, Watcher watcher, ChildrenCallback cb, Object ctx);
void getChildren(String path, boolean watch, ChildrenCallback cb, Object ctx);
List<String> getChildren(final String path, Watcher watcher, Stat stat);
List<String> getChildren(String path, boolean watch, Stat stat);
void getChildren(final String path, Watcher watcher, Children2Callback cb, Object ctx);
void getChildren(String path, boolean watch, Children2Callback cb, Object ctx);
```

```java
byte[] getData(final String path, Watcher watcher, Stat stat);
byte[] getData(String path, boolean watch, Stat stat);
void getData(final String path, Watcher watcher, DataCallback cb, Object ctx);
void getData(String path, boolean watch, DataCallback cb, Object ctx);
```

### 更新数据

```java
Stat setData(final String path, byte data[], int version);
void setData(final String path, byte data[], int version, StatCallback cb, Object ctx);
```

### 检测节点是否存在

```java
public Stat exists(final String path, Watcher watcher);
public Stat exists(String path, boolean watch);
public void exists(final String path, Watcher watcher, StatCallback cb, Object ctx);
public void exists(String path, boolean watch, StatCallback cb, Object ctx);
```

### 权限控制

```java
addAuthInfo(String scheme, byte[] auth);
```

## ZkClient

### 创建会话

```java
public ZkClient(String serverstring)
public ZkClient(String zkServers, int connectionTimeout)
public ZkClient(String zkServers, int sessionTimeout, int connectionTimeout)
public ZkClient(String zkServers, int sessionTimeout, int connectionTimeout, ZkSerializer zkSerializer)
public ZkClient(IZkConnection connection)
public ZkClient(IZkConnection connection, int connectionTimeout)
public ZkClient(IZkConnection zkConnection, int connectionTimeout, ZkSerializer zkSerializer)
```

### 创建节点

```java
String create(final String path, Object data, final CreateMode mode)
String create(final String path, Object data, final List<ACL> acl, final CreateMode mode)
void create(final String path, Object data, final CreateMode mode, final AsyncCallback.StringCallback callback, final Object context)
void createEphemeral(final String path)
void createEphemeral(final String path, final Object data)
void createPersistent(String path)
void createPersistent(String path, boolean createParents)
void createPersistent(String path, Object data)
void createPersistent(String path, List<ACL> acl, Object data)
String createPersistentSequential(String path, Object data)
String createEphemeralSequential(final String path, final Object data)
```

### 删除节点

```java
boolean delete(final String path)
delete(final String path, final AsyncCallback.VoidCallback callback, final Object context)
boolean deleteRecursive(String path)
```

### 读取数据

```java
List<String> getChildren(String path)

List<String> subscribeChildChanges(String path, IZkChildListener listener)
```

```java
<T extends Object> T readData(String path)
<T extends Object> T readData(String path, boolean returnNullIfPathNotExists)
<T extends Object> T readData(String path, Stat stat)

List<String> subscribeDataChanges(String path, IZkDataListener listener)
```

### 更新数据

```java
void writeData(String path, Object data)
void writeData(final String path, Object data, final int expectedVersion)
```

### 检测节点是否存在

```java
boolean exists(final String path)
```

## Curator

### 创建会话

### 创建节点

### 删除节点

### 读取数据

### 更新数据

### 异步接口

### 典型使用场景

#### 事件监听

#### Master选举

#### 分布式锁

#### 分布式计数器

#### 分布式Barrier

#### 工具

##### ZKPaths

##### EnsurePath

##### TestingServer


