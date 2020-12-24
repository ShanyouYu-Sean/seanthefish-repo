---
layout: post
title: juc之aqs
date: 2020-02-21 11:00:00
tags: 
- 并发
categories:
- 并发
---

今天说aqs，juc中的绝对核心。内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

AbstractQueuedSynchronizer（AQS）是一个同步器框架，抽象类。在实现锁的时候，一般会实现一个继承自AQS的内部类sync，作为我们的自定义同步器。AQS内部维护了一个state成员和一个队列。其中state标识了共享资源的状态，队列则记录了等待资源的线程。

以下这五个方法，在AQS中实现为直接抛出异常，这是我们自定义同步器需要重写的方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。

- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败返回false。

- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败返回false。

- tryAcquireShared(int)：共享方式。尝试获取资源。成功返回true，失败返回false。

- tryReleaseShared(int)：共享方式。尝试释放资源，成功则返回true，失败返回false。

例如ReentrantLock就是一种独占锁，CountDownLatch和Semaphore是共享锁。与CountDownLatch有一定相似性的CyclicBarrier并没有自己的共享同步器，而是使用Lock和Condition来实现的。

本质上aqs就是一个clh锁，AQS在CLH的基础上进行了变种：CLH是单向队列，其主要特点是自旋检查前驱节点的locked状态。而AQS同步队列是双向队列，每个节点也有状态waitStatus，而其并不是一直对前驱节点的状态自旋，在尝试自旋一次后会将线程阻塞让出CPU时间片，等到占有锁的线程主动唤醒后续节点的继续自旋。

     * <pre>
     *      +------+  prev +-----+       +-----+
     * head |      | <---- |     | <---- |     |  tail
     *      +------+       +-----+       +-----+
     * </pre>

