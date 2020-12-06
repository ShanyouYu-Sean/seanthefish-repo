---
layout: post
title: 用LockSupport实现一个先进先出的不可重入锁
date: 2020-11-25 14:00:00
tags: 
- 并发
categories:
- 并发
---

这是网上的解法，感觉不太好，还用了ConcurrentLinkedQueue

```java
public class FIFOMutex {

    private final AtomicBoolean locked = new AtomicBoolean(false);
    private final Queue<Thread> waiters = new ConcurrentLinkedQueue<Thread>();

    public void lock() {
        boolean wasInterrupted = false;
        Thread current = Thread.currentThread();
        waiters.add(current);

        // 只有自己在队首才可以获得锁，否则阻塞自己
        // cas 操作失败的话说明这里有并发，别人已经捷足先登了，那么也要阻塞自己的
        // 有了waiters.peek() != current判断如果自己队首了，为什么不直接获取到锁还要cas 操作呢？
        // 主要是因为接下来那个remove 操作把自己移除掉了额，但是他还没有真正释放锁，锁的释放在unlock方法中释放的
        while (waiters.peek() != current ||
            !locked.compareAndSet(false, true)) {
            // 这里就是使用LockSupport 来阻塞当前线程
            LockSupport.park(this);
            // 这里的意思就是忽略线程中断，只是记录下曾经被中断过
            // 大家注意这里的java 中的中断仅仅是一个状态，要不要退出程序或者抛异常需要程序员来控制的
            if (Thread.interrupted()) {
                wasInterrupted = true;
            }
        }
        // 移出队列，注意这里移出后，后面的线程就处于队首了，但是还是不能获取到锁的，locked 的值还是true,
        // 上面while 循环的中的cas 操作还是会失败进入阻塞的
        waiters.remove();
        // 如果被中断过，那么设置中断状态
        if (wasInterrupted) {
            current.interrupt();
        }

    }

    public void unlock() {
        locked.set(false);
        // 唤醒位于队首的线程
        LockSupport.unpark(waiters.peek());
    }

}
```

然后我自己又写了个不用ConcurrentLinkedQueue的版本，只用cas, 没有head，只有tail的尾插clh队列，前驱节点自旋，

```java
public class FIFOMutex {

    private final AtomicBoolean locked = new AtomicBoolean(false);

    private AtomicReference<Node> tail = new AtomicReference<>(new Node());

    public FIFOMutex(){
    }

    static class Node {
        Thread t;
        Node prev;
        Node next;
        public Node(Thread t){
            this.t = t;
        }
        public Node(){
        }
    }

    public void lock(){
        // 初始化节点
        Node node = new Node(Thread.currentThread());
        for (;;){
            // 拿锁
            if (!locked.compareAndSet(false, true)){
                    // 设置尾节点
                    if (node.prev == null){
                        for(;;){
                            Node oldTail = tail.get();
                            if (tail.compareAndSet(oldTail, node)){
                                node.prev = oldTail;
                                oldTail.next = node;
                                break;
                            }
                        }
                    }
                    // 如果前驱节点有线程，park
                    if (node.prev.t != null){
                        LockSupport.park(Thread.currentThread());
                    }

            }else {
                // 释放节点，使前驱节点有效自旋
                node.t = null;
                // 如果后驱节点不为空，unpark
                if (node.next != null){
                    LockSupport.unpark(node.next.t);
                }
                break;
            }
        }
    }

    public void unlock(){
        locked.set(false);
    }

}
```