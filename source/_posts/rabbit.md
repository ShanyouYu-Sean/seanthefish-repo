---
layout: post
title: rabbit-mq 读书笔记
date: 2019-09-17 10:00:00
tags: 
- mq
categories:
- mq
---

关于消息队列

https://tech.meituan.com/2016/07/01/mq-design.html

https://tech.meituan.com/2016/07/29/distributed-queue-based-programming.html

https://tech.meituan.com/2016/08/05/distributed-queue-based-programming-optimization.html

## 简介

![image.png](https://i.loli.net/2020/12/20/I93ia4RjxB2GM61.png)

RabbitMQ中的交换器有四种类型，不同的类型有着不同的路由策略，

- RoutingKey：路由键。生产者将消息发给交换器的时候，一般会指定一个RoutingKey，用来指定这个消息的路由规则，而这个RoutingKey需要与交换器类型和绑定键（BindingKey）联合使用才能最终生效。在交换器类型和绑定键（BindingKey）固定的情况下，生产者可以在发送消息给交换器时，通过指定RoutingKey来决定消息流向哪里。
- Binding：绑定。RabbitMQ中通过绑定将交换器与队列关联起来，在绑定的时候一般会指定一个绑定键（BindingKey），这样RabbitMQ就知道如何正确地将消息路由到队列了

![image.png](https://i.loli.net/2020/12/20/FQT5XSwkVfiZDAI.png)

![image.png](https://i.loli.net/2020/12/20/3W7dEQmoetF46jB.png)

### 交换器类型

RabbitMQ常用的交换器类型有fanout、direct、topic、headers这四种。AMQP协议里还提到另外两种类型：System和自定义，这里不予描述。对于这四种类型下面一一阐述。

- fanout：它会把所有发送到该交换器的消息路由到所有与该交换器绑定的队列中。
- direct：direct类型的交换器路由规则也很简单，它会把消息路由到那些BindingKey和RoutingKey完全匹配的队列中。
  ![image.png](https://i.loli.net/2020/12/20/1WfYsRjkwnTeoUP.png)
- topic：模糊匹配，
  - RoutingKey为一个点号“.”分隔的字符串（被点号“.”分隔开的每一段独立的字符串称为一个单词），如“com.rabbitmq.client”、“java.util.concurrent”、“com.hidden.client”
  - BindingKey和RoutingKey一样也是点号“.”分隔的字符串；
  - BindingKey中可以存在两种特殊字符串`*`和`#`，用于做模糊匹配，其中`*`用于匹配一个单词，`#`用于匹配多规格单词（可以是零个）。
  ![image.png](https://i.loli.net/2020/12/20/xyW23F9BwGAY5MS.png)
- headers：headers类型的交换器不依赖于路由键的匹配规则来路由消息，而是根据发送的消息内容中的headers属性进行匹配。在绑定队列和交换器时制定一组键值对，当发送消息到交换器时，RabbitMQ会获取到该消息的headers（也是一个键值对的形式），对比其中的键值对是否完全匹配队列和交换器绑定时指定的键值对，如果完全匹配则消息会路由到该队列，否则不会路由到该队列。headers类型的交换器性能会很差，而且也不实用，基本上不会看到它的存在。

### RabbitMQ运转流程

![image.png](https://i.loli.net/2020/12/20/Mdpw9Oau6tn73zh.png)

## amqp

![image.png](https://i.loli.net/2020/12/20/NfYhd1IeiJbtcTl.png)

push

![image.png](https://i.loli.net/2020/12/20/BHadxEqtXQsA2YU.png)

pull

![image.png](https://i.loli.net/2020/12/20/YdRawcSWgvNABO8.png)

不能将Basic.Get放在一个循环里来代替Basic.Consume，这样做会严重影响RabbitMQ的性能。如果要实现高吞吐量，消费者理应使用Basic.Consume方法。

## 客户端方法

### 发送

`void basicPublish（String exchange,String routingKey,BasicProperties props,byte[] body）throwsIOException;`

BasicProperties 可以设置deliverymode和priority

### 接收

push:
`String basicConsume（String queue,booleanautoAck,String consumerTag,boolean noLocal,boolean exclusive,Map<String,Object> arguments,Consumer callback）throws IOException;`

pull:
`GetResponse basicGet（String queue,boolean autoAck）throws IOException;`

### ack

当autoAck参数置为false，对于RabbitMQ服务端而言，队列中的消息分成了两个部分：一部分是等待投递给消费者的消息；一部分是已经投递给消费者，但是还没有收到消费者确认信号的消息。如果RabbitMQ一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则RabbitMQ会安排该消息重新进入队列，等待投递给下一个消费者，当然也有可能还是原来的那个消费者。

RabbitMQ不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否已经断开，这么设计的原因是RabbitMQ允许消费者消费一条消息的时间可以很久很久。

在消费者接收到消息后，如果想明确拒绝当前的消息而不是确认，那么应该怎么做呢？RabbitMQ在2.0.0版本开始引入了Basic.Reject这个命令，消费者客户端可以调用与其对应的channel.basicReject方法来告诉RabbitMQ拒绝这个消息。

`void basicReject（long deliveryTag,boolean requeue）throwsIOException;`

如果requeue参数设置为true，则RabbitMQ会重新将这条消息存入队列，以便可以发送给下一个订阅的消费者；如果requeue参数设置为false，则RabbitMQ立即会把消息从队列中移除，而不会把它发送给新的消费者。

Basic.Reject命令一次只能拒绝一条消息，如果想要批量拒绝消息，则可以使用Basic.Nack这个命令。消费者客户端可以调用channel.basicNack方法来实现，方法定义如下：

`void basicNack（long deliveryTag,boolean multiple,boolean requeue）throws IOException;`

将channel.basicReject或者channel.basicNack中的requeue设置为false，可以启用“死信队列”的功能。死信队列可以通过检测被拒绝或者未送达的消息来追踪问题。

对于requeue，AMQP中还有一个命令Basic.Recover具备可重入队列的特性。其对应的客户端方法为：

`Basic.RecoverOk basicRecover（）throws IOException;`

`Basic.RecoverOk basicRecover（boolean requeue）throws IOException;`
 
这个channel.basicRecover方法用来请求RabbitMQ重新发送还未被确认的消息。如果requeue参数设置为true，则未被确认的消息会被重新加入到队列中，这样对于同一条消息来说，可能会被分配给与之前不同的消费者。如果requeue参数设置为false，那么同一条消息会被分配给与之前相同的消费者。默认情况下，如果不设置requeue这个参数，相当于channel.basicRecover（true），即requeue默认为true。

## 进阶

### mandatory参数

当mandatory参数设为true时，交换器无法根据自身的类型和路由键找到一个符合条件的队列，那么RabbitMQ会调用Basic.Return命令将消息返回给生产者。当mandatory参数设置为false时，出现上述情形，则消息直接被丢弃。那么生产者如何获取到没有被正确路由到合适队列的消息呢？这时候可以通过调用channel.addReturnListener来添加ReturnListener监听器实现。

![image.png](https://i.loli.net/2020/12/20/udbErVMhFKgXBZc.png)

### immediate参数

当immediate参数设为true时，如果交换器在将消息路由到队列时发现队列上并不存在任何消费者，那么这条消息将不会存入队列中。当与路由键匹配的所有队列都没有消费者时，该消息会通过Basic.Return返回至生产者。

- mandatory参数告诉服务器至少将该消息路由到一个队列中，否则将消息返回给生产者。
- immediate参数告诉服务器，如果该消息关联的队列上有消费者，则立刻投递；如果所有匹配的队列上都没有消费者，则直接将消息返还给生产者，不用将消息存入队列而等待消费者了。

RabbitMQ3.0版本开始去掉了对immediate参数的支持，对此RabbitMQ官方解释是：immediate参数会影响镜像队列的性能，增加了代码复杂性，建议采用TTL和DLX的方法替代。

### 备份交换器

生产者在发送消息的时候如果不设置mandatory参数，那么消息在未被路由的情况下将会丢失；如果设置了mandatory参数，那么需要添加ReturnListener的编程逻辑，生产者的代码将变得复杂。

如果既不想复杂化生产者的编程逻辑，又不想消息丢失，那么可以使用备份交换器，这样可以将未被路由的消息存储在RabbitMQ中，再在需要的时候去处理这些消息。可以通过在声明交换器（调用channel.exchangeDeclare方法）的时候添加alternate exchange参数来实现，也可以通过策略（Policy）的方式实现。如果两者同时使用，则前者的优先级更高，会覆盖掉Policy的设置。

![image.png](https://i.loli.net/2020/12/20/pG67PhRoetauVny.png)

备份交换器其实和普通的交换器没有太大的区别，为了方便使用，建议设置为fanout类型，如若读者想设置为direct或者topic的类型也没有什么不妥。需要注意的是，消息被重新发送到备份交换器时的路由键和从生产者发出的路由键是一样的。

### ttl

目前有两种方法可以设置消息的TTL。

- 第一种方法是通过队列属性设置，队列中所有消息都有相同的过期时间。
- 第二种方法是对消息本身进行单独设置，每条消息的TTL可以不同。

如果两种方法一起使用，则消息的TTL以两者之间较小的那个数值为准。消息在队列中的生存时间一旦超过设置的TTL值时，就会变成“死信”（DeadMessage），消费者将无法再收到该消息

如果不设置TTL，则表示此消息不会过期；如果将TTL设置为0，则表示除非此时可以直接将消息投递到消费者，否则该消息会被立即丢弃，这个特性可以部分替代RabbitMQ3.0版本之前的immediate参数，之所以部分代替，是因为immediate参数在投递失败时会用Basic.Return将消息返回

对于第一种设置队列TTL属性的方法，一旦消息过期，就会从队列中抹去，而在第二种方法中，即使消息过期，也不会马上从队列中抹去，因为每条消息是否过期是在即将投递到消费者之前判定的。

为什么这两种方法处理的方式不一样？因为第一种方法里，队列中已过期的消息肯定在队列头部，RabbitMQ只要定期从队头开始扫描是否有过期的消息即可。而第二种方法里，每条消息的过期时间不同，如果要删除所有过期消息势必要扫描整个队列，所以不如等到此消息即将被消费时再判定是否过期，如果过期再进行删除即可。

### 死信队列 dlx

DLX，全称为DeadLetterExchange，可以称之为死信交换器，也有人称之为死信邮箱。当消息在一个队列中变成死信（deadmessage）之后，它能被重新被发送到另一个交换器中，这个交换器就是DLX，绑定DLX的队列就称之为死信队列。

消息变成死信一般是由于以下几种情况：

- 消息被拒绝（Basic.Reject/Basic.Nack），并且设置requeue参数为false；
- 消息过期；
- 队列达到最大长度。

DLX也是一个正常的交换器，和一般的交换器没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。当这个队列中存在死信时，RabbitMQ就会自动地将这个消息重新发布到设置的DLX上去，进而被路由到另一个队列，即死信队列。可以监听这个队列中的消息以进行相应的处理，这个特性与将消息的TTL设置为0配合使用可以弥补immediate参数的功能。

对于RabbitMQ来说，DLX是一个非常有用的特性。它可以处理异常情况下，消息不能够被消费者正确消费（消费者调用了Basic.Nack或者Basic.Reject）而被置入死信队列中的情况，后续分析程序可以通过消费这个死信队列中的内容来分析当时所遇到的异常情况，进而可以改善和优化系统。DLX配合TTL使用还可以实现延迟队列的功能。

### 延迟队列

在AMQP协议中，或者RabbitMQ本身没有直接支持延迟队列的功能，但是可以通过前面所介绍的DLX和TTL模拟出延迟队列的功能。

在图44中，不仅展示的是死信队列的用法，也是延迟队列的用法，对于queue.dlx这个死信队列来说，同样可以看作延迟队列。假设一个应用中需要将每条消息都设置为10秒的延迟，生产者通过exchange.normal这个交换器将发送的消息存储在queue.normal这个队列中。消费者订阅的并非是queue.normal这个队列，而是queue.dlx这个队列。当消息从queue.normal这个队列中过期之后被存入queue.dlx这个队列中，消费者就恰巧消费到了延迟10秒的这条消息。

![image.png](https://i.loli.net/2020/12/20/N4uDLq1plngKMrs.png)

在真实应用中，对于延迟队列可以根据延迟时间的长短分为多个等级，一般分为5秒、10秒、30秒、1分钟、5分钟、10分钟、30分钟、1小时这几个维度，当然也可以再细化一下。

参考图45，为了简化说明，这里只设置了5秒、10秒、30秒、1分钟这四个等级。根据应用需求的不同，生产者在发送消息的时候通过设置不同的路由键，以此将消息发送到与交换器绑定的不同的队列中。这里队列分别设置了过期时间为5秒、10秒、30秒、1分钟，同时也分别配置了DLX和相应的死信队列。当相应的消息过期时，就会转存到相应的死信队列（即延迟队列）中，这样消费者根据业务自身的情况，分别选择不同延迟等级的延迟队列进行消费。

![image.png](https://i.loli.net/2020/12/20/iTKAPf2SRNdpWD1.png)

### 优先级队列

优先级队列，顾名思义，具有高优先级的队列具有高的优先权，优先级高的消息具备优先被消费的特权。可以通过设置队列的xmaxpriority参数来实现。需要在发送时在消息中设置消息当前的优先级。

优先级高的消息可以被优先消费，这个也是有前提的：如果在消费者的消费速度大于生产者的速度且Broker中没有消息堆积的情况下，对发送的消息设置优先级也就没有什么实际意义。因为生产者刚发送完一条消息就被消费者消费了，那么就相当于Broker中至多只有一条消息，对于单条消息来说优先级是没有什么意义的。

### 持久化

RabbitMQ的持久化分为三个部分：交换器的持久化、队列的持久化和消息的持久化。

交换器的持久化是通过在声明交换器时将durable参数置为true实现的。如果交换器不设置持久化，那么在RabbitMQ服务重启之后，相关的交换器元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器中了。对一个长期使用的交换器来说，建议将其置为持久化的。

队列的持久化是通过在声明队列时将durable参数置为true实现的。如果队列不设置持久化，那么在RabbitMQ服务重启之后，相关队列的元数据会丢失，此时数据也会丢失。正所谓“皮之不存，毛将焉附”，队列都没有了，消息又能存在哪里呢？

队列的持久化能保证其本身的元数据不会因异常情况而丢失，但是并不能保证内部所存储的消息不会丢失。要确保消息不会丢失，需要将其设置为持久化。通过将消息的投递模式（BasicProperties中的deliveryMode属性）设置为2即可实现消息的持久化。

设置了队列和消息的持久化，当RabbitMQ服务重启之后，消息依旧存在。单单只设置队列持久化，重启之后消息会丢失；单单只设置消息的持久化，重启之后队列消失，继而消息也丢失。单单设置消息持久化而不设置队列的持久化显得毫无意义。

可以将所有的消息都设置为持久化，但是这样会严重影响RabbitMQ的性能。（随机）写入磁盘的速度比写入内存的速度慢得不只一点点。对于可靠性不是那么高的消息可以不采用持久化处理以提高整体的吞吐量。在选择是否要将消息持久化时，需要在可靠性和吐吞量之间做一个权衡。

### 生产者确认

#### 事务机制

RabbitMQ客户端中与事务机制相关的方法有三个：channel.txSelect、channel.txCommit和channel.txRollback。channel.txSelect用于将当前的信道设置成事务模式，channel.txCommit用于提交事务，channel.txRollback用于事务回滚。在通过channel.txSelect方法开启事务之后，我们便可以发布消息给RabbitMQ了，如果事务提交成功，则消息一定到达了RabbitMQ中，如果在事务提交执行之前由于RabbitMQ异常崩溃或者其他原因抛出异常，这个时候我们便可以将其捕获，进而通过执行channel.txRollback方法来实现事务回滚。注意这里的RabbitMQ中的事务机制与大多数数据库中的事务概念并不相同，需要注意区分。

commit：
![image.png](https://i.loli.net/2020/12/20/SmTkjuVh9a64Onx.png)

rollback：
![image.png](https://i.loli.net/2020/12/20/rOyCINsTMuKpnES.png)

####　发送方确认机制

生产者将信道设置成confirm（确认）模式，一旦信道进入confirm模式，所有在该信道上面发布的消息都会被指派一个唯一的ID（从1开始），一旦消息被投递到所有匹配的队列之后，RabbitMQ就会发送一个确认（Basic.Ack）给生产者（包含消息的唯一ID），这就使得生产者知晓消息已经正确到达了目的地了。如果消息和队列是持久化的，那么确认消息会在消息写入磁盘之后发出。RabbitMQ回传给生产者的确认消息中的deliveryTag包含了确认消息的序号，此外RabbitMQ也可以设置channel.basicAck方法中的multiple参数，表示到这个序号之前的所有消息都已经得到了处理，可以参考图410。注意辨别这里的确认和消费时候的确认之间的异同。

![image.png](https://i.loli.net/2020/12/20/oIc7D4mreFzvHGa.png)

事务机制在一条消息发送之后会使发送端阻塞，以等待RabbitMQ的回应，之后才能继续发送下一条消息。相比之下，发送方确认机制最大的好处在于它是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用程序便可以通过回调方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失，就会发送一条nack（Basic.Nack）命令，生产者应用程序同样可以在回调方法中处理该nack命令。

![image.png](https://i.loli.net/2020/12/20/PawT93UAfgn5md6.png)

事务机制和publisherconfirm机制确保的是消息能够正确地发送至RabbitMQ，这里的“发送至RabbitMQ”的含义是指消息被正确地发往至RabbitMQ的交换器，如果此交换器没有匹配的队列，那么消息也会丢失。所以在使用这两种机制的时候要确保所涉及的交换器能够有匹配的队列。更进一步地讲，发送方要配合mandatory参数或者备份交换器一起使用来提高消息传输的可靠性。

publisherconfirm的优势在于并不一定需要同步确认。这里我们改进了一下使用方式，总结有如下两种：

- 批量confirm方法：每发送一批消息后，调用channel.waitForConfirms方法，等待服务器的确认返回。
- 异步confirm方法：提供一个回调方法，服务端确认了一条或者多条消息后客户端会回调这个方法进行处理。

在批量confirm方法中，客户端程序需要定期或者定量（达到多少条），亦或者两者结合起来调用channel.waitForConfirms来等待RabbitMQ的确认返回。相比于前面示例中的普通confirm方法，批量极大地提升了confirm的效率，但是问题在于出现返回Basic.Nack或者超时情况时，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息数量，并且当消息经常丢失时，批量confirm的性能应该是不升反降的。

### 消息分发

当RabbitMQ队列拥有多个消费者时，队列收到的消息将以轮询（roundrobin）的分发方式发送给消费者。每条消息只会发送给订阅列表里的一个消费者。这种方式非常适合扩展，而且它是专门为并发程序设计的。如果现在负载加重，那么只需要创建更多的消费者来消费处理消息即可。

channel.basicQos方法允许限制信道上的消费者所能保持的最大未确认消息的数量。

举例说明，在订阅消费队列之前，消费端程序调用了channel.basicQos（5），之后订阅了某个队列进行消费。RabbitMQ会保存一个消费者的列表，每发送一条消息都会为对应的消费者计数，如果达到了所设定的上限，那么RabbitMQ就不会向这个消费者再发送任何消息。直到消费者确认了某条消息之后，RabbitMQ将相应的计数减1，之后消费者可以继续接收消息，直到再次到达计数上限。这种机制可以类比于TCP/IP中的“滑动窗口”。

Basic.Qos的使用对于拉模式的消费方式无效。

### 消息传输保障

消息可靠传输一般是业务系统接入消息中间件时首要考虑的问题，一般消息中间件的消息传输保障分为三个层级。

- At most once：最多一次。消息可能会丢失，但绝不会重复传输。
- At least once：最少一次。消息绝不会丢失，但可能会重复传输。
- Exactly once：恰好一次。每条消息肯定会被传输一次且仅传输一次。

RabbitMQ支持其中的“最多一次”和“最少一次”。其中“最少一次”投递实现需要考虑以下这个几个方面的内容：

- 消息生产者需要开启事务机制或者publisher confirm机制，以确保消息可以可靠地传输到RabbitMQ中。
- 消息生产者需要配合使用mandatory参数或者备份交换器来确保消息能够从交换器路由到队列中，进而能够保存下来而不会被丢弃。
- 消息和队列都需要进行持久化处理，以确保RabbitMQ服务器在遇到异常情况时不会造成消息丢失。
- 消费者在消费消息的同时需要将autoAck设置为false，然后通过手动确认的方式去确认已经正确消费的消息，以避免在消费端引起不必要的消息丢失。

那么RabbitMQ有没有去重的机制来保证“恰好一次”呢？答案是并没有，不仅是RabbitMQ，目前大多数主流的消息中间件都没有消息去重机制，也不保障“恰好一次”。去重处理一般是在业务客户端实现，比如引入GUID（GloballyUniqueIdentifier）的概念。针对GUID，如果从客户端的角度去重，那么需要引入集中式缓存，必然会增加依赖复杂度，另外缓存的大小也难以界定。建议在实际生产环境中，业务方根据自身的业务特性进行去重，比如业务消息本身具备幂等性，或者借助Redis等其他产品进行去重处理。

## 集群

### 普通模式

默认模式

RabbitMQ集群中的所有节点都会备份所有的元数据信息，包括以下内容。

- 队列元数据：队列的名称及属性；
- 交换器：交换器的名称及属性；
- 绑定关系元数据：交换器与队列或者交换器与交换器之间的绑定关系；
- vhost元数据：为vhost内的队列、交换器和绑定提供命名空间及安全属性。

但是不会备份消息（当然通过特殊的配置比如镜像队列可以解决这个问题，在第9.4节中会有详细的介绍）。基于存储空间和性能的考虑，在RabbitMQ集群中创建队列，集群只会在单个节点而不是在所有节点上创建队列的进程并包含完整的队列信息（元数据、状态、内容）。这样只有队列的宿主节点，即所有者节点知道队列的所有信息，所有其他非所有者节点只知道队列的元数据和指向该队列存在的那个节点的指针。因此当集群节点崩溃时，该节点的队列进程和关联的绑定都会消失。附加在那些队列上的消费者也会丢失其所订阅的信息，并且任何匹配该队列绑定信息的新消息也都会消失。

以两个节点（rabbit01、rabbit02）为例来进行说明。对于Queue来说，消息实体只存在于其中一个节点rabbit01（或者rabbit02），rabbit01和rabbit02两个节点仅有相同的元数据，即队列的结构。当消息进入rabbit01节点的Queue后，consumer从rabbit02节点消费时，RabbitMQ会临时在rabbit01、rabbit02间进行消息传输，把A中的消息实体取出并经过B发送给consumer。所以consumer应尽量连接每一个节点，从中取消息。即对于同一个逻辑队列，要在多个节点建立物理Queue。否则无论consumer连rabbit01或rabbit02，出口总在rabbit01，会产生瓶颈。当rabbit01节点故障后，rabbit02节点无法取到rabbit01节点中还未消费的消息实体。如果做了消息持久化，那么得等rabbit01节点恢复，然后才可被消费；如果没有持久化的话，就会产生消息丢失的现象。

![image.png](https://i.loli.net/2020/12/20/A8Q2Xuist5ole1v.png)

### 镜像模式

将需要消费的队列变为镜像队列，存在于多个节点，这样就可以实现RabbitMQ的HA高可用性。作用就是消息实体会主动在镜像节点之间实现同步，而不是像普通模式那样，在consumer消费数据时临时读取。缺点就是，集群内部的同步通讯会占用大量的网络带宽。

在通常的用法中，针对每一个配置镜像的队列（以下简称镜像队列）都包含一个主节点（master）和若干个从节点（slave），相应的结构可以参考图911。

![image.png](https://i.loli.net/2020/12/20/amy4G5VjYh7LOHk.png)

slave会准确地按照master执行命令的顺序进行动作，故slave与master上维护的状态应该是相同的。如果master由于某种原因失效，那么“资历最老”的slave会被提升为新的master。根据slave加入的时间排序，时间最长的slave即为“资历最老”。发送到镜像队列的所有消息会被同时发往master和所有的slave上，如果此时master挂掉了，消息还会在slave上，这样slave提升为master的时候消息也不会丢失。除发送消息（Basic.Publish）外的所有动作都只会向master发送，然后再由master将命令执行的结果广播给各个slave。

如果消费者与slave建立连接并进行订阅消费，其实质上都是从master上获取消息，只不过看似是从slave上消费而已。比如消费者与slave建立了TCP连接之后执行一个Basic.Get的操作，那么首先是由slave将Basic.Get请求发往master，再由master准备好数据返回给slave，最后由slave投递给消费者。读者可能会有疑问，大多的读写压力都落到了master上，那么这样是否负载会做不到有效的均衡？或者说是否可以像MySQL一样能够实现master写而slave读呢？注意这里的master和slave是针对队列而言的，而队列可以均匀地散落在集群的各个Broker节点中以达到负载均衡的目的，因为真正的负载还是针对实际的物理机器而言的，而不是内存中驻留的队列进程。

在图912中，集群中的每个Broker节点都包含1个队列的master和2个队列的slave，Q1的负载大多都集中在broker1上，Q2的负载大多都集中在broker2上，Q3的负载大多都集中在broker3上，只要确保队列的master节点均匀散落在集群中的各个Broker节点即可确保很大程度上的负载均衡（每个队列的流量会有不同，因此均匀散落各个队列的master也无法确保绝对的负载均衡）。至于为什么不像MySQL一样读写分离，RabbitMQ从编程逻辑上来说完全可以实现，但是这样得不到更好的收益，即读写分离并不能进一步优化负载，却会增加编码实现的复杂度，增加出错的可能，显得得不偿失。

![rabyOH.png](https://s3.ax1x.com/2020/12/20/rabyOH.png)

RabbitMQ的镜像队列同时支持publisherconfirm和事务两种机制。在事务机制中，只有当前事务在全部镜像中执行之后，客户端才会收到Tx.CommitOk的消息。同样的，在升为master的时候消息也不会丢失。除发送消息（Basic.Publish）外的所有动作都只会向master发送，然后再由master将命令执行的结果广播给各个slave。

新节点的加入过程可以参考图914，整个过程就像在链表中间插入一个节点。注意每当一个节点加入或者重新加入到这个镜像链路中时，之前队列保存的内容会被全部清空。

![xx](https://s3.ax1x.com/2020/12/20/raO9oR.png)

当slave挂掉之后，除了与slave相连的客户端连接全部断开，没有其他影响。当master挂掉之后，会有以下连锁反应：

- 与master连接的客户端连接全部断开。
- 选举最老的slave作为新的master，因为最老的slave与旧的master之间的同步状态应该是最好的。如果此时所有slave处于未同步状态，则未同步的消息会丢失。
- 新的master重新入队所有unack的消息，因为新的slave无法区分这些unack的消息是否已经到达客户端，或者是ack信息丢失在老的master链路上，再或者是丢失在老的master组播ack消息到所有slave的链路上，所以出于消息可靠性的考虑，重新入队所有unack的消息，不过此时客户端可能会有重复消息。
- 如果客户端连接着slave，并且Basic.Consume消费时指定了xcancelonhafailover参数，那么断开之时客户端会收到一个ConsumerCancellationNotification的通知，消费者客户端中会回调Consumer接口的handleCancel方法。如果未指定xcancelonhafailover参数，那么消费者将无法感知master宕机。

### Federation

Federation插件的设计目标是使RabbitMQ在不同的Broker节点之间进行消息传递而无须建立集群

Federation插件可以让多个交换器或者多个队列进行联邦。一个联邦交换器（federatedexchange）或者一个联邦队列（federatedqueue）接收上游（upstream）的消息，这里的上游是指位于其他Broker上的交换器或者队列。联邦交换器能够将原本发送给上游交换器（upstreamexchange）的消息路由到本地的某个队列中；联邦队列则允许一个本地消费者接收到来自上游队列（upstreamqueue）的消息。

#### 联邦交换器

假设图81中broker1部署在北京，broker2部署在上海，而broker3部署在广州，彼此之间相距甚远，网络延迟是一个不得不面对的问题。

![rajqJI.png](https://s3.ax1x.com/2020/12/20/rajqJI.png)

有一个在广州的业务ClientA需要连接broker3，并向其中的交换器exchangeA发送消息，此时的网络延迟很小，ClientA可以迅速将消息发送至exchangeA中，就算在开启了publisherconfirm机制或者事务机制的情况下，也可以迅速收到确认信息。此时又有一个在北京的业务ClientB需要向exchangeA发送消息，那么ClientB与broker3之间有很大的网络延迟，ClientB将发送消息至exchangeA会经历一定的延迟，尤其是在开启了publisher confirm机制或者事务机制的情况下，ClientB会等待很长的延迟时间来接收broker3的确认信息，进而必然造成这条发送线程的性能降低，甚至造成一定程度上的阻塞。

如图82所示，在broker3中为交换器exchangeA（broker3中的队列queueA通过“rkA”与exchangeA进行了绑定）与北京的broker1之间建立一条单向的Federationlink。此时Federation插件会在broker1上会建立一个同名的交换器exchangeA（这个名称可以配置，默认同名），同时建立一个内部的交换器“exchangeA→broker3B”，并通过路由键“rkA”将这两个交换器绑定起来。这个交换器“exchangeA→broker3B”名字中的“broker3”是集群名。

与此同时Federation插件还会在broker1上建立一个队列“federation:exchangeA→broker3B”，并与交换器“exchangeA→broker3B”进行绑定。Federation插件会在队列“federation:exchangeA→broker3B”与broker3中的交换器exchangeA之间建立一条AMQP连接来实时地消费队列“federation:exchangeA→broker3B”中的数据。这些操作都是内部的，对外部业务客户端来说这条Federationlink建立在broker1的exchangeA和broker3的exchangeA之间。

![rav0fI.png](https://s3.ax1x.com/2020/12/20/rav0fI.png)

回到前面的问题，部署在北京的业务ClientB可以连接broker1并向exchangeA发送消息，这样ClientB可以迅速发送完消息并收到确认信息，而之后消息会通过Federationlink转发到broker3的交换器exchangeA中。最终消息会存入与exchangeA绑定的队列queueA中，消费者最终可以消费队列queueA中的消息。经过Federationlink转发的消息会带有特殊的headers属性标记。

Federation不仅便利于消息生产方，同样也便利于消息消费方。假设某生产者将消息存入broker1中的某个队列queueB，在广州的业务ClientC想要消费queueB的消息，消息的流转及确认必然要忍受较大的网络延迟，内部编码逻辑也会因这一因素变得更加复杂，这样不利于ClientC的发展。不如将这个消息转发的过程以及内部复杂的编程逻辑交给Federation去完成，而业务方在编码时不必再考虑网络延迟的问题。Federation使得生产者和消费者可以异地部署而又让这两方感受不到过多的差异。

图82中broker1的队列“federation:exchangeA>broker3B”是一个相对普通的队列，可以直接通过客户端进行消费。假设此时还有一个客户端ClientD通过Basic.Consume来消费队列“federation:exchangeA→broker3B”的消息，那么发往broker1中exchangeA的消息会有一部分（一半）被ClientD消费掉，而另一半会发往broker3的exchangeA。所以如果业务应用有要求所有发往broker1中exchangeA的消息都要转发至broker3的exchangeA中，此时就要注意队列“federation:exchangeA→broker3B”不能有其他的消费者；而对于“异地均摊消费”这种特殊需求，队列“federation:exchangeA→broker3B”这种天生特性提供了支持。对于broker1的交换器exchangeA而言，它是一个普通的交换器，可以创建一个新的队列绑定它，对它的用法没有什么特殊之处。

如图84所示，一个federatedexchange同样可以成为另一个交换器的upstreamexchange。

![raxPBD.png](https://s3.ax1x.com/2020/12/20/raxPBD.png)

同样如图85所示，两方的交换器可以互为federatedexchange和upstreamexchange。

![raxe3t.png](https://s3.ax1x.com/2020/12/20/raxe3t.png)

#### 联邦队列

除了联邦交换器，RabbitMQ还可以支持联邦队列（federatedqueue）。联邦队列可以在多个Broker节点（或者集群）之间为单个队列提供均衡负载的功能。一个联邦队列可以连接一个或者多个上游队列（upstream queue），并从这些上游队列中获取消息以满足本地消费者消费消息的需求。

图89演示了位于两个Broker中的几个联邦队列（灰色）和非联邦队列（白色）。队列queue1和queue2原本在broker2中，由于某种需求将其配置为federatedqueue并将broker1作为upstream。Federation插件会在broker1上创建同名的队列queue1和queue2，与broker2中的队列queue1和queue2分别建立两条单向独立的Federationlink。当有消费者ClientA连接broker2并通过Basic.Consume消费队列queue1（或queue2）中的消息时，如果队列queue1（或queue2）中本身有若干消息堆积，那么ClientA直接消费这些消息，此时broker2中的queue1（或queue2）并不会拉取broker1中的queue1（或queue2）的消息；如果队列queue1（或queue2）中没有消息堆积或者消息被消费完了，那么它会通过Federationlink拉取在broker1中的上游队列queue1（或queue2）中的消息（如果有消息），然后存储到本地，之后再被消费者ClientA进行消费。

![raxfbD.png](https://s3.ax1x.com/2020/12/20/raxfbD.png)

消费者既可以消费broker2中的队列，又可以消费broker1中的队列，Federation的这种分布式队列的部署可以提升单个队列的容量。如果在broker1一端部署的消费者来不及消费队列queue1中的消息，那么broker2一端部署的消费者可以为其分担消费，也可以达到某种意义上的负载均衡。

和federatedexchange不同，一条消息可以在联邦队列间转发无限次。如图810中两个队列queue互为联邦队列。

![razqY9.png](https://s3.ax1x.com/2020/12/20/razqY9.png)

队列中的消息除了被消费，还会转向有多余消费能力的一方，如果这种“多余的消费能力”在broker1和broker2中来回切换，那么消费也会在broker1和broker2中的队列queue中来回转发。

federatedqueue并不具备传递性。考虑图811的情形，队列queue2作为federatedqueue与队列queue1进行联邦，而队列queue2又作为队列queue3的upstreamqueue，但是这样队列queue1与queue3之间并没有产生任何联邦的关系。如果队列queue1中有消息堆积，消费者连接broker3消费queue3中的消息，无论queue3处于何种状态，这些消费者都消费不到queue1中的消息，除非queue2有消费者。

![rdSpwD.png](https://s3.ax1x.com/2020/12/20/rdSpwD.png)

### Shovel

与Federation具备的数据转发功能类似，Shovel能够可靠、持续地从一个Broker中的队列（作为源端，即source）拉取数据并转发至另一个Broker中的交换器（作为目的端，即destination）。作为源端的队列和作为目的端的交换器可以同时位于同一个Broker上，也可以位于不同的Broker上。Shovel可以翻译为“铲子”，是一种比较形象的比喻，这个“铲子”可以将消息从一方“挖到”另一方。Shovel的行为就像优秀的客户端应用程序能够负责连接源和目的端、负责消息的读写及负责连接失败问题的处理。

- 松耦合。Shovel可以移动位于不同管理域中的Broker（或者集群）上的消息，这些Broker（或者集群）可以包含不同的用户和vhost，也可以使用不同的RabbitMQ和Erlang版本。
- 支持广域网。Shovel插件同样基于AMQP协议在Broker之间进行通信，被设计成可以容忍时断时续的连通情形，并且能够保证消息的可靠性。
- 高度定制。当Shovel成功连接后，可以对其进行配置以执行相关的AMQP命令。

图815展示的是Shovel的结构示意图。这里一共有两个Broker：broker1（IP地址：192.168.0.2）和broker2（IP地址：192.168.0.3）。broker1中有交换器exchange1和队列queue1，且这两者通过路由键“rk1”进行绑定；broker2中有交换器exchange2和队列queue2，且这两者通过路由键“rk2”进行绑定。在队列queue1和交换器exchange2之间配置一个Shovellink，当一条内容为“shoveltestpayload”的消息从客户端发送至交换器exchange1的时候，这条消息会经过图815中的数据流转最后存储在队列queue2中。如果在配置Shovellink时设置了add_forward_headers参数为true，则在消费到队列queue2中这条消息的时候会有特殊的headers属性标记

![rdS3pn.png](https://s3.ax1x.com/2020/12/20/rdS3pn.png)

#### 案例：消息堆积的治理

消息堆积是在使用消息中间件过程中遇到的最正常不过的事情。消息堆积是一把双刃剑，适量的堆积可以有削峰、缓存之用，但是如果堆积过于严重，那么就可能影响到其他队列的使用，导致整体服务质量的下降。对于一台普通的服务器来说，在一个队列中堆积1万至10万条消息，丝毫不会影响什么。但是如果这个队列中堆积超过1千万乃至一亿条消息时，可能会引起一些严重的问题，比如引起内存或者磁盘告警而造成所有Connection阻塞

当某个队列中的消息堆积严重时，比如超过某个设定的阈值，就可以通过Shovel将队列中的消息移交给另一个集群。

如图821所示，这里有如下几种情形。

- 情形1：当检测到当前运行集群cluster1中的队列queue1中有严重消息堆积，比如通过/api/queues/vhost/name接口获取到队列的消息个数（messages）超过2千万或者消息占用大小（messages_bytes）超过10GB时，就启用shovel1将队列queue1中的消息转发至备份集群cluster2中的队列queue2。
- 情形2：紧随情形1，当检测到队列queue1中的消息个数低于1百万或者消息占用大小低于1GB时就停止shovel1，然后让原本队列queue1中的消费者慢慢处理剩余的堆积。
- 情形3：当检测到队列queue1中的消息个数低于10万或者消息占用大小低于100MB时，就开启shovel2将队列queue2中暂存的消息返还给队列queue1。
- 情形4：紧随情形3，当检测到队列queue1中的消息个数超过1百万或者消息占用大小高于1GB时就将shovel2停掉。

![rdpmg1.png](https://s3.ax1x.com/2020/12/20/rdpmg1.png)



