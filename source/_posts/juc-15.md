---
layout: post
title: juc之ConcurrentHashMap
date: 2020-12-24 15:00:00
tags: 
- 并发
categories:
- 并发
---

搬运自 https://my.oschina.net/xiaolyuh/blog/3080609  略有修改

## jdk 1.7 

在jdk1.7中，用的是锁分段技术：

底层存储结构
在 JDK1.7中，本质上还是采用链表+数组的形式存储键值对的。但是，为了提高并发，把原来的整个 table 划分为 n 个 Segment 。所以，从整体来看，它是一个由 Segment 组成的数组。然后，每个 Segment 里边是由 HashEntry 组成的数组，每个 HashEntry之间又可以形成链表。我们可以把每个 Segment 看成是一个小的 HashMap，其内部结构和 HashMap 是一模一样的。

![image.png](https://i.loli.net/2020/11/24/KTmC3ynGUb69ZQO.png)

当对某个 Segment 加锁时，如图中 Segment2，并不会影响到其他 Segment 的读写。每个 Segment 内部自己操作自己的数据。这样一来，我们要做的就是尽可能的让元素均匀的分布在不同的 Segment中。最理想的状态是，所有执行的线程操作的元素都是不同的 Segment，这样就可以降低锁的竞争。

采用Segment数组结构和HashEntry数组结构组成，Segment数组的大小就是ConcurrentHashMap的并发度。Segment继承自ReentrantLock，所以他本身就是一个锁。Segment数组一旦初始化后就不会再进行扩容，这也是jdk1.8去掉他的原因。Segment里面又包含了一个table数组，这个数组是可以扩容的。

如图我们在定位数据的时候需要对key的hash值进行两次寻址操作，第一次找到在Segment数组的位置，第二次找到在table数组中的位置。

### Segment 类

```java
// 直接继承自ReentrantLock，所以一个Segment本身就是一个锁
static final class Segment<K,V> extends ReentrantLock implements Serializable { 
    ...
   // table数组  
   transient volatile HashEntry<K,V>[] table;

    // 一个Segment内的元素个数
    transient int count;

    // 扩容阈值
    transient int threshold;

    // 扩容因子
    final float loadFactor;

    Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
        this.loadFactor = lf;
        this.threshold = threshold;
        this.table = tab;
    }
...

```
我们发现Segment直接继承自ReentrantLock，所以一个Segment本身就是一个锁。所以Segment数组的长度大小直接影响了ConcurrentHashMap的并发度。还有每个Segment单独维护了扩容阈值，扩容因子，所以每个Segment的扩容操作时完全独立互不干扰的。

### HashEntry 类

```java
static final class HashEntry<K,V> {
    // 不可变
    final int hash;
    final K key;
    // volatile保证可见性，这样我们在get操作时就不用加锁了
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
...
}
```

### 构造函数

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    //  参数校验
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    // 并发度控制，最大是65536
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    // 等于ssize从1向左移位的 次数
    int sshift = 0;
    int ssize = 1;
    // 找出最接近concurrencyLevel的2的n次幂的数值
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 这里之所 以用32是因为ConcurrentHashMap里的hash()方法输出的最大数是32位的
    this.segmentShift = 32 - sshift;
    // 散列运算的掩码，等于ssize减1
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 里HashEntry数组的长度度，它等于initialCapacity除以ssize的倍数c，如果c大于1，就会取大于等于c的2的N次方值，所以cap不是1，就是2的N次方。
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    // 保证HashEntry数组大小一定是2的n次幂
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    // 初始化Segment数组，并实际只填充Segment数组的第0个元素。
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}`
```

通过代码我们可以看出，在构造ConcurrentHashMap的时候我们就会完成以下件事情：

- 确认ConcurrentHashMap的并发度，也就是Segment数组长度，并保证它是2的n次幂
- 确认HashEntry数组的初始化长度，并保证它是2的n次幂
- 将Segment数组初始化好并且只填充第0个元素

### put() 方法

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    // 1. 先获取key的hash值
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // 2. 定位到Segment
        s = ensureSegment(j);
    // 3.调用Segment的put方法
    return s.put(key, hash, value, false);
}
```

主要流程是：

- 先获取key的hash值
- 定位到Segment
- 调用Segment的put方法

### hash() 方法

```java
private int hash(Object k) {
        int h = hashSeed;

        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }

```

这个方法大致思路是：先拿到key的hashCode，然后对这个值进行再散列。

### Segment.put() 方法

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 1. 加锁
    HashEntry<K,V> node = tryLock() ? null :
            // scanAndLockForPut在没有获取到锁的情况下，去查询key是否存在，如果不存在就新建一个Node
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 确定元素在table数组上的位置
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                // 如果原来位置上有值并且key相同，那么直接替换原来的value
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                // 元素总数加一
                int c = count + 1;
                // 判断是否需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}

```

