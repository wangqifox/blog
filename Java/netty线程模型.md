---
title: Netty线程模型
date: 2018/08/28 10:16:00
---

Netty是一个高性能的网络异步框架，本文是对其线程模型的整理
<!-- more -->
# Reactor模型

无论是C++还是Java编写的网络框架，大多数都是基于Reactor模式进行设计和开发，Reactor模式基于事件驱动，特别适合处理海量的IO事件。

## 单线程模型

Reactor单线程模型，指的是所有的IO操作都在同一个NIO线程上面完成，NIO线程的职责如下：

1. 作为NIO服务端，接收客户端的TCP连接
2. 作为NIO客户端，向服务端发起TCP连接
3. 读取通信对端的请求或者应答消息
4. 向通信对端发送消息请求或者应答消息

由于Reactor模式使用的是异步非阻塞IO，所有的IO操作都不会导致阻塞，理论上一个线程可以独立处理所有IO相关的操作。从架构层面看，一个NIO线程确实可以完成其承担的职责。例如，通过Acceptor类接收客户端的TCP连接请求消息，链路建立成功之后，通过Dispatcher将对应的ByteBuffer派发到指定的Handler上进行消息解码。用户线程可以将消息编码通过NIO线程将消息发送给客户端。

对于一些小容量应用场景，可以使用单线程模型。但是对于高负载、大并发的应用场景却不合适，主要原因如下：

1. 一个NIO线程同时处理成百上千的链路，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取、发送
2. 当NIO线程负载过重之后，处理速度将变满，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，成为系统的性能瓶颈
3. 可靠性问题：一旦NIO线程意外跑飞，或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

## 多线程模型

Reactor多线程模型和单线程模型最大的区别就是有一组NIO线程处理IO操作

Reactor多线程模型的特点：

1. 有专门一个NIO线程——Acceptor线程用于监听服务端，接收客户端的TCP连接请求
2. 网络IO操作——读、写等由一个NIO线程池负责，线程池可以采用标准的JDK线程池实现，它包含一个任务队列和N个可用的线程，由这些NIO线程负责消息的读取、解码、编码、发送
3. 1个NIO线程可以同时处理N条链路，但是1个链路只对应1个NIO线程，防止发生并发操作问题

在绝大多数场景下，Reactor多线程模型都可以满足性能需求；但是，在极个别特殊场景中，一个NIO线程负责监听和处理所有的客户端连接可能会存在性能问题。例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。在这类场景下，单独一个Acceptor线程可能会存在性能不足问题，为了解决性能问题，产生了第三种Reactor线程模型——主从Reactor多线程模型。

## 主从多线程模型

主从Reactor线程模型的特点是：服务端用于接收客户端连接的不再是一个单独的NIO线程，而是一个独立的NIO线程池。Acceptor接收到客户端TCP连接请求处理完成后（可能包含接入认证等），将新创建的SocketChannel注册到IO线程池（sub reactor线程池）的某个IO线程上，由它负责SocketChannel的读写和编解码工作。Acceptor线程池仅仅只用于客户端的登录、握手、安全认证，一旦链路建立成功，就将链路注册到后端subReactor线程池的IO线程上，由IO线程负责后续的IO操作。

利用主从NIO线程模型，可以解决1个服务端监听线程无法有效处理所有客户端连接的性能不足问题。

它的工作流程总结如下：

1. 从主线程池中随机选择一个Reactor线程作为Acceptor线程，用于绑定监听端口，接收客户端连接
2. Acceptor线程接收客户端连接请求之后创建新的SocketChannel，将其注册到主线程池的其他Reactor线程上，由其负责接入认证、IP黑名单过滤、握手等操作。
3. 步骤2完成之后，业务层的链路正式建立，将SocketChannel从主线程池的Reactor线程的多路复用器上摘除，重新注册到Sub线程池的线程上，用于处理IO的读写操作

# Netty线程模型

## 服务端线程模型

一种比较流行的做法是服务端监听线程和IO线程分离，类似于Reactor的多线程模型

下面我们结合Netty的源码，对服务端创建线程工作流程进行介绍：

第一步，从用户线程发起创建服务端操作，代码如下：

```java
private static final int port = 9999;

public static void main(String[] args) throws InterruptedException {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    ServerBootstrap b = new ServerBootstrap();
    b.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 100)
            .handler(new LoggingHandler(LogLevel.DEBUG))
            .childHandler(new NettyServerFilter());

    Channel ch = b.bind(port).sync().channel();
    System.out.println("start server at port: " + port);
    ch.closeFuture().sync();
}
```

通常情况下，服务端的创建时在用户进程启动的时候进行，因此一般由Main函数或者启动类负责创建，服务端的创建由业务线程负责完成。在创建服务端的时候实例化了2个EventLoopGroup，1个EventLoopGroup实际就是一个EventLoop线程组，负责管理EventLoop的申请和释放。

