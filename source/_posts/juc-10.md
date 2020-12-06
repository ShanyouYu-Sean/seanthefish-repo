---
layout: post
title: juc之ArrayBlockingQueue
date: 2020-11-24 11:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自 https://juejin.cn/post/6844903989788540941 稍有改动

## BlockingQueue 介绍

BlockingQueue 继承自 Queue 接口,下面看看阻塞队列提供的接口；

```java
public interface BlockingQueue<E> extends Queue<E> {
    /**
     * 插入数据到队列尾部（如果立即可行且不会超过该队列的容量）
     * 在成功时返回 true，如果此队列已满，则抛IllegalStateException。(与offer方法的区别)
     */
    boolean add(E e);

    /**
     * 插入数据到队列尾部，如果没有空间，直接返回false;
     * 有空间直接插入，返回true。
     */
    boolean offer(E e);

    /**
     * 插入数据到队列尾部，如果队列没有空间，一直阻塞；
     * 有空间直接插入。
     */
    void put(E e) throws InterruptedException;

    /**
     * 插入数据到队列尾部，如果没有额外的空间，等待一定的时间，有空间即插入，返回true，
     * 到时间了，还是没有额外空间，返回false。
     */
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 取出和删除队列中的头元素，如果没有数据，会一直阻塞到有数据
     */
    E take() throws InterruptedException;

    /**
     * 取出和删除队列中的头元素，如果没有数据，需要会阻塞一定的时间，过期了还没有数据，返回null
     */
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;
    
    //除了上述方法还有继承自Queue接口的方法 
    /**
     * 取出和删除队列头元素，如果是空队列直接返回null。
     */
    E poll();

    /**
     * 取出但不删除头元素，该方法与peek方法的区别是当队列为空时会抛出NoSuchElementException异常
     */
    E element();

    /**
     * 取出但不删除头元素，空队列直接返回null
     */
    E peek();

    /**
     * 返回队列总额外的空间
     */
    int remainingCapacity();

    /**
     * 删除队列中存在的元素
     */
    boolean remove(Object o);

   /**
    * 判断队列中是否存在当前元素
    */
    boolean contains(Object o);

}
```

## ArrayBlockingQueue

ArrayBlockingQueue() 是一个用数组实现的有界阻塞队列，内部按先进先出的原则对元素进行排序；固定长度，不用扩容。
其中 put 方法和 take 方法为添加和删除元素的阻塞方法。
ArrayBlockingQueue 实现的生产者消费者的 Demo，代码只是一个简单的 ArrayBlockingQueue 的使用，Consumer 消费者和 Producer 生产者通过 ArrayBlockingQueue 来获取（take）和添加（put）数据。

```java
/**
 * 使用 ArrayBlockingQueue 实现的生产者消费者简单模型
 */
public class ArrayBlockingQueueDemo {

    private final static ExecutorService THREAD_POOL = Executors.newFixedThreadPool(4);
    private final static ArrayBlockingQueue<Data> QUEUE = new ArrayBlockingQueue<>(1);

    public static void main(String[] args) {
        THREAD_POOL.execute(new Producer(QUEUE));
        THREAD_POOL.execute(new Consumer(QUEUE));
        THREAD_POOL.execute(new Producer(QUEUE));
        THREAD_POOL.execute(new Consumer(QUEUE));
        THREAD_POOL.shutdown();
    }
}

class Data {

}

class Producer implements Runnable {

    private final ArrayBlockingQueue<Data> mAbq;

    public Producer(ArrayBlockingQueue<Data> mAbq) {
        this.mAbq = mAbq;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            produce();
        }

    }

    private void produce() {
        try {
            Data data = new Data();
            this.mAbq.put(data);
            System.out.println("生产了数据@" + data);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

class Consumer implements Runnable {

    private final ArrayBlockingQueue<Data> mAbq;

    public Consumer(ArrayBlockingQueue<Data> mAbq) {
        this.mAbq = mAbq;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            consumer();
        }
    }

    private void consumer() {
        try {
            Data data = this.mAbq.take();
            System.out.println("消费数据-" + data);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}
```


ArrayBlockingQueue 内部的阻塞队列是通过 ReentrantLock 和 Condition 条件队列实现的，
所以 ArrayBlockingQueue 中的元素存在公平和非公平访问的区别，这是因为 ReentrantLock 里面存在公平锁和非公平锁的原因，

下面对 ArrayBlockingQueue 构造方法进行分析：

```java
/**
 * 创建一个具体容量的队列，默认是非公平队列
 */
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

/**
 * 创建一个具体容量、是否公平的队列 
 */
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

ArrayBlockingQueue 除了实现上述 BlockingQueue 接口的方法，其他方法介绍如下：

```java
//返回队列剩余容量
public int remainingCapacity()

// 判断队列中是否存在当前元素o
public boolean contains(Object o) 

// 返回一个按正确顺序，包含队列中所有元素的数组
public Object[] toArray()

// 返回一个按正确顺序，包含队列中所有元素的数组；数组的运行时类型是指定数组的运行时类型
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a)


// 自动清空队列中的所有元素
public void clear()

// 移除队列中所有可用元素，并将他们加入到给定的 Collection 中    
public int drainTo(Collection<? super E> c)

// 从队列中最多移除指定数量的可用元素，并将他们加入到给定的 Collection 中    
public int drainTo(Collection<? super E> c, int maxElements)

