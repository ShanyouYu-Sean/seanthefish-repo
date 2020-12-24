---
layout: post
title: juc之ReentrantReadWriteLock
date: 2020-05-22 12:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

ReentrantLock提供了标准的互斥操作，但在应用中，我们对一个资源的访问有两种方式：读和写，读操作一般不会影响数据的一致性问题。但如果我们使用ReentrantLock，则在需要在读操作的时候也独占锁，这会导致并发效率大大降低。JUC包提供了读写锁ReentrantReadWriteLock，使得读写锁分离，在上述情境下，应用读写锁相对于使用独占锁，并发性能得到较大提高。

我们先来大致了解一下ReentrantReadWriteLock的性质：

- 基本性质：读锁是一个共享锁，写锁是一个独占锁。读锁能同时被多个线程获取，写锁只能被一个线程获取。读锁和写锁不能同时存在。

- 重入性：一个线程可以多次重复获取读锁和写锁。

- 锁降级：一个线程在已经获取写锁的情况下，可以再次获取读锁，如果线程又释放了写锁，就完成了一次锁降级。

- 锁升级：ReentrantReadWriteLock不支持锁升级。一个线程在获取读锁的情况下，如果试图去获取写锁，将会导致死锁（后面会详细说明）。

- 获取锁中断：提供了可中断的lock方法。

- 重入数：读锁和写锁的重入上限为65535（所有线程获取的锁的总数，为什么是这个值后面会详细说明）。

- 公平性：ReentrantReadWriteLock提供了公平&非公平两种工作模式。

ReentrantReadWriteLock实现了ReadWriteLock接口：

```java
public interface ReadWriteLock {  
    Lock readLock();  
    Lock writeLock();  
}
```

这个接口之有两个方法，分别返回读锁和写锁。ReentrantReadWriteLock定义了两个内部类：readLock&writeLock。

ReentrantReadWriteLock提供了两种自定义的同步器：FairSync&NonfairSync：

```java
　　static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }

    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

他们都继承自父类同步器Sync，而他们只定义了writerShouldBlock&readerShouldBlock方法。这两个方法用在获取锁的操作中，表示要获取锁的线程需要到等待队列中，还是可以直接尝试获取。后面我们会详细分析。

在自定义的同步器Sync中，定义了锁数量的记录方式:

```java
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

可见，ReentrantReadWriteLock用一个32位无符号数记录锁的数量，高16位记录共享锁（读锁）的数量，第16位记录独占锁（写锁）的数量，因此锁的数量上限都是65535。

## 写锁 

### lock 获取写锁

```java
public void lock() {
            sync.acquire(1);
        }
```
acquire方法不再赘述。这里重点关注自定义同步器Sync重写的tryAcquire方法： 

```java
protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
首先调用获取了一下state值，然后调用exclusiveCount方法获取当前写锁的数量。

然后做了一个判断，当c！=0时：如果w==0（即读锁的数量！=0），直接返回false。因为我们前面已经说过，读锁和写锁不能同时存在。当c！=0且W！=0的时候，有写锁存在，如果写锁不是由当前线程持有（注意，写锁是独占锁，只能由一个线程持有），直接返回false。如果是当前线程持有写锁，说明当前线程正在试图“重入”写锁。调用setState更新status值。注意，由于写锁是独占锁，因此执行到setState这一步时不可能出现竞争，因此不用调用CAS操作，直接setState即可。

注意：如果一个线程在持有读锁的情况下去申请写锁（试图锁升级），会导致死锁。tryAcquire在这种情况下返回false，AQS的acquire方法会将当前线程放入等待队列去等待写锁，在获取写锁之前不会释放锁持有的读锁，而读锁和写锁不能同时存在，发生死锁，他将永远不能获取这个写锁，其他线程也不能获取写锁，但读锁可被正常获取，只是永远不能获取写锁了。

如果c==0时，说明不存在任何锁。调用writerShouldBlock方法判断一下此时线程是否应该进入等待队列。注意：公平模式&非公平模式下的writerShouldBlock是不同的，非公平模式下，writerShouldBlock方法直接返回false，这也符合非公平的语义：

```java
 final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
```

而公平模式下，则调用方法，判断下等待队列中，当前线程之前是否有其他线程正在等待：

```java
final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }

```

注意，如果有，那么我们当时获取status的值的时候，这些线程还没来得及更改status值（因为我们当时获取的status为0），原因可能是应为刚到，或者刚被唤醒，在自旋中，还没有成功获取锁。 

```java
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

返回true必须满足两个条件：

- 队列非空
- 第一个等待线程（head.next）为空 或 不为空但不是当前线程。head.next为空的情形是：在我们获取head之后，head就被队列中下一个等待线程线程踢出队列了，next被置为空，那么踢他出去的这个线程一定不是当前线程，说明有其他线程等待在队列中。

