---
title: Netty——新连接接入
date: 2019/03/03 21:51:00
---

在前面的文章中，我们分别分析了Netty的启动过程以及`NioEventLoop`的工作流程。本文我们来分析Netty怎样处理新连接的接入。

<!-- more -->

经过前文[Netty——启动过程分析][1]的描述，我们知道Netty在服务端channel的启动过程中将selector注册到jdk的channel上，并将`NioServerSocketChannel`作为attachment。服务端channel绑定过程中注册一个`accept`事件，注册完成后Netty就可以接收新的连接了。

## OP_ACCEPT事件

当新连接接入时，select会接收到`accept`事件。`NioEventLoop`的运行过程中，判断是`OP_ACCEPT`事件，于是执行以下代码：

```java
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

服务端channel——`NioServerSocketChannel`持有的`unsafe`是`NioMessageUnsafe`。

`NioMessageUnsafe.read()`方法首先调用`NioServerSocketChannel.doReadMessages`方法:

### NioServerSocketChannel.doReadMessages

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```

`NioServerSocketChannel.doReadMessages`方法主要执行了以下两个步骤：

1. 调用`accept`方法从jdk的服务端channel——`ServerSocketChannel`中获取jdk的客户端channel——`SocketChannel`
2. 使用jdk的客户端channel新建netty的客户端channel——`NioSocketChannel`

`NioSocketChannel`的新建完成了以下几件事：

1. 创建`id`、`unsafe`、`pipeline`，其中客户端channel持有的unsafe是`NioSocketChannelUnsafe`
2. 保存jdk的客户端channel
3. 保存感兴趣的事件——`SelectionKey.OP_READ`
4. 设置阻塞模型为`false`，即非阻塞模式
5. 新建客户端channel的配置——`NioSocketChannelConfig`。设置`TcpNoDelay`为`true`。

### ServerBootstrapAcceptor.channelRead

回到`NioMessageUnsafe.read()`方法，生成客户端channel——`NioSocketChannel`之后，调用`DefaultChannelPipeline.fireChannelRead`方法传播读事件。

事件传播到`ServerBootstrapAcceptor.channelRead`方法。这是因为服务端Channel初始化过程中在channel的pipeline中添加了一个`ServerBootstrapAcceptor`。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

`channelRead`方法首先将`childHandler`中添加到pipeline中，并且设置`childOptions`。

然后调用`childGroup.register(child)`注册客户端channel，`childGroup`是我们在程序启动时新建的工作线程组（`NioEventLoopGroup`）。这个方法比较关键，将客户端channel分配到工作线程中去执行。

```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

`next()`调用`chooser`在线程组中挑选一个线程来执行`register`操作。执行`NioEventLoop`的父类`SingleThreadEventLoop`的`register`方法：

```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

最后调用客户端channel持有的unsafe——`NioSocketChannelUnsafe`的register方法，在eventLoop线程中执行`NioSocketChannelUnsafe.register0`方法。

#### NioSocketChannelUnsafe.register0

1. 调用`NioSocketChannel`父类`AbstractNioChannel`的`doRegister`方法。在jdk的channel上注册selector，将客户端channel——`NioSocketChannel`作为attachment

2. 调用pipeline的`fireChannelRegistered`方法传播注册事件

3. 调用pipeline的`fireChannelActive`方法传播active事件

    `HeadContext.channelActive`方法：
    
    ```java
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelActive();

        readIfIsAutoRead();
    }
    ```

    1. 继续传播active事件
    2. `readIfIsAutoRead`方法调用`TailContext.read`方法

        `TailContext.read`方法调用`NioSocketChannelUnsafe`的`beginRead`方法，`beginRead`方法调用`NioSocketChannel.doBeginRead()`方法。`NioSocketChannel.doBeginRead`方法注册感兴趣的读事件。
        

## OP_READ事件

`NioEventLoop`的运行过程中，判断是`OP_READ`事件，和`OP_ACCEPT`事件一样执行以下代码：

```java
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    unsafe.read();
}
```

注意，此时程序是在工作线程中执行的，当前的channel是客户端channel——`NioSocketChannel`，unsafe对应的是`NioSocketChannelUnsafe`。

其`read`方法执行步骤如下：

1. 分配`ByteBuf`
2. 调用`doReadBytes`方法从jdk的channel中读取数据保存在`ByteBuf`中
3. 调用pipeline的`fireChannelRead`方法传播read事件
4. 调用pipeline的`fireChannelReadComplete`方法传播读完成事件

## 总结

新连接接入的过程充分体现了netty的线程模型：

1. 在boss线程中由服务端channel——`NioServerSocketChannel`检测到accept事件
2. 使用jdk的客户端channel新建netty的客户端channel——`NioSocketChannel`
3. 将`NioSocketChannel`注册到工作线程中
4. 在`NioSocketChannel`中注册读事件

结果上面的步骤，客户端channel——`NioSocketChannel`就可以响应读事件。



















[1]: /articles/netty/Netty——启动过程分析.html