大致过程是：

- 加锁
- 定位key在table数组上的索引位置index，获取到头结点
- 判断是否有hash冲突
- 如果没有冲突直接将新节点node添加到数组index索引位
- 如果有冲突，先判断是否有相同key
- 有相同key直接替换对应node的value值
- 没有添加新元素到链表尾部
- 解锁
这里需要注意的是scanAndLockForPut方法，他在没有获取到锁的时候不仅会通过自旋获取锁，还会做一些其他的查找或新增节点的工，以此来提升put性能。

### Segment.scanAndLockForPut() 方法

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    //定位HashEntry数组位置，获取第一个节点
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    //扫描次数，循环标记位
    int retries = -1; // negative while locating node
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        // 表示遍历链表还没有结束
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    //  完成新节点初始化
                    node = new HashEntry<K,V>(hash, key, value, null);
                // 完成链表的遍历，还是没有找到相同key的节点
                retries = 0;
            }
            // 有hash冲突，开始查找是否有相同的key
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        // 断循环次数是否大于最大扫描次数
        else if (++retries > MAX_SCAN_RETRIES) {
            // 自旋获取锁
            lock();
            break;
        }
        // 每间隔一次循环，检查一次first节点是否改变
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            // 首节点有变动，更新first，重新扫描
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

scanAndLockForPut方法在当前线程获取不到segment锁的情况下，完成查找或新建节点的工作。当获取到锁后直接将该节点加入链表即可，提升了put操作的性能。大致过程：

- 定位key在HashEntry数组的索引位，并获取第一个节点
- 尝试获取锁，如果成功直接返回，否则进入自旋
- 判断是否有hash冲突，没有就直接完成新节点的初始化
- 有hash冲突，开始遍历链表查找是否有相同key
- 如果没找到相同key，那么就完成新节点的初始化
- 如果找到相同key，判断循环次数是否大于最大扫描次数
- 如果循环次数大于最大扫描次数，就直接CAS拿锁（阻塞式）
- 如果循环次数不大于最大扫描次数，判断头结点是否有变化
- 进入下次循环

### Segment.rehash() 扩容方法

```java
private void rehash(HashEntry<K,V> node) {
    // 复制老数组
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // table数组扩容2倍
    int newCapacity = oldCapacity << 1;
    // 扩容阈值也增加两倍
    threshold = (int)(newCapacity * loadFactor);
    // 创建新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    // 计算新的掩码
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 计算新的索引位
            int idx = e.hash & sizeMask;
            // 转移数据
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 将新的节点加到对应索引位
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```

在这里我们可以发现每次扩容是针对一个单独的Segment的，在扩容完成之前中不会对扩容前的数组进行修改，这样就可以保证get()不被扩容影响。大致过程是：

- 新建扩容后的数组，容量是原来的两倍
- 遍历扩容前的数组
- 通过e.hash & sizeMask;计算key新的索引位
- 转移数据
- 将扩容后的数组指向成员变量table

### get() 方法

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    // 计算出Segment的索引位
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 以原子的方式获取Segment
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        // 原子方式获取HashEntry
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            // key相同
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                // value是volatile所以可以不加锁直接取值返回
                return e.value;
        }
    }
    return null;
}
```

我们可以看到get方法是没有加锁的，因为HashEntry的value和next属性是volatile的，volatile直接保证了可见性，所以读的时候可以不加锁.

### size() 方法

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    // true表示size溢出32位（大于Integer.MAX_VALUE）
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            // retries 如果retries等于2则对所有Segment加锁
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            // 统计每个Segment元素个数
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        // 解锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    // 如果size大于Integer.MAX_VALUE值则直接返货Integer.MAX_VALUE
    return overflow ? Integer.MAX_VALUE : size;
}
```

size的核心思想是先进性两次不加锁统计，如果两次的值一样则直接返回，否则第三个统计的时候会将所有segment全部锁定，再进行size统计，所以size()尽量少用。因为这是在并发情况下，size其他线程也会改变size大小，所以size()的返回值只能表示当前线程、当时的一个状态，可以算其实是一个预估值。

### isEmpty() 方法

```java
public boolean isEmpty() {
    long sum = 0L;
    final Segment<K,V>[] segments = this.segments;
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j);
        if (seg != null) {
            // 只要有一个Segment的元素个数不为0则表示不为null
            if (seg.count != 0)
                return false;
            // 统计操作总数
            sum += seg.modCount;
        }
    }
    if (sum != 0L) { // recheck unless no modifications
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                if (seg.count != 0)
                    return false;
                sum -= seg.modCount;
            }
        }
        // 说明在统计过程中ConcurrentHashMap又被操作过，
        // 因为上面判断了ConcurrentHashMap不可能会有元素，所以这里如果有操作一定是新增节点
        if (sum != 0L)
            return false;
    }
    return true;
}
```

