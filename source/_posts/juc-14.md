---
layout: post
title: juc之CoryOnWriteArrayList、CopyOnWriteArraySet
date: 2020-12-24 14:01:00
tags: 
- 并发
categories:
- 并发
---

CopyOnWrite（简称：COW）：即复制再写入，就是在添加元素的时候，先把原 List 列表复制一份，再添加新的元素

CopyOnWriteArrayList 用于读场景远多于写场景的情况，它能够让读与读之间不互斥，读与写也不互斥，只有写与写之间才会互斥。它的思路也很简单，内部通过一个数组来维护数据，正常读数据时直接通过索引从数组中提取数据。

```java
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;

@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}

/**
 * {@inheritDoc}
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    return get(getArray(), index);
}

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}
```

而写数据时，需要将整个数组都复制一遍，然后在新数组的末尾添加最新的数据。最后替换掉原来的数组，这样原来的数组就会被回收。很显然，这种实现方式在减小竞争的同时，承担了数据空间 * 2 的压力。

```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取原始集合
        Object[] elements = getArray();
        int len = elements.length;
        
        // 复制一个新集合
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        
        // 替换原始集合为新集合
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}

/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}
```

CopyOnWriteArraySet
CopyOnWriteArraySet逻辑就更简单了，就是使用 CopyOnWriteArrayList 的 addIfAbsent 方法来去重的，添加元素的时候判断对象是否已经存在，不存在才添加进集合。

```java
/**
 * Appends the element, if not present.
 *
 * @param e element to be added to this list, if absent
 * @return {@code true} if the element was added
 */
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}
```