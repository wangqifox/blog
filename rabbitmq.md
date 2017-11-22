---
title: rabbitMQ
---

# rabbitMQ

Connection是RabbitMQ的socket链接，它封装了socket协议相关部分逻辑。

ConnectionFactory为Connection的制造工厂

Channel是我们与RabbitMQ打交道的最重要的一个接口，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等。

## Queue

Queue是RabbitMQ的内部对象，用于存储消息。

RabbitMQ中的消息都只能存储在Queue中，生产者生产消息并最终投递到Queue中，消费者可以从Queue中获取消息并消费。

多个消费者可以订阅同一个Queue，这时Queue中的消息会被平均分摊给多个消费者进行处理，而不是每个消费者都收到所有的消息并处理

## Message acknowledgment

在实际应用中，可能会发生消费者收到Queue中的消息，但没有处理完成就宕机(或出现其他意外)的情况，这种情况下就可能会导致消息丢失。为了避免这种情况发生，我们可以要求消费者在消费完消息后发送一个回执给RabbitMQ，RabbitMQ收到消息回执(Message acknowledgment)后才将消息从Queue中移除；如果RabbitMQ没有收到回执并检测到消息者的RabbitMQ连接断开，则RabbitMQ会将该消息发送给其他消费者(如果存在多个消费者)进行处理。这里不存在timeout概念，一个消费者处理消息时间再长也不会导致该消息被发送给其他消费者，除非它的RabbitMQ连接断开。

## Message durability

如果我们希望即使在RabbitMQ服务重启的情况下，也不会丢失消息，我们可以将Queue与Message都设置为可持久化的(durable)，这样可以保证绝大部分情况下我们的RabbitMQ消息不会丢失。但依然解决不了小概率丢失事件的发生(比如RabbitMQ服务器已经接收到生产者的消息，但还没来得及持久化该消息时RabbitMQ服务器就断电了)，如果我们需要对这种小概率事件也要管理起来，那么我们要用到事务。

## Prefetch count

前面我们讲到如果有多个消息者同时订阅同一个Queue中的消息，Queue中的消息会被平摊给多个消费者。这时如果每个消息的处理时间不同，就有可能会导致某些消费者一直在忙，而另外一些消费者很快就处理完手头工作并一直空闲的情况。我们可以通过设置prefetchCount来限制Queue每次发送给每个消费者的消息树，比如我们设置prefetchCount=1，则Queue每次给每个消费者发送一条消息；消费者处理完这条消息后Queue会再给该消费者发送一条消息。

## Exchange

在上一节我们看到生产者将消息投递到Queue中，实际上这在RabbitMQ中这种事情永远都不会发生。实际的情况是，生产者将消息发送到Exchange，由Exchange将消息路由到一个或多个Queue中(或者丢弃)。

Exchange是按照什么逻辑将消息路由到Queue的？这个将在Binding一节介绍。
RabbitMQ中的Exchange有四种类型，不同的类型有着不同的路由策略，这将在Exchange Types一节介绍

## routing key

生产者在将消息发送给Exchange的时候，一般会指定一个routing key，来指定这个消息的路由规则，而这个routing key需要与Exchange Type及binding key联合使用才能最终生效。

在Exchange Type与binding key固定的情况下(在正常使用时一般这些内容都是固定配置好的)，我们的生产者就可以再发送消息给Exchange时，通过指定routing key来决定消息流向哪里。

RabbitMQ为routing key设定的长度限制255bytes

## Binding

RabbitMQ中通过Binding将Exchange与Queue关联起来，这样RabbitMQ就知道如何正确地将路由指定的Queue了。

## Binding key

在绑定(Binding)Exchange与Queue的同时，一般会指定一个binding key；消费者将消息发送给Exchange时，一般会指定一个routing key；当binding key与routing key相匹配时，消息将会被路由到对应的Queue中。这个将在Exchange Types章节会列举实际的例子加以说明。

在绑定多个Queue到同一个Exchange的时候，这些Binding允许使用相同的binding key。

binding key并不是在所有情况下都生效，它依赖于Exchange Type，比如fanout类型的Exchange就会无视binding key，而是将消息路由到所有绑定到该Exchange的Queue

## Exchange Types

RabbitMQ常用的Exchange Type有fanout、direct、topic、headers这四种

### fanout

fanout类型的Exchange路由规则非常简单，它会把所有消息发送到该Exchange的消息路由到所有与它绑定的Queue中

### direct

direct类型的Exchange路由规则也很简单，它会把消息路由到那些binding key与routing key完全匹配的Queue中

### topic

前面讲到direct类型的Exchange路由规则是完全匹配binding key与routing key，但这种严格的匹配方法在很多情况下不能满足实际业务需求。topic类型的Exchange在匹配规则上进行了扩展，它与direct类型的Exchange相似，也是将消息路由到binding key与routing key相匹配的Queue中，但这里的匹配规则有些不同，它约定：

- routing key为一个句点号"."分隔的字符串，如"stock.usd.nyse"、"nyse.vmw"、"quick.orange.rabbit"
- binding key与routing key一样也是句点号"."分隔的字符串
- binding key中可以存在两种特殊字符"*"与"#"，用于做模糊匹配，其中"*"用于匹配一个单词，"#"用于匹配多个单词(可以是零个)

### headers

headers类型的Exchange不依赖于routing key与binding key的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。

在绑定Queue与Exchange时指定一组键值对；当消息发送到Exchange时，RabbitMQ会取到该消息的headers(也是一个键值对的形式)，对比其中的键值对是否完全匹配Queue与Exchange绑定时指定的键值对；如果完全匹配则消息会路由到该Queue，否则不会路由到该Queue。

## RPC

MQ本身是基于异步的消息处理，前面的示例中所有的生产者将消息发送到RabbitMQ后不会知道消费者处理成功或者失败(甚至连有没有消费者来处理这条消息都不知道)。

但实际的应用场景中，我们很可能需要一些同步处理，需要同步等待服务端将我的消息处理完成后再进行下一步处理。这相当于RPC。在RabbitMQ中也支持RPC。

RabbitMQ中实现RPC的机制是：

- 客户端发送请求(消息)时，在消息的属性(MessageProperties)中设置两个值replyTo(一个Queue名称，用于告诉服务器处理完成后将通过我的消息发送到这个Queue中)和correlationId(此次请求的标识号，服务器处理完成后需要将此属性返还，客户端将根据这个id了解哪条请求被成功执行了或执行失败)
- 服务器端收到消息并处理
- 服务器端处理完消息后，将生产一条应答消息到replyTo指定的Queue，同时带上correlationId属性
- 客户端之前已订阅replyTo指定的Queue，从中收到服务器的应答消息后，根据其中的correlationId属性分析哪条请求被执行了，根据执行结果进行后续业务处理


