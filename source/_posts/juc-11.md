---
layout: post
title: juc之LinkedBlockingQueue、SynchronousQueue
date: 2020-11-24 12:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自   https://www.jianshu.com/p/28c9d9e34b29   稍有改动

## LinkedBlockingQueue

LinkedBlockingQueue 底层基于单向链表实现的阻塞队列，可以当做无界队列也可以当做有界队列来使用，满足 FIFO 的特性，为了防止 LinkedBlockingQueue 容量迅速增，损耗大量内存。通常在创建 LinkedBlockingQueue 对象时，会指定其大小，如果未指定，容量等于 Integer.MAX_VALUE。因为Integer.MAX_VALUE数值很大，所以可以称为”无界”队列(这里的无界并不是真正意义上的无界)。

LinkedBlockingQueue比ArrayBlockingQueue具有更高的吞吐量，但在大多数并发应用程序中的预测性能较差。

队列中存在最久的元素存在于 head.next 节点上(PS: head 节点永远存在, 且是一个 dummy 节点), 存储时间最短的节点存储在tail上; 通常情况下 LinkedBlockingQueue 的吞吐量要好于 ArrayBlockingQueue.
主要特点:

基于两个lock的 queue, putLock, takeLock; 并且两个锁都有相关联的 condition 用于相应的 await; 每次进行 put/offer 或 take/poll 之后会根据queue的容量进行判断是否需要进行对应的唤醒
队列中总是存在一个 dummy 节点, 每次 poll 节点时获取的是 head.next 节点中的值

为了实现阻塞效果并保证线程安全，它的内部用到了两个锁和两个 Condition。（ArrayBlockingQueue只有一个锁，这导致了LinkedBlockingQueue可以一边取一边放，而ArrayBlockingQueue不行。为啥？因为固定长度的count是个int，不能保证原子性和可见性，至于为什么这么设计，我也不清楚）

queue中的数据存储在 Node 对象中, 且 Node 具有以下的特点:

- head.item 永远是 null, head是一个 dummy 节点, 所以进行 poll 时获取的是 head.next 的值
- tail.next = null

```java
/** Linked list node class */
/**
 * Linked 的数据节点, 这里有个特点, LinkedBlockingQueue 开始构建时会创建一个dummy节点(类似于 ConcurrentLinkedQueue)
 * 而整个队列里面的头节点都是 dummy 节点
 * @param <E>
 */
static class Node<E>{
    E item;

    /**
     * One of:
     * - the real successor Node
     * - this Node, meaning the successor is head.next
     * - null, meaning there is no successor (this is the last node)
     */
    /**
     * 在进行 poll 时 会将 node.next = node 来进行 help gc
     * next == null 指的是要么是队列的 tail 节点
     */
    Node<E> next;
    Node(E x){
        item = x;
    }
}

/** The capacity bound, or Integer.MAX_VALUE if none */
private final int capacity;

/** Current number of elements */
private final AtomicInteger count = new AtomicInteger();

/**
 * Head of linked list
 * Invariant: head.item == null
 * Head 节点的不变性 head.item == null <- 这是一个 dummy 节点(dummy 节点什么作用呢, 主要是方便代码, 不然的话需要处理一些 if 的判断, 加大代码的复杂度, 尤其是非阻塞的实现)
 */
transient Node<E> head;

/**
 * Tail of linked list
 * Invariant: last.next == null
 * Tail 节点的不变性 last.next == null <- 尾节点的 next 是 null
 */
private transient Node<E> last;

/** ReentrantLock Condition 的常见使用方式 */
/** Lock held by take, poll, etc */
private final ReentrantLock takeLock = new ReentrantLock();

/** Wait queue for waiting takes */
private final Condition notEmpty = takeLock.newCondition();

/** Lock held by put, offer, etc */
private final ReentrantLock putLock = new ReentrantLock();

/** Wait queue for waiting puts */
private final Condition notFull = putLock.newCondition();
```

