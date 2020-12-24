---
layout: post
title: redis进阶之消息队列
date: 2020-12-27 21:00:00
tags: 
- redis
categories:
- redis
---

文章转载自 https://medium.com/@cheukfung/redis%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97-f4f75404647f  有删改

## FIFO队列

![image.png](https://i.loli.net/2020/12/01/4qLiXpeadZ3GNC9.png)

FIFO（先进先出）队列是队列的基本形式，也是最简单的消息队列。使用一个列表和LPUSH、RPOP两个命令即可实现一条FIFO队列。
生产者通过LPUSH将消息发送到Redis：

```
127.0.0.1:6379> LPUSH queue "message-1"
127.0.0.1:6379> LPUSH queue "message-2"
```

消费者使用RPOP从队列中依次获取消息：

```
127.0.0.1:6379> RPOP queue
"message-1"
127.0.0.1:6379> RPOP queue
"message-2"
```

当队列为空时，RPOP会直接返回空。如果希望在队列为空时能够阻塞连接等待消息，直到队列接收到新的消息或超时，可以使用BRPOP。(建议不要这么做？因为会阻塞redis，建议在程序中控制)

RPOP或BRPOP之后，消息就会从队列中删除。因此，当有多个消费者时，一条消息只能传递给其中一个消费者。我们不用担心在并发下同一条消息会传递给多个消费者，因为Redis本身是一个单线程的程序，相当于所有操作都有一把天然的排他锁。

## 可靠队列

FIFO队列中的消息一经发送出去，便从队列里删除。如果由于网络原因消费者没有收到消息，或者消费者在处理这条消息的过程中崩溃了，就再也无法还原出这条消息。也就是说，FIFO队列不能保证消息会传递成功。

究其原因，在于FIFO队列缺乏消息确认机制，即消费者向队列报告消息已收到或已处理的机制。可靠队列便是加入了这一机制的消息队列。

Redis在RPOPLPUSH命令的文档中提供了一种利用这一命令实现可靠队列的方式。RPOPLPUSH命令可以在从一个list中获取消息的同时把这条消息复制到另一个list里，并且这个过程是原子的。

![image.png](https://i.loli.net/2020/12/01/2oVuQdHD3PBhtLl.png)

利用RPOPLPUSH实现的可靠队列由两个列表组成，一个存储待处理的消息（pending list），另一个存储处理中的消息（processing list）。

生产者通过LPUSH将消息发送到待处理列表：

```
127.0.0.1:6379> LPUSH queue:pending "message"
```

消费者使用RPOPLPUSH从待处理列表获取消息，同时将它加入处理中列表：

```
127.0.0.1:6379> RPOPLPUSH queue:pending queue:processing
"message"
```

此时这条消息已经从待处理列表中删除，并且复制到了处理中列表：

```
127.0.0.1:6379> LRANGE queue:pending 0 -1
(empty list or set)
127.0.0.1:6379> LRANGE queue:processing 0 -1
1) "message"
```

消费者在收到消息或者处理完消息后，使用LREM命令从处理中列表删除这条消息，即完成了消息确认：

```
127.0.0.1:6379> LREM queue:processing 1 "message"
```

使用LREM而不是RPOP的原因在于，在并发时，不能保证处理中的消息能按加入列表的先后顺序被确认；而RPOP会按顺序删除消息。

没有被确认的消息会一直存储在处理中列表。如果一个消息在处理中列表呆的时间过长，那么可以认为这个消息的传递或处理失败了。我们可以设定一个超时时间，定时扫描处理中列表，将超时的消息重新放回待处理列表等待重新传递。

## 延迟队列

不同的微服务之间做异步通迅时通常会使用Kafka，它非常适用于对消费次序或时间没有强一致性需要的场景。如果消息需要在指定的时间才可以被消费，Kafka并没有原生支持此类消费场景，需要较复杂的实现。我们把支持此类场景的队列称为延迟队列。

延迟队列的主要特性是进入队列的消息会被推迟到指定的时间才出队被消费。而对于Kafka队列，消息入队后会排在队尾等待被消费，并不能指定出队时间。因此，延迟队列中的一条消息，除了消息本身外，还需要附加一个“何时出队”的信息。

Redis的Sorted Set可被用于实现简单的延迟队列。利用Redis的Lua支持我们也可以将基建设成一个功能全面的延迟队列服务。

下文将介绍如何使用Redis的Sorted Set实现简单的延迟队列。

![image.png](https://i.loli.net/2020/12/01/Ntz1dgQyEPuIKAk.png)

Sorted Set是一个有序的集合，元素的排序基于加入集合时指定的score。通过ZRANGEBYSCORE命令，我们可以取得score在指定区间内的元素。将集合中的元素做为消息，score视为延迟的时间(时间戳)，这便是一个延迟队列的模型。

生产者通过ZADD将消息发送到队列中：

```
127.0.0.1:6379> ZADD delay-queue 1520985600 "publish article"
```

消费者通过ZRANGEBYSCORE获取消息，以当前时间戳为score。如果时间未到，将得不到消息；当时间已到或已超时，都可以得到消息：

```
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985599
(empty list or set)
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985600 WITHSCORES
1) "publish article"
2) "1520985600"
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985601 WITHSCORES
1) "publish article"
2) "1520985600"
```

使用ZRANGEBYSCORE取得消息后，消息并没有从集合中删出。需要调用ZREM删除消息：

```
127.0.0.1:6379> ZREM delay-queue "publish article"
```

美中不足的是，消费者组合使用ZRANGEBYSCORE和ZREM的过程不是原子的，当有多个消费者时会存在竞争，可能使得一条消息被消费多次。此时需要使用Lua脚本保证消费操作的原子性：

```lua
local message = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'WITHSCORES', 'LIMIT', 0, 1);
if #message > 0 then
  redis.call('ZREM', KEYS[1], message[1]);
  return message;
else
  return {};
end
```

----

说一下redisson中的延时队列，除了sort set以外，还有有一个destination queue，redisson保证sort set中的延时操作，确定到延时时间了会自动放进destination queue中供消费者使用。

实际上我们常用的数据结构，队列，分布式锁，在Java的redis client redisson中都有相应的实现，这里不做具体讨论，详情请看redisson源码。

jvm中的数据模型还是要比redis操作快很多的，现实中需要合理结合业务使用redisson中的数据结构。