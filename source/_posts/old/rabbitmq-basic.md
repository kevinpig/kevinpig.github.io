title: Rabbitmq基础指南
date: 2015-11-17 17:12:12
tags:
  - 云计算
  - 转载
---

声明:
本博客欢迎转发
内容系本人学习, 研究和总结,有些是摘抄自互联网,如有雷同,实属荣幸.

## RabbitMQ 的基本概念

RabbitMQ 是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）。

AMQP 是一个定义了在应用或者组织之间传送消息的协议的开放标准 （an open standard for passing business messages between applications or organizations），它最新的版本是 1.0。AMQP 目标在于解决在两个应用之间传送消息存在的下列问题：
 * 网络是不可靠的 =>消息需要保存后再转发并有出错处理机制
 * 与本地调用相比，网络速度慢 =>得异步调用
 * 应用之间是不同的（比如不同语言实现、不同操作系统等） =>得与应用无关
 * 应用会经常变化 =>同上

AMQP 使用异步的、应用对应用的、二进制数据通信来解决这些问题。

RabbitMQ 是 AMQP 的一种实现，它包括Server （服务器端）、Client （客户端） 和 Plugins （插件）。RabbitMQ 服务器是用 Erlang 语言编写的，其最新版本是刚刚（2015/02/11）发布的 3.4.4，而 OpenStack Juno 中使用的 Server 是 2014年3月发布的 3.2.4 版本。现在 RabbitMQ 支持的 AMQP 版本依然是0.9.1。

### RabbitMQ 的概念非常清晰、简洁

其基本概念参见下图：

