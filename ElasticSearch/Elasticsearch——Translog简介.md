---
title: ElasticSearch——Translog简介
date: 2019/11/29 11:10:00
---

本文是对`Elasticsearch`中`Translog`（事务日志）的一个简单整理。

<!-- more -->

我们知道`Elasticsearch`是一个`near-realtime`（近实时）的搜索引擎。这个近实时指的是数据添加到`Elasticsearch`后并不能马上被搜索到，而是要等待一定时间（默认为1秒）。

这里的原因是`ElasticSearch`底层依赖lucene来提供搜索能力，一个`lucene index`由许多独立的`Segments`组成。

![lucene_index](media/lucene_index.jpeg)

`Segment`是最小的数据存储单元。其中包含了文档中的词汇字典、词汇字典的倒排索引以及Document的字段数据。因此`Segment`直接提供了搜索功能。但是`Segment`能提供搜索的前提是数据必须被提交，即文档经过一系列的处理之后生成倒排索引等一系列数据。可以想见这个过程是比较耗时的。因此`Elasticsearch`并不会每接收到一条数据就提交到一个`Segment`中，一方面是因为这样耗时太长，另一方面是这样会生成巨量的`Segment`，降低了IO性能。

`Elasticsearch`采取的机制是将数据添加到`lucene`，`lucene`内部会维护一个数据缓冲区，此时数据都是不可搜索的。每隔一段时间（默认为1秒），`Elasticsearch`会执行一次`refresh`操作：`lucene`中所有的缓存数据都被写入到一个新的`Segment`，清空缓存数据。此时数据就可以被搜索。当然，每次执行`refresh`操作都会生成一个新的`Segment`文件，这样一来`Segment`文件有大有小，相当碎片化。`Elasticsearch`内部会开启一个线程将小的`Segment`合并（Merge）成大的`Segment`，减少碎片化，降低文件打开数，提升IO性能。

不过这样也带来一个问题。数据写入缓冲区中，没有及时保存到磁盘中，一旦发生程序崩溃或者服务器宕机，数据就会发生丢失。为了保证可靠性，`Elasticsearch`引入了`Translog`（事务日志）。每次数据被添加或者删除，都要在`Translog`中添加一条记录。这样一旦发生崩溃，数据可以从`Translog`中恢复。

不过，不要以为数据被写到`Translog`中就已经被保存到磁盘了。一般情况下，对磁盘文件的write操作，更新的只是内存中的页缓存，而脏页面不会立即更新到磁盘中，而是由操作系统统一调度，如由专门的flusher内核线程在满足一定条件时（如一定时间间隔、内存中的脏页面达到一定比例）将脏页面同步到磁盘上。因此如果服务器在write之后、磁盘同步之前宕机，则数据会丢失。这时需要调用操作系统提供的`fsync`功能来确保文件所有已修改的内容被正确同步到磁盘上。

`Elasticsearch`提供了几个参数配置来控制`Translog`的同步：

- `index.translog.durability`

    该参数控制如何同步`Translog`数据。有两个选项：
    
    - `request`（默认）：每次请求（包括index、delete、update、bulk）都会执行`fsync`，将`Translog`的数据同步到磁盘中。
    - `async`：异步提交`Translog`数据。和下面的`index.translog.sync_interval`参数配合使用，每隔`sync_interval`的时间将`Translog`数据提交到磁盘，这样一来性能会有所提升，但是在间隔时间内的数据就会有丢失的风险。

- `index.translog.sync_interval`：该参数控制`Translog`的同步时间间隔。默认为5秒。
- `index.translog.flush_threshold_size`：该参数控制`Translog`的大小，默认为512MB。防止过大的`Translog`影响数据恢复所消耗的时间。一旦达到了这个大小就会触发`flush`操作，生成一个新的`Translog`。

下面我们来看一看涉及到`Translog`的操作。

## 索引操作

对于数据操作最终的目标类都是`InternalEngine`。这个类关联了`Elasticsearch`与`lucene`，通过它来操作`lucene`的接口。

索引操作的方法为`InternalEngine.index()`，主要步骤如下：

1. 调用`indexIntoLucene`方法将数据写入到`lucene`中。实际的工作是调用`IndexWriter`将索引数据添加到`lucene`中

    需要注意的是：数据添加到lucene之后，并不能立即被搜索到，必须写入到segment中。这个步骤是由后面的`refresh`操作完成的。