LinkedBlockingQueue 构造函数
LinkedBlockingQueue的构造函数比较简单, 主要是初始化一下容量(默认 Integer.MAX_VALUE), 及 head, tail

```java
/**
 * Creates a {@code KLinkedBlockingQueue} with the given (fixed) capacity
 *
 * @param capacity the capacity of this queue
 * @throws IllegalArgumentException if {@code capacity} is not greater
 *                                  than zero
 */
public KLinkedBlockingQueue(int capacity){
    if(capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity; // 指定 queue 的容量
    last = head = new Node<E>(null); // 默认的在 queue 里面 创建 一个 dummy 节点
}
```

添加元素 put方法
put 方法是将元素添加到队列尾部, queue满时进行await, 添加成功后容量还未满, 则进行 signal

```java
/**
 * Inserts the specified element at the tail of this queue, waiting if
 * necessary for space to become available
 *
 *  将元素加入到 queue 的尾部
 * @param e
 * @throws InterruptedException
 */
public void put(E e) throws InterruptedException{
    if(e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negativeto indicate failure unless set.
    // 有趣的 变量 c 下面会有对它的讲解
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLocK = this.putLock;
    final AtomicInteger count = this.count;  // 获取 queue 的数量 count 
    putLocK.lockInterruptibly(); // 获取 put 的lock
    try {
        /**
         * Note that count is used in wait guard even though it is
         * not protected by lock. This works because count can
         * only decrease at this point (all other puts are shut
         * out by lock), and we (or some other waiting put) are
         * signalled if it ever changes from capacity. Similarly
         * for all other uses of count in other wait guards
         */
        /**
         * 若 queue 的容量满了 则进行 await,直到有人进行通知
         * 那何时进行通知呢?
         * 有两种情况进行通知,
         *      (1) 有线程进行 put/offer 成功后且 (c + 1) < capacity 时
         *      (2) 在线程进行 take/poll 成功 且 (c == capacity) (PS: 这里的 c 指的是 在进行 take/poll 之前的容量)
         */

        while(count.get() == capacity){     // 容量满了, 进行等待
            notFull.await();
        }
        enqueue(node);                        // 进行节点的入队操作
        c = count.getAndIncrement();          // 进行节点个数的增加1, 返回原来的值
        if(c + 1 < capacity){               // 说明 现在的 put 操作后 queue 还没满
            notFull.signal();               // 唤醒其他在睡的线程
        }

    }finally {
        putLock.unlock();                   // 释放锁
    }
    if(c == 0){                             // c == 0 说明 原来queue是空的, 所以这里 signalNotEmpty 一下, 唤醒正在 poll/take 等待中的线程
        signalNotEmpty();
    }
}

/**
 * Links node at end of queue
 * 节点 入队列 (PS: 这里因为有个 dummy 节点, 不需要判空 <- 现在有点感觉 dummy 节点的作用了吧)
 * @param node the node
 */
private void enqueue(Node<E> node){
    // assert putLock.isHeldByCurrentThread()
    // assert last.next == null
    last = last.next = node;
}

/**
 * Signals a waiting take. Called only from put/offer (which do not
 * otherwise ordinarily lock takeLock.)
 */
private void signalNotEmpty(){
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    }finally {
        takeLock.unlock();
    }
}
```

代码的注释中基本把操作思想都说了, 有几个注意的地方

- 当queue满时, 会调用 notFull.await() 进行等待, 而相应的唤醒的地方有两处, 一个是 "有线程进行 put/offer 成功后且 (c + 1) < capacity 时", 另一处是 "在线程进行 take/poll 成功 且 (c == capacity) (PS: 这里的 c 指的是 在进行 take/poll 之前的容量)"
- 代码中的 "signalNotEmpty" 这时在原来queue的数量 c (getAndIncrement的返回值是原来的值) ==0 时对此时在调用 take/poll 方法的线程进行唤醒

添加元素offer 方法

offer与put都是添加元素到queue的尾部, 只不过 put 方法在队列满时会进行阻塞, 直到成功; 而 offer 操作在容量满时直接返回 false.