- 先判断Segment里面是否有元素，如果有直接返回，如果没有则统计操作总数；
- 为了保证在统计过程中ConcurrentHashMap里面的元素没有发生变化，再对所有的Segment的操作数做了统计；
- 最后 sum==0 表示ConcurrentHashMap里面确实没有元素返回true，否则一定进行过新增元素返回false。
和size方法一样这个方法也是一个若一致方法，最后的结果也是一个预估值。



## jdk1.8

![image.png](https://i.loli.net/2020/11/25/cM5TOhxaEjqLQid.png)

这个结构和HashMap一样

### 核心属性

```java
//最大容量
private static final int MAXIMUM_CAPACITY = 1 << 30;
//初始容量
private static final int DEFAULT_CAPACITY = 16;
//数组最大容量
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
//默认并发度，兼容1.7及之前版本
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
//加载/扩容因子，实际使用n - (n >>> 2)
private static final float LOAD_FACTOR = 0.75f;
//链表转红黑树的节点数阀值
static final int TREEIFY_THRESHOLD = 8;
//红黑树转链表的节点数阀值
static final int UNTREEIFY_THRESHOLD = 6;
//当数组长度还未超过64,优先数组的扩容,否则将链表转为红黑树
static final int MIN_TREEIFY_CAPACITY = 64;
//扩容时任务的最小转移节点数
private static final int MIN_TRANSFER_STRIDE = 16;
//sizeCtl中记录stamp的位数
private static int RESIZE_STAMP_BITS = 16;
//帮助扩容的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
//size在sizeCtl中的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

// ForwardingNode标记节点的hash值（表示正在扩容）
static final int MOVED     = -1; // hash for forwarding nodes
// TreeBin节点的hash值，它是对应桶的根节点
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

//存放Node元素的数组,在第一次插入数据时初始化
transient volatile Node<K,V>[] table;
//一个过渡的table表,只有在扩容的时候才会使用
private transient volatile Node<K,V>[] nextTable;
//基础计数器值(size = baseCount + CounterCell[i].value)
private transient volatile long baseCount;
/**
 * 控制table数组的初始化和扩容，不同的值有不同的含义：
 * -1:表示正在初始化
 * -n:表示正在扩容
 * 0:表示还未初始化，默认值
 * 大于0：表示下一次扩容的阈值
 */
private transient volatile int sizeCtl;
//节点转移时下一个需要转移的table索引
private transient volatile int transferIndex;
//元素变化时用于控制自旋
private transient volatile int cellsBusy;
// 保存table中的每个节点的元素个数 长度是2的幂次方，初始化是2，每次扩容为原来的2倍
// size = baseCount + CounterCell[i].value
private transient volatile CounterCell[] counterCells;
```

### Node 类

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }
...
```

链表节点，保存着key和value的值。

### TreeNode类

```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }
...

```

红黑树节点，包含了树的信息。

### TreeBin类
TreeBins中使用的节点
```java
static final class TreeBin<K,V> extends Node<K,V> {
	TreeNode<K,V> root;
	volatile TreeNode<K,V> first;
	// 锁的持有者
	volatile Thread waiter;
	// 锁状态
	volatile int lockState;
	// values for lockState
	// 表示持有写锁
	static final int WRITER = 1; // set while holding write lock
	// 表示等待
	static final int WAITER = 2; // set when waiting for write lock
	// 表示读锁的增量值
	static final int READER = 4; // increment value for setting read lock
...

```
与HashMap有点区别的是，他不直接使用TreeNode作为数的根节点，而是使用TreeBins对其做了装饰后成为了根节点；同时它还记录了锁的状态；需要注意的是：

- TreeBins节点的hash值是 -2
- 我们对红黑树添加节点后，红黑树的根节点有可能会因为旋转而发生变化，所以我们在添加树节点的时候在putTreeVal()方法里面我们使用cas在加了一次锁。

### ForwardingNode 类

```java
/**
 * A node inserted at head of bins during transfer operations.
 */
static final class ForwardingNode<K,V> extends Node<K,V> {
	final Node<K,V>[] nextTable;
	ForwardingNode(Node<K,V>[] tab) {
		super(MOVED, null, null, null);
		this.nextTable = tab;
	}