// 返回此队列中按正确顺序进行迭代的，包含所有元素的迭代器
public Iterator<E> iterator()
```

ArrayBlockingQueue 源码和实现原理分析

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /** 存储数据的数组 */
    final Object[] items;

    /** 获取数据的索引，用于下次 take, poll, peek or remove 等方法 */
    int takeIndex;

    /** 添加元素的索引， 用于下次 put, offer, or add 方法 */
    int putIndex;

    /** 队列元素的个数 */
    int count;

    /*
     * 并发控制使用任何教科书中的经典双条件算法
     */

    /** 控制并发访问的锁 */
    final ReentrantLock lock;

    /** 非空条件对象，用于通知 take 方法中在等待获取数据的线程，队列中已有数据，可以执行获取操作 */
    private final Condition notEmpty;

    /** 未满条件对象，用于通知 put 方法中在等待添加数据的线程，队列未满，可以执行添加操作 */
    private final Condition notFull;

    /** 迭代器 */
    transient Itrs itrs = null;
}
```

添加(阻塞添加)的实现分析

```java
/**
 * 在当前 put 位置插入数据，put 位置前进一位，
 * 同时唤醒 notEmpty 条件对象等待队列(链表)中第一个可用线程去 take 数据。
 * 当然这一系列动作只有该线程获取锁的时候才能进行，即只有获取锁的线程
 * 才能执行 enqueue 操作。
 */
// 元素统一入队操作
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = x; // putIndex 位置添加数据
    //putIndex 进行自增，当达到数组长度的时候，putIndex 重头再来，即设置为0
    //因为线程会阻塞，所以进到这个方法的线程肯定是OK的，有空间的，直接冲头再来就好了
    if (++putIndex == items.length) 
        putIndex = 0;
    count++; //元素个数自增
    notEmpty.signal(); //添加完数据后，说明数组中有数据了，所以可以唤醒 notEmpty 条件对象等待队列(链表)中第一个可用线程去 take 数据
}

// 添加数据，数组中元素已满时，直接返回 false。
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获取锁，保证线程安全
    lock.lock();
    try {
        // 当数组元素个数已满时，直接返回false
        if (count == items.length)
            return false;
        else {
            // 执行入队操作，enqueue 方法在上面分析了
            enqueue(e);
            return true;
        }
    } finally {
        // 释放锁，保证其他等待锁的线程可以获取到锁
        // 为什么放到 finally (避免死锁)
        lock.unlock();
    }
}

// add 方法其实就是调用了 offer 方法来实现，
// 与 offer 方法的区别就是 offer 方法数组满，抛出 IllegalStateException 异常。
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

/**
 * 插入数据到队列尾部，如果队列已满，阻塞等待空间
 */
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    // 获取锁，期间线程可以打断，打断则不会添加
    lock.lockInterruptibly();
    try {
        // 通过上述分析，我们通过 count 来判断数组中元素个数
        while (count == items.length)
            notFull.await(); // 元素已满，线程挂起，线程加入 notFull 条件对象等待队列(链表)中，等待被唤醒
        enqueue(e); // 队列未满，直接执行入队操作
    } finally {
        lock.unlock();
    }
}

```

提取(阻塞提取)的实现分析

```java
/**
 * 提取 takeIndex 位置上的元素， 然后 takeIndex 前进一位，
 * 同时唤醒 notFull 等待队列(链表)中的第一个可用线程去 put 数据。
 * 这些操作都是在当前线程获取到锁的前提下进行的，
 * 同时也说明了 dequeue 方法线程安全的。
 */
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items; 
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex]; // 提取 takeIndex位置上的数据
    items[takeIndex] = null; // 同时清空数组在 takeIndex 位置上的数据
    // takeIndex 向前前进一位，如果前进后位置超过了数组的长度，则将其设置为0；
    // 为什么设置为0，理由在 putIndex 设置为0的时候介绍过了，原因是一样的。
    if (++takeIndex == items.length) 
        takeIndex = 0;
    count--; // 同时数组的元素个数进行减1
    if (itrs != null)
        itrs.elementDequeued(); // 同时更新迭代器中的元素，迭代器的具体分析会在下面单独整理
    notFull.signal(); // 提取完数据后，说明数组中有空位，所以可以唤醒 notFull 条件对象的等待队列(链表)中的第一个可用线程去 put 数据
    return x;
}

// 提取数据，数组中数据为空时，直接返回 null
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 加锁，前面也分析过，要执行 dequeue操作时，当前线程必须获取锁，保证线程安全
    try {
        return (count == 0) ? null : dequeue(); // 元素个数为0时，直接返回 null，不为0时，元素出队
    } finally {
        // 释放锁，在 finally 中释放可以避免死锁
        lock.unlock();
    }
}


// 返回数组上第 i 个元素
final E itemAt(int i) {
    return (E) items[i];
}

/**
 * 通过代码可以看到，peek 是获取元素，而不是提取， 不会删除 takeIndex 位置上的数据。
 * 内部通过 itemAt 方法实现，而不是 dequeue 方法。
 */
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
     return itemAt(takeIndex); //当队列为空时，返回 null
    } finally {
     lock.unlock();
    }
}

// 从队列头部提取数据，队列中没有元素则阻塞，阻塞期间线程可中断
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); //获取锁，期间线程可以打断，打断则不会提取
    try {
        // 元素为0时，当有线程提取元素，则将该线程加入到 notEmpty 条件对象的等待队列中，
        // 直到当队列中有数据之后，会唤醒该线程去提取数据。
        while (count == 0)
            notEmpty.await();
        return dequeue(); // 若有数据，直接调用 dequeue 提取数据
    } finally {
        lock.unlock();
    }
}

```