```java
/**
 * Inserts the specified element at the tail of this queue, waiting if
 * necessary up to the specified wait time for space to become available
 *
 *  支持中断和超时的 offer 节点
 *
 * @param e
 * @param timeout
 * @param unit
 * @return {@code true} if successful, or {@code false} if
 *          the specified waiting time elapses before space is available
 * @throws InterruptedException
 */
public boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException{
    if(e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;     // 获取 put lock
    final AtomicInteger count = this.count;         // 获取 queue 的容量
    putLock.lockInterruptibly();
    try {
        while(count.get() == capacity){             // queue的容量满了进行 带 timeout 的 await
            if(nanos <= 0){                           //  用光了 timeout 直接 return false
                return false;
            }
            nanos = notFull.awaitNanos(nanos);      // 直接 await (PS: 返回值 nanos <= 0 说明 等待是超时了, 正常 await 并且 被 signal nanos > 0; 具体详情会在 Condition 那一篇中详细说明)
        }
        enqueue(new Node<E>(e));                    // 节点若队列
        c = count.getAndIncrement();                // 获取入队列之前的容量
        if(c + 1 < capacity){                     // c + 1 < capacity 说明 现在的 offer 成功后 queue 还没满
            notFull.signal();                     // 唤醒其他正在 await 的线程
        }
    }finally {
        putLock.unlock();                           // 释放锁
    }
    if(c == 0){
        signalNotEmpty();                            // c == 0 说明 原来queue是空的, 所以这里 signalNotEmpty 一下, 唤醒正在 poll/take 等待中的线程
    }
    return true;
}
```

offer 整个操作和 put 差不多, 唯一变化的是多了一个 notFull.awaitNanos(nanos), 这个函数的返回值若是负数, 则说明等待超时, 则直接 return false (关于 Condition.awaitNanos 方法会在后续再说)

获取queue头元素 take 方法
此方法是获取 queue 中呆着时间最长的节点的值(head.next)

```java
/**
 * 取走 queue 中呆着时间最长的节点的 item (其实就是 head.next.item 的值)
 * @return
 * @throws InterruptedException
 */
public E take() throws InterruptedException{
    E x;
    int c = -1;
    final AtomicInteger count = this.count;          // 获取 queue 的容量
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();                      // 获取 lock
    try {
        while(count.get() == 0){                      // queue 为空, 进行 await
            notEmpty.await();
        }
        x = dequeue();                                 // 将 head.next.item 的值取出, head = head.next
        c = count.getAndDecrement();                   // queue 的容量计数减一
        if(c > 1){
            notEmpty.signal();                        // c > 1 说明 进行 take 后 queue 还有值
        }
    }finally {
        takeLock.unlock();                              // 释放 lock
    }
    if(c == capacity){                                // c == capacity 说明一开始 queue 是满的, 调用 signalNotFull 进行唤醒一下 put/offer 的线程
        signalNotFull();
    }
    return x;
}

/**
 * Removes a node from head of queue
 * 节点出队列 这里有个注意点 head 永远是 dummy 节点, dequeue 的值是 head.next.item 的值
 * 在 dequeue 后 将 原  head 的后继节点设置为 head(成为了 dummy 节点)
 * @return the node
 */
private E dequeue(){
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;       // 这里的 head 是一个 dummy 节点
    Node<E> first = h.next; // 获取真正的节点
    h.next = h;             // help GC
    head = first;           // 重行赋值 head
    E x = first.item;       // 获取 dequeue 的值
    first.item = null;      // 将 item 置 空
    return x;
}

/** Signal a waiting put. Called only from take/poll */
private void signalNotFull(){
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    }finally {
        putLock.unlock();
    }
}
```

操作过程: 将 head.next 的值取出, 将 head.next 设置为新的head; 操作的步骤比较少, 只有两处 condition 的唤醒需要注意一下：

当 take 结束时, 判断 queue 是否还有元素 (c > 1) 来进行 notEmpty.signal()
当 take 结束时, 判断原先的容量是否已经满 (c == capacity) 来决定是否需要调用 signalNotFull 进行唤醒此刻还在等待 put/offer 的线程

