---
layout: post
title: juc之Semaphore
date: 2020-06-22 13:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

## Semaphore 是什么
Semaphore是JUC包提供的一个共享锁，一般称之为信号量。可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。

Semaphore通过自定义的同步器维护了一个或多个共享资源，线程通过调用acquire获取共享资源，通过调用release释放。

注意Semaphore并非lock的子类，所以加锁和解锁，用的是acquire，release，tryacquire，tryrelease

可以把它简单的理解成我们停车场入口立着的那个显示屏，每有一辆车进入停车场显示屏就会显示剩余车位减1，每有一辆车从停车场出去，显示屏上显示的剩余车辆就会加1，当显示屏上的剩余车位为0时，停车场入口的栏杆就不会再打开，车辆就无法进入停车场了，直到有一辆车从停车场出去为止。

## 用semaphore 实现停车场提示牌功能

业务场景 ：

- 停车场容纳总停车量10。
- 当一辆车进入停车场后，显示牌的剩余车位数响应的减1.
- 每有一辆车驶出停车场后，显示牌的剩余车位数响应的加1。
- 停车场剩余车位不足时，车辆只能在外面等待。

```java
public class TestCar {
​
    //停车场同时容纳的车辆10
    private  static  Semaphore semaphore=new Semaphore(10);
​
    public static void main(String[] args) {
​
        //模拟100辆车进入停车场
        for(int i=0;i<100;i++){
​
            Thread thread=new Thread(new Runnable() {
                public void run() {
                    try {
                        System.out.println("===="+Thread.currentThread().getName()+"来到停车场");
                        if(semaphore.availablePermits()==0){
                            System.out.println("车位不足，请耐心等待");
                        }
                        semaphore.acquire();//获取令牌尝试进入停车场
                        System.out.println(Thread.currentThread().getName()+"成功进入停车场");
                        Thread.sleep(new Random().nextInt(10000));//模拟车辆在停车场停留的时间
                        System.out.println(Thread.currentThread().getName()+"驶出停车场");
                        semaphore.release();//释放令牌，腾出停车场车位
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            },i+"号车");
​
            thread.start();
​
        }
​
    }
}
```

## 源码

```java
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
```

初始化Semaphore时需要指定共享资源的个数。Semaphore提供了两种模式：公平模式&非公平模式。如果不指定工作模式的话，默认工作在非公平模式下。后面我们将看到，两种模式的区别在于获取共享资源时的排序策略。Semaphore有三个内部类：Sync&NonfairSync&FairSync。后两个继承自Sync，Sync继承自AQS。除了序列化版本号之外，Semaphore只有一个成员变量sync，公平模式下sync初始化为FairSync，非公平模式下sync初始化为NonfairSync。

## acquire 响应中断获取资源

Semaphore提供了两种获取资源的方式：响应中断&不响应中断。我们先来看一下响应中断的获取。

```java
public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

acquire方法由同步器sync调用上层AQS提供的acquireSharedInterruptibly方法获取：

```java
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

acquireSharedInterruptibly方法先检测中断。然后调用tryAcquireShared方法试图获取共享资源。这时公平模式和非公平模式的代码执行路径发生分叉，FairSync和NonfairSync各自重写了tryAcquireShared方法。

我们先来看下非公平模式下的tryAcquireShared方法：

```java
     protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
```

它直接代用了父类Sync提供的nonfairTryAcquireShared方法：

```java
final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

注意，这里是一个CAS自旋。因为Semaphore是一个共享锁，可能有多个线程同时申请共享资源，因此CAS操作可能失败。直到成功获取返回剩余资源数目，或者发现没有剩余资源返回负值代表申请失败。有一个问题，为什么我们不在CAS操作失败后就直接返回失败呢？因为这样做虽然不会导致错误，但会降低效率：在还有剩余资源的情况下，一个线程因为竞争导致CAS失败后被放入等待序列尾，一定在队列头部有一个线程被唤醒去试图获取资源，这比直接自旋继续获取多了操作等待队列的开销。(有剩余资源的时候就不断的自旋获取，没有剩的了再入队)

这里“非公平”的语义体现在：如果一个线程通过nonfairTryAcquireShared成功获取了共享资源，对于此时正在等待队列中的线程来说，可能是不公平的：队列中线程先到，却没能先获取资源。

如果tryAcquireShared没能成功获取，acquireSharedInterruptibly方法调用doAcquireSharedInterruptibly方法将当前线程放入等待队列并开始自旋检测获取资源：

```java
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

我们注意到，doAcquireSharedInterruptibly中，当一个线程从parkAndCheckInterrupt方法中被中断唤醒之后，直接抛出了中断异常。还记得我们分析AQS时的doAcquireShared方法吗，它在这里的处理方式是用一个局部变量interrupted记录下这个异常但不立即处理，而是等到成功获取资源之后返回这个中断标志，并在上层调用selfInterrupt方法补上中断。
这正是两个方法的关键区别：是否及时响应中断。

我们再来看公平模式下的tryAcquireShared方法：

