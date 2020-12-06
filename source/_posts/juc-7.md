---
layout: post
title: juc之CountDownLatch
date: 2020-11-22 14:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

## CountDownLatch是什么

闭锁，CountDownLatch能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

可见CountDownLatch跟之前我们所有的juc对象都有着本质性区别，之前的都是线程能拿到资源，也可以归还资源，CountDownLatch则是只能拿，不能还，而且在资源花完之前，父线程会一直阻塞。

例子：每天起早贪黑的上班，父母每天也要上班，话说今天定了个饭店，一家人一起吃个饭，通知大家下班去饭店集合。假设：3个人在不同的地方上班，必须等到3个人到场才能吃饭，用程序如何实现呢？

```java
public static void main(String[] args) throws InterruptedException{
 
		new Thread()
		{
			public void run()
			{
				fatherToRes();
				latch.countDown();
			};
		}.start();
		new Thread()
		{
			public void run()
			{
				motherToRes();
				latch.countDown();
			};
		}.start();
		new Thread()
		{
			public void run()
			{
				meToRes();
				latch.countDown();
			};
		}.start();
 
		latch.await();
		togetherToEat();
	}

```

CountDownLatch就是一个使用共享模式的自定义同步器实现的共享锁。

## await方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

CountDownLatch提供了两种await方法：有等待时长限制和一直等待。两种均能响应中断（归根到底是UNSAFE.park可响应中断。但是如果是定时的park，则不能判断被唤醒的原因是超时还是被中断，因此需要isInterrupted判断下，而此方法会清除中断标志，因此如果是延迟处理要“补上”）。

await（）方法调用了同步器的acquireSharedInterruptibly方法，这个方法由上层AQS提供，它调用了我们重写的tryAcquireShared方法而封装了排队等待、唤醒、响应中断的细节，我们只关注自定义同步器中的tryAcquireShared方法即可：

```java
protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
```

注意，tryAcquireShared方法的返回值的意义在AQS是这样规定的：负值代表获取资源失败，非负值代表成功获取资源后剩余资源的数量。而这里当getState返回值为0的时候，我们却总是返回1，表示仍有剩余资源。这看上去并不合理，但这确实是正确的：因为可能有多个线程调用了await，同时在队列中等待资源，CountDownLatch的语义要求我们在倒计时结束有唤醒所有等待线程。因此我们在成功获取资源后，总是要告诉AQS“还有剩余”，这样AQS便会继续唤醒队列中的其他等待线程（由AQS中的setHeadAndPropagate方法调用doReleaseShared来唤醒）。一句话：成功获取总返回1是为了保证唤醒的“延续性”。

有等待时长限制的await(long, TimeUnit)方法调用了同步器的tryAcquireSharedNanos方法：

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
```

这个方法首先检测中断，然后试图获取，失败后进入“自旋-等待”阶段，直到成功获取或被中断。这是AQS的内容，不再赘述。

## countDown方法

```java
 public void countDown() {
        sync.releaseShared(1);
    }
```

countDown方法调用releaseShared释放资源：

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

releaseShared会调用tryReleaseShared方法：

```java
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

方法一直自旋，直到成功释放或倒计时完毕。因为可能有超过count的线程调用countDown，因此releaseShared是可能失败的。当然在释放过程中也可能发生竞争，CAS自旋保证竞争发生时的正确执行。

## 总结

CountDownLatch是一个共享锁，但有些特别：他在初始化的时候锁住了所有共享资源，任何线程都可以调用countDown方法释放一个资源，当所有资源都被释放后，所有等待线程被唤醒。从而实现了倒计时的效果。

CountDownLatch是一次性的，计数值不可恢复。