获取queue头元素 poll 方法
poll 与 take 都是获取头节点的元素, 唯一的区别是 take在queue为空时进行await, poll 则直接返回

```java
/**
 * 带 timeout 的poll 操作, 获取 head.next.item 的值
 * @param timeout
 * @param unit
 * @return
 * @throws InterruptedException
 */
public E poll(long timeout, TimeUnit unit) throws InterruptedException{
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);             //  计算超时时间
    final AtomicInteger count = this.count;       // 获取 queue 的容量
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();                   // 获取 lock
    try{
        while(count.get() == 0){                   // queue 为空, 进行 await
            if(nanos <= 0){                        // timeout 用光了, 直接 return null
                return null;
            }
            nanos = notEmpty.awaitNanos(nanos);   // 调用 condition 进行 await, 在 timeout之内进行 signal -> nanos> 0
        }
        x = dequeue();                             // 节点出queue
        c = count.getAndDecrement();               // 计算器减一
        if(c > 1){                                 // c > 1 说明 poll 后 容器内还有元素, 进行 换新 await 的线程
            notEmpty.signal();
        }
    }finally {
        takeLock.unlock();                         // 释放锁
    }
    if(c == capacity){                           // c == capacity 说明一开始 queue 是满的, 调用 signalNotFull 进行唤醒一下 put/offer 的线程
        signalNotFull();
    }
    return x;
}
```

LinkedBlockingQueue 是一个基于链表实现的阻塞queue, 它的性能好于 ArrayBlockingQueue, 但是差于 ConcurrentLinkeQueue; 并且它非常适于生产者消费者的环境中, 比如 Executors.newFixedThreadPool() 就是基于这个队列的。

LinkedBlockingQueue的静态工厂方法
使用LinkedBlockingQueue的好处：

因为线程大小固定的线程池，其线程的数量是不具备伸缩性的，当任务非常繁忙的时候，就势必会导致所有的线程都处于工作状态，导致队列满的情况发生，从而导致任务无法提交而抛出RejectedExecutionException；
而使用”无界”队列由于其良好的存储容量的伸缩性，可以很好的去缓冲任务繁忙情况下场景，即使任务非常多，也可以进行动态扩容，当任务被处理完成之后，队列中的节点也会被随之被GC回收，非常灵活。

FixedThreadPool的execute()说明

