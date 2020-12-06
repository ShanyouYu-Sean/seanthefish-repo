---
layout: post
title: juc之ReentrantLock
date: 2020-11-22 10:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

ReentrantLock是JUC包提供的一种可重入独占锁，它实现了Lock接口。与Semaphore类似，ReentrantLock也提供了两种工作模式：公平模式&非公平模式，也是通过自定义两种同步器FairSync&NonfairSync来实现的。

## lock 不响应中断获取锁

```java
public void lock() {
        sync.lock();
    }
```
lock方法通过调用自定义同步器的同名方法来获取锁。注意：ReentrantLock自定义了两种同步器：FairSync&NonfairSync，分别对应公平模式&非公平模式。

我们先来看一下非公平模式下的lock方法：

```java
final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

lock方法并没有直接调用AQS提供的acquire方法，而是先试探地获取了一下锁，CAS操作失败再去调用acquire方法。我的理解是为了提升性能。因为可能很多时候我们能在第一次试探获取时成功，而不需要经过acquire->tryAcquire->nonfairAcquire的调用过程：

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

AQS提供的acquire方法首先调用了我们自定义同步器重写的tryAcquire方法试图获取锁，如果失败的话先调用addWaiter方法将当前线程加入等待队列，然后对掉用acquireQueued方法进行自旋、检测获取锁的操作，直到成功获取锁。在自旋、检测的过程中如果被中断（注意：acquireQueued延迟处理中断），要在成功获取锁之后调用selfInterrupt方法“补上”这次中断。这里我们主要关注ReentrantLock重写的tryAcquire方法：

```java
 protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

nonfairSync的tryAcquire方法通过调用其父类Sync的nonfairTryAcquire方法实现：

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

nonfairTryAcquire方法首先判断锁是否被占用，如果锁可用，通过调用CAS操作试图获取锁，如果失败直接返回false；但如果锁被占用（state==0），并不代表没有机会，因为有可能占用锁的正是当前线程。如果正是当前线程占用了锁，让state做+1操作，然后返回true：这正是可重入的概念，一个已经获取锁的线程可以重复获取锁。

我们再来看一下公平模式下的lock方法：

```java
final void lock() {
            acquire(1);
        }
```

fairSync的lock方法直接调用acquire，而没有想NonfairSync一样先试图获取，因为这样可能导致违反“公平”的语义：在已等待在队列中的线程之前获取了锁。

由上面的分析可知，AQS的acquire方法调用了fairSync重写的tryAcquire方法：

```java
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

这与nonfairTryAcquire方法大同小异，主要区别在于，当发现锁未被占用的时候，还要判断一下等待队列中是否有先到的线程正在等待锁，如果有，直接返回false。这保证了公平性：线程按照申请锁的顺序获取锁。

## lockInterruptibly 可响应中断获取锁

```java
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```

lockInterruptibly方法通过调用AQS提供的acquireInterruptibly方法实现：

```java
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
```

acquireInterruptibly方法首先检测一下中断，然后调用重写的tryAcquire方法试图获取锁，如果失败，调用doAcquireInterruptibly方法进行自旋、检测获取锁操作：

```java
private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
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

doAcquireInterruptibly方法与acquireQueued方法的区别在于：

- doAcquireInterruptibly方法将addWaiter的调用写在了方法里，而acquireQueued方法没有；
- doAcquireInterruptibly在当前线程从park中被中断唤醒时，直接抛出中断异常，而acquireQueued方法则是用一个局部变量记录下这次中断，但不立即处理，等到成功获取锁/共享资源之后，反馈给上层，由上层调用selfInterrupt方法“补上”这次中断。

这些区别与doAcquireSharedInterruptibly&doAcquireShared方法之间的区别一致。

## tryLock & tryLock(Timeout) 尝试获取锁

```java
public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
```

ReentrantLock提供了两种tryLock方法：限时&不限时。我们注意到，不限时（立即返回）的tryLock方法，不管在公平还是非公平模式下，调用的都是Sync中的nonfairTryAcquire方法。因此，如果在公平模式下调用tryLock，即使队列中有等待线程，也可能获取成功。

而限时（不立即返回）的tryLock(Timeout)方法则公国tryAcquireNanos提供了公平&非公平两种模式的tryLock(Timeout)操作：

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
```

可以看到，tryAcquireNanos方法通过调用不同的重写的tryAcquire方法提供了两种模式下的不同操作。tryAcquire方法已经分析过，不再赘述。这里重点关注doAcquireNanos方法：

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
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

可以看到，doAcquireNanos方法是立即响应中断的（事实上doAcquireSharedNanos方法也是立即响应中断的），即限时（不立即返回）的尝试获取的方法都是及时响应中断的，没有延迟处理中断的版本。

还有一点需要注意，当线程从park中被唤醒时，我们无法确定唤醒原因是被中断还是超时，因此需要检测一下中断标志。

## unlock 释放锁

公平&非公平模式的unlock操作是一致的：

```java
public void unlock() {
        sync.release(1);
    }
```

```java

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

```

tryRelease是在FairSync和NonfairSync的父类Sync中定义的，因此公平&非公平模式下的release操作是统一的。tryRelease方法首先检测当前线程是否持有锁，然后计算一下释放之后锁是否可用（计数值state是否等于0），如果可用，释放&设置持有锁线程为null&返回true，如果不可用，释放&返回返回false。