![D8ZfXR.png](https://s3.ax1x.com/2020/11/22/D8ZfXR.png)

每个线程都会包装成一个node，从队列尾部入队，头节点占有锁的线程（一个虚节点），当前线程会通知检查其前驱节点，看其是否是头节点，如果是就自旋去获取锁（线程可能休息，但自旋会一直进行），其他节点都可以安全阻塞等待被唤醒。

## 独占模式

下面是一个简单的独占锁的实现，它是不可重入的。它重写了AQS的tryAcquire方法和tryRelease方法：

```java
class Mutex implements Lock, Serializable {
    //自定义同步器，继承自AQS
   private static class Sync extends AbstractQueuedSynchronizer {
     //试图获取锁，当state为0时能成功获取，
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; //这是一个对于state进行操作的量，含义自定义
       if (compareAndSetState(0, 1)) {    //注意：这是一个原子操作
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }
     //释放锁，此时state应为1，Mutex处于被独占状态
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }
     //返回一个Condition
     Condition newCondition() { return new ConditionObject(); }
   }

   private final Sync sync = new Sync();

   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```

### acquire 获取锁

我们先来看一下Mutex重写的tryAcquire方法：

```java
//试图获取锁，当state为0时能成功获取，
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; //这是一个对于state进行操作的量，含义自定义
       if (compareAndSetState(0, 1)) {    //注意：这是一个原子操作
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }
```

注意：当我们初始化一个Sync的时候，如果没有指定state的初值（无参数），那么state的默认初值是0。可以看到，方法开头首先有一个断言acquires==1，参数acquires代表要在state上做的改变的量（减去或增加），在Mutex中，我们定义state只有两个状态：0或1，0代表共享资源可以被获取，1表示共享资源正在被占用，因此Mutex是不可重入的。实际上，自定义同步器通过重写tryAcquire和tryRelease来定义state代表的意义和资源的共享方式，这是同步器的主要任务。Mutex的tryAcquire使用一个原子操作compareAndSetState来试图获取资源，这个原子操作由上层的AQS提供，如果成功，将当前线程设置为独占线程并返回true。

Mutex的lock方法调用了aqs的acquire方法，acquire方法实现为：

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

它首先调用tryAquire去获取共享资源，如果失败，调用addWaiter将当前线程放入等待队列，返回持有当前线程的Node对象，然后调用acquireQueued方法来监视等待队列并获取资源。acquireQueued方法会阻塞线程，直到成功获取。注意，acquire方法不能及时响应中断，只能在成功获取锁之后，再来处理。中断当前线程的操作跑出的异常在acquireQueued方法中被捕获，外部调用者没能看到这个异常，因此调用selfInterrupt来重置中断标识。

我们需要详细了解addWaiter方法和acquireQueued方法，之后再来回顾acquire的过程，才能对整个获取锁的流程有比较详细的了解。

（ps：enqueue：入队，简称enq；dequeue：出队，简称dnq）

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

addWaiter首先将当前线程包装在一个Node对象node中，然后获取了一下队列的尾节点，如果队列不为空（tail不为null）的话，调用一个CAS函数试图将node放入等待队列的尾部，注意，此时可能发生竞争，如果有另外一个线程在两个if之间抢先更新的队列的尾节点，CAS操作将会失败，这时会调用enq方法，继续试图将node放入队列：

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

enq方法会循环检测队列，如果队列为空，则调用CAS函数初始化队列（此时node==head==tail），否则调用CAS函数将node放入队列尾。

请注意，初始化的头结点并不是当前线程节点，而是调用了无参构造函数的节点。如果经历了初始化或者并发导致队列中有元素，则与之前的方法相同。其实，addWaiter就是一个在双端链表添加尾节点的操作，需要注意的是，双端链表的头结点是一个无参构造函数的头结点。头节点虚节点。

如果CAS失败，enq会继续循环检测，直到成功将node入列。这里便是入队操作中的CAS自旋，因为这里还没有涉及锁的竞争，所以不需要响应中断。这里有一个隐含的知识点，即tail是一个volatile成员，确保某个线程更新队列后对其他线程的可见性。

注意：队列为空的时候，第一个线程进入队列的情况有点tricky：第一个发现队列为空并初始化队列（head节点）的线程不一定优先拿到资源。head节点被初始化后，当前线程需要下一次旋转才有机会进入队列，在这期间，完全有可能半路杀出程咬金，将当前线程与它初始化出的head节点无情分开。我们来总结一下，当队列只有一个节点时（head=tail），有两种情况：第一种是这个队列刚刚被初始化，head并没有持有任何线程对象。这个状态不会持续太久，初始化队列的线程有很大机会在下次自旋时把自己接到队尾。第二种情况是，所有等待线程都已经获得资源并继续执行下去了，队列仅有的节点是最后一个获取共享资源的线程，等到下一个线程到达等待队列并将它踢出队列之后，它才有机会被回收。

enq执行完毕，我们已经成功把当前线程放入等待队列，接下来的任务就是监视队列，等待获取资源。这个过程由acquireQueued方法实现：

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
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

acquireQueued方法是一个很重要的方法，在分析这个方法之前，我们先来说一下AQS中的那个等待队列。这个队列实际上是一个CLH队列，它保证了竞争资源的线程按到达顺序来获取资源，避免了饥饿的发生。CLH队列的工作过程，就是acquireQueued方法的工作过程。很明显，这又是一个自旋。首先，我们调用predecessor方法获取当前线程的前驱节点，如果这个前驱是head节点，就紧接着调用tryAcquire去获取共享资源，当然这是有可能失败的，因为head节点可能刚刚“上位”，持有锁的线程还没有释放资源。如果很幸运，我们拿到了资源，就调用setHead将node设置为队列的头结点，setHead方法同时会将node的prev置为null，并把thread置为null（前驱节点出队，当前虚节点head占有锁），紧接着将原先head的next也置为null，显然这是为了让其后续被回收。注意：acquireQueued方法在自旋过程中是不可被中断的，当然它会检测到中断（在parkAndCheckInterrupt方法中检测中断标志），但并不会因此结束自旋，只能在获得资源退出方法后，反馈给上层的方法：我刚刚被中断了。还记得acquire方法中的selfInterrupt的调用吗，就是为了“补上”这里没有响应的中断。

好，我们继续往下。获取资源失败后，调用shouldParkAfterFailedAcquire方法检测是否该去“休息”下，毕竟一直自旋很累嘛。如果可以休息就调用parkAndCheckInterrupt放心去休息。我们先来看一下shuldParkAfterFailedAcquire：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

我们首先了解一下waitStatus。Node对象维护了一个int成员waitStatus，他的可能取值如下：

```java
// 因为超时或者中断，结点会被设置为取消状态，被取消状态的结点不应该去竞争锁，只能保持取消状态不变，不能转换为其他状态。
// 处于这种状态的结点会被踢出队列，被GC回收；
static final int CANCELLED =  1;
// 表示这个结点的继任结点被阻塞了，到时需要通知它；
static final int SIGNAL    = -1;
// 表示这个结点在条件队列中，因为等待某个条件而被阻塞；
static final int CONDITION = -2;
// 使用在共享模式头结点有可能处于这种状态，表示锁的下一次获取可以无条件传播；
static final int PROPAGATE = -3;
// 0：None of the above，新结点会处于这种状态。
```

在我们的Mutex的例子中，节点的waitStatus只可能有CANCELLED、SIGNAL和0三中状态（事实上，独占模式下所有不使用Condition的同步器都是这样）。

我们继续来分析shouldParkAfterFailedAcquire方法：

首先检测下node的前驱节点pred，如果pred状态已经被置为SIGNAL，直接返回true。否则，从node的前驱继续往前找，直到找到一个waitStatus小于等于0的节点，设置该点为node的前驱（注意：此时node与这个节点之间的节点从等待队列中被“摘下”，等待被回收了）并返回false。

返回之后，上层的acquireQueued方法继续自旋，再次进入shouldParkAfterFailedAcquire方法之后，如果发现node前驱不是取消状态且waitStatus不等于SIGNAL，调用CAS函数进行注册（从ws->SIGNAL，标注为可以休息，等待被唤醒）。注意：这个操作可能失败，因此不能直接返回true，而是返回false由上层的自旋再次调用shouldParkAfterFailedAcquire直到确认注册成功。

历尽曲折，我们终于可以安心休息了：

```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

parkAndCheckInterrupt方法十分简单，他调用LockSupport的静态方法park阻塞当前线程，直到被中断，这次中断会被acquireQueued记录，但不会立即响应，直到自旋完成。注意：返回操作中的interrupted方法会将中断标志复位，因此我们在上层需要将这个中断“补上”，再一次：还记得大明湖边的selfInterrupt吗？

注意aqs的线程中断：

- 当中断线程被唤醒时，并不知道被唤醒的原因，可能是当前线程在等待中被中断，也可能是释放了锁以后被唤醒。因此我们通过Thread.interrupted()方法检查中断标记（该方法返回了当前线程的中断状态，并将当前线程的中断标识设置为False），并记录下来，如果发现该线程被中断过，就再中断一次。

- 线程在等待资源的过程中被唤醒，唤醒后还是会不断地去尝试获取锁，直到抢到锁为止。也就是说，在整个流程中，并不响应中断，只是记录中断记录。最后抢到锁返回了，那么如果被中断过的话，就需要补充一次中断。

### release 释放锁

我们先来看一下Mutex中重写的tryRelease方法：

```java
//释放锁，此时state应为1，Mutex处于被独占状态
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }
```

逻辑比较简单，首先将独占线程置为null，紧接着将state设置为0，这里不会发生资源竞争，因此不需要用CAS去设置state值，直接置0即可。

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

好，我们开始分析release方法。首先调用tryRelease试图释放共享资源，紧接着检测head的waitStatus是否为SIGNAL，如果是的话，调用unparkSuccessor唤醒队列中的head。独占模式下，waitStatus！=0与waitStatus==-1等价（这里waitStatus不会为CANCELLED，因为已经获取资源了）。如果不为SIGNAL，说明如果有下个等待线程，它正在自旋。所以直接返回true即可。我们来看下unparkSuccessor方法：

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

unparkSuccessor方法也会在共享模式的工作流程中被调用，因此方法开始做的判断是有必要的。对于独占模式而言，ws应该都是0。然后找到下一个需要被唤醒的线程并调用LockSupport的静态方法unpark唤醒等待线程。

至此，我们比较详细地了解了acquire&release的工作流程。

## 共享模式

### acquireShared 获取锁

下面，我们来学习下共享模式下的获取&释放锁的工作流程。

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```