	Node<K,V> find(int h, Object k) {
		// loop to avoid arbitrarily deep recursion on forwarding nodes
		outer: for (Node<K,V>[] tab = nextTable;;) {
			Node<K,V> e; int n;
			// 1. 判断新的数组是否是null，
			// 2. 如果不为NULL给那就找到对应索引位上的头结点
			// 3. 判断头节点是否为NULL
			if (k == null || tab == null || (n = tab.length) == 0 ||
				(e = tabAt(tab, (n - 1) & h)) == null)
				return null;
			// 自旋找节点
			for (;;) {
				int eh; K ek;
				if ((eh = e.hash) == h &&
					((ek = e.key) == k || (ek != null && k.equals(ek))))
					return e;
				if (eh < 0) {
					// 如果又变成了ForwardingNode标记节点，那说明有发生了扩容，需要跳出循环从新查找
					if (e instanceof ForwardingNode) {
						tab = ((ForwardingNode<K,V>)e).nextTable;
						continue outer;
					}
					else
						return e.find(h, k);
				}
				if ((e = e.next) == null)
					return null;
			}
		}
	}
}
```

ForwardingNode 节点是一个扩容标记节点，只要在数组上发现对应索引位上是ForwardingNode 节点时，表示正在扩容。当get方法调用时，如果遇到ForwardingNode 节点，那么它将会到扩容后的数据上查找数据，否则还是在扩容前的数组上查找数据。这个要注意两点：

- 这个节点的hash值是 -1
- 这个节点的find方法是在对扩容后的数组进行查找

### 构造函数

```java
public ConcurrentHashMap18() {
}
```

与HashMap一样，构造函数啥都没干，初始化操作是在第一次put完成的。

### put() 方法

```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

### spread() 方法

```java
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```
计算key.hashCode（）并将更高位的散列扩展（XOR）降低。采用位运算主要是是加快计算速度。

### putVal() 方法

```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
	if (key == null || value == null) throw new NullPointerException();
	// 计算hash值
	int hash = spread(key.hashCode());
	int binCount = 0;
	for (Node<K,V>[] tab = table;;) {
		Node<K,V> f; int n, i, fh;
		// 判断是否需要初始化
		if (tab == null || (n = tab.length) == 0)
			tab = initTable();
		// 找出key对应的索引位上的第一个节点
		else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
			// 如果该索引位为null，则直接将数据放到该索引位
			if (casTabAt(tab, i, null,
						 new Node<K,V>(hash, key, value, null)))
				break;                   // no lock when adding to empty bin
		}
		// 正在扩容
		else if ((fh = f.hash) == MOVED)
			tab = helpTransfer(tab, f);
		else {
			V oldVal = null;
			// 加内置锁锁定一个数组的索引位，并添加节点
			synchronized (f) {
				if (tabAt(tab, i) == f) {
					// 表示链表节点
					if (fh >= 0) {
						binCount = 1;
						for (Node<K,V> e = f;; ++binCount) {
							K ek;
							// key相同直接替换value值
							if (e.hash == hash &&
								((ek = e.key) == key ||
								 (ek != null && key.equals(ek)))) {
								oldVal = e.val;
								if (!onlyIfAbsent)
									e.val = value;
								break;
							}
							// 将新节点添加到链表尾部
							Node<K,V> pred = e;
							if ((e = e.next) == null) {
								pred.next = new Node<K,V>(hash, key,
														  value, null);
								break;
							}
						}
					}
					// 表示树节点
					else if (f instanceof TreeBin) {
						Node<K,V> p;
						binCount = 2;
						// 添加数节点
						if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
													   value)) != null) {
							oldVal = p.val;
							if (!onlyIfAbsent)
								p.val = value;
						}
					}
				}
			}
			if (binCount != 0) {
				// 尝试将链表转换成红黑树
				if (binCount >= TREEIFY_THRESHOLD)
					treeifyBin(tab, i);
				if (oldVal != null)
					return oldVal;
				break;
			}
		}
	}
	addCount(1L, binCount);
	return null;
}
```

主要流程：

- 计算key的hash值
- 判断是否需要初始化，如果是则调用initTable() 方法完成初始化
- 判断是否有hash冲突，如没有直接设置新节点到对饮索引位，如果有获取头结点
- 根据头结点的hash值判断是否正在扩容，如果是则帮助扩容
- 如果没有扩容则对头结点加锁，添加新节点
- fh >= 0根据头结点hash值判断是否是链表节点，如果是新增链表节点，否则新增树节点
- 新增树节点putTreeVal()需要注意，红黑树的根节点有可能会因为旋转而发生变化，所以我们在添加节点的时候还需要对根节点使用cas在加了一次锁。
- 判断是否需要尝试由链表转换成树结构
- addCount(1L, binCount);新增count数，并判断是否需要扩容或者帮助扩容

sizeCtl值含义：
-1:表示正在初始化
-n:表示正在扩容
0:表示还未初始化，默认值
大于0：表示下一次扩容的阈值

### initTable() 初始化方法

```java
private final Node<K,V>[] initTable() {
	Node<K,V>[] tab; int sc;
	while ((tab = table) == null || tab.length == 0) {
		// 正在初始化
		if ((sc = sizeCtl) < 0)
			// 让出CPU执行权，然后自旋
			Thread.yield(); // lost initialization race; just spin
		// CAS替换标志位（相当于获取锁）
		else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
			try {
				// 二次判断
				if ((tab = table) == null || tab.length == 0) {
					int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
					@SuppressWarnings("unchecked")
					Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
					table = tab = nt;
					// 相当于sc=n*3/4
					sc = n - (n >>> 2); 
				}
			} finally {
				// 扩容阈值
				sizeCtl = sc;
			}
			break;
		}
	}
	return tab;
}
```

主要过程：

- 根据sizeCtl判断是否正在初始化
- 如果其他线程正在初始化就让出CPU执行权，进入下一次CPU执行权的竞争Thread.yield();
- 如果没有进行初始化的线程则，CAS替换sizeCtl标志位（相当于获取锁）
- 获取到锁后再次判断是否初始化
- 如果没有则初始化Node数组,并设置sizeCtl值为下一次扩容阈值

### helpTransfer()帮助扩容

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
	Node<K,V>[] nextTab; int sc;
	// ForwardingNode标记节点，表示正在扩容
	if (tab != null && (f instanceof ForwardingNode) &&
		(nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
		int rs = resizeStamp(tab.length);
		while (nextTab == nextTable && table == tab &&
			   (sc = sizeCtl) < 0) {
			if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
				sc == rs + MAX_RESIZERS || transferIndex <= 0)
				break;
			if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
				transfer(tab, nextTab);
				break;
			}
		}
		return nextTab;
	}
	return table;
}
```
判断是否正在扩容，如果是就帮助扩容。

### transfer() 扩容方法

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {\
	// n原来数组长度
	int n = tab.length, stride;
	if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
		stride = MIN_TRANSFER_STRIDE; // subdivide range
	// 判断是发起扩容的线程还是帮助扩容的线程，如果是发起扩容的需要初始化新数组
	if (nextTab == null) {            // initiating
		try {
			@SuppressWarnings("unchecked")
			Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
			nextTab = nt;
		} catch (Throwable ex) {      // try to cope with OOME
			sizeCtl = Integer.MAX_VALUE;
			return;
		}
		nextTable = nextTab;
		transferIndex = n;
	}
	int nextn = nextTab.length;
	// 扩容期间的数据节点（用于标志位，hash值是-1）
	ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
	// 当advance == true时，表明该节点已经处理过了
	boolean advance = true;
	// 在扩容完成之前保证get不被影响
	boolean finishing = false; // to ensure sweep before committing nextTab
	// 1. 从右往左找到第一个有数据的索引位节点（有hash冲突的桶）
	// 2. 如果找到的节点是NULL节点（没有hash冲突的节点），那么将该索引位的NULL替换成ForwardingNode标记节点，这个节点的hash是-1
	// 3. 如果找到不为NULL的节点（有hash冲突的桶），则对这个节点进行加锁
	// 4. 开始进进移动节点数据
	for (int i = 0, bound = 0;;) {
		//f:当前处理i位置的node（头结点或者根节点）;
		Node<K,V> f; int fh;
		// 通过while循环获取本次需要移动的节点索引i
		while (advance) {
			// nextIndex:下一个要处理的节点索引; nextBound:下一个需要处理的节点的索引边界
			int nextIndex, nextBound;
			// i是老数组索引位，通过--i来讲索引位往前一个索引位移动，直到0索引位
			if (--i >= bound || finishing)
				advance = false;
			// 节点已全部转移
			else if ((nextIndex = transferIndex) <= 0) {
				i = -1;
				advance = false;
			}
			// transferIndex（初值为最后一个节点的索引），表示从transferIndex开始后面所有的节点都已分配，
			// 每次线程领取扩容任务后，需要更新transferIndex的值(transferIndex-stride)。
			// CAS修改transferIndex，并更新索引边界
			else if (U.compareAndSwapInt
					 (this, TRANSFERINDEX, nextIndex,
					  nextBound = (nextIndex > stride ?
								   nextIndex - stride : 0))) {
				bound = nextBound;
				// 老数组最后一个索引位置
				i = nextIndex - 1;
				advance = false;
			}
		}
		if (i < 0 || i >= n || i + n >= nextn) {
			int sc;
			// 已经完成所有节点复制了
			if (finishing) {
				nextTable = null;
				table = nextTab;
				// sizeCtl阈值为原来的1.5倍
				sizeCtl = (n << 1) - (n >>> 1);
				// 结束自旋
				return;
			}
			// CAS 更新扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
			if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
				if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
					return;
				finishing = advance = true;
				i = n; // recheck before commit
			}
		}
		// 将以前老数组上为NULL的节点（还没有元素的桶或者说成没有hash冲突的数据节点），用ForwardingNode标记节点补齐
		// 主要作用是：其他线程在put元素，发现找到的索引位是fwd节点则表示正在扩容，那么该线程会来帮助扩通，而不是在那里等待
		else if ((f = tabAt(tab, i)) == null)
			advance = casTabAt(tab, i, null, fwd);
		// 表示处理过该节点了
		else if ((fh = f.hash) == MOVED)
			advance = true; // already processed
		else {
			// 对应索引位加锁
			synchronized (f) {
				// 再次校验一下老数组对应索引位节点是否是我们找到的节点f
				if (tabAt(tab, i) == f) {
					// 低索引位头节点(i位)， 高位索引位头节点（i+tab.length）
					Node<K,V> ln, hn;
					// fh >=0 表示链表节点，TreeBin节点的hash值-2
					if (fh >= 0) {
						// fh & n算法可以算出新的节点该分配到那个索引位（runBit要么为0放低位ln，要么为n放高位hn），
						// runBit表示链表中最后一个元素的hash值&n的值
						int runBit = fh & n;
						// lastRun表示链表中最后一个元素
						Node<K,V> lastRun = f;
						// 找到链表中最后一个节点，并赋值给lastRun
						for (Node<K,V> p = f.next; p != null; p = p.next) {
							int b = p.hash & n;
							if (b != runBit) {
								runBit = b;
								lastRun = p;
							}
						}
						// 判断原来的最后一个节点应该添加到高位还是低位
						if (runBit == 0) {
							ln = lastRun;
							hn = null;
						}
						else {
							hn = lastRun;
							ln = null;
						}
						// f表示头结点，如果p不是尾节点，则转移节点
						// 如果以前节点顺序是 1 2 3 4 转移后就是 3 2 1 4 
						for (Node<K,V> p = f; p != lastRun; p = p.next) {
							int ph = p.hash; K pk = p.key; V pv = p.val;
							if ((ph & n) == 0)
								// 转移节点时都是新建节点,以免破坏原来数组结构影响get方法
								ln = new Node<K,V>(ph, pk, pv, ln);
							else
								hn = new Node<K,V>(ph, pk, pv, hn);
						}
						// 设置新数组低索引位头节点(i位)
						setTabAt(nextTab, i, ln);
						// 设置新数组高位索引位头节点（i+tab.length）
						setTabAt(nextTab, i + n, hn);
						// 设置老数组i位为标记节点，表示已经处理过了
						setTabAt(tab, i, fwd);
						advance = true;
					}
					else if (f instanceof TreeBin) {
						TreeBin<K,V> t = (TreeBin<K,V>)f;
						TreeNode<K,V> lo = null, loTail = null;
						TreeNode<K,V> hi = null, hiTail = null;
						int lc = 0, hc = 0;
						for (Node<K,V> e = t.first; e != null; e = e.next) {
							int h = e.hash;
							TreeNode<K,V> p = new TreeNode<K,V>
								(h, e.key, e.val, null, null);
							if ((h & n) == 0) {
								if ((p.prev = loTail) == null)
									lo = p;
								else
									loTail.next = p;
								loTail = p;
								++lc;
							}
							else {
								if ((p.prev = hiTail) == null)
									hi = p;
								else
									hiTail.next = p;
								hiTail = p;
								++hc;
							}
						}
						ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
							(hc != 0) ? new TreeBin<K,V>(lo) : t;
						hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
							(lc != 0) ? new TreeBin<K,V>(hi) : t;
						// 设置新数组低索引位头节点(i位)
						setTabAt(nextTab, i, ln);
						// 设置新数组高位索引位头节点（i+tab.length）
						setTabAt(nextTab, i + n, hn);
						// 设置老数组i位为标记节点，表示已经处理过了
						setTabAt(tab, i, fwd);
						advance = true;
					}
				}
			}
		}
	}
}
```