我们回到tryAcquire方法中，当发现writerShouldBlock为true，或者writerShouldBlock为false但在CAS操作中失败时（由于这里的获取写锁不是重入，因此可能有多个线程同时竞争写锁），返回false。如果CAS成功，则调用setExclusiveOwnerThread将当前持有写锁的线程设置为当前线程。

### release 释放写锁

```java
public void unlock() {
            sync.release(1);
        }
```

与ReentrantLock一样，unlock方法调用AQS提供的release方法,这里重点关注自定义同步器Sync重写的tryRelease方法：

```java
protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```

首先，我们需要清楚一点：tryRelease方法的返回值表示当前释放操作完成后，剩余写锁数量是否等于0（即完成此释放后，写锁是否可用）。这与同样是可重入的ReentrantLock的tryRelease方法一样，ReentrantLock的tryRelease方法返回值的意义也是剩余写锁数量是否等于0（即完成此释放后，写锁是否可用）。

### tryLock 获取写锁

```java
public boolean tryLock( ) {
            return sync.tryWriteLock();
        }
```

WriteLock的tryLock方法调用自定义同步器Sync的tryWriteLock方法实现：

```java
final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

tryWriteLock方法看上去跟tryAcquire方法真的很像。唯一的区别在于，tryWriteLock忽略的writerShouldBlock方法，即，默认调用tryLock方法的时机，就是需要我们去“抢”写锁的时机。

## 读锁

### lock 获取读锁

```java
public void lock() {
            sync.acquireShared(1);
        }
```
ReadLock的lock方法调用AQS提供的acquireShared方法来实现, 我们重点关注自定义同步器Sync重写的tryAcquireShared方法： 
```java
protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.
             * 2. Otherwise, this thread is eligible for
             *    lock wrt state, so ask if it should block
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }
```

方法首先检测了一下当前是否有其他线程持有写锁，如果是的话，直接返回-1，表示获取失败。后续AQS的acquireShared方法会将当前线程放入等待队列中。

然后方法做了这样一个判断，如果当前线程可以直接参与竞争读锁的话，就调用CAS操作将status值加一个SHARED_UNIT，注意，这里不是加1, 是因为status的高16位代表读锁的数量。

OK，我们必须在这里暂停一下，我们需要详细解释一下几个成员变量：

```java
static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    final long tid = getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}

private transient ThreadLocalHoldCounter readHolds;
private transient HoldCounter cachedHoldCounter;

private transient Thread firstReader = null;
private transient int firstReaderHoldCount;
```

HoldCounter是一个final的内部类，有两个成员：tid&count，分别代表一个线程ID和线程对应的一个计数值。

ThreadLocalHoldCounter是一个final的内部类，它继承自`ThreadLocal<HoldCounter>`，它重写了initialValue方法，ThreadLocalHoldCounter对象对某一个线程第一次调用get方法是，会调用initialValue方法初始化这个线程响应的本地变量，并加入到map中。

readHolds存在的作用是：记录所有持有读锁的线程所持有读锁的数量。对于写锁来说，它是独占锁，我们可以通过status的低16位+独占写锁的线程来记录关于写锁的所有信息，即它被谁持有&被重入的数量。而读锁是一个共享锁，任何线程都可能持有它，因此，我们必须对每个线程都记录一下它所持有的共享锁（读锁）的数量。本地变量ThreadLocal来实现这个记录是非常合适的。

cachedHoldCounter是一个缓存。很多情况下，一个线程获取读锁之后要更新一下它对应的记录值（线程对应的HoldCounter对象），然后有很大可能在很短的时间内就释放掉读锁，这时候需要再次更新HoldCounter，甚至需要从readHolds中删除（如果重入的读锁都被释放掉的话），需要调用readHolds的get方法，这是有一定开销的。因此，设置cachedHoldCounter作为一个缓存，在某个线程需要这个记录值的时候，先检查cachedHoldCounter对应的线程是否是这个线程自己，如果不是的话，再熊readHolds中get出来，这提高了效率。

firsReader&firstReaderHoldCount，这两个值记录了第一个获取读锁的线程和它持有的读锁的数量（可重入的嘛），这两个值在读锁全部释放之后要清空，以便记录下一次首先获取读锁的线程和其锁数目。这两个值存在的意义是：很多时候，读锁只被一个线程获取，这时候我们规定，第一个获取读锁的线程的计数不放入readHolds中，而是单独用这两个计数值来记录，这就避免了当只有一个线程操作读锁的时候，频繁地在readHolds上读取，提高了效率。

注意区别：

- cachedHoldCounter提高的是一个线程获取-释放之间没有其他线程来获取或释放锁时的效率；
- firsReader&firstReaderHoldCount提高的是只有一个线程操作锁时的效率。

这时候我们再回到tryAcquireShared方法，当CAS操作成功后，需要去更新刚刚说过的计数值。具体细节代码已经很清楚，不再赘述。

如果CAS失败或readerShouldBlock方法返回true，我们调用fullTryAcquireShared方法继续试图获取读锁。fullTryAcquireShared方法是tryAcquireShared方法的完整版，或者叫升级版，它处理了CAS失败的情况和readerShouldBlock返回true的情况。

在分析fullTryAcquireShared方法之前，我们先来看一下readerShouldBlock方法：

在公平模式下，根据等待队列中在当前线程之前有没有等待线程来判断：

```java
 final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