```java
protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

相比较非公平模式的nonfairTryAcquireShared方法，公平模式下的tryAcquireShared方法在试图获取之前做了一个判断，如果发现等对队列中有线程在等待获取资源，就直接返回-1表示获取失败。当前线程会被上层的acquireSharedInterruptibly方法调用doAcquireShared方法放入等待队列中。这正是“公平”模式的语义：如果有线程先于我进入等待队列且正在等待，就直接进入等待队列，效果便是各个线程按照申请的顺序获得共享资源，具有公平性。

## acquireUnInterruptibly 不响应中断获取资源

```java
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }
```

acquireUnInterruptibly方法调用AQS提供的acquireShared方法：

```java
 public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

acquireShared方法首先试图获取资源，这与acquireSharedInterruptibly方法相比，没有先检测中断的这一步。紧接着调用doAcquireShared方法，由于这个方法在AQS中已经详细分析过，这里我们只关注它与doAcquireSharedInterruptibly方法的区别：

```java
 private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

正如刚刚说过的，区别只在线程从parkAndCheckInterrupt方法中因中断而返回时的处理：在这里它没有抛出异常，而是用一个局部变量interrupted记录下这个异常但不立即处理，而是等到成功获取资源之后返回这个中断标志，并在上层调用selfInterrupt方法补上中断。

## acquire(int) & acquireUninterruptibly(int) 指定申请的资源数目的获取

```java
public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }
```

可以看到，与不指定数目时的获取的区别仅在参数值，不再赘述。

## release 释放资源

公平模式和非公平模式的释放资源操作是一样的：

```java
public void release() {
        sync.releaseShared(1);
    }
    
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }
```

调用AQS提供的releaseShared方法：

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

releaseShared方法首先调用我们重写的tryReleaseShared方法试图释放资源。然后调用doReleaseShared方法唤醒队列之后的等待线程。我们主要关注tryReleaseShared方法：

```java
protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

这个方法也是一个CAS自旋，原因是应为Semaphore是一个共享锁，可能有多个线程同时释放资源，因此CAS操作可能失败。最后方法总会成功释放并返回true（如果不出错的话）。

## tryAcquire & tryAcquire(timeout) 方法

```java
public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }
```

没有指定等待时间的tryAcquire调用的是nonfairTryAcquireShared方法，我们已经分析过，不再赘述。我们重点关注指定等待时长的方法。限时等待是通过调用AQS提供的tryAcquireSharedNanos方法实现的：

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
```

注意：限时等待默认都是及时响应中断的。方法开始先检测中断，然后调用tryAcquireShared方法试图获取资源，如果成功的话直接返回true，不成功则调用doAcquireSharedNanos方法：

```java
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return true;
                    }
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

方法在自旋之前先计算了一个结束等待的时间节点deadline，然后便开始自旋，每次自旋都要计算一下剩余等待时间nanosTimeout，如果nanosTimeout小于等于0，说明已经到达deadline，直接返回false表示超时。

有一点值得注意，spinForTimeoutThreshold这个值规定了一个阈值，当剩余等待时间小于这个值的时候，线程将不再被park，而是一直在自旋试图获取资源。关于这个值的作用Doug Lea是这样注释的：

```java
/**
     * The number of nanoseconds for which it is faster to spin
     * rather than to use timed park. A rough estimate suffices
     * to improve responsiveness with very short timeouts.
     */
```

park和unpark操作需要一定的开销，当nanosTimeout很小的时候，这个开销就相对很大了。这个阈值的设置可以让短时等待的线程一直保持自旋，可以提高短时等待的反应效率，而由于nanosTimeout很小，自旋又不会有过多的开销。

除此之外，doAcquireSharedNanos方法与不限时等待的doAcquireShared方法还有两点重要区别：

- 由于有等待时限，所以线程从park方法返回时我们不能确定返回的原因是中断还是超时，因此需要调用interrupted方法检测一下中断标志；
- doAcquireSharedNanos方法是及时响应中断的，而doAcquireShared方法延迟处理中断。

## drainPermits & reducePermits 修改剩余共享资源数量

Semaphore提供了“耗尽”所有剩余共享资源的操作：

```java
public int drainPermits() {
        return sync.drainPermits();
    }
```

drainPermits调用了自定义同步器Sync的同名方法：

```java
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
```

用CAS自旋将剩余资源清空。

我们再来看看“缩减”剩余共享资源的操作：

```java
protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }
```

首先，缩减必须是单向的，即只能减少不能增加，然后调用Sync的同名方法：

```java
final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }
```

用CAS自旋在剩余共享资源上做缩减。

上述两个对共享资源数量的修改操作有两点需要注意：

- 是不可逆的
- 是对剩余资源的操作而不是全部资源，当剩余资源数目不足或已经为0时，方法就返回，正咋被占用的资源不参与。

## 总结：

Semaphore是JUC包提供的一个典型的共享锁，它通过自定义两种不同的同步器（FairSync&NonfairSync）提供了公平&非公平两种工作模式，两种模式下分别提供了限时/不限时、响应中断/不响应中断的获取资源的方法（限时获取总是及时响应中断的），而所有的释放资源的release操作是统一的。