---
layout: post
title: juc之ConcurrentLinkedQueue
date: 2020-12-24 14:00:00
tags: 
- 并发
categories:
- 并发
---


内容转载自 https://www.beikejiedeliulangmao.top/java/concurrent/concurrent-linked-queue/ 稍有改动

Java 提供的线程安全的 Queue 可以分为阻塞队列和非阻塞队列，其中阻塞队列的典型例子是 BlockingQueue，非阻塞队列的典型例子是 ConcurrentLinkedQueue，在实际应用中要根据实际需要选用阻塞队列或者非阻塞队列。 阻塞队列可以通过加锁来实现，非阻塞队列可以通过 CAS 操作实现。

ConcurrentLinkedQueue 使用了链表作为其数据结构．内部使用 CAS 来进行链表的维护。ConcurrentLinkedQueue 适合在对性能要求相对较高，同时对队列的读写存在多个线程同时进行的场景，即如果对队列加锁的成本较高则适合使用无锁的 ConcurrentLinkedQueue 来替代。

接下来我们简单地看一下 ConcurrentLinkedQueue 的实现，在 ConcurrentLinkedQueue 中所有数据通过单向链表存储，同时我们还会保存该链表的头指针和尾指针。

```java
// 链表中的节点
private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
    //...
}
/**
 * A node from which the first live (non-deleted) node (if any)
 * can be reached in O(1) time.
 * Invariants:
 * - all live nodes are reachable from head via succ()
 * - head != null
 * - (tmp = head).next != tmp || tmp != head
 * Non-invariants:
 * - head.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 */
private transient volatile Node<E> head;

/**
 * A node from which the last node on list (that is, the unique
 * node with node.next == null) can be reached in O(1) time.
 * Invariants:
 * - the last node is always reachable from tail via succ()
 * - tail != null
 * Non-invariants:
 * - tail.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 * - tail.next may or may not be self-pointing to tail.
 */
private transient volatile Node<E> tail;
```

在对象实例化时，会创建一个虚节点。看到后面你会发现，如果想通过 CAS 维护一个链表，一般都会使用到虚节点。

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

介绍完内部数据结构，我们看一下增删节点的实现方式。先来看一下增加数据的逻辑：

- 入队操作是在一个循环中尝试 CAS 操作，首先判断，尾结点 p.next 是不是 null，是的话就通过 CAS 将 null-> newNode，如果 CAS 成功，说明该节点就已经算是加入到队列中了
  - 但是这里并没有直接修改尾结点，因为 ConcurrentLinkedQueue 中 tail 并不一定是实际上的尾结点，在并发很大时，如果所有线程都要去竞争修改尾结点的话，对性能会有影响，所以，当实际的尾结点（代码中的变量 p）不等于 tail 时，才会进行更新。
  - 在 ConcurrentLinkedQueue 中会出现 Node1 (head)->Node2 (tail)->null 以及 Node1 (head)->Node2 (tail)->Node3->null 这样的情况甚至 Node1 (head)->Node2 (tail)->Node3->Node4 这样的情况，虽然 tail 指针没有直接指向尾结点会导致将新节点加入链表时，需要从 tail 向后查找实际的尾结点，但是这个过程相较于对 tail 节点的竞争来说，影响较小，最终效率也更高
- 如果发现当前 p 节点不是实际上的尾结点，会先检查它的 next 指针是否指向自己，在出队函数 poll 中，将一个元素出队后会把它的 next 指针指向自己，所以这一步实际上是判断当前的 p 节点是否已经出队
  - 如果满足上述情况，我们需要重新获取 tail 指针，如果发现在上述过程中 tail 指针发生了变化，这说明期间已经好有个并发插入过程完成了，我们直接从最新的 tail 对象开始上述流程即可，所以这里就将 p 赋为最新的 tail 指向的对象，
  - 如果整个过程中 tail 指针都没变，说明当前的情况类似于 Node1 (head，tail)-> Node2->null, 但是在判断 p == q 之前，发生了出队操作，状态变成了 Node1 (tail, 已经出队的对象) Node2 (head)->null，这个时候我们要将 p 设置为 head 然后从 head 开始向后遍历
- 最后就是单纯的没有遍历到尾结点的情况了， Node1 (head)->Node2 (tail，当前 p 变量)->Node3（当前 q 变量）->null
  - 如果发现已经进行过一次向后遍历的过程，即 p != t ，并且 tail 指针发生了变化，我们就直接使用 tail 指针，不再向后遍历了 p = t (最新的 tail 指针)
  - 如果不满足上述情况，比如还从来没遍历过，或者虽然遍历过但是 tail 指针没变，我们就继续遍历 p = q (p.next)