2. 调用`index.origin().isFromTranslog()`方法判断数据是否来自日志

    如果数据来自日志的话就不再重复记录日志，否则记录日志。
    
    如果index操作成功，则在`translog`中添加一个`INDEX`日志。如果index操作失败，则在`translog`中添加一个`NO_OP`日志。

## 删除操作

删除操作的方法为`InternalEngine.delete()`，主要步骤如下：

1. 调用`deleteInLucene`方法将删除的数据写入到`lucene`。

    `Segment`文件是不可变更的。当一个`Document`删除的时候，实际上就是将旧的文档标记为删除。在合并`Segment`的过程中再将旧的`Document`删除掉。

2. 调用`index.origin().isFromTranslog()`方法判断数据是否来自日志

    如果数据来自日志的话就不再重复记录日志，否则记录日志。
    
    如果delete操作成功，则在`translog`中添加一个`DELETE`日志。

## refresh操作：

前面我们提到过，`refresh`就是将缓冲区中的数据写入到一个新的`Segment`。这样缓存中的数据就处于可被搜索状态。

默认情况下，`Elasticsearch`每隔1秒钟执行一次`refresh`操作。这个配置在`IndexSettings`中：

```java
public static final TimeValue DEFAULT_REFRESH_INTERVAL = new TimeValue(1, TimeUnit.SECONDS);
public static final Setting<TimeValue> INDEX_REFRESH_INTERVAL_SETTING =
        Setting.timeSetting("index.refresh_interval", DEFAULT_REFRESH_INTERVAL, new TimeValue(-1, TimeUnit.MILLISECONDS),
```

`refresh`操作在`InternalEngine.refresh()`中：

```java
ReferenceManager<IndexSearcher> referenceManager = getReferenceManager(scope);
// it is intentional that we never refresh both internal / external together
if (block) {
    referenceManager.maybeRefreshBlocking();
    refreshed = true;
} else {
    refreshed = referenceManager.maybeRefresh();
}
```

可以看到`refresh`操作调用`lucene`的`ReferenceManager`，真正的`refresh`操作由`lucene`执行。`maybeRefreshBlocking`和`maybeRefresh`的区别是：当有另外一个线程在执行时，`maybeRefreshBlocking`会阻塞直到另外的线程执行结束，`maybeRefresh`直接返回。


## flush操作

前面我们提到了，`flush`操作是为了防止`translog`文件变得太大而影响数据的恢复（当然也可以手动触发`flush`操作）。

`flush`操作在`InternalEngine.flush()`中：

```java
translog.rollGeneration();
commitIndexWriter(indexWriter, translog, null);
refresh("version_table_flush", SearcherScope.INTERNAL, true);
translog.trimUnreferencedReaders();

refreshLastCommittedSegmentInfos();
```

1. 调用`Translog.rollGeneration`方法。生成一个新的`Translog`。
2. 调用`commitIndexWriter`方法。向`lucene`写入`commitData`信息。将`lucene`中所有的修改都提交到索引中，然后同步（sync）索引数据到磁盘文件中。这样索引数据就彻底保存到磁盘中了。
3. 调用`refresh`方法将缓冲区中数据写入一个新的`Segment`
4. 调用`Translog.trimUnreferencedReaders`删除掉未被引用的日志reader。
5. 调用`refreshLastCommittedSegmentInfos`更新最新的`Segment`信息

## recover操作

`recover`操作的作用是从事务日志中恢复数据，在`InternalEngine.recoverFromTranslog()`方法中，其主要功能调用`recoverFromTranslogInternal`来实现。步骤如下：

1. 首先从`lastCommittedSegmentInfos`（最后提交的`SegmentInfo`，记录在`lucene`中）中获取事务日志的`generation`。
2. 通过这个`generation`生成事务日志的`snapshot`（快照），根据这个`snapshot`来恢复数据

数据恢复的方法是`IndexShard.runTranslogRecovery()`。它的功能是从`snapshot`中遍历`Operation`（表示记录在事务日志中的数据操作），根据`operation`重新执行一遍操作，达到恢复数据的目的。


# 总结

在大部分场景下，性能和可靠性都是需要进行权衡和取舍的，`Translog`就是这种权衡的产物，它能在性能轻微损失的同时保证数据的可靠性。在许多用于数据存取的软件设计中都能看到事务日志。学习事务日志是非常有意义的事。接下来的文章中我们进一步来分析`Translog`的原理。



























> https://www.toutiao.com/i6749080730841645580/
> https://leonlibraries.github.io/2017/04/27/ElasticSearch内部机制浅析三/
> https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-translog.html
> https://www.cnblogs.com/promise6522/archive/2012/05/27/2520028.html