主要过程：

- tab为扩容前的数组
- 判断是否是第一个发起扩容的线程，如果是需要初始化扩容后的数组nextTable
- fwd = new ForwardingNode<K,V>(nextTab)初始化扩容标记节点
- 进入扩容循环
- 在扩容前的数组tab上从右往左（从高索引位到低索引位）遍历所有头结点，索引位为i
- 如果找到的头结点是NULL(没有hash冲突)则tab[i]=fwd。
- 找到的头结点不为NULL则（有hash冲突）则锁定头结点synchronized (f)
- 再次校验头结点是否发生改变，如果改变直接结束
- 初始化高索引位和第索引位的头结点
- 移动节点到相应索引位
- 设置扩容后的数组低索引位头节点(i位)
- 设置扩容后的数组高位索引位头节点（i+tab.length）
- 设置扩容前的数组i位为标记节点（tab[i]=fwd），表示已经处理过了
- 进入第3步直到完成
  
注意：

- 第5点有tab[i]=fwd有两层含义：1,表示对应索引位已经处理过了；2,当其他线程拿到该头结点的时候能知晓正在扩容，这时在put的时候帮助扩容，在get的时候去扩容后的数组上找相应的key
- int runBit = fh & n;算法可以算出新的节点该分配到那个索引位（runBit要么为0放低位ln，要么为n放高位hn）
- 如果是链表节点，以前节点顺序是 1 2 3 4 扩容后会变成 3 2 1 4