![](http://images.cnitblog.com/blog/697113/201502/160955370968420.jpg)

RabbitMQ 官网 和其它网站上有很多文章来描述其基本概念。简单说明如下：

 * Message （消息）：RabbitMQ 转发的二进制对象，包括Headers（头）、Properties （属性）和 Data （数据），其中数据部分不是必要的。具体见 1.2 部分的描述。
 * Producer（生产者）： 消息的生产者，负责产生消息并把消息发到交换机 Exhange的应用。
 * Consumer （消费者）：使用队列 Queue 从 Exchange 中获取消息的应用。
 * Exchange （交换机）：负责接收生产者的消息并把它转到到合适的队列 Queue 。下面有 1.3 部分描述。
 * Queue （队列）：一个存储Exchange 发来的消息的缓冲，并将消息主动发送给Consumer，或者 Consumer 主动来获取消息。见 1.4 部分的描述。

 * Binding （绑定）：队列 和 交换机 之间的关系。Exchange 根据消息的属性和 Binding 的属性来转发消息。绑定的一个重要属性是 binding_key。
 * Connection （连接）和 Channel （通道）：生产者和消费者需要和 RabbitMQ 建立 TCP 连接。一些应用需要多个connection，为了节省TCP 连接，可以使用 Channel，它可以被认为是一种轻型的共享 TCP 连接的连接。连接需要用户认证，并且支持 TLS (SSL)。连接需要显式关闭。

![](http://images.cnitblog.com/blog/697113/201502/151659179171057.jpg)

 * Virtual Host （虚拟主机） ：RabbitMQ 用来进行资源隔离的机制。一个虚机主机会隔离用户、exchange、queue 等。默认的虚拟主机为 "/"。

### 关于消息 message

#### 消息结构

![](http://images.cnitblog.com/blog/697113/201502/151703080897171.jpg)

#### 消息的几个重要属性：

 * routing\_key：Direct 和 Topic 类型的 exchange 会根据本属性来转发消息。
 * delivery\_mode: 将其值设置为 2 将用于消息的持久化，持久化的消息会被保存到磁盘上来防止其丢失。下面章节 3 有描述。

 * reply\_to：一般用来表示RPC实现中客户端的回调队列的名字。下面章节 4 有描述。
 * correlation\_id：用于使用 RabbitMQ 来实现 RPC的情形。下面章节 4 有描述。
 * content\_type：表示消息data的编码格式名称。实际上RabbitMQ只负责原样传送消息因此不会使用该属性，该属性只被 Publisher 和 Consumer 使用。

#### 消息的确认/删除机制：

Consumer 处理消息可能会失败，那么 RabbitMQ 怎么知道什么时候来删除 queue 中的消息呢？它使用两种机制：

  * 当 RabbitMQ 主动将消息发给 Consumer 以后，它会删除消息
  * 当 Consumer 发回一个确认后，RabbitMQ 会删除消息。
第二种情况下，如果 RabbitMQ 没收到确认，它会把消息重新放进队列（re-queued）并添加标识 'redelivered' 表明该消息之前已经发送过 ，如果没有Consumer的话，消息将保持到有下一个 Consumer 为止。

Consumer 可以主动告诉 RabbitMQ 消息处理失败了（拒绝消息），并告知RabbitMQ 是删除消息还是重新放进队列。

### exchange 交换机

exchange 有几个重要的属性：

 * Name 名字：交换机名字。空字符串名字的exchange为默认的exchange。
 * Type 类型：Direct, Fanout, Topic, Headers。类型决定 exchange 的消息转发能力。下面 章节2 有描述。
 * durable：值为 True/False。值为 true 的 exchange 在 rabbitmq 重启后会被自动创建。OpenStack 使用的exchange的该值都为false。
 * auto_delete：值为 True/False。设置为 true 的话，当所有消费者的连接都关闭后，该 exchange 会被自动删除。OpenStack 使用的exchange的该值都为false。
 * exclusive：值为 True/False。设置为 true 的话，该 exchange 只允许被创建的connection使用，并且在该 connection 关闭后它会被自动删除。

RabbitMQ 默认会为每一种类型生成一个或者两个的默认的 exchange：

 * Fanout 类型：名字为 amq.fanout
 * Topic 类型: 名字为 amq.topic
 * Headers 类型：名字为 amq.match 和 amq.headers
 * Direct 类型：名字为空字符串的exchange 以及 amq.direct。其中名字为空的exchange比较特殊。在一个 Queue 被创建后，RabbitMQ 会自动建立它和该 exchange 之间的binding，并且设置其 binding\_key 为该queue 的名字。这样，该语句 channel.basic\_publish(exchange='', routing\_key='hello',             body=message) 会让该默认的 exchange 将该 message 转发到名字为 'hello' 的队列中。

### 队列 Queue

队列同样有类似于 exchange 的 name、durable、auto_delete 和 exclusive 等属性，并且含义相同。

Exchange 会将消息分发（copy）到符合要求的所有队列中。

Consumer 可以主动获取或者被动接受Queue里面的消息：

```
被动接收消息（订阅消息 "push API"）：使用 basic.consume(short reserved-1, queue-name queue, consumer-tag consumer-tag,no-local no-local, no-ack no-ack, bit exclusive, no-wait no-wait,table arguments) 方法。见 5.1 示例代码。

主动获取消息 （"pull API"）: 使用 basic.get(short reserved-1, queue-name queue, no-ack no-ack) 方法。
```

一个 Queue 允许有多个 Consumer，比如利用 RabbitMQ 来实现一个简单的 load balancer。这时候，消息会在这些 Consumer 之间根据 channel 的 prefetch level 做分发（请参见AQMP： QoS or message prefetching），如果该值一样的话，消息会被平均分发给这些Consumer。

### rabbitmqctl  Cli

RabbitMQ 提供Cli  rabbitmqctl [-n <node>] [-q] <command> [<command options>] 来进行管理和配置。常用到的命令有：

```
# stop/start_app
# add/delete/list_vhosts
# list_queues/exchanges/bindings/connections/channels
# trace_on/off
```

## 消息转发机制

Exchange 根据它自身的类型 type、消息的属性 routing\_key 或者 headers，以及 Binding 的属性 binding\_key 来转发消息。

 Exchange 的类型 Type |使用的消息属性 |使用的消息属性|转发模式|
 ----- |----- |---- |---- |
Fanout|- (忽略消息的转发属性）|-（忽略binding的转发属性）|Exchange 将消息转发到所有与它有 binding 关系的队列中。这种方法转发效率较高。<br /> OpenStack 大量使用这种类型的 exchange。|
Direct|routing\_key （任意的字符串，比如 "abc"）|binding\_key （任意的字符串，比如 "abc"）|Exchange 只将消息转到 binding 的 binding\_key 等于消息的 routing\_key 的队列中。|
Topic|Exchange 只将消息转到 binding 的 binding\_key 等于消息的 routing\_key 的队列中。|binding\_key （包含 "#" 和 "\*" 的以 “.” 分割的多单词字符串，比如 \*.efg.\*|Exchange 只将消息转到消息的 routing\_key 和 binding 的 binding\_key 匹配的队列中。匹配规则如下：<br /> 1. 两者以"."分割的单词数目相同<br /> 2. "*"可代表一个单词 <br /> 3. "#“可代表零个或多个单词 |
Headers|headers （消息头）|binding\_key|Exchange 只将消息转到消息的 headers 和 binding 的 binding\_key 匹配的队列中。匹配规则待研究。|


## 持久化

消息的持久化意味着在 RabbitMQ 被重启后，消息依然还在。要实现持久化，得实现几个相关组件的持久化：

 1. 交换机的持久化，需要将其 durable 属性设为 true。chan.exchange\_declare(exchange="sorting\_room", type="direct", durable=True, auto\_delete=False,)
 2. 队列的持久化，需要将其 durable 属性设置为 true。chan.queue\_declare(queue="po\_box", durable=True, exclusive=False, auto\_delete=False)
 3. 消息的持久化，需要将其 Delivery Mode 属性设置成2 。msg.properties["delivery\_mode"] = 2

## RPC

可以使用 RabbitMQ 来实现 RPC 机制，这里说说其实现原理：

![](http://images.cnitblog.com/blog/697113/201502/151536557764975.jpg)

过程：

 1. 客户端 Client 设置消息的 routing key 为 Service 的队列 op\_q；设置消息的 reply-to 属性为返回的 response 的目标队列 reponse\_q，设置其 correlation\_id 为以随机UUID，然后将消息发到 exchange。比如: <br \>
```
channel.basic_publish(exchange='', routing_key='op_q',
     properties=pika.BasicProperties(reply_to = reponse_q, correlation_id = self.corr_id),
     body=request)
```
 2. Exchange 将消息转发到 Service 的 op_q
 3. Service 收到该消息后进行处理，然后将response 发到 exchange，并设置消息的 routing\_key 为原消息的 reply\_to 属性，以及设置其 correlation\_id 为原消息的 correlation\_id 。<br \>
```
ch.basic_publish(exchange='', routing_key=props.reply_to,
    properties=pika.BasicProperties(correlation_id = props.correlation_id),
    body=str(response))
```
 4. Exchange 将消息转发到 reponse_q
 5. Client 逐一接受 response\_q 中的消息，检查消息的 correlation\_id 是否为等于它发出的消息的correlation\_id，是的话表明该消息为它需要的response。

[这里](https://www.rabbitmq.com/tutorials/tutorial-six-python.html)有详细的描述
