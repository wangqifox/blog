---
title: Elasticsearch——stream读写的实践
date: 2019/12/30 18:40:00
---

之前我们分析了`Elasticsearch`中网络数据读写的过程。本文我们来学习并实践一下其中针对`stream`的读写。

<!-- more -->

前文进行了网络数据读写的分析，我们注意到三个重要的接口和类：

- `Streamable`：定义`stream`读写的接口。实现该接口的类可以向`StreamOutput`中写入数据，也可以从`StreamInput`中读取数据。
- `StreamInput`：读取`stream`的抽象类。定义了从`stream`读取各种数据类型(包括`byte`、`int`、`short`、`long`、`double`、`list`、`map`等等)的方法。
- `StreamOutput`：写入`stream`的抽象类。定义了向`stream`写入各种数据类型的方法。

针对`stream`的功能测试，我从`Elasticsearch`中分离出核心的代码，详见：[elasticsearch-stream](https://github.com/wangqifox/elasticsearch-stream)

# 使用

某个类需要添加`stream`的读写功能可以继承`Streamable`接口，实现`readFrom`和`writeTo`方法。如[User.java](https://github.com/wangqifox/elasticsearch-stream/blob/master/src/test/java/love/wangqi/User.java)所示，在`readFrom`和`writeTo`中自定义对`User`类的读写。

定义了`Streamable`接口的实现类，可以非常方便地将该实现类写入`StreamOutput`或者从`StreamInput`中读取该类的内容。如[RWTest.java](https://github.com/wangqifox/elasticsearch-stream/blob/master/src/test/java/love/wangqi/RWTest.java)所示。

# 性能测试

从直觉上看，`stream`因为直接读写二进制流，性能上会比较好。为了验证我们的直觉，我们设计测试用例来对比`stream`操作与`json`操作的性能。代码如[PerformanceTest.java](https://github.com/wangqifox/elasticsearch-stream/blob/master/src/test/java/love/wangqi/PerformanceTest.java)。

该测试是一个不太严格的测试，仅仅通过它对`stream`的性能有个直观的认识。

对于写操作（序列化），`stream`的性能大概是`json`的**2.4**倍，`stream`生成的数据大小大概是`json`的**41%**。

对于读操作（反序列化），`stream`的性能大概是`json`的**2.6**倍。

# 总结

`Elasticsearch`设计的这套数据流读写框架，无论从性能还是从需要传输的数据量角度来看都胜过`json`，缺点在于每个实现类都要自定义序列化、分序列化的过程。在性能要求高、数据交换频繁的内部服务中可以借鉴该设计。