扩容的大致过程图解：

1. 发起扩容，扩容前数组tab

![image.png](https://i.loli.net/2020/11/25/WL3vCiXeB7tqyfU.png)

2. 在扩容前的数组tab上从右往左（从高索引位到低索引位）遍历所有头结点，索引位为i，如果找到的头结点是NULL则直接赋值成````fwd``` 标记节点。

![image.png](https://i.loli.net/2020/11/25/luVcF6AwN2TtaUK.png)

3. 扩容前的数组上找到不为NULL的节点，则还是移动节点到扩容后的额数组

![image.png](https://i.loli.net/2020/11/25/cZKgYvChXGIirf9.png)

### addCount() 方法

```java
private final void addCount(long x, int check) {
	// CounterCell[] as;使用计数器数组因该是为了提升并发量，减小冲突概率
	CounterCell[] as; long b, s;
	// 计数器表不为NULL（counterCells当修改baseCount有冲突时，需要将size增量放到这个计数器数组里面）
	if ((as = counterCells) != null ||
		// 使用CAS更新baseCount的值（+1）如果失败说明存在竞争
		!U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
		CounterCell a; long v; int m;
		// CounterCell是否存在竞争的标记位
		boolean uncontended = true;
		// CounterCell[] as为NULL表示as没有竞争
		if (as == null || (m = as.length - 1) < 0 ||
			// 随机一个数组位置来验证是否为NULL，如果a是null表示没有竞争，随机也是为了减小冲突概率
			(a = as[ThreadLocalRandom.getProbe() & m]) == null ||
			// CAS替换a的value，如果失败表示存在竞争
			!(uncontended =
			  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
			// 将size增量值存到as上
			fullAddCount(x, uncontended);
			return;
		}
		if (check <= 1)
			return;
		// 统计size
		s = sumCount();
	}
	// 检查是否需要扩容
	if (check >= 0) {
		Node<K,V>[] tab, nt; int n, sc;
		// size大于阈值sizeCtl，tab数组长度小于最大值1<<30
		while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
			   (n = tab.length) < MAXIMUM_CAPACITY) {
			int rs = resizeStamp(n);
			// 表示正在扩容
			if (sc < 0) {
				if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
					sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
					transferIndex <= 0)
					break;
				if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
					// 帮助扩容
					transfer(tab, nt);
			}
			// sc = (rs << RESIZE_STAMP_SHIFT) + 2，移位后是负数
			else if (U.compareAndSwapInt(this, SIZECTL, sc,
										 (rs << RESIZE_STAMP_SHIFT) + 2))
				// 发起扩容，此时nextTable=null
				transfer(tab, null);
			s = sumCount();
		}
	}
}
```

在put()方法执行最后会对当前Map的size+1，ConcurrentHashMap中size由baseCount和CounterCell[] as组成，size=baseCount+as[i].value。addCount的大致过程如下：

- CAS替换baseCount值，如果失败说明对size的增量（size++）存在竞争
- 如果存在竞争，我们会使用到CounterCell[] as数组
- as[ThreadLocalRandom.getProbe() & m]随机取一个索引位，使用CAS完成size++
- 如果as[i]也存在竞争会调用fullAddCount(x, uncontended);方法完成size++
- size++完成后通过size=baseCount+as[i].value公式计算出元素总数
- 判断是否需要扩容
- 如果需要扩容，在判断一下是帮助扩容还是发起扩容

注意：

- CounterCell[] as：这个的只要目的是分散对baseCount的单一竞争，提示size++的并发率，这里和table数组一样使用了锁分离技术，as的长度也是2的n次方，初始长度是2
- 在第三步中使用随机数也是为了提升并发效率，ThreadLocalRandom类是JDK7在JUC包下新增的随机数生成器，它解决了Random类在多线程下，多个线程竞争内部唯一的原子性种子变量，而导致大量线程自旋重试的不足
- fullAddCount(x, uncontended);方法里面完成了as的初始化和扩容
- 元素总数的计算公式是size=baseCount+as[i].value

### sumCount() 方法

```java
final long sumCount() {
	CounterCell[] as = counterCells; CounterCell a;
	long sum = baseCount;
	if (as != null) {
		for (int i = 0; i < as.length; ++i) {
			if ((a = as[i]) != null)
				sum += a.value;
		}
	}
	return sum;
}
```

元素总数的计算公式是size=baseCount+as[i].value

### get() 方法

```java
public V get(Object key) {
	Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
	int h = spread(key.hashCode());
	// table 不为NULL并且对饮索引位不为NULL
	if ((tab = table) != null && (n = tab.length) > 0 &&
		(e = tabAt(tab, (n - 1) & h)) != null) {
		if ((eh = e.hash) == h) {
			// 头节点key相同
			if ((ek = e.key) == key || (ek != null && key.equals(ek)))
				return e.val;
		}
		// 树节点或者ForwardingNode标记节点
		else if (eh < 0)
			return (p = e.find(h, key)) != null ? p.val : null;
		// 链表节点
		while ((e = e.next) != null) {
			// key相同
			if (e.hash == h &&
				((ek = e.key) == key || (ek != null && key.equals(ek))))
				return e.val;
		}
	}
	return null;
}
```

主要流程：

- 判断table和key对应索引位是否为NULL
- 判断头节点是否是要找的节点
- eh < 0表示是树节点或ForwardingNode标记节点，直接通过find方法找对应的key
- 否则是链表节点，挨个链表节点找相应的key
- 返回结果

注意：

- get 方法没有加锁，原因是节点的value是volatile的，已经保证了可见性，只要value有更新，那么我们一定能读到最新数据。
- e.find(h, key)这里：如果对应索引位头结点是ForwardingNode节点，那么会直接去扩通后的数组找对应的key，可以参见上面ForwardingNode.find()方法

### size()方法
```java
public int size() {
	long n = sumCount();
	return ((n < 0L) ? 0 :
			(n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
			(int)n);
}
```

### 弱一致性
get方法和containsKey方法都是遍历对应索引位上所有节点，来判断是否存在key相同的节点以及获得该节点的value。但由于遍历过程中其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据，这一点是ConcurrentHashMap在弱一致性上的体现。

### JDK1.8与JDK1.7的不同点

- 去掉了Segment 数组：这样做锁的粒度更小，减少了并发冲突的概率；查找数据时不用计算两次hash；
- 存储数据是采用了链表+红黑树的形式：当一个桶内数据量很大的时候，红黑树的查询效率远高于链表。
- 1.8直接使用了内置锁synchronized：简化了加锁操作
- 1.8的初始化是在第一次put时完成的，1.7的时候再构造的时候完成的
- 在put过程中当发现正在扩容，1.8的线程会帮助扩容，1.7的只是会检查key是否存在或者完成新节点的初始化工作
- 1.8的hash值计算更简单了
- 1.8扩容过程中会修改扩容前的数组，1.7扩容过程中不会修改原来数组
- 1.8在get()时如果判断到当前索引位正在扩容，那么直接在扩容后的数组中去找对应的key
- 1.7的size计算使用的三次计算的方式，1.8使用了锁分离技术

1、整体结构
1.7：Segment + HashEntry + Unsafe

1.8: 移除Segment，使锁的粒度更小，Synchronized + CAS + Node + Unsafe

2、put（）
1.7：先定位Segment，再定位桶，put全程加锁，没有获取锁的线程提前找桶的位置，并最多自旋64次获取锁，超过则挂起。

1.8：由于移除了Segment，类似HashMap，可以直接定位到桶，拿到first节点后进行判断，1、为空则CAS插入；2、为-1则说明在扩容，则跟着一起扩容；3、else则加锁put（类似1.7）

3、get（）
基本类似，由于value声明为volatile，保证了修改的可见性，因此不需要加锁。

4、resize（）
1.7：跟HashMap步骤一样，只不过是搬到单线程中执行，避免了HashMap在1.7中扩容时死循环的问题，保证线程安全。

1.8：支持并发扩容，HashMap扩容在1.8中由头插改为尾插（为了避免死循环问题），ConcurrentHashmap也是，迁移也是从尾部开始，扩容前在桶的头部放置一个hash值为-1的节点，这样别的线程访问时就能判断是否该桶已经被其他线程处理过了。

5、size（）
1.7：很经典的思路：计算两次，如果不变则返回计算结果，若不一致，则锁住所有的Segment求和。

1.8：用baseCount来存储当前的节点个数，这就设计到baseCount并发环境下修改的问题