---
layout: post
title: juc之Condition
date: 2020-11-22 11:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

Condition在JUC框架下提供了传统Java监视器风格的wait、notify和notifyAll相似的功能。

![DGpqBD.png](https://s3.ax1x.com/2020/11/22/DGpqBD.png)

Condition队列与Sync队列（锁等待队列）有几点不同：

- Condition队列是一个单向链表，而Sync队列是一个双向链表；
- Sync队列在初始化的时候，会在队列头部添加一个空的dummy节点，它不持有任何线程，而Condition队列初始化时，头结点就开始持有等待线程了。
- Condition永远都是一个公平锁（顺序）的实现

Condition必须被绑定到一个独占锁上使用。ReentrantLock中获取Condition的方法为：

```java
public Condition newCondition() {
    return sync.newCondition();
}

final ConditionObject newCondition() {
    return new ConditionObject();
}
```

直接初始化并返回了一个AQS提供的ConditionObject对象。因此，Condition实际上是AQS框架的内容。ConditionObject通过维护两个成员变量：

```java
   /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;
```

下面我们就来分析下Condition的工作流程。

## await 在条件变量上等待

分别是Condition队列的头结点和尾节点。Condition在调用await方法之前，必须先获取锁，注意，这个锁必须是一个独占锁。我们先来看一下await中用到的几个方法：

addConditionWaiter:

```java
private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

顾名思义，此方法在Condition队列中添加一个等待线程。首先，方法先检查一下队列尾节点是否还在等待Condition（如果被signal或者中断，waitStatus会被修改为0或者CANCELLED）。如果尾节点被取消或者中断，调用unlinkCancelledWaiters方法删除Condition队列中被cancel的节点。然后将当前线程封装在一个Node中，添加到Condition队列的尾部。这里由于我们在操纵Condition队列的时候已经获取了一个独占锁，因此不会发生竞争。

我们有必要在这里提一下Node对象中的nextWaiter成员、SHARED成员和EXCLUSIVE成员：

```java
/** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        Node nextWaiter;
```

nextWaiter在共享模式下，被设置为SHARED，SHARED为一个final的空节点，用来表示当前模式是共享模式；默认情况下nextWaiter是null，EXCLUSIVE成员是一个final的null，因此默认模式是独占模式。在Condition队列中nextWaiter被用来指向队列里的下一个等待线程。在一个线程从Condition队列中被移除之后，nextWaiter被设置为空（EXCLUSIVE）。这再次表明：Condition必须被绑定在一个独占锁上使用。

我们来看一下unlinkCancelledWaiters方法：

```java
private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

unlinkCancelledWaiters方法很简单，从头到尾遍历Condition队列，移除被cancel或被中断的节点。由于这里我们在操纵Condition队列的时候已经获取了所绑定的独占锁，因此不用担心竞争的发生。

我们再来看一下fullyRelease方法，这个方法用来释放锁：

```java
final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

方法首先获取了state的值，这个值表示可锁被“重入”深度，并调用release释放全部的重入获取，如果成功，返回这个深度，如果失败，要将当前线程的waitStatus设置为CANCELLED。

我们再来看一下isOnSyncQueue方法，这个方法返节点是否在Sync队列中等待锁：

```java
final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }
```

node从Condition队列移除的第一步，就是设置waitStatus为其他值，因此是否等于Node.CONDITON可以作为判断标志，如果等于，说明还在Condition队列中，即不再Sync队列里。在node被放入Sync队列时，第一步就是设置node的prev为当前获取到的尾节点，所以如果发现node的prev为null的话，可以确定node尚未被加入Sync队列。

相似的，node被放入Sync队列的最后一步是设置node的next，如果发现node的next不为null，说明已经完成了放入Sync队列的过程，因此可以返回true。

当我们执行完两个if而仍未返回时，node的prev一定不为null，next一定为null，这个时候可以认为node正处于放入Sync队列的执行CAS操作执行过程中。而这个CAS操作有可能失败，因此我们再给node一次机会，调用findNodeFromTail来检测：

```java
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
```

findNodeFromTail方法从尾部遍历Sync队列，如果检查node是否在队列中，如果还不在，此时node也许在CAS自旋中，在不久的将来可能会进到Sync队列里。但我们已经等不了了，直接放回false。

我们再来看一下checkInterruptWhileWaiting方法：

```java
/** Mode meaning to reinterrupt on exit from wait */
        private static final int REINTERRUPT =  1;
        /** Mode meaning to throw InterruptedException on exit from wait */
        private static final int THROW_IE    = -1;

        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
```

此方法在线程从park中醒来后调用，它的返回值有三种：0代表在park过程中没有发生中断；THORW_IE代表发生了中断，且在后续我们需要抛出中断异常；REINTERRUPT表示发生了中断，但在后续我们不抛出中断异常，而是“补上”这次中断。当没有发生中断时，我们返回0即可，当中断发生时，返回THROW_IE or REINTERRUPT由transferAfterCancelledWait方法判断：

```java
final boolean transferAfterCancelledWait(Node node) {
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
```

transferAfterCancelledWait方法并不在ConditionObject中定义，而是由AQS提供。这个方法根据是否中断发生时，是否有signal操作来“掺和”来返回结果。方法调用CAS操作将node的waitStatus从CONDITION设置为0，如果成功，说明当中断发生时，说明没有signal发生（signal的第一步是将node的waitStatus设置为0），在调用enq将线程放入Sync队列后直接返回true，表示中断先于signal发生，即中断在await等待过程中发生，根据await的语义，在遇到中断时需要抛出中断异常，返回true告诉上层方法返回THROW_IT，后续会根据这个返回值做抛出中断异常的处理。

如果CAS操作失败，是否说明中断后于signal发生呢？只能说这时候我们不能确定中断和signal到底谁先发生，只是在我们做CAS操作的时候，他们俩已经都发生了（中断->interrupted检测->signal->CAS，或者signal->中断->interrupted检测->CAS都有可能），这时候我们无法判断到底顺序是怎样，这里的处理是不管怎样都返回false告诉上层方法返回REINTERRUPT，当做是signal先发生（线程被signal唤醒）来处理，后续根据这个返回值做“补上”中断的处理。在返回false之前，我们要先做一下等待，直到当前线程被成功放入Sync锁等待队列。

因此，我们可以这样总结：transferAfterCancelledWait的返回值表示了线程是否因为中断从park中唤醒。

至此，我们终于可以正式来看await方法了：

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                // 有趣的是这里this是conditionobject，而非thread
                // 我试图debug代码，发现线程并不会stuck在这里
                // LockSupport.park(this)会神奇的做一个移出condition队列，入队sync队列的操作，然后正常跳出循环获取锁
                // 之后再dubug下去，发现代码并不能继续，而是又回到了await()方法里做与刚才同样的操作
                // 这里百思不得其解，求大神解答
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```

await方法是及时响应中断的。它首先检查了一下中断标志。然后调用addConditionWaiter将当前线程放入Condition队列的尾，并顺手清理了一下队列里的无用节点。紧接着调用fullyRelease方法释放当前线程持有的锁。然后是一个while循环，这个循环会循环检测线程的状态，直到线程被signal或者中断唤醒且被放入Sync锁等待队列。如果中断发生的话，还需要调用checkInterruptWhileWaiting方法，根据中断发生的时机确定后去处理这次中断的方式，如果发生中断，退出while循环。

退出while循环后，我们调用acquireQueued方法来获取锁，注意，acquireQueued方法的返回值表示在等待获取锁的过程中是否发生中断，如果发生中断 且 原来没有需要做抛出处理的中断发生时，我们将后续处理方式设置为REINTERRUPT（如果原来在await状态有中断发生，即interrruptMode==THROW_IE，依然保持THROW_IE）。

如果是应为中断从park中唤醒（interruptMode==THROT_IE），当前线程仍在Condition队列中，但waitStatus已经变成0了，这里在调用unlinkCancelledWaiters做一次清理。

最后，根据interruptMode的值，调用reportInterruptAfterWait做出相应处理：

```java

        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

```

如果interruptMod==0，donothing，如果是THROW_IE，说明在await状态下发生中断，抛出中断异常，如果是REINTERRUPT，说明是signal“掺和”了中断，我们无法分辨具体的先后顺序，于是统一按照先signal再中断来处理，即成功获取锁之后要调用selfInterrupt“补上”这次中断。

## awaitNanos 限时的在条件变量上等待

```java
public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }
```

awaitNanos方法与await方法大致相同，区别在于每次park是定时的，当被唤醒时，比较一下剩余等待时间Timeout与spinForTimeoutThreshold阈值的大小，如果小于，将不再park。

注意：当已经到达等待的deadline时，调用transferAfterCancelledWait方法，注意，此时可能发生中断（上次调用checkInterruptWhileWaiting之后被中断），再次的，我们无法判断这次中断与到时这两个的先后顺序，我们在这里的处理方式是直接忽略这次中断，统一认为是先到时后中断（体现在没有记录transferAfterCancelledWait方法的返回值），但在transferAfterCancelledWait方法中的处理是考虑了被中断的情况的，只不过这个中断标志位没有检测，留给后续来处理了。这个中断标志将会在调用acquireQueued方法并成功获取锁之后被检测并返回，最终影响interruptMode的值，并在reportInterruptAfterWait方法中被处理。可见，这次中断最终没有被遗漏，只是我们先处理的signal，回过头来再去处理它。

最后方法的返回值是拍唤醒后的剩余等待时间，这个时间可能小于0。

await(long time, TimeUnit unit)方法与awaitNanos方法十分类似，不再赘述。

## awaitUtil 指定结束时刻的在条件变量上等待

```java
public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```

awaitUtil方法在原理上与awaitNanos方法是也十分相似，只不过park操作调用的是LockSupportparkUtil方法，且没有spinForTimeoutThreshold阈值的应用。在返回值上也有些许差别：返回值timedout记录了transferAfterCancelledWait方法的返回值——线程是否因为中断从park中唤醒，如果是的话，表示还没有到等待的deadline。

### signal 唤醒Condition队列的头节点持有的线程

```java
public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```

调用signal之前也需要获取锁，因此signal方法首先检测了一下当前线程是否获取了独占锁。然后调用doSignal唤醒队列中第一个等待线程。注意，这里的“唤醒”意思是将线程从Condition队列移到Sync队列，表示已经完成Condition的等待，具有了去竞争锁的资格。至此，我们可以发现，由于await会直接把线程放入Condition等待队列的尾部，因此Condition是公平的，即按照入列的顺序来signal。

```java
private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

doSignal方法先将first节点从队列中摘下，然后调用transferForSignal去改变first节点的waitStatus（所谓唤醒线程），这个方法有可能失败，因为等待线程可能已经到时或者被中断，因此while循环这个操作直到成功唤醒或队列为空。我们来看下transferForSignal方法：

```java
final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

这个方法并不在ConditionObject中定义，而是由AQS提供。方法首先调用CAS操作修改node的waitStatus，如果失败，表示线程已经放弃等待（到时或被中断），直接返回false。如果成功，调用enq方法将它放入Sync锁等待队列，返回值p是node在Sync队列中的前驱节点。紧接着检测一下前驱p的waitStatus，如果发现不为SIGNAL，需要将node持有的线程（注意不是当前线程）unpark，这里必须搞清楚，node线程是在哪里park的，显然，他还在await方法的那个while循环里。unpark之后，node线程将会从while循环中退出，然后去调用acquireQueued方法，这个方法是一个自旋，弄得线程会在自旋过程中清除已经为CANCELLED状态的前驱，然后注册前驱节点的waitStatus为SIGNAL。

至此，signal方法已经完成了所有该做的，“唤醒”的线程已经成功加入Sync队列并已经参与锁的竞争了，返回true。

## signalAll 唤醒Condition队列的所有等待线程

```java
public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
```

signalAll方法同样先检测是否持有独占锁，然后对奥用doSignalAll方法：

```java
private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```
doSignalAll方法循环调用transferForSignal方法“唤醒”队列的头结点，直到队列为空。

总结：ConditionObject由AQS提供，它实现了类似wiat、notify和notifyAll类似的功能。Condition必须与一个独占锁绑定使用，在await或signal之前必须现持有独占锁。Condition队列是一个单向链表，他是公平的，按照先进先出的顺序从队列中被“唤醒”，所谓唤醒指的是完成Condition对象上的等待，被移到Sync锁等待队列中，有参与竞争锁的资格（Sync队列有公平&非公平两种模式，注意区别）。

------

加餐：

摘自  https://juejin.cn/post/6844903984197533704

## Thread.sleep()和Object.wait()的区别

- Thread.sleep()不会释放占有的锁，Object.wait()会释放占有的锁；
- Thread.sleep()必须传入时间，Object.wait()可传可不传，不传表示一直阻塞下去；
- Thread.sleep()到时间了会自动唤醒，然后继续执行；
- Object.wait()不带时间的，需要另一个线程使用Object.notify()唤醒；
- Object.wait()带时间的，假如没有被notify，到时间了会自动唤醒，这时又分好两种情况，一是立即获取到了锁，线程自然会继续执行；二是没有立即获取锁，线程进入同步队列等待获取锁；

## Thread.sleep()和Condition.await()的区别

这个题目的回答思路跟Object.wait()是基本一致的，不同的是Condition.await()底层是调用LockSupport.park()来实现阻塞当前线程的。

实际上，它在阻塞当前线程之前还干了两件事，一是把当前线程添加到条件队列中，二是“完全”释放锁，也就是让state状态变量变为0，然后才是调用LockSupport.park()阻塞当前线程。

## Thread.sleep()和LockSupport.park()的区别

- 从功能上来说，Thread.sleep()和LockSupport.park()方法类似，都是阻塞当前线程的执行，且都不会释放当前线程占有的锁资源；
- Thread.sleep()没法从外部唤醒，只能自己醒过来；
- LockSupport.park()方法可以被另一个线程调用LockSupport.unpark()方法唤醒；
- Thread.sleep()方法声明上抛出了InterruptedException中断异常，所以调用者需要捕获这个异常或者再抛出；
- LockSupport.park()方法不需要捕获中断异常；
- Thread.sleep()本身就是一个native方法；
- LockSupport.park()底层是调用的Unsafe的native方法；

## Object.wait()和LockSupport.park()的区别

- Object.wait()方法需要在synchronized块中执行；
- LockSupport.park()可以在任意地方执行；
- Object.wait()方法声明抛出了中断异常，调用者需要捕获或者再抛出；
- LockSupport.park()不需要捕获中断异常；
- Object.wait()不带超时的，需要另一个线程执行notify()来唤醒，但不一定继续执行后续内容；
- LockSupport.park()不带超时的，需要另一个线程执行unpark()来唤醒，一定会继续执行后续内容；
- 如果在wait()之前执行了notify()会怎样？抛出IllegalMonitorStateException异常；
- 如果在park()之前执行了unpark()会怎样？线程不会被阻塞，直接跳过park()，继续执行后续内容；

ps：LockSupport.park()不能重入，会死锁