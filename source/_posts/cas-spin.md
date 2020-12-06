---
layout: post
title: 自旋锁
date: 2020-11-19 12:14:47
tags: 
- 并发
categories:
- 并发
---

## 自旋锁

自旋锁是采用让当前线程不停地的在循环体内执行实现的，当循环的条件被其他线程改变时 才能进入临界区。如下

```java
public class SpinLock {

  private AtomicReference<Thread> sign =new AtomicReference<>();

  public void lock(){
    Thread current = Thread.currentThread();
    while(!sign .compareAndSet(null, current)){
    }
  }

  public void unlock (){
    Thread current = Thread.currentThread();
    sign .compareAndSet(current, null);
  }
}
```

## Ticket锁

Ticket锁主要解决的是访问顺序的问题, 每次都要查询一个serviceNum服务号，影响性能（必须要到主内存读取，并阻止其他cpu修改）。

```java
package com.alipay.titan.dcc.dal.entity;

import java.util.concurrent.atomic.AtomicInteger;

public class TicketLock {
    private AtomicInteger                     serviceNum = new AtomicInteger();
    private AtomicInteger                     ticketNum  = new AtomicInteger();
    private static final ThreadLocal<Integer> LOCAL      = new ThreadLocal<Integer>();

    public void lock() {
        int myticket = ticketNum.getAndIncrement();
        LOCAL.set(myticket);
        while (myticket != serviceNum.get()) {
        }

    }

    public void unlock() {
        int myticket = LOCAL.get();
        serviceNum.compareAndSet(myticket, myticket + 1);
    }
}
```

## CLHLock

CLHlock是连表结构，不停的查询前驱变量，导致不适合在NUMA 架构下使用（在这种结构下，每个线程分布在不同的物理内存区域）

```java
public class CLHLock {

    public static class QNode {
        volatile boolean locked;
    }

    // 尾巴，是所有线程共有的一个。所有线程进来后，把自己设置为tail
    private final AtomicReference<QNode> tail;
    // 前驱节点，每个线程独有一个。
    private final ThreadLocal<QNode> myPred;
    // 当前节点，表示自己，每个线程独有一个。
    private final ThreadLocal<QNode> myNode;

    public CLHLock() {
        // 初始状态 tail指向一个node(head)节点
        this.tail = new AtomicReference<>(new QNode());
        this.myNode = ThreadLocal.withInitial(QNode::new);
        this.myPred = new ThreadLocal<>();
    }

    public void lock() {
        // 获取当前线程的代表节点
        QNode node = myNode.get();
        // 将自己的状态设置为true表示获取锁。
        node.locked = true;
        // 将自己放在队列的尾巴，并且返回以前的值。第一次进将获取构造函数中的那个new QNode
        QNode pred = tail.getAndSet(node);
        // 把旧的节点放入前驱节点。
        myPred.set(pred);
        // 判断前驱节点的状态，然后走掉。
        while (pred.locked) {
        }
    }

    public void unlock() {
        // unlock. 获取自己的node。把自己的locked设置为false。
        QNode node = myNode.get();
        node.locked = false;
    }
}
```

CLH队列锁的优点是空间复杂度低（如果有n个线程，L个锁，每个线程每次只获取一个锁，那么需要的存储空间是O（L+n），n个线程有n个myNode，L个锁有L个tail），CLH的一种变体被应用在了JAVA并发框架中。唯一的缺点是在NUMA系统结构下性能很差，在这种系统结构下，每个线程有自己的内存，如果前趋结点的内存位置比较远，自旋判断前趋结点的locked域，性能将大打折扣，但是在SMP系统结构下该法还是非常有效的。一种解决NUMA系统结构的思路是MCS队列锁。

## MCSLock

MCS 的实现是基于链表的，每个申请锁的线程都是链表上的一个节点，这些线程会一直轮询自己的本地变量，来知道它自己是否获得了锁。已经获得了锁的线程在释放锁的时候，负责通知其它线程，这样 CPU 之间缓存的同步操作就减少了很多，仅在线程通知另外一个线程的时候发生，降低了系统总线和内存的开销。实现如下所示：

```java
public class MCSLock {

    public static class Node{
        // 后继节点
        volatile   Node next;
        // 默认false
        volatile   boolean locked;
    }

    // 指向最后加入的线程
    final AtomicReference<Node> tail = new AtomicReference<>(null);

    ThreadLocal<Node> current;

    public MCSLock(){
        //初始化当前节点的node
        current= ThreadLocal.withInitial(Node::new);
    }
    public void lock() throws InterruptedException {

        // 获取当前线程的Node
        Node own = current.get();

        // 把自己设为尾节点， 并取回旧节点
        Node preNode = tail.getAndSet(own);

        // 如果旧节点不为null，说明有线程已经占用
        if(preNode != null){
            // 设置当前节点为需要占用状态；
            own.locked = true;
            // 把前面节点的next指向自己
            preNode.next = own;

            // 在自己的节点上自旋等待前驱通知
            while(own.locked){

            }


        }

    }

    public void unlock(){
        // 获取自己的节点
        Node own=current.get();
        //
        if(own.next==null){
            // 判断是不是自身是不是最后一个线程
            if(tail.compareAndSet(own,null)){
                // 是的话就结束
                return;
            }

        }
        // 在判断过程中，又有线程进来
        while (own.next==null){

        }
        // 本身解锁，通知它的后继节点可以工作了，不用再自旋了
        own.next.locked = false;
        own.next=null;// for gc
    }
}
```

MCS 的能够保证较高的效率，降低不必要的性能消耗，并且它是公平的自旋锁。

CLH 锁与 MCS 锁的原理大致相同，都是各个线程轮询各自关注的变量，来避免多个线程对同一个变量的轮询，从而从 CPU 缓存一致性的角度上减少了系统的消耗。
CLH 锁与 MCS 锁最大的不同是，MCS 轮询的是当前队列节点的变量，而 CLH 轮询的是当前节点的前驱节点的变量，来判断前一个线程是否释放了锁。