![DNQXj0.png](https://s3.ax1x.com/2020/11/24/DNQXj0.png)

![DNQxBT.png](https://s3.ax1x.com/2020/11/24/DNQxBT.png)

FixedThreadPool的execute()执行流程

- 如果当前运行的线程数少于corePoolSize，则会创建新线程来执行任务；

- 在线程池完成预热之后(当前运行的线程数等于corePoolSize)，将任务加入LinkedBlockingQueue；

- 线程执行完步骤A中的任务后，会在循环中反复从LinkedBlockingQueue获取任务来执行。

使用Executors.newFixedThreadPool相关的方法，LinkedBlockingQueue的capacity为Integer.MAX_VALUE，所以会对线程池有以下影响：

- 当线程池中的线程数达到corePoolSize后，新任务将在无界队列中等待，因此线程池中的线程数不会超过corePoolSize；

- 由于A影响，使用”无界”队列时maximumPoolSize将是一个无效参数；

- 由于A、B影响，使用”无界”队列时keepAliveTime将是一个无效参数；

- 由于使用”无界队列”，运行中的FixedThreadPool(未执行shutdown()或shutdownNow())不会拒绝任务(不会调用RejectedExecutionException.rejectedExecution方法)。

SingleThreadExecutor的execute()说明

![DNlDvq.png](https://s3.ax1x.com/2020/11/24/DNlDvq.png)

![DNl6bT.png](https://s3.ax1x.com/2020/11/24/DNl6bT.png)

SingleThreadExecutor的execute()执行流程

- 如果当前运行的线程数少于corePoolSize(即线程池中无运行的线程)，则会创建一个新线程来执行任务；

- 在线程池完成预热之后(当前线程池中有一个运行的线程)，将任务加入LinkedBlockingQueue；

- 线程执行完步骤A中的任务后，会在一个无线循环中反复从LinkedBlockingQueue获取任务来执行。

SingleThreadExecutor使用”无界”的工作队列对线程池带来的影响与使用Executors.newFixedThreadPool相同。

ArrayBlockingQueue和LinkedBlockingQueue的比较

- ArrayBlockingQueue由于其底层基于数组，并且在创建时指定存储的大小，在完成后就会立即在内存分配固定大小容量的数组元素，因此其存储通常有限；
- 而LinkedBlockingQueue可以由用户指定最大存储容量，也可以无需指定，如果不指定则最大存储容量将是Integer.MAX_VALUE，由于其节点的创建都是动态创建，并且在节点出队列后可以被GC所回收，因此其具有灵活的伸缩性。
- 但是由于ArrayBlockingQueue的有界性，因此其能够更好的对于性能进行预测，
- 而LinkedBlockingQueue由于没有限制大小，当任务非常多的时候，不停地向队列中存储，就有可能导致内存溢出的情况发生。

- ArrayBlockingQueue中在入队列和出队列操作过程中，使用的是同一个lock，所以即使在多核CPU的情况下，其读取和操作的都无法做到并行，而LinkedBlockingQueue的读取和插入操作所使用的锁是两个不同的lock，它们之间的操作互相不受干扰，因此两种操作可以并行完成，故LinkedBlockingQueue的吞吐量要高于ArrayBlockingQueue。

## SynchronousQueue

每个插入操作必须等待另一个线程的对应移除操作 ，反之亦然。同步队列没有任何内部容量，甚至连一个队列的容量都没有。
SynchronousQueue的支持公平策略和非公平策略，所以底层可能两种数据结构：队列（实现公平策略）和栈（实现非公平策略），队列与栈都是通过链表来实现的。

SynchronousQueue的静态工厂方法

SynchronousQueue的execute()说明

![DNG0oT.png](https://s3.ax1x.com/2020/11/24/DNG0oT.png)

![DNGrYF.png](https://s3.ax1x.com/2020/11/24/DNGrYF.png)

可以看到CacheThreadPool的corePoolSize被设置为0，即corePool为空，maximumPoolSize被设置为Integer.MAX_VALUE，即maximumPool是”无界”的，这意味着，如果主线程提交任务的速度高于maximumPool中线程处理任务的速度，CacheThreadPool会不断创建新的线程，极端情况下，CacheThreadPool会因为创建过多线程而耗尽CPU和内存资源；keepAliveTime设置为60L，意味着CacheThreadPool中的空闲线程等待新任务的最长时间为60s，空闲线程超过60s将被终止。

SynchronousQueue的execute()执行流程

- 首先执行SynchronousQueue.offer(E o)。如果当前maximumPool中有空闲线程正在执行SynchronousQueue.poll(long timeout, TimeUnit unit)，那么主线程执行offer操作与空闲线程执行的poll操作配对成功，主线程把任务交给空闲线程执行，execute()方法执行完成；否则执行下面的步骤2；

- 当初始maximumPool为空，或者maximumPool中当前没有空闲线程，将没有线程执行SynchronousQueue.poll(long timeout, TimeUnit unit)。这种情况下步骤1将失败。此时CacheThreadPool会创建一个新线程执行任务，execute()方法执行完成；

- 在步骤2中创建的线程将任务执行完成后，会执行SynchronousQueue.poll(long timeout, TimeUnit unit)。这个poll操作会让线程最多在SynchronousQueue中等待60s。如果60s内主线程就提交了一个新任务(主线程执行步骤1)，那么这个空闲线程将执行主线程提交的新任务；否则，这个空闲线程将终止。由于60s的空闲线程会被终止，因此长时间保持空闲的CacheThreadPool不会使用任何资源。
