---
layout: post
title: netty入门浅析
date: 2020-12-18 11:00:00
tags: 
- nio
categories:
- nio
---

转自 https://cloud.tencent.com/developer/article/1488120  https://hhbbz.github.io/2018/07/22/Netty%E5%85%A5%E9%97%A8%E6%B5%85%E6%9E%90(1)/ https://www.cnblogs.com/demingblog/p/9970772.html

## Reactor模式

### 服务端的线程模型

对于支持多连接的服务器，一般可以总结为2种fd和3种事件：

![r066ED.png](https://s3.ax1x.com/2020/12/21/r066ED.png)

2种fd

- listenfd：一般情况，只有一个。用来监听一个特定的端口(如80)。
- connfd：每个连接都有一个connfd。用来收发数据。

3种事件

- listenfd进行accept阻塞监听，创建一个connfd
- 用户态/内核态copy数据。每个connfd对应着2个应用缓冲区：readbuf和writebuf。
- 处理connfd发来的数据。业务逻辑处理，准备response到writebuf。

### Reactor模型

无论是C++还是Java编写的网络框架，大多数都是基于Reactor模型进行设计和开发，Reactor模型基于事件驱动，特别适合处理海量的I/O事件。

Reactor模型中定义的三种角色：

- Reactor：负责监听和分配事件，将I/O事件分派给对应的Handler。新的事件包含连接建立就绪、读就绪、写就绪等。
- Acceptor：处理客户端新连接，并分派请求到处理器链中。
- Handler：将自身与事件绑定，执行非阻塞读/写任务，完成channel的读入，完成处理业务逻辑后，负责将结果写出channel。可用资源池来管理。

Reactor处理请求的流程：

读取操作：

- 应用程序注册读就绪事件和相关联的事件处理器
- 事件分离器等待事件的发生
- 当发生读就绪事件的时候，事件分离器调用第一步注册的事件处理器

写入操作类似于读取操作，只不过第一步注册的是写就绪事件。

#### 单Reactor单线程模型

Reactor线程负责多路分离套接字，accept新连接，并分派请求到handler。Redis使用单Reactor单进程的模型。

![r0cTQ1.png](https://s3.ax1x.com/2020/12/21/r0cTQ1.png)

消息处理流程：

- Reactor对象通过select监控连接事件，收到事件后通过dispatch进行转发。
- 如果是连接建立的事件，则由acceptor接受连接，并创建handler处理后续事件。
- 如果不是建立连接事件，则Reactor会分发调用Handler来响应。
- handler会完成read->业务处理->send的完整业务流程。

单Reactor单线程模型只是在代码上进行了组件的区分，但是整体操作还是单线程，不能充分利用硬件资源。handler业务处理部分没有异步。

对于一些小容量应用场景，可以使用单Reactor单线程模型。但是对于高负载、大并发的应用场景却不合适，主要原因如下：

- 即便Reactor线程的CPU负荷达到100%，也无法满足海量消息的编码、解码、读取和发送。
- 当Reactor线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重Reactor线程的负载，最终会导致大量消息积压和处理超时，成为系统的性能瓶颈。
- 一旦Reactor线程意外中断或者进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。

为了解决这些问题，演进出单Reactor多线程模型。

### 单Reactor多线程模型

该模型在事件处理器（Handler）部分采用了多线程（线程池）。

![r0cxWd.png](https://s3.ax1x.com/2020/12/21/r0cxWd.png)

消息处理流程：

- Reactor对象通过Select监控客户端请求事件，收到事件后通过dispatch进行分发。
- 如果是建立连接请求事件，则由acceptor通过accept处理连接请求，然后创建一个Handler对象处理连接完成后续的各种事件。
- 如果不是建立连接事件，则Reactor会分发调用连接对应的Handler来响应。
- Handler只负责响应事件，不做具体业务处理，通过Read读取数据后，会分发给后面的Worker线程池进行业务处理。
- Worker线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给Handler进行处理。
- Handler收到响应结果后通过send将响应结果返回给Client。

相对于第一种模型来说，在处理业务逻辑，也就是获取到IO的读写事件之后，交由线程池来处理，handler收到响应后通过send将响应结果返回给客户端。这样可以降低Reactor的性能开销，从而更专注的做事件分发工作了，提升整个应用的吞吐。

但是这个模型存在的问题：

- 多线程数据共享和访问比较复杂。如果子线程完成业务处理后，把结果传递给主线程Reactor进行发送，就会涉及共享数据的互斥和保护机制。
- Reactor承担所有事件的监听和响应，只在主线程中运行，可能会存在性能问题。例如并发百万客户端连接，或者服务端需要对客户端握手进行安全认证，但是认证本身非常损耗性能。

为了解决性能问题，产生了第三种主从Reactor多线程模型。

### 主从Reactor多线程模型

比起第二种模型，它是将Reactor分成两部分：

- mainReactor负责监听server socket，用来处理网络IO连接建立操作，将建立的socketChannel指定注册给subReactor。
- subReactor主要做和建立起来的socket做数据交互和事件业务处理操作。通常，subReactor个数上可与CPU个数等同。

Nginx、Swoole、Memcached和Netty都是采用这种实现。

![r0gtp9.png](https://s3.ax1x.com/2020/12/21/r0gtp9.png)

消息处理流程：

- 从主线程池中随机选择一个Reactor线程作为acceptor线程，用于绑定监听端口，接收客户端连接
- acceptor线程接收客户端连接请求之后创建新的SocketChannel，将其注册到主线程池的其它Reactor线程上，由其负责接入认证、IP黑白名单过滤、握手等操作
- 步骤2完成之后，业务层的链路正式建立，将SocketChannel从主线程池的Reactor线程的多路复用器上摘除，重新注册到Sub线程池的线程上，并创建一个Handler用于处理各种连接事件
- 当有新的事件发生时，SubReactor会调用连接对应的Handler进行响应
- Handler通过Read读取数据后，会分发给后面的Worker线程池进行业务处理
- Worker线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给Handler进行处理
- Handler收到响应结果后通过Send将响应结果返回给Client

Reactor模型具有如下的优点：

- 响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；
- 编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；
- 可扩展性，可以方便地通过增加Reactor实例个数来充分利用CPU资源；
- 可复用性，Reactor模型本身与具体事件处理逻辑无关，具有很高的复用性。

## Netty Reactor模式

Netty是典型的Reactor模型结构。Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，Boss线程池和Work线程池，其中Boss线程池的线程负责处理请求的accept事件，当接收到accept事件的请求时，把对应的socket封装到一个NioSocketChannel中，并交给Work线程池，其中Work线程池负责请求的read和write事件。 流程图：

![r0grkD.png](https://s3.ax1x.com/2020/12/21/r0grkD.png)

## Netty核心组件

### Channel

这里的Channel与Java的Channel不是同一个，是netty自己定义的通道；Netty的Channel是对网络连接处理的抽象，负责与网络进行通讯，支持NIO和OIO两种方式；内部与网络socket连接，通过channel能够进行I/O操作，如读、写、连接和绑定。 通过Channel可以执行具体的I/O操作，如read, write, connect, 和bind。在Netty中，所有I/O操作都是异步的；Netty的服务器端处理客户端连接的Channel创建时可以设置父Channel。例如：ServerSocketChannel接收到请求创建SocketChannel，SocketChannel的父为ServerSocketChannel。

### ChannelHandler与ChannelPipeline

ChannelHandler是通道处理器，用来处理I/O事件或拦截I/O操作，ChannelPipeline字如其名，是一个双向流水线，内部维护了多个ChannelHandler，服务器端收到I/O事件后，每次顺着ChannelPipeline依次调用ChannelHandler的相关方法。 ChannelHandler是个接口，通常我们在Netty中需要使用下面的子类：

- ChannelInboundHandler 用来处理输入的I/O事件
- ChannelOutboundHandler 用来处理输出的I/O事件

另外，下面的adapter类提供了:

- ChannelInboundHandlerAdapter 用来处理输入的I/O事件
- ChannelOutboundHandlerAdapter 用来处理输出的I/O事件
- ChannelDuplexHandler 可以用来处理输入和输出的I/O事件

Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器/拦截器，每次收到请求会依次调用配置好的拦截器链。Netty服务器收到消息后，将消息在ChannelPipeline中流动和传递，途经的ChannelHandler会对消息进行处理，ChannelHandler分为两种inbound和outbound，服务器read过程中只会调用inbound的方法，write时只寻找链中的outbound的Handler。 ChannelPipeline内部维护了一个双向链表，Head和Tail分别代表表头和表尾，Head作为总入口和总出口，负责底层的网络读写操作；用户自己定义的ChannelHandler会被添加到链表中，这样就可以对I/O事件进行拦截和处理；这样的好处在于用户可以方便的通过新增和删除链表中的ChannelHandler来实现不同的业务逻辑，不需要对已有的ChannelHandler进行修改

![r0gzAU.png](https://s3.ax1x.com/2020/12/21/r0gzAU.png)

如图所示，在服务器初始化后，ServerSocketChannel的会创建一个Pipeline，内部维护了ChannelHanlder的双向链表，读取数据时，会依次调用ChannelInboundHandler子类的channelRead()方法，例如：读取到客户端数据后，依次调用解码-业务逻辑-直到Tail。而写入数据时，会从用户自定义的ChannelHandler出发查找ChannelOutboundHandler的子类，调用channelWrite()，最终由Head的write()向socket写入数据。例如：写入数据会通过业务逻辑的组装–编码–写入socket（Head的write）

### EventLoop与EventLoopGroup

EventLoop是事件循环，EventLoopGroup是运行在线程池中的事件循环组，Netty使用了Reactor模型，服务器的连接和读写放在线程池之上的事件循环中执行，这是Netty获得高性能的原因之一。事件循环内部会打开selector，并将Channel注册到事件循环中，事件循环不断的进行select()查找准备就绪的描述符；此外，某些系统任务也会被提交到事件循环组中运行。

### ServerBootstrap

ServerBootstrap是辅助启动类，用于服务端的启动，内部维护了很多用于启动和建立连接的属性。包括：

- EventLoopGroup group 线程池组
- channel是通道
- channelFactory 通道工厂，用于创建channel
- localAddress 本地地址
- options 通道的选项，主要是TCP连接的属性
- attrs 用来设置channel的属性，
- handler 通道处理器

## netty简单案例

用Netty搭建一个HttpServer，从实际开发中了解netty框架的一些特性和概念。

### 本文HttpServer的实现目标

本文只是为了演示如何使用Netty来实现一个HTTP服务器，如果要实现一个完整的，那将是十分复杂的。所以，我们只实现最基本的，请求-响应。具体来说是这样的：

```
1. 启动服务
2. 客户端访问服务器，如：http://localhost:8081/index
3. 服务器返回 ： 你请求的uri为：/index
```

### 创建server

netty 的api设计非常好，具有通用性，几乎就是一个固定模式的感觉。server端的启动和客户端的启动代码十分相似。启动server的时候指定初始化器，在初始化器中，我们可以放一个一个的handler，而具体业务逻辑处理就是放在这一个个的handler中的。写好的server端代码如下：

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;
 
import java.net.InetSocketAddress;
 
/**
 * netty server
 * 2018/11/1.
 */
public class HttpServer {
 
    int port ;
 
    public HttpServer(int port){
        this.port = port;
    }
 
    public void start() throws Exception{
        ServerBootstrap bootstrap = new ServerBootstrap();
        EventLoopGroup boss = new NioEventLoopGroup();
        EventLoopGroup work = new NioEventLoopGroup();
        bootstrap.group(boss,work)
                .handler(new LoggingHandler(LogLevel.DEBUG))
                .channel(NioServerSocketChannel.class)
                .childHandler(new HttpServerInitializer());
 
        ChannelFuture f = bootstrap.bind(new InetSocketAddress(port)).sync();
        System.out.println(" server start up on port : " + port);
        f.channel().closeFuture().sync();
 
    }
 
}
```

server端代码就这么多，看起来很长，但是这就是一个样板代码，你需要着重留意的就是childHandler(new HttpServerInitializer());这一行。如果你对netty还不是十分熟悉，那么你不需要着急把每一行的代码都看懂。这段代码翻译成可以理解的文字是这样的：

1. bootstrap为启动引导器。
2. 指定了使用两个时间循环器。EventLoopGroup
3. 指定使用Nio模式。（NioServerSocketChannel.class）
4. 初始化器为HttpServerInitializer

server启动代码就是这么多，我们注意看 HttpServerInitializer 做了什么。

### 在HttpServerInitializer 中添加server配置

HttpServerInitializer 其实就是一个ChannelInitializer，在这里我们可以指定我们的handler。前面我们说过handler是用来承载我们具体逻辑实现代码的地方，我们需要在ChannelInitializer中加入我们的特殊实现。代码如下：

```java
public class HttpServerInitializer extends ChannelInitializer<SocketChannel>{
 
    @Override
    protected void initChannel(SocketChannel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        pipeline.addLast(new HttpServerCodec());// http 编解码
        pipeline.addLast("httpAggregator",new HttpObjectAggregator(512*1024)); // http 消息聚合器,512*1024为接收的最大contentlength
        pipeline.addLast(new HttpRequestHandler());// 请求处理器
 
    }
}
```

上面代码很简单，需要解释的点如下：

1. channel 代表了一个socket.
2. ChannelPipeline 就是一个“羊肉串”，这个“羊肉串”里边的每一块羊肉就是一个 handler.
   handler分为两种，inbound handler,outbound handler 。顾名思义，分别处理 流入，流出。
3. HttpServerCodec 是 http消息的编解码器。
4. HttpObjectAggregator是Http消息聚合器，Aggregator这个单次就是“聚合，聚集”的意思。http消息在传输的过程中可能是一片片的消息片端，所以当服务器接收到的是一片片的时候，就需要HttpObjectAggregator来把它们聚合起来。
5. 接收到请求之后，你要做什么，准备怎么做，就在HttpRequestHandler中实现。

### httpserver处理请求

上面的展示了 server端启动的代码，然后又展示了 server端初始化器的代码。下面我们来看看，请求处理的handler的代码：

```java
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
 
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
 
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) throws Exception {
        // 100 Continue
        if (is100ContinueExpected(req)) {
            ctx.write(new DefaultFullHttpResponse(
                           HttpVersion.HTTP_1_1,            
                           HttpResponseStatus.CONTINUE));
        }
		// 获取请求的uri
        String uri = req.uri();
        Map<String,String> resMap = new HashMap<>();
        resMap.put("method",req.method().name());
        resMap.put("uri",uri);
        String msg = "<html><head><title>test</title></head><body>你请求uri为：" + uri+"</body></html>";
       // 创建http响应
        FullHttpResponse response = new DefaultFullHttpResponse(
                                        HttpVersion.HTTP_1_1,
                                        HttpResponseStatus.OK,
                                        Unpooled.copiedBuffer(msg, CharsetUtil.UTF_8));
       // 设置头信息
        response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/html; charset=UTF-8");
        //response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain; charset=UTF-8");
       // 将html write到客户端
        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }
}
```

上面代码的意思，我用注释标明了，逻辑很简单。无论是FullHttpResponse，还是 ctx.writeAndFlush都是netty的api，看名知意即可。半蒙半猜之下，也可以看明白这个handler其实是：

1. 获取请求uri，
2. 组装返回的响应内容，
3. 响应对融到客户端。需要解释的是100 Continue的问题:
   100 Continue含义
   HTTP客户端程序有一个实体的主体部分要发送给服务器，但希望在发送之前查看下服务器是否会接受这个实体，所以在发送实体之前先发送了一个携带100 Continue的Expect请求首部的请求。服务器在收到这样的请求后，应该用 100 Continue或一条错误码来进行响应。