EventLoopGroup管理的线程数可以通过构造函数设置，如果没有设置，默认去`-Dio.netty.eventLoopThreads`，如果该系统参数也没有指定，则为可用的CPU内核数 * 2

bossGroup线程组实际就是Acceptor线程组，负责处理客户端的TCP连接请求，如果系统只有一个服务端端口需要监听，则建议bossGroup线程组线程数设置为1

workerGroup是真正负责IO读写操作的线程组，通过ServerBootstrap的group方法进行设置，用于后续的Channel绑定。

## 客户端线程模型

相比于服务端，客户端的线程模型简单以下，它的工作原理如下：

第一步，由用户线程发起客户端连接：

```java
public static void main(String[] args) throws InterruptedException {
    EventLoopGroup group = new NioEventLoopGroup();
    Bootstrap b = new Bootstrap();
    b.group(group)
            .channel(NioSocketChannel.class)
            .option(ChannelOption.TCP_NODELAY, true)
            .handler(new NettyClientFilter());
    ch = b.connect(host, port).sync().channel();
    start();
}
```

客户端线程模型如下：

1. 由用户线程负责初始化客户端资源，发起连接操作
2. 如果连接成功，将SocketChannel注册到IO线程组的NioEventLoop线程中，监听读操作位
3. 如果没有立即连接成功，将SocketChannel注册到IO线程组的NioEventLoop线程中，监听连接操作位
4. 连接成功后，修改监听位为READ，但是不需要切换线程

# Reactor线程NioEventLoop

NioEventLoop是Netty的Reactor线程，它的职责如下：

1. 作为服务端Acceptor线程，负责处理客户端的请求接入
2. 作为客户端Connector线程，负责注册监听连接操作位，用于判断异步连接结果
3. 作为IO线程，监听网络读操作位，负责从SocketChannel中读取报文
4. 作为IO线程，负责向SocketChannel写入报文发送给对方，如果发生写半包，会自动注册监听写事件，用于后续继续发送半包数据，直到数据全部发送完成。
5. 作为定时任务线程，可以执行定时任务，例如链路空闲检测和发送心跳消息等
6. 作为线程执行器可以执行普通的任务线程

## 串行化设计避免线程竞争

我们知道当系统在运行过程中，如果频繁的进行线程上下文切换，会带来额外的性能损耗。多线程并发执行某个业务流程，业务开发者还需要时刻对线程安全保持警惕，哪些数据可能会被并发修改，如何保护？这不仅降低了开发效率，也会带来额外的性能损耗。

为了解决上述问题，Netty采用了串行化设计理念，从消息的读取、编码以及后续的Handler的执行，始终都由IO线程NioEventLoop负责，这就意味着整个流程不会进行线程上下文的切换，数据也不会面临被并发修改的风险，对于用户而言，甚至不需要了解Netty的线程细节。

一个NioEventLoop聚合了一个多路复用器Selector，因此可以处理成百上千的客户端连接，Netty的处理策略是每当有一个新的客户端接入，则从NioEventLoop线程组中顺序获取一个可用的NioEventLoop，当达到数组上限之后，重新返回到0，通过这种方式，可以基本保障各个NioEventLoop的负载均衡。一个客户端连接只注册到一个NioEventLoop上，这样就避免了多个IO线程去并发操作它。

Netty通过串行化设计理念降低了用户的开发难度，提升了处理性能。利用线程组实现了多个串行化线程水平并行执行，线程之间并没有交集，这样既可以充分利用多核提示并行处理能力，同时避免了线程上下文的切换和并发保护带来的额外性能损耗。

# Netty线程开发最佳实践

## 时间可控的简单业务直接在IO线程上处理

如果业务非常简单，执行时间非常短，不需要与外部网络交互、访问数据库、磁盘，不需要等待其它资源，则建议直接在业务ChannelHandler中执行，不需要再启动业务线程或线程池。避免线程上下文切换，也不存在线程并发问题。

## 复杂和时间不可控的业务建议投递到后端业务线程池统一处理

对于此类业务，不建议直接在业务ChannelHandler中启动线程或者线程池处理，建议将不同的业务统一封装成Task，统一投递到后端的业务线程池中进行处理。

过多的业务ChannelHandler会带来开发效率和可维护性问题，不要把Netty当做业务容器，对于大多数复杂的业务产品，仍然需要集成或者开发自己的业务容器，做好和Netty的架构分层。

## 业务线程避免直接操作ChannelHandler

对于ChannelHandler，IO线程和运维线程都可能会操作，因为业务通常是多线程模型，这样就会存在多线程操作ChannelHandler。为了尽量避免多线程并发问题，建议按照Netty自身的做法，通过将操作封装成独立的Task由NioEventLoop统一执行，而不是业务线程直接操作。









































> http://www.infoq.com/cn/articles/netty-threading-model#mainLogin