acquireShared方法首先调用tryAcquireShared试图获取共享资源。tryAcquireShared的返回值表示剩余资源个数，负值表示获取失败，0表示获取成功但已无剩余资源。如果获取失败，调用doAcquireShared方法完成独占模式下类似的操作，后面我们会详细分析。注意，doAcquireShared方法在等待资源的过程中也是不响应中断的，它能觉察到中断，但在成功获取资源之前不会处理。

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

doAcquireShared方法与acquireQueued方法相似，不同的地方在于，共享模式下成功获取资源并将head指向自己之后，要检查并试图唤醒之后的等待线程。因为共享资源可能剩余，可以被后面的等待线程获取。

```java
private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

setHeadAndPropagate中有一个长长的if，来判断是否应该去试图唤醒后面的线程。propagate大于0，表示尚有资源可被获取，显然应该继续判断；h == null, 说明没有节点；而当h.waitStatus小于0时，它有两种取值可能，SIGNAL和PROPAGATE，我们将在后面看到，这几种情况都是应该继续判断。后续是对node的后继进行的判断，注意，node此时可能已经不是head节点了，因为这是共享模式，所以可能有一个node的后继成功获取资源后，把自己设为head，将node踢出了队列。这种情况下node的后继s是可能为null的，但貌似这种情况doReleaseShared的调用没有意义。s.isShared的判断主要是考虑到读写锁的情况，在读写锁的使用过程中，申请写锁（独占模式）和申请读锁（共享模式）的线程可能同时存在，这个判断发现后即线程是共享模式的时候，调用doReleaseShared方法唤醒他。

```java
private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
又是一个自旋。我们首先获取head节点h，然后检查它的waitStatus是否为SIGNAL，如果是的话，调用CAS将h的waitStatus设置为0，并调用unparkSuccessor唤醒下一个等待线程。注意，这里调用CAS方法而不是直接赋值，是因为在共享模式下，这里可能发生竞争。doReleaseShared方法可能由head节点在使用完共享资源后主动调用（后续在releaseShared方法中可以看到），也可能由刚刚“上位”的等待线程调用，在上位之后，原来的head线程已被踢出队列。