```

而在非公平模式下：

```java
  final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
```

调用了apparentlyFirstQueuedIsExclusive方法：

```java
final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

这个方法返回是否队列的head.next正在等待独占锁（写锁）。当然这个方法执行的过程中队列的形态可能发生变化。这个方法的意思是：读锁不应该让写锁始终等待。

好了，我们现在来看fullTryAcquireShared方法：

```java
/**
         * Full version of acquire for reads, that handles CAS misses
         * and reentrant reads not dealt with in tryAcquireShared.
         */
        final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }
```

我们可以看到：fullTryAcquireShared方法是tryAcquireShared方法的完整版，或者叫升级版，它处理了CAS失败的情况和readerShouldBlock返回true的情况。

跟tryAcquireShared方法一样，首先检查是否有其他线程正在持有写锁，如果是，直接返回false。如果没有线程正在持有写锁，则调用readerShouldBlock检测当前线程是否应该进入等待队列。就算readerShouldBlock方法返回true，原因可能因为当前是公平模式或者队列的第一个等待线程（head.next）正在等待写锁，我们也不能直接返回false，因为返回false意味着当前线程将要进入等待队列（见AQS的acquireShared方法），原因是：

- 如果当前线程正在持有读锁，且这次读锁的重入被放入等待队列，万一之前队列中有线程正在等待写锁，将会导致死锁；
- 另一种情况是当前线程正在持有写锁，且这次读锁的“降级申请”被放入等待队列，如果队列中之前有线程正在等待锁，不论等待的是写锁还是读锁，都将导致死锁。

因此，我们需要做一个判断，如果这次申请读锁是对读锁的一次重入（因为我们已经检测过没有写锁，因此只考虑上述第①种情况），我们将不能返回false（返回false意味着进队列），而是调用CAS操作去获取读锁，如果CAS失败，则一直自旋，直到成功获取，或者可以返回false去队列的时机的到来。

我们可以这样提fullTryAcquireShared方法说句话：不是我不想进队列休息，实在是因为进队列有可能死锁，所以我才一直自旋！

注意：判断重入的时候firstReader==当前线程即说明是一次重入，因为firstReader线程释放最后一个读锁的时候会将firstReader置为null，这里还不是null，说明依然持有读锁。

另外还记得我们提过apparentlyFirstQueuedIsExclusive方法是不可靠的吗，它在检测的过程中队列结构可能被更改，head可能被踢出，方法可能因为head.next为null而返回false。而且它也只是检测第一个等待线程（head.next），如果有等待写锁的线程在后面，它也不能检测出来。不过没关系，这些都导致它返回false，返回false意味着fullTryAcquireShared可以去抢“锁”并不会影响正确性。

### unlock 释放读锁

```java
public void unlock() {
            sync.releaseShared(1);
        }
```

readLock的unlock方法调用AQS提供的releaseShared方法实现, 这里我们关注自定义同步器Sync重写的tryReleaseShared方法：

```java
protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```

分为三部分：

- 如果是firstReader，对firstReader修改；②
- 如果不是firstReader，修改readHolds；
- CAS自旋更新status值。

注意：tryReleaseShared方法的返回值如果为true，表示status为0，即已经不存在任何锁，both读锁&写锁。

### tryLock 获取读锁

```java
public boolean tryLock() {
            return sync.tryReadLock();
        }
```
ReadLock的tryLock调用自定义同步器Sync的tryReadLock方法实现：

```java
final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }
```

与写锁的tryWriteLock方法类似，tryReadLock同样忽略了readerShouldBlock方法，因为调用这个方法就意味着：现在是适合抢占的时机。

tryReadLock方法与tryAcquireShared方法十分类似，不同在于：当CAS失败时，tryAcquireShared方法调用fullAcquireShared处理CAS失败，而tryReadLock方法遇到CAS失败时，直接返回false，毕竟只是try嘛。

 

总结：

ReentrantReadWriteLock相比于其他锁，还是比较复杂的，因为他结合了共享锁和独占锁，并混合使用了他们。虽然ReentrantReadWriteLock通过精巧的设计尽量避免死锁的发生，但如果我们使用不当仍然可能发生死锁，比如我们在持有读锁的情况下去申请写，企图做锁升级。