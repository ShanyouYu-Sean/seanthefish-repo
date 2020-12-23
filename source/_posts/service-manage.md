---
layout: post
title: 服务治理
date: 2020-12-5 10:00:00
tags: 
- 微服务
categories:
- 微服务
---

本文部分内容转自何小锋老师的极客时间专栏

## 服务发现

为什么zookeeper不适合做注册中心？ https://zhuanlan.zhihu.com/p/98591694

zookeeper为了一致性而牺牲可用性，导致网络分区时注册中心不可用，这时很难被接受的。

### 基于消息总线的最终一致性的注册中心

使用最终一致性，而非强一致性。

我们可以考虑采用消息总线机制。注册数据可以全量缓存在每个注册中心内存中，通过消息总线来同步数据。
当有一个注册中心节点，接收到服务节点注册时，会产生一个消息推送给消息总线，再通过消息总线通知给其它注册中心节点更新数据并进行服务下发，从而达到注册中心间数据最终一致性，具体流程如下图所示：

![image.png](https://i.loli.net/2020/12/16/AVxYzdsv5BXyEl9.png)

- 当有服务上线，注册中心节点收到注册请求，服务列表数据发生变化，会生成一个消息，推送给消息总线，每个消息都有整体递增的版本。
- 消息总线会主动推送消息到各个注册中心，同时注册中心也会定时拉取消息。对于获取到消息的在消息回放模块里面回放，只接受大于本地版本号的消息，小于本地版本号的消息直接丢弃，从而实现最终一致性。
- 消费者订阅可以从注册中心拿到指定接口的全部服务实例，并缓存到消费者的内存里面。
- 采用推拉模式，消费者可以及时地拿到服务实例增量变化情况，并和内存中的缓存数据进行合并。
 
为了性能，这里采用了两级缓存，注册中心和消费者的内存缓存，通过异步推拉模式来确保最终一致性。

## 健康检测

向注册中心发送心跳包，使用可用率作为健康检测的标准。

可用率的计算方式是某一个时间窗口内接口调用成功次数的百分比（成功次数 / 总调用次数）。

## 路由策略

节点上线灰度发布，比如我们可以先发布少量实例观察是否有异常，后续再根据观察的情况，选择发布更多实例还是回滚已经上线的实例。

注册中心只会把刚上线的服务 IP 地址推送到选择指定的调用方，而其他调用方是不能通过服务发现拿到这个 IP 地址的。

或者通过负载均衡策略，注册中心把负载均衡规则下发到服务调用方。

在流量切换的过程中，为了保证整个流程的完整性，我们必须保证某个主题对象的所有请求都使用同一种应用（老应用/新应用）来承接。给所有的服务提供方节点都打上标签，用来区分新老应用节点。在服务调用方发生请求的时候，我们可以很容易地拿到请求参数。通过参数来决定调用老应用or新应用。

## 负载均衡

为什么不采用添加负载均衡设备或者 TCP/IP 四层代理，域名绑定负载均衡设备的 IP 或者四层代理 IP 的方式？

- 搭建负载均衡设备或 TCP/IP 四层代理，需要额外成本；
- 请求流量都经过负载均衡设备，多经过一次网络传输，会额外浪费一些性能；
- 负载均衡添加节点和摘除节点，一般都要手动添加，当大批量扩容和下线时，会有大量的人工操作，“服务发现”在操作上是个问题；
- 我们在服务治理的时候，针对不同接口服务、服务的不同分组，我们的负载均衡策略是需要可配的，如果大家都经过这一个负载均衡设备，就不容易根据不同的场景来配置不同的负载均衡策略了。

RPC 的负载均衡完全由 RPC 框架自身实现，RPC 的服务调用者会与“注册中心”下发的所有服务节点建立长连接，在每次发起 RPC 调用时，服务调用者都会通过配置的负载均衡插件，自主选择一个服务节点，发起 RPC 调用请求。

![image.png](https://i.loli.net/2020/12/16/XTvVe2nkl15jtuC.png)

### 负载均衡算法

转自 https://zhibo.blog.csdn.net/article/details/90265243

#### 轮询法（Round Robin）

算法描述

假设有 N 台服务器 S = {S0, S1, S2, …, Sn}，算法可以描述为：
1、从 S0 开始依次调度 S1, S2, …, Sn；
2、若所有服务器都已被调度过，则从头开始调度；

轮训算法与服务器权重没有关系，每个服务器有序的会被轮训到。

代码实现

```java
public class ServerManager {
    public static Map<String, Integer> serverMap = new TreeMap<>();

    static {
        serverMap.put("192.168.1.1", 1);
        serverMap.put("192.168.1.2", 2);
        serverMap.put("192.168.1.3", 3);
        serverMap.put("192.168.1.4", 4);
    }
}


public class RoundRobin {
    private static AtomicInteger indexAtomic = new AtomicInteger(0);

    public static String getServer() {
        Set<String> serverSet = ServerManager.serverMap.keySet();
        ArrayList<String> serverList = new ArrayList<>(serverSet);

        if (indexAtomic.get() >= serverList.size()) {
            indexAtomic.set(0);
        }
        String server = serverList.get(indexAtomic.getAndIncrement());
        return server;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            String server = getServer();
            System.out.println(server);
        }
    }
}
```

#### 加权轮询法（Weight Round Robin）

算法描述

假设有 N 台服务器 S = {S0, S1, S2, …, Sn}，默认权重为 W = {W0, W1, W2, …, Wn}，服务器列表为 serverList，算法可以描述为：

1、初始化 serverList，将 W0 个 S0 加入至serverList，将 W1 个 S1 加入至serverList，依据此规则将所有的服务器加入至 serverList 中；
2、从 serverList 的从 S0 开始依序调度；
3、若所有服务器都已被调度过，则从头开始调度；

加权轮训算法通过在服务器列表中增加相应的权重个数的服务器来达到加权的效果，每个服务器会依序被轮训到。

代码实现

```java
public class ServerManager {
    public volatile static Map<String, Integer> serverMap = new TreeMap<>();

    static {
        serverMap.put("192.168.1.1", 1);
        serverMap.put("192.168.1.2", 2);
        serverMap.put("192.168.1.3", 3);
        serverMap.put("192.168.1.4", 4);
    }
}

public class WeightRoundRobin {
    private static AtomicInteger indexAtomic = new AtomicInteger(0);

    public static String getServer() {
        ArrayList<String> serverList = new ArrayList<>();
        Set<String> serverSet = ServerManager.serverMap.keySet();
        Iterator<String> iterator = serverSet.iterator();
        while(iterator.hasNext()){
            String server = iterator.next();
            Integer weight = ServerManager.serverMap.get(server);
            for (int i = 0; i < weight; i++) {
                serverList.add(server);
            }
        }

        if (indexAtomic.get() >= serverList.size()) {
            indexAtomic.set(0);
        }
        String server = serverList.get(indexAtomic.getAndIncrement());
        return server;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 20; i++) {
            String server = getServer();
            System.out.println(server);
        }
    }
}
```

#### 随机法（Random）

算法描述

假设有 N 台服务器 S = {S0, S1, S2, …, Sn}，算法可以描述为：
1、通过随机函数生成 0 到 N 之间的任意整理，将该数字作为索引，从 S 中获取对应的服务器；

随机算法与服务器权重没有关系，每个服务器会被随机的访问到，由概率论可以得知，当样本量足够大时，每台服务器被访问到的几率近似是相等的，随机算法的效果就越趋近于轮询算法。

```java
public class ServerManager {
    public volatile static Map<String, Integer> serverMap = new TreeMap<>();

    static {
        serverMap.put("192.168.1.1", 1);
        serverMap.put("192.168.1.2", 2);
        serverMap.put("192.168.1.3", 3);
        serverMap.put("192.168.1.4", 4);
    }
}

public class RandomBalance {

    public static String getServer() {
        Set<String> serverSet = ServerManager.serverMap.keySet();
        ArrayList<String> serverList = new ArrayList<>(serverSet);

        Random random = new Random();
        String server = serverList.get(random.nextInt(serverList.size()));
        return server;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            String server = getServer();
            System.out.println(server);
        }
    }
}
```

#### 加权随机法（Weight Random）

算法描述

假设有 N 台服务器 S = {S0, S1, S2, …, Sn}，默认权重为 W = {W0, W1, W2, …, Wn}，权重之和为 weightSum， 服务器列表为 serverList，算法可以描述为：
1、初始化 serverList，将 W0 个 S0 加入至serverList，将 W1 个 S1 加入至serverList，依据此规则将所有的服务器加入至 serverList 中；
2、通过随机函数生成 0 到 weightSum 之间的任意整理，将该数字作为索引，从 serverList 中获取对应的服务器；

与加权轮询算法类似，根据权重的不同向 serverList 添加相应的服务器，只不过服务器是通过随机算法获取的。

```java
public class ServerManager {
    public volatile static Map<String, Integer> serverMap = new TreeMap<>();

    static {
        serverMap.put("192.168.1.1", 1);
        serverMap.put("192.168.1.2", 2);
        serverMap.put("192.168.1.3", 3);
        serverMap.put("192.168.1.4", 4);
    }
}

public class WeightRandom {

    public static String getServer() {
        ArrayList<String> serverList = new ArrayList<>();
        Set<String> serverSet = ServerManager.serverMap.keySet();
        Iterator<String> iterator = serverSet.iterator();

        Integer weightSum = 0;
        while(iterator.hasNext()){
            String server = iterator.next();
            Integer weight = ServerManager.serverMap.get(server);
            weightSum += weight;
            for (int i = 0; i < weight; i++) {
                serverList.add(server);
            }
        }

        Random random = new Random();
        String server = serverList.get(random.nextInt(weightSum));
        return server;
    }

    public static void main(String[] args) {
        for (int i = 0; i < 20; i++) {
            String server = getServer();
            System.out.println(server);
        }
    }
}
```

#### 源地址哈希法（Hash）

算法描述

假设有 N 台服务器 S = {S0, S1, S2, …, Sn-1}，算法可以描述为：
1、通过指定的哈希函数，计算请求来源的地址的哈希值
2、对哈希值进行求余，底数为 N
3、将余数作为索引值，从 S 中获取对应的服务器；

```java
public class ServerManager {
    public volatile static Map<String, Integer> serverMap = new TreeMap<>();

    static {
        serverMap.put("192.168.1.1", 1);
        serverMap.put("192.168.1.2", 2);
        serverMap.put("192.168.1.3", 3);
        serverMap.put("192.168.1.4", 4);
    }
}

public class HashBalance {

    public static String getServer(String sourceIp) {
        if(sourceIp == null){
            return "";
        }
        Set<String> serverSet = ServerManager.serverMap.keySet();
        ArrayList<String> serverList = new ArrayList<>(serverSet);

        Integer index = sourceIp.hashCode() % serverList.size();
        return serverList.get(index);
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            String server = getServer("ip" + new Random().nextInt(1000));
            System.out.println(server);
        }
    }
}

```

### 自适应的负载均衡

采用一种打分的策略，服务调用者收集与之建立长连接的每个服务节点的指标数据，如服务节点的负载指标、CPU 核数、内存大小、请求处理的耗时指标（如请求平均耗时、TP99、TP999）、服务节点的状态指标（如正常、亚健康）。通过这些指标，计算出一个分数，比如总分 10 分，如果 CPU 负载达到 70%，就减它 3 分，当然了，减 3 分只是个类比，需要减多少分是需要一个计算策略的。我们可以配合随机权重的负载均衡策略去控制，通过最终的指标分数修改服务节点最终的权重。

![image.png](https://i.loli.net/2020/12/16/8MAc4Oj2q59WzUR.png)

## 异常重试

要确保被调用的服务的业务逻辑是幂等的，才能考虑根据事件情况开启 RPC 框架的异常重试功能。

连续的异常重试可能会出现一种不可靠的情况，那就是连续的异常重试并且每次处理的请求时间比较长，最终会导致请求处理的时间过长，超出用户设置的超时时间。

因此，如果发送请求发生异常并触发了异常重试，我们可以先判定下这个请求是否已经超时，如果已经超时了就直接返回超时异常，否则就先重置下这个请求的超时时间，之后再发起重试。

我们还需要在所有发起重试、负载均衡选择节点的时候，去掉重试之前出现过问题的那个节点，以保证重试的成功率。

我们还需要加个重试异常的白名单，用户可以将允许重试的异常加入到这个白名单中。

![image.png](https://i.loli.net/2020/12/16/8MAc4Oj2q59WzUR.png)

## 优雅关闭

先要简述下上线的大概流程：当服务提供方要上线的时候，一般是通过部署系统完成实例重启。在这个过程中，服务提供方的团队并不会事先告诉调用方他们需要操作哪些机器，从而让调用方去事先切走流量。而对调用方来说，它也无法预测到服务提供方要对哪些机器重启上线，因此负载均衡就有可能把要正在重启的机器选出来，这样就会导致把请求发送到正在重启中的机器里面，从而导致调用方不能拿到正确的响应结果。

在服务重启的时候，对于调用方来说，这时候可能会存在以下几种情况：

- 调用方发请求前，目标服务已经下线。对于调用方来说，跟目标节点的连接会断开，这时候调用方可以立马感知到，并且在其健康列表里面会把这个节点挪掉，自然也就不会被负载均衡选中。
- 调用方发请求的时候，目标服务正在关闭，但调用方并不知道它正在关闭，而且两者之间的连接也没断开，所以这个节点还会存在健康列表里面，因此该节点就有一定概率会被负载均衡选中。

我们可以这么处理：

当服务提供方正在关闭，如果这之后还收到了新的业务请求，服务提供方直接返回一个特定的异常给调用方（比如 ShutdownException）。这个异常就是告诉调用方“我已经收到这个请求了，但是我正在关闭，并没有处理这个请求”，然后调用方收到这个异常响应后，RPC 框架把这个节点从健康列表挪出，并把请求自动重试到其他节点，因为这个请求是没有被服务提供方处理过，所以可以安全地重试到其他节点，这样就可以实现对业务无损。

但如果只是靠等待被动调用，就会让这个关闭过程整体有点漫长。因为有的调用方那个时刻没有业务请求，就不能及时地通知调用方了，所以我们可以加上主动通知流程（通过注册中心），这样既可以保证实时性，也可以避免通知失败的情况。

在 Java 语言里面，对应的是 Runtime.addShutdownHook 方法，可以注册关闭的钩子。

![image.png](https://i.loli.net/2020/12/16/Yoj2y1NzsqpD4mS.png)

感觉有点像四次挥手。

## 优雅启动

在 Java 里面，在运行过程中，JVM 虚拟机会把高频的代码编译成机器码，被加载过的类也会被缓存到 JVM 缓存中，再次使用的时候不会触发临时加载，这样就使得“热点”代码的执行不用每次都通过解释，从而提升执行速度。但是这些“临时数据”，都在我们应用重启后就消失了。

### 启动预热

就是让刚启动的服务提供方应用不承担全部的流量，而是让它被调用的次数随着时间的移动慢慢增加，最终让流量缓和地增加到跟已经运行一段时间后的水平一样。

首先对于调用方来说，我们要知道服务提供方启动的时间，我们需要把这个时间作用在负载均衡上，我们要让负载均衡权重变成动态的，并且是随着时间的推移慢慢增加到服务提供方设定的固定值。

![image.png](https://i.loli.net/2020/12/16/8Ps1QgVb6CR4eXh.png)

### 延迟暴露

利用服务提供方把接口注册到注册中心的那段时间。我们可以在服务提供方应用启动后，接口注册到注册中心前，预留一个 Hook 过程，让用户可以实现可扩展的 Hook 逻辑。用户可以在 Hook 里面模拟调用逻辑，从而使 JVM 指令能够预热起来，并且用户也可以在 Hook 里面事先预加载一些资源，只有等所有的资源都加载完成后，最后才把接口注册到注册中心。

![image.png](https://i.loli.net/2020/12/16/liEI8KgzwYZjvnf.png)

如果是大批量重启，可以通过：

1、分时分批启动，就和灰度发布一样；
2、在请求低峰，在热点的应用肯定是有使用低峰的；
3、如果必须同时大批量重启，为了保证服务的可用性，可以在低峰时期，限流，为PLUS服务，非PLUS的就提醒暂时不可用之类的友好提示。

## 熔断限流

### 限流

限流就是服务端的自我保护

这里只说一下ratelimiter

#### RateLimiter

令牌桶算法

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。

![image.png](https://i.loli.net/2020/12/16/cvNp42eEkMSGXY1.png)

实现原理

RateLimiter有两种限流模式，一种为稳定模式(SmoothBursty:令牌生成速度恒定)，一种为渐进模式(SmoothWarmingUp:令牌生成速度缓慢提升直到维持在一个稳定值)。

集体实现参照 https://mp.weixin.qq.com/s/zGSvz6LyoenQw1TeWFLpuQ

### 熔断

熔断就是调用端的自我保护

熔断器的工作机制主要是关闭、打开和半打开这三个状态之间的切换。在正常情况下，熔断器是关闭的；当调用端调用下游服务出现异常时，熔断器会收集异常指标信息进行计算，当达到熔断条件时熔断器打开，这时调用端再发起请求是会直接被熔断器拦截，并快速地执行失败逻辑；当熔断器打开一段时间后，会转为半打开状态，这时熔断器允许调用端发送一个请求给服务端，如果这次请求能够正常地得到服务端的响应，则将状态置为关闭状态，否则设置为打开。

### 业务分组

我们可以尝试把应用提供方这个大池子划分出不同规格的小池子，再分配给不同的调用方，而不同小池子之间的隔离带，就是我们在 RPC 里面所说的分组，它可以实现流量隔离。

为了实现分组隔离逻辑，我们需要重新改造下服务发现的逻辑，调用方去获取服务节点的时候除了要带着接口名，还需要另外加一个分组参数，相应的服务提供方在注册的时候也要带上分组参数。

## 问题定位

### 借助合理封装的异常信息

![image.png](https://i.loli.net/2020/12/16/vEJ98wihZqNWtzR.png)

### 借助分布式链路跟踪

在整合分布式链路跟踪需要做的最核心的两件事就是“埋点”和“传递”。

所谓“埋点”就是说，分布式链路跟踪系统要想获得一次分布式调用的完整的链路信息，就必须对这次分布式调用进行数据采集，而采集这些数据的方法就是通过 RPC 框架对分布式链路跟踪进行埋点。

RPC 调用端在访问服务端时，在发送请求消息前会触发分布式跟踪埋点，在接收到服务端响应时，也会触发分布式跟踪埋点，并且在服务端也会有类似的埋点。这些埋点最终可以记录一个完整的 Span，而这个链路的源头会记录一个完整的 Trace，最终 Trace 信息会被上报给分布式链路跟踪系统。

那所谓“传递”就是指，上游调用端将 Trace 信息与父 Span 信息传递给下游服务的服务端，由下游触发埋点，对这些信息进行处理，在分布式链路跟踪系统中，每个子 Span 都存有父 Span 的相关信息以及 Trace 的相关信息。

![image.png](https://i.loli.net/2020/12/16/6CUyRvwzVbpJlNt.png)