---
layout: post
title: juc之原子类
date: 2020-01-21 10:00:00
tags: 
- 并发
categories:
- 并发
---

原子类是对Java中不能保证原子类型的指令的扩展。

原子类是典型的cas无锁的应用，并在一些累加方法中加入了自旋锁。原子类相比于锁，有一定优势：
粒度更细：原子类把竞争范围缩小到了变量级别
效率较高：通常情况下更高，但是高度竞争的情况下效率更低

看了一下源码都差不太多，放在一起说吧

以AtomicInteger为例：

compareAndSet（）：

```java
public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

直接调用unsafe中的cas方法，没有自旋，一次返回。

getAndSet():
```java
public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
```
需要注意一下貌似原子类的代码在新版本的jdk中是有改动过的，网上有些解析都是在原子类中自旋，现在都是把自旋的操作隐藏在unsafe里了。因此这个getAndSet方法实际上是有自旋操作的，点进unsafe看：
```java
public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }
```
每次都会先取出原子类旧的value，然后做cas操作，直到操作成功。

getAndUpdate（）：
```java
public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }
```
跟getAndSet差不多，只不过这里参数是一个函数式的接口，并且把自旋操作拿到了原子类里。

addAndGet（）、decrementAndGet（）、incrementAndGet（），getAndAdd（）、getAndDecrement（）、getAndIncrement（）：
都是一样的，全都调用unsafe的getAndAddInt（）,一个自旋锁，也没啥好说的：
```java
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

原子类基本上就是这些，基本类型和对象类型的方法源码基本上都是大同小异，不多说了，AtomicReferenceFieldUpdater就是原子更新对象中的某一个字段，带array的原子类就是在更新时加一个数组下标。