因此，doReleaseShared方法的执行情况变得比较复杂，需要细致分析。

第一种情况，只有刚刚释放资源的head线程调用，这时候没有竞争，waitStatus是SIGNAL，就去唤醒下个线程，是0，就重置为PROPAGATE。

第二种情况，刚刚释放完资源的旧head，和刚刚上位的新head同时调用doReleaseShared方法，这时候最新的head获取的都是自己，若干被踢出的旧head获取的可能是旧head，也可能是新head，这些被踢出的旧head线程也在根据自己获取的head（不管新旧）的状态进行CAS操作和unparkSuccessor操作，幸运的是，这些操作不会造成错误，只是多了一些唤醒而已（这些唤醒可能导致一个线程获得资源，也可能是一个“虚晃”）。

我们可以发现，不管head引用怎样更迭，最终新head的waitStatus都会被顺利处理。注意，可能有多个旧head同时参与这个过程，都不影响正确性。

我们注意到，一个新head，在他刚上位的时候有机会调用一次setHeadAndPropagate进而调用doReleaseShared，在他释放资源之后，又一次调用doReleaseShared（这次是必然的）。第一次调用时，不管新head的waitStatus是0还是SIGNAL，最终状态都被PROPAGATE（当然，被踢出队列的head可能还没来得及设置成PROPAGATE，但新上位的head最终会被设置），这也符合PROPAGATE的语义：使用在共享模式头结点有可能处于这种状态，表示锁的下一次获取可以无条件传播。

还有一个问题，它是由SIGNAL-->0-->PROPAGATE变化而来的，为什么不是SIGNAL-->PROPAGA这样直接变化呢？原因是unparkSuccessor方法会试图将当前node的waitStatus复位成0，如果我们直接SIGNAL-->PROPAGA后，那么又被复位成0，还需要一次CAS操作置为PROPAGATE。

### releaseShared 释放锁

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

我们可以看到，调用tryReleaseShared成功释放共享资源之后，最终要再次调用doReleaseShared试图唤醒后面的等待线程。

-----

之前自己实现过一个分布式锁：https://github.com/IBM/distributed-lock-spring-boot-starter

redis实现的，具体怎么实现以后会单独开文章说。

这里主要说下aqs：

单实例的aqs，头节点永远是占有锁的节点，所以它的后驱节点永远在自旋。其他节点都可以放心休息。
而多实例aqs，肯定会有实例的等待队列的头节点得不到锁，从而错误的进入了休息，这样这个队列就永远不会被唤醒。
因此，要解决问题，只需要保持每个实例的队列的头节点即使不是占有锁的节点，也得保证其后驱节点永远在自旋即可。
简单改一下aqs中的一行就能实现：
```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null;
                    failed = false;
                    return interrupted;
                }
                // 就是这一行
                if (p != head && shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()) {
                    interrupted = true;
                }
            }
        } finally {
            if (failed) {
                cancelAcquire(node);
            }
        }
    }
```