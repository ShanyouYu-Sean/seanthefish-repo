---
layout: post
title: 认识socket和Linux io模型
date: 2019-09-1 11:00:00
tags: 
- nio
categories:
- nio
---

## 什么是socket

我们知道两个进程如果需要进行通讯最基本的一个前提能能够唯一的标示一个进程，在本地进程通讯中我们可以使用PID来唯一标示一个进程，但PID只在本地唯一，网络中的两个进程PID冲突几率很大，这时候我们需要另辟它径了，我们知道IP层的ip地址可以唯一标示主机，而TCP层协议和端口号可以唯一标示主机的一个进程，这样我们可以利用ip地址＋协议＋端口号唯一标示网络中的一个进程。

能够唯一标示网络中的进程后，它们就可以利用socket进行通信了，什么是socket呢？我们经常把socket翻译为套接字，socket是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用已实现进程在网络中通信。

![image.png](https://i.loli.net/2020/12/04/qjp1MLaVeWCwyO5.png)

socket起源于UNIX，在Unix一切皆文件哲学的思想下，socket是一种"打开—读/写—关闭"模式的实现，服务器和客户端各自维护一个"文件"，在建立连接打开后，可以向自己文件写入内容供对方读取或者读取对方内容，通讯结束时关闭文件。（Linux认为所有操作都是基于文件的，所以每建立一个socket链接都会返回一个文件名描述符，即socket描述符）

所以每建立一个socket链接，其实就是不同计算机上的两个进程通过socket这种形式进行通信，而通信的过程伴随着缓冲区的读写，操作系统会创建一个由文件系统管理的socket对象，进行文件的读和写，这就被称为网络io。

## socket通信流程

socket是"打开—读/写—关闭"模式的实现，以使用TCP协议通讯的socket为例，其交互流程大概是这样子的

![image.png](https://i.loli.net/2020/12/04/a2WIDYql8Bv4yMO.png)

![image.png](https://i.loli.net/2020/12/06/VPRHSDWou9hBjyX.png)

可以看到跟三次握手是能对应上的

![image.png](https://i.loli.net/2020/12/04/nFvcRCQMWtUeu9J.png)


## linux 上的网络编程接口

### socket()函数

`int socket(int domain, int type, int protocol);`

socket函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述符，socket()用于创建一个socket描述符（socket descriptor），它唯一标识一个socket。这个socket描述符跟文件描述符一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。

[什么是文件描述符?](https://segmentfault.com/a/1190000009724931)

正如可以给fopen的传入不同参数值，以打开不同的文件。创建socket的时候，也可以指定不同的参数创建不同的socket描述符，socket函数的三个参数分别为：

domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。

type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等等。

protocol：故名思意，就是指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。

注意：并不是上面的type和protocol可以随意组合的，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当protocol为0时，会自动选择type类型对应的默认协议。

当我们调用 socket 创建一个socket时，没有一个具体的地址。如果想要给它赋值一个地址，就必须调用bind()函数。

### bind()函数

正如上面所说bind()函数把一个地址族中的特定地址赋给socket。例如对AF_INET、AF_INET6就是把一个ipv4或ipv6地址和端口号组合赋给socket。

`int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

函数的三个参数分别为：

sockfd：即socket描述符，它是通过socket()函数创建了，唯一标识一个socket。bind()函数就是将给这个描述符绑定一个名字。

addr：一个const struct sockaddr *指针，指向要绑定给sockfd的协议地址。这个地址结构根据地址创建socket时的地址协议族的不同而不同。

通常服务器在启动的时候都会绑定一个众所周知的地址（如ip地址+端口号），用于提供服务，客户就可以通过它来接连服务器；而客户端就不用指定，有系统自动分配一个端口号和自身的ip地址组合。这就是为什么通常服务器端在listen之前会调用bind()，而客户端就不会调用，而是在connect()时由系统随机生成一个。

### listen()函数

如果作为一个服务器，在调用socket()、bind()之后就会调用listen()来监听这个socket，

`int listen(int sockfd, int backlog);`

listen函数的第一个参数即为要监听的socket描述符，第二个参数为相应socket可以排队的最大连接个数。socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。

### connect()函数

`int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);`

如果客户端这时调用connect()发出连接请求，服务器端就会接收到这个请求。

connect函数的第一个参数即为客户端的socket描述符，第二参数为服务器的socket地址，第三个参数为socket地址的长度。客户端通过调用connect函数来建立与TCP服务器的连接。

### accept()函数

TCP服务器端依次调用socket()、bind()、listen()之后，就会监听指定的socket地址了。TCP客户端依次调用socket()、connect()之后就想TCP服务器发送了一个连接请求。TCP服务器监听到这个请求之后，就会调用accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络I/O操作了，即类同于普通文件的读写I/O操作。

`int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);`

accept函数的第一个参数为服务器的socket描述符，第二个参数为指向struct sockaddr *的指针，用于返回客户端的协议地址，第三个参数为协议地址的长度。如果accpet成功，那么其返回值是由内核自动生成的一个全新的描述符，代表与返回客户的TCP连接。

注意：accept的第一个参数为服务器的socket描述符，是服务器开始调用socket()函数生成的，称为监听socket描述符；而accept函数返回的是已连接的socket描述符。一个服务器通常通常仅仅只创建一个监听socket描述符，它在该服务器的生命周期内一直存在。内核为每个由服务器进程接受的客户连接创建了一个已连接socket描述符，当服务器完成了对某个客户的服务，相应的已连接socket描述符就被关闭。

### read()、write()等函数

服务器与客户已经建立好连接了。可以调用网络I/O进行读写操作了，即实现了网咯中不同进程之间的通信！网络I/O操作有下面几组：

- read()/write()
- recv()/send()
- readv()/writev()
- recvmsg()/sendmsg()
- recvfrom()/sendto()

read函数是负责从fd中读取内容。当读成功时，read返回实际所读的字节数，如果返回的值是0表示已经读到文件的结束了，小于0表示出现了错误。如果错误为EINTR说明读是由中断引起的，如果是ECONNREST表示网络连接出了问题。

write函数将buf中的nbytes字节内容写入文件描述符fd.成功时返回写的字节数。失败时返回-1，并设置errno变量。 在网络程序中，当我们向套接字文件描述符写时有俩种可能。

- write的返回值大于0，表示写了部分或者是全部的数据。
- 返回的值小于0，此时出现了错误。我们要根据错误类型来处理。如果错误为EINTR表示在写的时候出现了中断错误。如果为EPIPE表示网络连接出现了问题(对方已经关闭了连接)。

其它的就不一一介绍了。这几对I/O函数了，具体参见man文档或者使用搜索引擎，下面的例子中将使用到send/recv。

### close()函数

在服务器与客户端建立连接之后，会进行一些读写操作，完成了读写操作就要关闭相应的socket描述符，好比操作完打开的文件要调用fclose关闭打开的文件。

`int close(int fd);`

close一个TCP socket的缺省行为时把该socket标记为以关闭，然后立即返回到调用进程。该描述字不能再由调用进程使用，也就是说不能再作为read或write的第一个参数。

close操作只是使相应socket描述字的引用计数-1，只有当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。

## IO模型

首先说一下同步异步，阻塞非阻塞的问题 看这里

怎样理解阻塞非阻塞与同步异步的区别？ - 萧萧的回答 - 知乎
https://www.zhihu.com/question/19732473/answer/241673170

其实阻塞和非阻塞描述的是进程的一个操作是否会使得进程转变为“等待”的状态， 但是为什么我们总是把它和 IO 连在一起讨论呢？原因是， 阻塞这个词是与系统调用 System Call 紧紧联系在一起的， 因为要让一个进程进入 等待（waiting） 的状态, 要么是它主动调用 wait() 或 sleep() 等挂起自己的操作， 另一种就是它调用 System Call, 而 System Call 因为涉及到了 I/O 操作， 不能立即完成， 于是内核就会先将该进程置为等待状态， 调度其他进程的运行， 等到 它所请求的 I/O 操作完成了以后， 再将其状态更改回 ready 。

非阻塞I/O 系统调用( nonblocking system call ) 和 异步I/O系统调用 （asychronous system call）的区别是：

- 一个非阻塞I/O 系统调用 read() 操作立即返回的是任何可以立即拿到的数据， 可以是完整的结果， 也可以是不完整的结果， 还可以是一个空值。
- 而异步I/O系统调用 read（）结果必须是完整的， 但是这个操作完成的通知可以延迟到将来的一个时间点。

《unix网络编程》里总结了五类IO模型。为了更好地理解，我会举一个叫外卖的例子来说明。

### 阻塞模型

第一种是完全阻塞模型，它的示意图如下：

![image.png](https://i.loli.net/2020/12/03/WAuepSV81YTRtGH.png)

就是说，如果我客户端发起了connect请求，那么当前线程就会休眠，等待服务端响应完毕，返回消息，才会继续走下去。

就好比，我叫个外卖，然后我就去大门口傻等着，外卖不送到，我也什么都不做，就坐在门口打盹，直到外卖小哥过来，把我叫醒，我才拿着外卖回家去吃。这样做显然效率不高，显得脑子有水。

### 非阻塞式IO

把非阻塞的文件描述符称为非阻塞I/O。可以通过设置SOCK_NONBLOCK标记创建非阻塞的socket fd，或者使用fcntl将fd设置为非阻塞。

对非阻塞fd调用系统接口时，不需要等待事件发生而立即返回，事件没有发生，接口返回-1，此时需要通过errno的值来区分是否出错，有过网络编程的经验的应该都了解这点。不同的接口，立即返回时的errno值不尽相同，如，recv、send、accept errno通常被设置为EAGIN 或者EWOULDBLOCK，connect 则为EINPRO- GRESS 。

![image.png](https://i.loli.net/2020/12/03/AJevsGaucQbYj9w.png)

就是说，客户端程序会不停地去尝试读取数据，但是不会阻塞在那个读方法里，如果读的时候，没有读到内容，也会立即返回。这就允许我们在客户端里，读到不数据的时候可以搞点其他的事情了。
仍然以外卖举例，就相当于，我一边扫地，一边等外卖。我不再像原来一样，在门口傻等了，而是扫两下，就跑到门口看看外卖到了没有。一直这样循环，直到我取到外卖，才从这个循环中跳出来，进入吃的流程。

### IO多路复用

最常用的I/O事件通知机制就是I/O复用(I/O multiplexing)。Linux 环境中使用select/poll/epoll_wait 实现I/O复用，I/O复用接口本身是阻塞的（epoll本身要阻塞在那里等待socket返回），在应用程序中通过I/O复用接口向内核注册fd所关注的事件，当关注事件触发时，通过I/O复用接口的返回值通知到应用程序。I/O复用接口可以同时监听多个I/O事件以提高事件处理效率。

![image.png](https://i.loli.net/2020/12/03/rw8SdgpQUYcIRPO.png)

还是外卖的例子，如果我们整栋楼的人，很多人叫了外卖，都有下楼来看外卖到没到的需求，于是物业就出了个招，让门卫小哥帮大家看着，整栋楼上的，不管是谁的外卖到了，先放到门卫小哥那里，然后门卫小哥再通知你下来拿自己的外卖。这样一来，我们就把本来多个人要跑去看自己的外卖到了这件事交给门卫小哥去做了。而我们解放出来，就可以继续看电视，打扫卫生，刷知乎了。由于我们可以继续做自己的事情，外卖小哥和门卫小哥在同时也在工作，互不干扰，所以这种工作方式就被称为异步模型。

### SIGIO

除了I/O复用方式通知I/O事件，还可以通过SIGIO信号来通知I/O事件，如图所示。两者不同的是，在等待数据达到期间，I/O复用是会阻塞应用程序，而SIGIO方式是不会阻塞应用程序的。

![image.png](https://i.loli.net/2020/12/03/3TDm1NUaHlZ2Rju.png)

上面这张图，就是我们现实生活中真正的外卖。数据到达以后，给客户端发一个消息，让客户端过来取数据。这就像外卖小哥到你家门口给你打电话，让你出来取一下。显然，这种是最方便的，也是最合理的。

但实际上，在真正的编程中，我们很少使用这种模型。

### async io

POSIX规范定义了一组异步操作I/O的接口，不用关心fd 是阻塞还是非阻塞，异步I/O是由内核接管应用层对fd的I/O操作。异步I/O向应用层通知I/O操作完成的事件，这与前面介绍的I/O 复用模型、SIGIO模型通知事件就绪的方式明显不同。以aio_read 实现异步读取IO数据为例，如图所示，在等待I/O操作完成期间，不会阻塞应用程序。

![image.png](https://i.loli.net/2020/12/03/D4usxU8mARNQbjg.png)

这个图，如果对应到外卖有点不合适了，比较像网购空调，我所要做的，只是下单，而快递小哥会把空调送过来，他也不会让你自己去取，他会让安装师傅直接帮你安装。这个过程中，你什么都不需要做。你所要做的仅仅是发起一个请求。这种IO模型就是纯正的异步IO。

这种纯异步IO的最典型例子就是node.js中的callback。