```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            // 找到了最后一个节点，通过 CAS 将其 next 指向新节点
            if (p.casNext(null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                // 如果 tail.next 为null就不修改tail，tail.next != null 时才会修改
                // 这里会出现多个线程同时发现 tail.next != null 的情况，所以 tail 指针和实际的尾结点的距离不一定是1
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK. 因为没有要求 tail 指针和实际的尾结点的距离是1
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // 如果发现当前p节点不是实际上的尾结点，会先检查它的next 指针是否指向自己，在出队函数poll中，将一个元素出队后会把它的next指针指向自己，所以这一步实际上是判断当前的 p 节点是否已经出队
            // 如果 tail 指针发生了变化，就从最新的 tail 开始遍历
            // 否则，从 head 开始遍历，因为这时候 tail 可能指向了一个死掉(next 指向自己，已经从队列中移除)的节点
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // 最后就是单纯的没有遍历到尾结点的情况了
            // 如果发现已经进行过一次向后遍历的过程，并且 tail 指针发生了变化，我们就直接使用 tail 指针
            // 如果还从来没遍历过，或者虽然遍历过但是 tail 指针没变，我们就继续遍历
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

最后，我们介绍一下出队的操作，整个出队过程也是在一个 CAS 循环中进行：

- 首先我们检查头指针的 p (head).item 是不是 null，不是的话才说明该节点是一个有效节点，因为初始化是创建的虚节点 item 才等于 null，这里通过 item 是不是 null 来判断是不是虚节点也就是说 ConcurrentLinkedQueue 中不能添加值为 null 的节点
  - 找到有效节点后，通过 cas 将 item 改为 null，后续的操作和添加元素时类似，因为 head 指针也是一个竞争点，所以这里并没有直接修改 head 指针，而是发现从 head 至少向后遍历过一次时，才会修改 head 指针，这和 offer 中的方式类似
  - 如果当前线程要负责修改 head 指针，会判断 刚删掉的节点 p 的 next 是不是 null，是的话就让 p 作为 head（此时 p 充当新的虚节点），如果不是的话，就让 p.next 作为 next（此时 head 就是实际上的头结点）
- 如果 p 的 item == null 或者 cas 失败（别的线程已经把 p.item 置为 null），我们要检查一下 p.next 是不是 null，如果是的话说明 p 已经是最后一个节点了，我们需要返回 null，但是在此之前，我们不妨把 p 设为新的 head 来减少其他线程的遍历开销
- 检查当前 p 节点的 next 指针是不是指向自己，是的话说明当前检查的这个节点已经被别的线程从队列中移除了，那我们就重新开始执行 poll
- 否则，让 p = q (p.next)，也就是说这是从 head 开始向后遍历的过程

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            // item != null 说明该节点是一个有效节点, 通过 CAS 将其item改为 null
            if (item != null && p.casItem(item, null)) {
                // CAS 成功说明已经移除一个节点了，后续的操作和添加元素时类似，因为 head 指针也是一个竞争点
                // 所以这里并没有直接修改 head 指针，而是发现从 head 至少向后遍历过一次时，才会修改 head 指针，这和 offer 中的方式类似
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    // 判断刚删掉的节点 p 的 next 是不是null，是的话就让 p 作为 head（此时p充当新的虚节点），
                    // 如果不是的话，就让 p.next 作为 next（此时head就是实际上的头结点）
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                // 说明 p已经是最后一个节点了，我们需要返回 null
                // 但是在此之前，我们不妨把p设为新的head来减少其他线程的遍历开销
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                // 说明当前检查的这个节点已经被别的线程从队列中移除了，那我们就重新开始执行 poll
                continue restartFromHead;
            else
                // p = q(p.next)，也就是说这是从 head 开始向后遍历的过程
                p = q;
        }
    }
}
```

updateHead 的过程中先会检查是不是真的有必要重置 head 指针，有必要的话在通过 CAS 修改 head 指针，如果 CAS 失败了也无妨，毕竟我们不要求 head 一定指向实际的头结点，poll 中的遍历过程能 cover 这种情况。如果 CAS 成功，会将删掉的 head 指针指向自己。

```java
/**
 * Tries to CAS head to p. If successful, repoint old head to itself
 * as sentinel for succ(), below.
 */
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}

void lazySetNext(Node<E> val) {
    UNSAFE.putOrderedObject(this, nextOffset, val);
}
```

这里大家可能会有疑问，为什么要 lazySet next 指针呢？要想理解这个问题，我们需要先理解 putOrderedObject 和 putObjectVolatile 的区别。因为 Node 中的 next 属性是用 volatile 修饰的，而 volatile 有什么特点呢？一个是防止指令重拍，一个是将其他 CPU cache 中的相关数据无效化，迫使这些 CPU 重新从主存中拉取最新数据。这是通过 Fence (内存屏障) 实现的，在 linux x86 架构中一般是 lock; addl $0,0(%%esp). , 通过加锁来保证互斥。

而 putObjectVolatile 函数等效于声明一个 volatile 变量，然后直接对该变量进行修改。也就是说，无论是 putObjectVolatile 还是对 volatile 变量的直接修改，都依赖与 StoreLoad barriers ，这里 StoreLoad barriers 就是说如果指令的顺序是 Store1; StoreLoad; Load2 ，就需要确保 Store1 保存的数据在 Load2 访问数据之前，一定要能够对所有线程可见。关于内存屏障的解释，可以参考这篇手册, 其中介绍了各个内存屏障的要求，以及在不同架构上的实现方式。

而 putOrderedObject 函数呢，只需要保证当前 cpu 内指令是有序的，不会出现非法的内存访问即可，这也就是说，putOrderedObject 没有多处理期间的可见性保证，也就不会有多余的开销。在我们 ConcurrentLinkedQueue 的场景中，最终将 next 指针指向自己并不需要这么高的可见性需求，而且 next 是修饰为 volatile 的，所以，我们需要显式地调用 putOrderedObject 才能达到 “去 volatile 特性” 的效果，从而提升效率。