---
layout: post
title: redis基础
date: 2020-11-28 15:00:00
tags: 
- redis
categories:
- redis
---

## 为啥快

- 纯内存访问，Redis将所有数据放在内存中，内存的响应时长大 约为100纳秒，这是Redis达到每秒万级别访问的重要基础。
- 非阻塞I/O，Redis使用epoll作为I/O多路复用技术的实现，再加上 Redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不 在网络I/O上浪费过多的时间
- 单线程避免了线程切换和竞态产生的消耗。

但是单线程会有一个问题:对于每个命令的执行时间是有要求的。如果 某个命令执行过长，会造成其他命令的阻塞，对于Redis这种高性能的服务 来说是致命的，所以Redis是面向快速执行场景的数据库。

![image.png](https://i.loli.net/2020/11/29/KpSQHNqYJzBxwf4.png)

![image.png](https://i.loli.net/2020/11/29/YWqGg65uVjm72z8.png)

![image.png](https://i.loli.net/2020/11/29/mAcMrYKdpg2TnL7.png)

![image.png](https://i.loli.net/2020/11/29/NfIqOXMiJp1893B.png)


## 5种数据结构

![image.png](https://i.loli.net/2020/11/29/D7meIb8K3V2dH91.png)

![image.png](https://i.loli.net/2020/11/29/4WJHSETuw1ChPUn.png)

### 字符串

![image.png](https://i.loli.net/2020/11/29/xFDuY9B1EhHt7sW.png)

![image.png](https://i.loli.net/2020/11/29/MTJ57FwLBKPhelj.png)

使用场景

- 缓存
- 计数
- 限速
- 共享session

### 哈希

![image.png](https://i.loli.net/2020/11/29/AJ9enD2aUGViy35.png)

![image.png](https://i.loli.net/2020/11/29/ApEsFftxNSq5czv.png)

使用场景

- 关系型数据库非序列化场景下的缓存，不用序列化，可以直接改变某一个field

### 列表

- 列表中的元素是有序的
- 列表中的元素可以是重复的

![image.png](https://i.loli.net/2020/11/29/Zwb8qGvlrkaROUD.png)

![image.png](https://i.loli.net/2020/11/29/8vMaShxCgncJfQT.png)

使用场景

- 消息队列 lpush+brpop命令组合即可实现阻塞队列
- 文章列表
- ·lpush+lpop=Stack(栈) ·lpush+rpop=Queue(队列) ·lpsh+ltrim=Capped Collection(有限集合) ·lpush+brpop=Message Queue(消息队列)

### 集合

- 集合中不允许有重复元素
- 集合中的元素是无序的

![image.png](https://i.loli.net/2020/11/29/agSjHbEMQxAJ7pF.png)

![image.png](https://i.loli.net/2020/11/29/1H73vWUgbwErMt6.png)

使用场景

- 标签(tag)
- ·sadd=Tagging(标签) ·spop/srandmember=Random item(生成随机数，比如抽奖) ·sadd+sinter=Social Graph(社交需求)

### 有序集合

它保留了集合不能有重复成员的特性， 但不同的是，有序集合中的元素可以排序。但是它和列表使用索引下标作为 排序依据不同的是，它给每个元素设置一个分数(score)作为排序的依 据.有序集合中的元素不能重复，但是score可以重复。

![image.png](https://i.loli.net/2020/11/29/zxjVYJUpXC96aFl.png)

![image.png](https://i.loli.net/2020/11/29/47XaVExQt2mouiS.png)

![image.png](https://i.loli.net/2020/11/29/yaj1SrQDT6ghVGC.png)

使用场景

- 排行榜系统

## 键管理

### 键过期

- expire key seconds:键在seconds秒后过期。
- expireat key timestamp:键在秒级时间戳timestamp后过期。
- setex命令作为set+expire的组合，是原子执行
- persist命令可以将键的过期时间清除

### 迁移键

- move move命令用于在Redis内部进行数据迁移

![image.png](https://i.loli.net/2020/11/29/hH3tdnmVjL5Y4Tc.png)

- dump+restore可以实现在不同的Redis实例之间进行数据迁移的功能，整个迁移的过程分为两步:
  - 在源Redis上，dump命令会将键值序列化，格式采用的是RDB格式。 
  - 在目标Redis上，restore命令将上面序列化的值进行复原，其中ttl参数代表过期时间，如果ttl=0代表没有过期时间。

![image.png](https://i.loli.net/2020/11/29/YJNKocSlU6FEZtq.png)

- migrate migrate命令也是用于在Redis实例间进行数据迁移的，实际上migrate命 令就是将dump、restore、del三个命令进行组合，从而简化了操作流程。 migrate命令具有原子性

![image.png](https://i.loli.net/2020/11/29/NAr9Oiwk1GYs5zL.png)

![image.png](https://i.loli.net/2020/11/29/7DZrTPytzSs1dJN.png)

### 遍历键

- keys，执行keys命令很可能会造成Redis阻塞，所以一般建议不要在生 产环境下使用keys命令
- scan，和keys命令执行时会遍历所有键不同，scan采用渐进式遍历的方式来解决keys命令可能带来的阻塞问题，每次scan命令的时间复杂度是 O(1)，但是要真正实现keys的功能，需要执行多次scan。
- scan cursor [match pattern] [count number]，cursor是必需参数，实际上cursor是一个游标，第一次遍历从0开始，每次scan遍历完都会返回当前游标的值，直到游标值为0，表示遍历结束。
- 除了scan以外，Redis提供了面向哈希类型、集合类型、有序集合的扫描遍历命令，解决诸如hgetall、smembers、zrange可能产生的阻塞问题，对 应的命令分别是hscan、sscan、zscan，它们的用法和scan基本类似
- 如果在scan的过程中如果有键的变化(增加、删除、修改)，那么遍历效果可能会碰到如下问题:新增的键可能没有遍历到，遍历出了重 复的键等情况，也就是说scan并不能保证完整的遍历出来所有的键，这些是 我们在开发时需要考虑的。

## Pipeline

Redis客户端执行一条命令分为如下四个过程: 

- 1)发送命令
- 2)命令排队
- 3)命令执行
- 4)返回结果
  
其中1)+4)称为Round Trip Time(RTT，往返时间)。
Redis提供了批量操作命令(例如mget、mset等)，有效地节约RTT。但大部分命令是不支持批量操作的，例如要执行n次hgetall命令，并没有mhgetall命令存在，需要消耗n次RTT。Redis的客户端和服务端可能部署在不同的机器上。例如客户端在北京，Redis服务端在上海，两地直线距离约为1300公里，那么1次RTT时间=1300×2/(300000×2/3)=13毫秒(光在真空中 传输速度为每秒30万公里，这里假设光纤为光速的2/3)，那么客户端在1秒 内大约只能执行80次左右的命令，这个和Redis的高并发高吞吐特性背道而驰。

Pipeline(流水线)机制能改善上面这类问题，它能将一组Redis命令进 行组装，通过一次RTT传输给Redis，再将这组Redis命令的执行结果按顺序返回给客户端。

![image.png](https://i.loli.net/2020/11/29/tDGyNISYPg34oZ9.png)

原生批量命令与Pipeline对比 

可以使用Pipeline模拟出批量操作的效果，但是在使用时要注意它与原生批量命令的区别，具体包含以下几点: 

- 原生批量命令是原子的，Pipeline是非原子的。 
- 原生批量命令是一个命令对应多个key，Pipeline支持多个命令。
- 原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端的共同实现。

## 事务与Lua

### 事务

Redis提供了简单的事务功能，将一组需要一起执行的命令放到multi和 exec两个命令之间。multi命令代表事务开始，exec命令代表事务结束，它们之间的命令是原子顺序执行的。它不支持事务中的回滚特性，同时无法实现命令之间的逻辑关系计算

### Redis与Lua

在Redis中执行Lua脚本有两种方法:eval和evalsha。

![image.png](https://i.loli.net/2020/11/29/bW3udJXgyYacK2Z.png)

![image.png](https://i.loli.net/2020/11/29/tmbEZxIvLrp6ClB.png)

- Lua脚本在Redis中是原子执行的，执行过程中间不会插入其他命令。
- Lua脚本可以帮助开发和运维人员创造出自己定制的命令，并可以将这 些命令常驻在Redis内存中，实现复用的效果。
- Lua脚本可以将多条命令一次性打包，有效地减少网络开销。

## 发布订阅

![image.png](https://i.loli.net/2020/11/29/QVlWunEhxotZ6m9.png)

### 发布消息

publish channel message

### 订阅消息

subscribe channel [channel ...]

- 客户端在执行订阅命令之后进入了订阅状态，只能接收subscribe、psubscribe、unsubscribe、punsubscribe的四个命令。
- 新开启的订阅客户端，无法收到该频道之前的消息，因为Redis不会对 发布的消息进行持久化。

![image.png](https://i.loli.net/2020/11/29/h9q47CfjcmDlbvX.png)

## 持久化

### RDB

RDB持久化是把当前进程数据生成快照保存到硬盘的过程

触发机制

手动触发分别对应save和bgsave命令:

- save命令: 阻塞当前Redis服务器，直到RDB过程完成为止，对于内存比较大的实例会造成长时间阻塞，线上环境不建议使用。运行save命令对应 的Redis日志如下:
  ```
  DB saved on disk
  ```
- bgsave命令: Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生在fork阶段，一般时间很短。运行 bgsave命令对应的Redis日志如下:
  ```
  Background saving started by pid 3151
  DB saved on disk
  RDB: 0 MB of memory used by copy-on-write Background saving terminated with success
  ```
 used by copy-on-write Background saving 说明是cow

自动触发RDB

- 1)使用save相关配置，如“save m n”。表示m秒内数据集存在n次修改 时，自动触发bgsave。
- 2)如果从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件并发送给从节点。
- 3)执行debug reload命令重新加载Redis时，也会自动触发save操作。
- 4)默认情况下执行shutdown命令时，如果没有开启AOF持久化功能则 自动执行bgsave。

![image.png](https://i.loli.net/2020/11/29/gb2IsEAV7xilOB4.png)

RDB的优点:

- RDB是一个紧凑压缩的二进制文件，代表Redis在某个时间点上的数据快照。非常适用于备份，全量复制等场景。比如每6小时执行bgsave备份， 并把RDB文件拷贝到远程机器或者文件系统中(如hdfs)，用于灾难恢复。
- Redis加载RDB恢复数据远远快于AOF的方式。

RDB的缺点:

- RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高。
- RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，存在老版本Redis服务无法兼容新版RDB格式的问题。

针对RDB不适合实时持久化的问题，Redis提供了AOF持久化方式来解决。

### AOF

AOF(append only file)持久化:以独立日志的方式记录每次写命令， 重启时再重新执行AOF文件中的命令达到恢复数据的目的。AOF的主要作用 是解决了数据持久化的实时性，目前已经是Redis持久化的主流方式。

使用AOF

开启AOF功能需要设置配置:appendonly yes，默认不开启。AOF文件名 通过appendfilename配置设置，默认文件名是appendonly.aof。

AOF的工作流程操作:

![image.png](https://i.loli.net/2020/11/29/b8Fle6xLkzvJZw1.png)

- 1) 所有的写入命令会追加到aof_buf(缓冲区)中。
  AOF命令写入的内容直接是文本协议格式，在AOF缓冲区中追加。AOF为什么把命令追加到aof_buf中?Redis使用单线程响应命令，如果每次写AOF文件命令都直接追加到硬盘，那么性能完全取决于当前硬盘负载。先写入缓冲区aof_buf中，还有另一个好处，Redis可以提供多种缓冲区同步硬盘的策略，在性能和安全性方面做出平衡。
- 2) AOF缓冲区根据对应的策略向硬盘做同步操作。
  Redis提供了多种AOF缓冲区同步文件策略，由参数appendfsync控制
  ![image.png](https://i.loli.net/2020/11/29/KP81qa4wm69euyt.png)
  系统调用write和fsync说明:
  - write操作会触发延迟写(delayed write)机制。Linux在内核提供页缓冲区用来提高硬盘IO性能。write操作在写入系统缓冲区后直接返回。同步 硬盘操作依赖于系统调度机制，例如:缓冲区页空间写满或达到特定时间周 期。同步文件之前，如果此时系统故障宕机，缓冲区内数据将丢失。
  - fsync针对单个文件操作(比如AOF文件)，做强制硬盘同步，fsync将 阻塞直到写入硬盘完成后返回，保证了数据持久化。
  - 配置为always时，每次写入都要同步AOF文件，在一般的SATA硬盘 上，Redis只能支持大约几百TPS写入，显然跟Redis高性能特性背道而驰，不建议配置。
  - 配置为no，由于操作系统每次同步AOF文件的周期不可控，而且会加 大每次同步硬盘的数据量，虽然提升了性能，但数据安全性无法保证。
  - 配置为everysec，是建议的同步策略，也是默认配置，做到兼顾性能和 数据安全性。理论上只有在系统突然宕机的情况下丢失1秒的数据。
- 3) 随着AOF文件越来越大，需要定期对AOF文件进行重写，达到压缩的目的。
  AOF重写流程
  ![image.png](https://i.loli.net/2020/11/29/W2mNvLDqCXhkU7g.png)
- 4)当Redis服务器重启时，可以加载AOF文件进行数据恢复。
  Redis持久化文件加载流程
  ![image.png](https://i.loli.net/2020/11/29/CRP3ayxXeJHw1VQ.png)

### Redis持久化功能问题定位和优化

#### fork操作
当Redis做RDB或AOF重写时，一个必不可少的操作就是执行fork操作创 建子进程，对于大多数操作系统来说fork是个重量级错误。如何改善fork操作的耗时: 

- 1) 优先使用物理机或者高效支持fork操作的虚拟化技术，避免使用Xen。 
- 2) 控制Redis实例最大可用内存，fork耗时跟内存量成正比，线上建议每个Redis实例内存控制在10GB以内。 
- 3) 合理配置Linux内存分配策略，避免物理内存不足导致fork失败。
- 4) 降低fork操作的频率，如适度放宽AOF自动触发时机，避免不必要 的全量复制等。

#### 子进程开销监控和优化 
子进程负责AOF或者RDB文件的重写，它的运行过程主要涉及CPU、内
存、硬盘三部分的消耗。

CPU

- 子进程负责把进程内的数据分批写入文件，这个过程属于CPU密集操作，通常子进程对单核CPU利用率接近90%. Redis是CPU密集型服务，不要做绑定单核CPU操作。 由于子进程非常消耗CPU，会和父进程产生单核资源竞争。
- 不要和其他CPU密集型服务部署在一起，造成CPU过度竞争。 如果部署多个Redis实例，尽量保证同一时刻只有一个子进程执行重写工作。

内存

- 子进程通过fork操作产生，占用内存大小等同于父进程，理论上需要两倍的内存来完成持久化操作，但Linux有写时复制机制 (copy-on-write)。父子进程会共享相同的物理内存页，当父进程处理写请求时会把要修改的页创建副本，而子进程在fork操作过程中共享整个父进程内存快照。
- 同CPU优化一样，如果部署多个Redis实例，尽量保证同一时刻只有 一个子进程在工作。
- 避免在大量写入时做子进程重写操作，这样将导致父进程维护大量 页副本，造成内存消耗。

硬盘

- 子进程主要职责是把AOF或者RDB文件写入硬盘持久 化。势必造成硬盘写入压力。
- 不要和其他高硬盘负载的服务部署在一起。如:存储服务、消息队列服务等。
- AOF重写时会消耗大量硬盘IO，可以开启配置no-appendfsync-on-rewrite，默认关闭。表示在AOF重写期间不做fsync操作。
- 当开启AOF功能的Redis用于高流量写入场景时，如果使用普通机械 磁盘，写入吞吐一般在100MB/s左右，这时Redis实例的瓶颈主要在AOF同步 硬盘上。
- 对于单机配置多个Redis实例的情况，可以配置不同实例分盘存储 AOF文件，分摊硬盘写入压力。

#### AOF追加阻塞

当开启AOF持久化时，常用的同步硬盘的策略是everysec，用于平衡性 能和数据安全性。对于这种方式，Redis使用另一条线程每秒执行fsync同步 硬盘。当系统硬盘资源繁忙时，会造成Redis主线程阻塞。

![image.png](https://i.loli.net/2020/11/29/HPUG56g318LCcSl.png)

通过对AOF阻塞流程可以发现两个问题:

- 1) everysec配置最多可能丢失2秒数据，不是1秒。
- 2) 如果系统fsync缓慢，将会导致Redis主线程阻塞影响效率。

优化AOF追加阻塞问题主要是优化系统硬盘负载，优化方式见上一节。

### 多实例部署

Redis单线程架构导致无法充分利用CPU多核特性，通常的做法是在一 台机器上部署多个Redis实例。当多个实例开启AOF重写后，彼此之间会产 生对CPU和IO的竞争。本节主要介绍针对这种场景的分析和优化。

对于单机多Redis部署，如果 同一时刻运行多个子进程，对当前系统影响将非常明显，因此需要采用一种 措施，把子进程工作进行隔离。Redis在info Persistence中为我们提供了监控 子进程运行状况的度量指标

![image.png](https://i.loli.net/2020/11/29/P1I4yTKNLmHiO3C.png)

我们基于以上指标，可以通过外部程序轮询控制AOF重写操作的执行

![image.png](https://i.loli.net/2020/11/29/mqJcOpeldIkjatQ.png)

流程说明:

- 1) 外部程序定时轮询监控机器(machine)上所有Redis实例。
- 2) 对于开启AOF的实例，查看(aof_current_size- aof_base_size)/aof_base_size确认增长率。
- 3) 当增长率超过特定阈值(如100%)，执行bgrewriteaof命令手动触发 当前实例的AOF重写。
- 4) 运行期间循环检查aof_rewrite_in_progress和 aof_current_rewrite_time_sec指标，直到AOF重写结束。
- 5) 确认实例AOF重写完成后，再检查其他实例并重复2)~4)步操作。 从而保证机器内每个Redis实例AOF重写串行化执行。