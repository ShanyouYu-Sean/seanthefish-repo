---
layout: post
title: redis进阶之哨兵
date: 2019-04-10 18:00:00
tags: 
- redis
categories:
- redis
---

## 哨兵

Redis的主从复制模式下，一旦主节点由于故障不能提供服务，需要人 工将从节点晋升为主节点，同时还要通知应用方更新主节点地址，对于很多 应用场景这种故障处理的方式是无法接受的。可喜的是Redis从2.8开始正式 提供了Redis Sentinel(哨兵)架构来解决这个问题。

![DguZod.png](https://s3.ax1x.com/2020/11/29/DguZod.png)

### 主从复制的问题

Redis的主从复制模式可以将主节点的数据改变同步给从节点，这样从节点就可以起到两个作用: 

- 作为主节点的一个备份，一旦主节点出了故障不可达的情况，从节点可以作为后备“顶”上来，并且保证数据尽量不丢失(主从复制是最终一致性)。
- 从节点可以扩展主节点的读能力，一旦主节点不能支撑住大并发量的读操作，从节点可以在一定程度上帮助主节点分担读压力。

但是主从复制也带来了以下问题:

- 一旦主节点出现故障，需要手动将一个从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令其他从节点去复制新的主节点，整个过程都需要人工干预。
- 主节点的写能力受到单机的限制。
- 主节点的存储能力受到单机的限制。
  
其中第一个问题就是Redis的高可用问题，就是哨兵解决的。 第二、三个问题属于Redis的分布式问题，是集群解决的。

### 高可用

当主节点出现故障时，Redis Sentinel能自动完成故障发现和故障转移，并通知应用方，从而实现真正的高可用。

Redis Sentinel是一个分布式架构，其中包含若干个Sentinel节点和Redis 数据节点，每个Sentinel节点会对数据节点和其余Sentinel节点进行监控，当 它发现节点不可达时，会对节点做下线标识。如果被标识的是主节点，它还 会和其他Sentinel节点进行“协商”，当大多数Sentinel节点都认为主节点不可 达时，它们会选举出一个Sentinel节点来完成自动故障转移的工作，同时会 将这个变化实时通知给Redis应用方。整个过程完全是自动的，不需要人工 来介入，所以这套方案很有效地解决了Redis的高可用问题。

Redis Sentinel与Redis主从复制模式只是多了若干Sentinel 节点，所以Redis Sentinel并没有针对Redis节点做了特殊处理。

![DgKkpq.png](https://s3.ax1x.com/2020/11/29/DgKkpq.png)

从逻辑架构上看，Sentinel节点集合会定期对所有节点进行监控，特别是对主节点的故障实现自动转移。

下面以1个主节点、2个从节点、3个Sentinel节点组成的Redis Sentinel为例子进行说明，拓扑结构如图9-7所示。

![DgKMN9.png](https://s3.ax1x.com/2020/11/29/DgKMN9.png)

整个故障转移的处理逻辑有下面4个步骤:

- 1) 如图9-8所示，主节点出现故障，此时两个从节点与主节点失去连接，主从复制失败。
  ![DgK8c6.png](https://s3.ax1x.com/2020/11/29/DgK8c6.png)
- 2) 如图9-9所示，每个Sentinel节点通过定期监控发现主节点出现了故障。
  ![DgKw4A.png](https://s3.ax1x.com/2020/11/29/DgKw4A.png)
- 3) 如图9-10所示，多个Sentinel节点对主节点的故障达成一致，选举出 sentinel-3节点作为领导者负责故障转移。
  ![DgKsjf.png](https://s3.ax1x.com/2020/11/29/DgKsjf.png)
- 4) 如图9-11所示，Sentinel领导者节点执行了故障转移，整个过程和主从复制手动处理是完全一致的，只不过是自动化完成的。
  ![DgKWNj.png](https://s3.ax1x.com/2020/11/29/DgKWNj.png)
- 5) 故障转移后整个Redis Sentinel的拓扑结构图9-12所示。
  ![DgKovV.png](https://s3.ax1x.com/2020/11/29/DgKovV.png)

通过上面介绍的Redis Sentinel逻辑架构以及故障转移的处理，可以看出Redis Sentinel具有以下几个功能:

- 监控:Sentinel节点会定期检测Redis数据节点、其余Sentinel节点是否可达。
- 通知:Sentinel节点会将故障转移的结果通知给应用方。
- 主节点故障转移: 实现从节点晋升为主节点并维护后续正确的主从关系。
- 配置提供者: 在Redis Sentinel结构中，客户端在初始化的时候连接的是Sentinel节点集合，从中获取主节点信息。

同时看到，Redis Sentinel包含了若个Sentinel节点，这样做也带来了两个好处:

- 对于节点的故障判断是由多个Sentinel节点共同完成，这样可以有效地 防止误判。
- Sentinel节点集合是由若干个Sentinel节点组成的，这样即使个别Sentinel节点不可用，整个Sentinel节点集合依然是健壮的。

但是Sentinel节点本身就是独立的Redis节点，只不过它们有一些特殊， 它们不存储数据，只支持部分命令。

### sentinel配置

#### `sentinel monitor <master-name> <ip> <port> <quorum>`

Sentinel节点会定期监控主节点，所以从配置上必然也会有所体现，本 配置说明Sentinel节点要监控的是一个名字叫做`<master-name>`，ip地址和端口为`<ip>` `<port>`的主节点。`<quorum>`代表要判定主节点最终不可达所需要的 票数。但实际上Sentinel节点会对所有节点进行监控，但是在Sentinel节点的配置中没有看到有关从节点和其余Sentinel节点的配置，那是因为Sentinel节 点会从主节点中获取有关从节点以及其余Sentinel节点的相关信息.

例如某个Sentinel初始节点配置如下:

```java
port 26379
daemonize yes
logfile "26379.log"
dir /opt/soft/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2 
sentinel down-after-milliseconds mymaster 30000 
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

当所有节点启动后，配置文件中的内容发生了变化，体现在三个方面: 

- Sentinel节点自动发现了从节点、其余Sentinel节点。 
- 去掉了默认配置，例如parallel-syncs、failover-timeout参数。 
- 添加了配置纪元相关参数。

启动后变化为:

```java
port 26379
daemonize yes
logfile "26379.log"
dir "/opt/soft/redis/data"
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
#发现两个slave节点
sentinel known-slave mymaster 127.0.0.1 6380
sentinel known-slave mymaster 127.0.0.1 6381
#发现两个sentinel节点
sentinel known-sentinel mymaster 127.0.0.1 26380 282a70ff56c36ed56e8f7ee6ada741
24140d6f53
sentinel known-sentinel mymaster 127.0.0.1 26381 f714470d30a61a8e39ae031192f1fe
ae7eb5b2be
sentinel current-epoch 0
```

`<quorum>`参数用于故障发现和判定，例如将quorum配置为2，代表至少 有2个Sentinel节点认为主节点不可达，那么这个不可达的判定才是客观的。 对于`<quorum>`设置的越小，那么达到下线的条件越宽松，反之越严格。一 般建议将其设置为Sentinel节点的一半加1。

同时`<quorum>`还与Sentinel节点的领导者选举有关，至少要有`max(quorum，num(sentinels)/2+1)`个Sentinel节点参与选举，才能选出领导者Sentinel，从而完成故障转移。例如有5个Sentinel节点，quorum=4，那么 至少要有`max(4，5/2+1)=4`个在线Sentinel节点才可以进行领导者选举。

#### `sentinel down-after-milliseconds <master-name> <times>`

每个Sentinel节点都要通过定期发送ping命令来判断Redis数据节点和其 余Sentinel节点是否可达，如果超过了down-after-milliseconds配置的时间且没 有有效的回复，则判定节点不可达，`<times>`(单位为毫秒)就是超时时 间。这个配置是对节点失败判定的重要依据。

优化说明: down-after-milliseconds越大，代表Sentinel节点对于节点不可达的条件越宽松，反之越严格。条件宽松有可能带来的问题是节点确实不可达了，那么应用方需要等待故障转移的时间越长，也就意味着应用方故障时间可能越长。条件严格虽然可以及时发现故障完成故障转移，但是也存在一定的误判率。

#### `sentinel parallel-syncs <master-name> <nums>`

当Sentinel节点集合对主节点故障判定达成一致时，Sentinel领导者节点会做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，parallel-syncs就是用来限制在一次故障转移之后，每次向新的主节点发起复制操作的从节点个数。

如果这个参数配置的比较大，那么多个从节点会向新的主节点同时发起复制操作，尽管复制操作通常不会阻塞主节点， 但是同时向主节点发起复制，必然会对主节点所在的机器造成一定的网络和磁盘IO开销。

#### `sentinel failover-timeout <master-name> <times>`

failover-timeout通常被解释成故障转移超时时间，但实际上它作用于故障转移的各个阶段:

- a) 选出合适从节点。
- b) 晋升选出的从节点为主节点。
- c) 命令其余从节点复制新的主节点。
- d) 等待原主节点恢复后命令它去复制新的主节点。

failover-timeout的作用具体体现在四个方面:

- 1) 如果Redis Sentinel对一个主节点故障转移失败，那么下次再对该主节点做故障转移的起始时间是failover-timeout的2倍。
- 2) 在 b) 阶段时，如果Sentinel节点向 a) 阶段选出来的从节点执行slaveof no one一直失败(例如该从节点此时出现故障)，当此过程超过 failover-timeout时，则故障转移失败。
- 3) 在 b) 阶段如果执行成功，Sentinel节点还会执行info命令来确认 a) 阶段选出来的节点确实晋升为主节点，如果此过程执行时间超过failover- timeout时，则故障转移失败。
- 4) 如果 c) 阶段执行时间超过了failover-timeout(不包含复制时间)， 则故障转移失败。注意即使超过了这个时间，Sentinel节点也会最终配置从节点去同步最新的主节点。

#### `sentinel auth-pass <master-name> <password>`

如果Sentinel监控的主节点配置了密码，sentinel auth-pass配置通过添加主节点的密码，防止Sentinel节点对主节点无法监控。

#### `sentinel notification-script <master-name> <script-path>`

sentinel notification-script的作用是在故障转移期间，当一些警告级别的 Sentinel事件发生(指重要事件，例如-sdown:客观下线、-odown:主观下线)时，会触发对应路径的脚本，并向脚本发送相应的事件参数。

#### `sentinel client-reconfig-script <master-name> <script-path>`

sentinel client-reconfig-script的作用是在故障转移结束后，会触发对应路 径的脚本，并向脚本发送故障转移结果的相关参数。

### 如何监控多个主节点

Redis Sentinel可以同时监控多个主节点，具体拓扑图类似于图9-18。
配置方法也比较简单，只需要指定多个masterName来区分不同的主节点 即可，例如下面的配置监控monitor master-business-1(10.10.xx.1:6379)和 monitor master-business-2(10.10.xx.2:6379)两个主节点:

```java
sentinel monitor master-business-1 10.10.xx.1 6379 2 
sentinel down-after-milliseconds master-business-1 60000 
sentinel failover-timeout master-business-1 180000 
sentinel parallel-syncs master-business-1 1

sentinel monitor master-business-2 10.16.xx.2 6380 2 
sentinel down-after-milliseconds master-business-2 10000 
sentinel failover-timeout master-business-2 180000 
sentinel parallel-syncs master-business-2 1
```

![xx](https://i.bmp.ovh/imgs/2020/11/0de94e8cde09823f.png)

### 调整配置

![aa](https://i.bmp.ovh/imgs/2020/11/d1f68a21f86d8620.png)

- 1) sentinel set命令只对当前Sentinel节点有效。
- 2) sentinel set命令如果执行成功会立即刷新配置文件，这点和Redis普 通数据节点设置配置需要执行config rewrite刷新到配置文件不同。
- 3) 建议所有Sentinel节点的配置尽可能一致，这样在故障发现和转移时 比较容易达成一致。

### 部署技巧

- Sentinel节点不应该部署在一台物理“机器”上。

- 部署至少三个且奇数个的Sentinel节点。
 
- 只有一套Sentinel，还是每个主节点配置一套Sentinel?
  - 方案一: 一套Sentinel，很明显这种方案在一定程度上降低了维护成 本，因为只需要维护固定个数的Sentinel节点，集中对多个Redis数据节点进 行管理就可以了。但是这同时也是它的缺点，如果这套Sentinel节点集合出 现异常，可能会对多个Redis数据节点造成影响。还有如果监控的Redis数据 节点较多，会造成Sentinel节点产生过多的网络连接，也会有一定的影响。
  - 方案二: 多套Sentinel，显然这种方案的优点和缺点和上面是相反的， 每个Redis主节点都有自己的Sentinel节点集合，会造成资源浪费。但是优点 也很明显，每套Redis Sentinel都是彼此隔离的。

### 客户端

- 1) 遍历Sentinel节点集合获取一个可用的Sentinel节点，后面会介绍 Sentinel节点之间可以共享数据，所以从任意一个Sentinel节点获取主节点信息都是可以的，如图9-22所示。

![image.png](https://i.loli.net/2020/11/30/SLxJpyUd2Cz8hoK.png)

- 2) 通过sentinel get-master-addr-by-name master-name这个API来获取对应 主节点的相关信息，如图9-23所示。

![image.png](https://i.loli.net/2020/11/30/uD4XcxQ5mrhpW1i.png)

- 3) 验证当前获取的“主节点”是真正的主节点，这样做的目的是为了防 止故障转移期间主节点的变化，如图9-24所示。

![image.png](https://i.loli.net/2020/11/30/sKqfVJQznahX4OZ.png)

- 4) 保持和Sentinel节点集合的“联系”，时刻获取关于主节点的相关“信 息”，如图9-25所示。

![image.png](https://i.loli.net/2020/11/30/F91RMJyTsKQ7f6A.png)

从上面的模型可以看出，Redis Sentinel客户端只有在初始化和切换主节点时需要和Sentinel节点集合进行交互来获取主节点信息，所以在设计客户端时需要将Sentinel节点集合考虑成配置(相关节点信息和变化)发现服务。

### 实现原理

#### 三个定时监控任务

一套合理的监控机制是Sentinel节点判定节点不可达的重要保证，Redis
Sentinel通过三个定时监控任务完成对各个节点发现和监控: 

- 1) 每隔10秒，每个Sentinel节点会向主节点和从节点发送info命令获取最新的拓扑结构，如图9-26所示。这个定时任务的作用具体可以表现在三个方面:

  - 通过向主节点执行info命令，获取从节点的信息，这也是为什么 Sentinel节点不需要显式配置监控从节点。
  - 当有新的从节点加入时都可以立刻感知出来。 
  - 节点不可达或者故障转移后，可以通过info命令实时更新节点拓扑信息。

![D2eJUA.png](https://s3.ax1x.com/2020/11/30/D2eJUA.png)

- 2) 每隔2秒，每个Sentinel节点会向Redis数据节点 `__sentinel__:hello` 频道上发送该Sentinel节点对于主节点的判断以及当前Sentiel节点的信息 (如图9-27所示)，同时每个Sentinel节点也会订阅该频道，来了解其他 Sentinel节点以及它们对主节点的判断，所以这个定时任务可以完成以下两个工作:
  - 发现新的Sentinel节点: 通过订阅主节点的`__sentinel__:hello`了解其他 的Sentinel节点信息，如果是新加入的Sentinel节点，将该Sentinel节点信息保 存起来，并与该Sentinel节点创建连接。
  - Sentinel节点之间交换主节点的状态，作为后面客观下线以及领导者选举的依据。

![image.png](https://i.loli.net/2020/11/30/WIqNTC5uosA7DEb.png)

- 3) 每隔1秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点 发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达。如图9- 28所示。通过上面的定时任务，Sentinel节点对主节点、从节点、其余 Sentinel节点都建立起连接，实现了对每个节点的监控，这个定时任务是节点失败判定的重要依据。

![D2wiJx.png](https://s3.ax1x.com/2020/11/30/D2wiJx.png)

#### 主观下线和客观下线

##### 主观下线

上一小节介绍的第三个定时任务，每个Sentinel节点会每隔1秒对主节点、从节点、其他Sentinel节点发送ping命令做心跳检测，当这些节点超过 down-after-milliseconds没有进行有效回复，Sentinel节点就会对该节点做失败判定，这个行为叫做主观下线。从字面意思也可以很容易看出主观下线是当前Sentinel节点的一家之言，存在误判的可能。

##### 客观下线

当Sentinel主观下线的节点是主节点时，该Sentinel节点会通过sentinel is- master-down-by-addr命令向其他Sentinel节点询问对主节点的判断，当超过 `<quorum>`个数，Sentinel节点认为主节点确实有问题，这时该Sentinel节点会 做出客观下线的决定，这样客观下线的含义是比较明显了，也就是大部分 Sentinel节点都对主节点的下线做了同意的判定，那么这个判定就是客观的.

注意: 从节点、Sentinel节点在主观下线后，没有后续的故障转移操作。

#### 领导者Sentinel节点选举

假如Sentinel节点对于主节点已经做了客观下线，那么是不是就可以立 即进行故障转移了?当然不是，实际上故障转移的工作只需要一个Sentinel 节点来完成即可，所以Sentinel节点之间会做一个领导者选举的工作，选出 一个Sentinel节点作为领导者进行故障转移的工作。Redis使用了Raft算法实现领导者选举:

- 每个在线的Sentinel节点都有资格成为领导者，当它确认主节点客观下线时候，会向其他Sentinel节点发送sentinel is-master-down-by-addr命令， 要求将自己设置为领导者。
- 收到命令的Sentinel节点，如果没有同意过其他Sentinel节点的sentinel is-master-down-by-addr命令，将同意该请求，否则拒绝。
- 如果该Sentinel节点发现自己的票数已经大于等于max(quorum， num(sentinels)/2+1)，那么它将成为领导者。
- 如果此过程没有选举出领导者，将进入下一次选举。

#### 故障转移

领导者选举出的Sentinel节点负责故障转移，具体步骤如下:

- 1) 在从节点列表中选出一个节点作为新的主节点，选择方法如下:
  - a) 过滤: “不健康”(主观下线、断线)、5秒内没有回复过Sentinel节点ping响应、与主节点失联超过down-after-milliseconds*10秒。
  - b) 选择slave-priority(从节点优先级)最高的从节点列表，如果存在则 返回，不存在则继续。
  - c) 选择复制偏移量最大的从节点(复制的最完整)，如果存在则返回，不存在则继续。
  - d) 选择runid最小的从节点。
- 2) Sentinel领导者节点会对第一步选出来的从节点执行slaveof no one命令让其成为主节点。
- 3) Sentinel领导者节点会向剩余的从节点发送命令，让它们成为新主节点的从节点，复制规则和parallel-syncs参数有关。
- 4) Sentinel节点集合会将原来的主节点更新为从节点，并保持着对其关注，当其恢复后命令它去复制新的主节点。

### 高可用读写分离

从节点的作用
从节点一般可以起到两个作用:

- 当主节点出现故障时，作为主节点的后备“顶”上来实现故障转移，Redis Sentinel已经实现了该功能的自动化，实现了真正的高可用。
- 扩展主节点的读能力，尤其是在读多写少的场景非常适用。

![DRSLl9.png](https://s3.ax1x.com/2020/11/30/DRSLl9.png)

但上述模型中，从节点不是高可用的，如果slave-1节点出现故障，首先
客户端client-1将与其失联，其次Sentinel节点只会对该节点做主观下线，因为Redis Sentinel的故障转移是针对主节点的。所以很多时候，Redis Sentinel 中的从节点仅仅是作为主节点一个热备，不让它参与客户端的读操作，就是 为了保证整体高可用性，但实际上这种使用方法还是有一些浪费，尤其是在有很多从节点或者确实需要读写分离的场景，所以如何实现从节点的高可用 是非常有必要的。

Redis Sentinel读写分离设计思路

Redis Sentinel在对各个节点的监控中，如果有对应事件的发生，都会发
出相应的事件消息。在设计Redis Sentinel的从节点高可用时，只要能够实时掌握所有从节点的状态，把所有从节点看做一个资源池，无论是上线 还是下线从节点，客户端都能及时感知到(将其从资源池中添加或者删 除)，这样从节点的高可用目标就达到了。

![DRpY7V.png](https://s3.ax1x.com/2020/11/30/DRpY7V.png)

### sentinel重点回顾

- 1) Redis Sentinel是Redis的高可用实现方案:故障发现、故障自动转
移、配置中心、客户端通知。
- 2) Redis Sentinel从Redis2.8版本开始才正式生产可用，之前版本生产不 可用。
- 3) 尽可能在不同物理机上部署Redis Sentinel所有节点。
- 4) Redis Sentinel中的Sentinel节点个数应该为大于等于3且最好为奇数。
- 5) Redis Sentinel中的数据节点与普通数据节点没有区别。
- 6) 客户端初始化时连接的是Sentinel节点集合，不再是具体的Redis节 点，但Sentinel只是配置中心不是代理。
- 7) Redis Sentinel通过三个定时任务实现了Sentinel节点对于主节点、从 节点、其余Sentinel节点的监控。
- 8) Redis Sentinel在对节点做失败判定时分为主观下线和客观下线。 
- 9) 看懂Redis Sentinel故障转移日志对于Redis Sentnel以及问题排查非常
有帮助。
- 10) Redis Sentinel实现读写分离高可用可以依赖Sentinel节点的消息通
知，获取Redis数据节点的状态变化。

