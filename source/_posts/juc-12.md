---
layout: post
title: juc之PriorityBlockingQueue
date: 2020-11-24 13:00:00
tags: 
- 并发
categories:
- 并发
---
内容转载自  [ https://www.jianshu.com/p/28c9d9e34b29](https://www.jianshu.com/p/bf6ef56646e5)   稍有改动


## PriorityBlockingQueue定义

PriorityBlockingQueue 是基于 二叉堆, ReentrantLock, Condition 实现的并发安全的优先级队列.
主要有以下特点:

- 数据容量没有界限(最大值 Integer.MAX_VALUE - 8)
- 居于ReentrantLock 实现并发安全, 基于 Condition 实现线程等待唤醒
- 数据底层存放在居于数组实现的二叉堆上, 注意这里没有实现堆排序, 只是每次有数据变更时将最小/大放在了堆的最上面的节点上

## 初始化方法 堆化(heapify)
实现思路: 从堆的最后一个 parent 开始, 将最小/大值放在 parent位置, 直到最顶层的parent为止；
直接看代码

```java
/**
 * Establishes the heap invariant (described above) in the entire tree
 * assuming nothing about the order of the elements prior to the call
 */
private void heapify(){
    /**
     * 将这个数组进行 堆化 (heapify)
     *
     */
    Object[] array = queue;
    int n = size;
    int half = (n >>> 1) -1;                // 1. 这里有个注意点 n 是数组的 length,
    Comparator<? super E> cmp = comparator; // 2. 获取 比较器, 若这里的 comparator是空, 则用元素自己实现的比较接口进行比较
    if(cmp == null){
        for(int i = half; i >= 0; i--){     // 3. 从整个数组的最后一颗树开始, 将二叉树的最小值放置在parent位置, 一直到最上面的那颗二叉树
            siftDownComparable(i, (E)array[i], array, n);
        }
    }else{
        for(int i = half; i >= 0; i--){
            siftDownUsingComparator(i, (E)array[i], array, n, cmp);
        }
    }
                                            // 4. 经过这个 heapify 方法后, 整个二叉堆中的最小值已经放在的 index=0 的位置上(注意: 这时不保证 左子树一定小于右子树)
                                            // 5. 若要进行二叉堆的排序, 则需要将 index=0的位置排查在外 从 index= 1的位置开始, 到最后一个位置, 再进行上面的操作
                                            // 其实思路就是 每次将最小值放在数组的最上面, 然后排除这个节点在外, 将下面的数组作为一个整体, 然后重复上面的步骤, 直到最后一个元素
}
```

这个方法其实从最后一个 parent 开始进行与子节点比较, 将最小/大值放在 parent 位置, 直到 顶层的 parent 为止
我们发现代码中有个 siftDownComparable 方法, 这是实现 堆化的重要步骤

```java
/**
 * Inserts item x at position k, maintaining heap invariant by
 * demoting x down the tree repeatedly until it is less than or
 * equal to its children or is a leaf
 *
 * @param k     the position to fill
 * @param x     the item to insert
 * @param array the heap array
 * @param n     the heap array
 * @param <T>
 */
private static <T> void siftDownComparable(int k, T x, Object[] array, int n){
    /**
     * 从整个数组的 k 位置开始向下进行 比较更换操作
     * 1. 获取这个数组的中间值(大于等于它其实就是说已经没有子节点)
     *      举例: 数组 array 含有元素 : 1,2,3,4,5,6,7,8,9,10 共10个元素
     *          其中的之间 half = n >>> 1 = 10 >>> 1 = 5; (就是下面代码中的 half, 堆中所有parent的 index 均小于 5)
     *          而最大 parent 的index 是 : (9 - 1) >>> 1 = 4;
     *          再parent调整好后, 再下面的代码中获取的 k 就变成 9/10, 但是 9/10 > 5 (就是下面代码的 while(k < half))
     * 2. 从k位子开始不断向下比较, 将最小值放到 parent位置, 直到 k >= half
     * 3. 经过这个方法比较后, 从k往下 都是最小值上parent上的一个棵二叉树
     */
    if(n > 0){
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;                 // 1. 获取整个数组的中间坐标
        while(k < half){                    // 2. k这里其实表示 parent 在数组中的 index, k >= half 其实就说明 k 在数组中已经没有子节点
            int child = (k << 1) + 1;       // 3. 获取 k 的左子节点的 index
            Object c = array[child];        // 4. 获取左子节点的值
            int right = child + 1;          // 5. 获取右子节点的 index
            if(right < n &&                 // 6. 这个 if 判断其实是 判断左右子节点的大小, 并且找到其中的最小值, 赋值给 c;
                    ((Comparable<? super T>)c).compareTo((T)array[right]) > 0
                    ){
                c = array[child = right];
            }
            if(key.compareTo((T)c) <= 0){   // 7. key <= c 则说明, 进行下面 sift 已经完成 (父节点k已经小于等于子节点), 直接 break 出
                break;
            }
            array[k] = c;                   // 8. 代码运行到这里说明 k > c， 则将子数据c赋值到k的位置
            k = child;                      // 9. 将上次的子节点 child作为父节点, 再次下面进行比较, 直到 k >= half
        }
        array[k] = key;                     // 10. 将key值赋值给最后一次进行 siftdown 比较的  父节点上
    }
}
```

操作思路:

```java
 从整个数组的 k 位置开始向下进行 比较更换操作
  1. 获取这个数组的中间值(大于等于它其实就是说已经没有子节点)
       举例: 数组 array 含有元素 : 1,2,3,4,5,6,7,8,9,10 共10个元素
           其中的之间 half = n >>> 1 = 10 >>> 1 = 5; (就是下面代码中的 half, 堆中所有parent的 index 均小于 5)
           而最大 parent 的index 是 : (9 - 1) >>> 1 = 4;
           再parent调整好后, 再下面的代码中获取的 k 就变成 9/10, 但是 9/10 > 5 (就是下面代码的 while(k < half))
  2. 从k位子开始不断向下比较, 将最小值放到 parent位置, 直到 k >= half
  3. 经过这个方法比较后, 从k往下 都是最小值上parent上的一个棵二叉树
```

## 添加元素 offer 方法
主要思路: 将添加的元素放置到数组的最尾端, 然后调用 siftUp 进行向上调整

```java
  /**
 * Inserts the specified element into this priority queue
 * As the queue is unbounded, his method will never return {@code false}
 *
 * @param e the lement to add
 * @return {@code true} (as specified element cannot be compared
 *          with elements currently in the priority queue according to the
 *          priority queue's ordering)
 * @throws NullPointerException if the specified element is null
 */
@Override
public boolean offer(E e) {
    if(e != null){
        throw new NullPointerException();
    }
    final ReentrantLock lock = this.lock;       // 1. 获取全局共享的锁
    lock.lock();
    int n, cap;
    Object[] array;                             // 2. 判断容器是否需要扩容
    while((n = size) >= (cap = (array = queue).length)){
        tryGrow(array, cap);                    // 3. 进行扩容操作
    }

    try{
        Comparator<? super E> cmp = comparator;
        if(cmp == null){                        // 4. 进行 保持 heap 性质的 siftUp 操作
            siftUpComparable(n, e, array);
        }else{
            siftUpUsingComparator(n, e, array, cmp);
        }
        size = n + 1;                           // 5. 数据插入后, 整个容量值 + 1;
        notEmpty.signal();                      // 6. Condition 释放信号, 告知其他等待的线程: 容器中已经有元素
    }finally {
        lock.unlock();                          // 7. 释放锁
    }
    return true;
}
```

在代码中我们看到了 tryGrow, 这个调整堆存储空间的方法
在里面运用了 先进行锁的释放 lock.unlock, 然后 根据 allocationSpinLock 这个指标判断是否其他线程在进行扩容, 基本上每次扩容都是 * 1.5

```java
/**
 * Tries to grow array to accommodate at least one more element
 * (but normally expend by about 50%), giving up (allowing retry)
 * on contention (which we expect to be race). Call only this while
 * holding lock
 *
 * @param array the heap array
 * @param oldCap    the length of the array
 */
private void tryGrow(Object[] array, int oldCap){
    /**
     * tryGrow 数组容量扩容操作
     * 整个方法的执行是在已经 ReentrantLock 获取锁的情况下进行的
     */

    lock.unlock(); // must release and then re-acquire main lock // 1. 释放全局的锁(为什么呢? 原因也非常简单, 这个 lock 是全局方法共享的, 为的是更好的并发性能, 而扩容操作的并发是通过简单的乐观锁 allocationSpinLock 来进行控制de)
    Object[] newArray = null;
    if(allocationSpinLock == 0 &&                                // 2. 居于CAS操作, 在 allocationSpinLock 实现乐观锁, 这个也是为了在扩容时不影响容器的其他并发操作
            unsafe.compareAndSwapInt(this, allocationSpinLockOffset, 0, 1)){
        try{
            int newCap = oldCap + ((oldCap < 64)?                // 3. 容量若小于 64则直接 double + 2; 大于的话, 直接 ＊ 1.5
                    (oldCap + 2): // grow faster if small
                    (oldCap >> 1)
                                    );
            if(newCap - MAX_ARRAY_SIZE > 0){ // possible overflow
                int minCap = oldCap + 1;                         // 4. 扩容后超过最大容量处理
                if(minCap < 0 || minCap > MAX_ARRAY_SIZE){
                    throw new OutOfMemoryError();
                }
                newCap = MAX_ARRAY_SIZE;
            }
            if(newCap > oldCap && queue == array){              // 5. queue == array 若数组没变化, 直接进行新建数组
                newArray = new Object[newCap];
            }
        }finally {
            allocationSpinLock = 0;
        }
    }
                                                                // 6. newArray == null 说明上面的操作过程中, 有其他的线程进行了扩容的操作
    if(newArray == null){ // back off if another thread is allocating
        Thread.yield();                                         // 7. 让出 CPU 调度(因为其他线程扩容后必定有其他的操作)
    }
    lock.lock();                                                // 8. 重新获取锁
    if(newArray != null && queue == array){                     // 9. 判断数组 queue 有没有在其他线程中变化过
        queue = newArray;                                       // 10. 未变化, 直接进行赋值操作
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

在进行offer元素时主要还调用了 siftUpComparable 方法
思路: 将元素与上面的 parent 进行比较, 直到 parent >= 这个元素

```java
    /**
     * Insert item x at position k, maintaining heap invariant by
     * promoting x up the tree until it is greater than or equal to
     * its parent, or is the root
     *
     * To simplify and speed up coercions and comparisons. the
     * Comparable and Comparator versions are separated into different
     * method that are otherwise identical. (Similarly for siftDown)
     * These methods are statics, with heap state as arguments, to
     * simplify use in light og possible comparator exceptions
     *
     * @param k the position to fill
     * @param x the item to insert
     * @param array the heap array
     * @param <T>
     */
    private static <T> void siftUpComparable(int k, T x, Object[] array){
        /**
         * 简单的 siftUp 操作: 大体操作就是将元素x放置到k位置, 然后对k的parent进行比较, 直到 k>=parent为止
         */
        Comparable<? super T> key = (Comparable<? super T>)x;
        while(k > 0){                           // 1. k是否到达二叉树的顶端
            int parent = (k - 1) >>> 1;         // 2. 寻找 k 的parent位置
            Object e = array[parent];           // 3. 获取parent的值
            if(key.compareTo((T)e) >= 0){       // 4. key >= e说明 parent >=子节点, 则不需要 siftUp 操作
                break;
            }
            array[k] = e;                       // 5. 将上次比较中 parent节点的值放在子节点上
            k = parent;                         // 6. 将这次比较中的 parent 当作下次比价的k(k是下次比较的子节点)
        }
        array[k] = key;                         // 7. 将值key放置合适的位置上
    }
```

## 删除元素 poll 方法
思路: 将元素的首节点拿出, 作为返回, 末尾节点放置到 index=0位置, 开始 siftDown直到 满足 parent >= child

```java
@Override
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue(){
    int n = size - 1;
    if(n < 0){                              // 1. 判断元素是否未空
        return null;                        // 2. 容器中没有元素, 直接返回 null
    }
    else{
        Object[] array = queue;
        E result = (E)array[0];             // 3. 取出数组中的第一个元素, 作为返回值
        E x = (E)array[n];                  // 4. 将数组的最后一个元素取出
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if(cmp == null){                    // 5. 将刚才取出的数组中最后一个元素放到第一个index位置, 进行siftDown操作(就是向下堆化操作)
            siftDownComparable(0, x, array, n);
        }else{
            siftDownUsingComparator(0, x, array, n, cmp);
        }
        size = n;                           // 6. 重新赋值 size值
        return result;                      // 7. 返回取出的值
    }
}
```

PriorityBlockingQueue的注意要点

PriorityBlockingQueue底层由数组维护，是基于PriorityQueue实现的阻塞队列。构造PriorityBlockingQueue最好传参数capacity(容量)，防止过度扩张，默认为11。

但因为PriorityBlockingQueue可以扩容，可以被无限扩容到Integer.MAX_VALUE - 8，所以可以看做是一个”无界”阻塞队列。注意资源耗竭问题(会产生OOM)。

PriorityBlockingQueue底层对于元素的入队列和出队列使用的是同一个lock对象。

因为PriorityBlockingQueue是”无界”阻塞队列，所以put()不需要阻塞，直接调用offer()方法。因为，put方法在队列满时阻塞，take方法在队列空时阻塞，由于PriorityBlockingQueue是”无界”阻塞队列，所以不需要阻塞。

PriorityBlockingQueue和ArrayBlockingQueue都是由数组维护。二者区别是，PriorityBlockingQueue支持扩容，ArrayBlockingQueue并不支持。由于LinkedBlockingQueue由链表维护，没有扩容的概念，只是会有容量限制。PriorityBlockingQueue的扩容使用CAS自旋锁。

PriorityBlockingQueue的offer()方法(add()、put()方法相当于调用offer()方法)，poll()方法调用lock.lock()获取锁，不响应中断，因为这两个方法不是阻塞的；take()方法都调用lock.lockInterruptibly();方法获取锁，注意：该方法支持线程中断响应。

PriorityBlockingQueue的插入移除参考二叉堆中的最小堆，因为数字越小优先级越高，数字越大优先级越低。所以数字最小的是在队头。

