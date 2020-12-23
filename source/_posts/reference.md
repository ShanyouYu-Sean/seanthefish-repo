---
layout: post
title: 强软弱虚引用及其回收
date: 2020-11-4 15:00:00
tags: 
- jvm
categories:
- jvm
---

之前总结对外内存回收感觉总结的不太好，把这个话题单拎出来，我们再说一次

## 四种引用

- 强引用：代码中普遍存在的类似`object obj = new object()`的引用，只要强引用存在，垃圾处理器就不会回收。
- 软引用：描述有些还有用但非必须的对象。在系统将要发生内存溢出时，会将这些对象列为回收范围进行二次回收，如果这次回收后还没有足够的内存，才会oom。Java中SoftReference类表示软引用。
- 弱引用：描述非必须对象，被弱引用关联的对象只能生存到下一次回收之前，垃圾收集器工作之后，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。Java中WeakReference类表示弱引用。
- 虚引用：这个引用存在的唯一目的，就是在这个对象被收集器回收时得到一个系统通知，被虚引用关联的对象，和其生存时间完全没有关系。Java中PhantomReference类表示虚引用。

### 弱引用

首先, 弱引用(weak reference) 是可以被GC强制回收的。当垃圾收集器发现一个弱可达对象(weakly reachable,即指向该对象的引用只剩下弱引用) 时, 就会将其置入相应的ReferenceQueue 中, 变成可终结的对象. 之后可能会遍历这个 reference queue, 并执行相应的清理。典型的示例是weakhashmap中不再引用的KEY。

当然, 在这个时候, 我们还可以将该对象赋值给新的强引用, 在最后终结和回收前, GC会再次确认该对象是否可以安全回收。因此, 弱引用对象的回收过程是横跨多个GC周期（至少两个，一次标记为引用对象不可达，自己入队，第二次再由gc回收）的。

当你在构造WeakReference时传入一个ReferenceQueue对象，当该引用指向的对象被标记为垃圾的时候，这个引用对象会自动地加入到引用队列里面。接下来，你就可以在固定的周期，处理传入的引用队列，比如做一些清理工作来处理这些没有用的引用对象。

### 软引用

软引用基本上和弱引用差不多，只是相比弱引用，它阻止垃圾回收期回收其指向的对象的能力强一些。如果一个对象是弱引用可到达，那么这个对象会被垃圾回收器接下来的回收周期销毁。但是如果是软引用可以到达，那么这个对象会停留在内存更时间上长一些。当内存不足时垃圾回收器才会回收这些软引用可到达的对象。

软引用比弱引用更难被垃圾收集器回收. 回收软引用没有确切的时间点, 由JVM自己决定. 一般只会在即将耗尽可用内存时, 才会回收软引用,以作最后手段。这意味着, 可能会有更频繁的 full GC, 暂停时间也比预期更长, 因为老年代中的存活对象会很多。

### 虚引用

与软引用，弱引用不同，虚引用指向的对象十分脆弱，我们不可以通过get方法来得到其指向的对象。它的唯一作用就是当其指向的对象被回收之后，自己被加入到引用队列，用作记录该引用指向的对象已被销毁。

使用虚引用时, 必须手动进行内存管理, 以标识这些对象是否可以安全地回收。表面上看起来很正常, 但实际上并不是这样。 javadoc 中写道:

为了防止可回收对象的残留, 虚引用对象不应该被获取: phantom reference 的 get 方法返回值永远是 null。

令人惊讶的是, 很多开发者忽略了下一段内容(这才是重点):

与软引用和弱引用不同, 虚引用不会被 GC 自动清除, 因为他们被存放到队列中. 通过虚引用可达的对象会继续留在内存中, 直到引用自身变为不可达。（如果虚引用为clearer对象，会主动调用其clean方法进行引用自身变为不可达的操作）

也就是说,我们必须手动使虚引用变为不可达状态（把引用置为null）, 否则可能会造成 OutOfMemoryError 而导致 JVM 挂掉. 使用虚引用的理由是, 对于用编程手段来跟踪某个对象何时变为不可达对象, 这是唯一的常规手段。 和软引用/弱引用不同的是, 我们不能复活虚可达(phantom-reachable)对象。

## 四种引用的例子

### 强引用

```java
public class Student {
    @Override
    protected void finalize() throws Throwable {
        System.out.println("Student 被回收了");
    }
}

public class Strong {
    public static void main(String[] args) {
        Student student = new Student();
        // help gc
        student = null;
        System.gc();
    }
}
```

结果： 
```
> Task :Strong.main()
Student 被回收了
```

### 软引用

```java
public class Soft {
    public static void main(String[] args) {
        SoftReference<byte[]> softReference = new SoftReference<byte[]>(new byte[1024*1024*10]);
        System.out.println(softReference.get());
        System.gc();
        System.out.println(softReference.get());
        byte[] bytes = new byte[1024 * 1024 * 10];
        System.out.println(softReference.get());
    }
}
```

-Xmx20M 运行结果：

```
> Task :Soft.main()
[B@7852e922
[B@7852e922
null
```

设置reference queue的软引用：

```java
public class Soft {
    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        Object obj = new Object();
        SoftReference softRef = new SoftReference<Object>(obj,queue);
        //删除强引用
        obj = null;
        //调用gc
        System.gc();
        System.out.println("gc之后的值: " + softRef.get()); // 对象依然存在
        //申请较大内存使内存空间使用率达到阈值，强迫gc
        byte[] bytes = new byte[100 * 1024 * 1024];
        //如果obj被回收，则软引用会进入引用队列
        Reference<?> reference = queue.remove();
        if (reference != null){
            System.out.println("对象已被回收: "+ reference.get());  // 对象为null
        }
    }
}
```

### 弱引用

```java
public class Weak {
    public static void main(String[] args) {
        WeakReference<byte[]> weakReference = new WeakReference<byte[]>(new byte[1]);
        System.out.println(weakReference.get());
        System.gc();
        System.out.println(weakReference.get());
    }
}
```

运行结果：

```
> Task :Weak.main()
[B@7852e922
null
```

设置reference queue的弱引用：

```java
public class Weak {
    public static void main(String[] args) throws InterruptedException {
        ReferenceQueue<Object> queue = new ReferenceQueue<>();
        Object obj = new Object();
        WeakReference weakRef = new WeakReference<Object>(obj,queue);
        //删除强引用
        obj = null;
        System.out.println("gc之后的值: " + weakRef.get()); // 对象依然存在
        //调用gc
        System.gc();
        //如果obj被回收，则软引用会进入引用队列
        Reference<?> reference = queue.remove();
        if (reference != null){
            System.out.println("对象已被回收: "+ reference.get());  // 对象为null
        }
    }
}
```

运行结果：

```
> Task :Weak.main()
gc之后的值: java.lang.Object@7852e922
对象已被回收: null
```

### 虚引用

参考  https://blog.csdn.net/shijiujiu33/article/details/104547837

## reference源码

参考 https://blog.csdn.net/zqz_zqz/article/details/79474314

### Reference构造方法

我们先来看看这些引用类型的父类都给我们提供了什么？

其内部提供2个构造函数,除了传入referent对象外，区别就是要不要传入一个ReferenceQueue队列

```java
Reference(Treferent) {

    this(referent,null);

}
Reference(T referent,ReferenceQueue<?super T>queue){

    this.referent =referent;

    this.queue = (queue==null)? ReferenceQueue.NULL:queue;

}
```
ReferenceQueue队列的作用就是Reference引用的对象被回收时，Reference对象能否进入到pending队列，最终由ReferenceHander线程处理后，Reference就被放到了这个队列里面（Cleaner对象除外），然后我们就可以在这个ReferenceQueue里拿到reference,执行我们自己的操作（至于什么操作就看你想怎么用了），所以这个队列起到一个对象被回收时通知的作用；

如果不带ReferenceQueue的话,要想知道Reference持有的对象是否被回收，就只有不断地轮训reference对象,通过判断里面的get是否为null(phantomReference对象不能这样做,其get始终返回null,因此它只有带queue的构造函数).这两种方法均有相应的使用场景,取决于实际的应用.如weakHashMap中就选择去查询queue的数据,来判定是否有对象将被回收.而ThreadLocalMap,则采用判断get()是否为null来作处理;

对于带ReferenceQueue参数的构造方法，如果传入的队列为null，那么就会给成员变量queue赋值为ReferenceQueue.NULL队列,这个NULL是ReferenceQueue对象的一个继承了ReferenceQueue的内部类，它重写了入队方法enqueue，这个方法只有一个操作，直接返回 false，也就是这个对列不会存取任何数据,它起到状态标识的作用；

### Reference的重要成员变量

```java
//在构造方法传入java对象最后就赋值给了referent，也就是它引用到的对象

private Treferent;        /* Treated specially by GC */

// pending队列
// pending成员变量与后面的discovered对象一起构成了一个pending单向链表
// 注意这个成员变量是一个静态对象，所以是全局唯一的
// pending为链表的头节点，discovered为链表当前Reference节点指向下一个节点的引用
// 这个队列是由jvm的垃圾回收器构建的，当对象除了被reference引用之外没有其它强引用了，jvm的垃圾回收器就会将指向需要回收的对象的Reference都放入到这个队列里面
//（好好理解一下这句话，注意是指向要回收的对象的Reference，要回收的对象就是Reference的成员变量refernt持有的对象，是refernt持有的对象要被回收，而不是Reference对象本身）
// 这个队列会由ReferenceHander线程来处理
// ReferenceHander线程是jvm的一个内部线程，它也是Reference的一个内部类，它的任务就是将pending队列中要被回收的Reference对象移除出来，
// 如果Reference对象在初始化的时候传入了ReferenceQueue队列，那么就把从pending队列里面移除的Reference放到它自己的ReferenceQueue队列里
//（为什么是它自己的？pending队列是全局唯一的队列，但是Reference的queue却不是，它是在构造方法里面指定的，前面说过这里Cleaner对象是个特例）
// 如果没有ReferenceQueue队列，那么其关联的对象就不会进入到Pending队列中，会直接被回收掉，除此之外ReferenceHander线程还会做一些其它操作，后面会讲到

/* List of References waiting to beenqueued.  The collector adds
* References to this list, while theReference-handler thread removes
* them. This list is protected by the above lock object. The
* list uses the discovered field to linkits elements.
*/
privatestatic Reference<Object>pending =null;

//与成员变量pending一起组成pending队列，指向链表当前节点的下一个节点
/* When active:   next element in a discovered reference listmaintained by GC (or this if last)
*    pending:   next element in thepending list (or null if last)
*  otherwise:   NULL
*/

transientprivate Reference<T>discovered; /* used by VM */

// ReferenceQueue队列，这就是我们前面提到的起到通知作用的ReferenceQueue，
// 需要注意的是ReferenceQueue并不是一个链表数据结构，它只持有这个链表的表头对象header，
// 这个链表是由Refence对象里面的next成员变量构建起来的，next也就是链表当前节点的下一个节点(只有next的引用，它是单向链表)，
// 所以Reference对象本身就是一个链表的节点，
// 这个链表数据来源前面讲pending队列的时候已经提到了，它是由ReferenceHander线程从pending队列中取的数据构建的
//（需要注意的是，这个对象并不是一个全局的，它是在构造方法里面传入进来的，所以Reference对象需要进入那个队列是我们自己指定的，
// 也有特例，如FinalReference，Cleaner就是一个内部全局唯一的，无法指定，后面有两篇专门讲他们俩的源码），
// 一旦Reference对象放入了队列里面，那么queue就会被设置为ReferenceQueue.ENQUEUED，来标识当前Reference已经进入到队里里面了；
volatileReferenceQueue<?super T>queue;

//用来与queue成员变量一同组成ReferenceQueue队列，见上面queue的说明；
/* When active:   NULL
*     pending:  this
*   Enqueued:   next reference inqueue (or this if last)
*   Inactive:   this
*/
@SuppressWarnings("rawtypes")
Reference next;

// lock成员变量是pending队列的全局锁，
// 如果你搜索这个lock变量会发现它只在ReferenceHander线程run方法里面用到了，不要忘了jvm垃圾回收器线程也会操作pending队列，往pending里面添加Reference对象，
// 所以需要加锁；
/* Object used to synchronize with thegarbage collector.  The collector
* must acquire this lock at the beginningof each collection cycle.  It is
* therefore critical that any code holdingthis lock complete as quickly
* as possible, allocate no new objects,and avoid calling user code.
*/
staticprivateclass Lock { };

privatestatic Locklock =new Lock();
```

这里一定要理解pending队列，什么样的Reference对象会进入这个队列。

进入这个队列的Reference对象需要满足两个条件：

- Reference所引用的对象已经不存在其它强引用；
- Reference对象在创建的时候，指定了ReferenceQueue；

### Reference状态及其转换
如果去查看Reference源码，会发现这个类的开头有一段很长的注释，说明了Reference对象的四种状态：

```java
/* A Reference instance is in one of four possible internal states:
     *
     *     Active: Subject to special treatment by the garbage collector.  Some
     *     time after the collector detects that the reachability of the
     *     referent has changed to the appropriate state, it changes the
     *     instance's state to either Pending or Inactive, depending upon
     *     whether or not the instance was registered with a queue when it was
     *     created.  In the former case it also adds the instance to the
     *     pending-Reference list.  Newly-created instances are Active.
     *
     *     Pending: An element of the pending-Reference list, waiting to be
     *     enqueued by the Reference-handler thread.  Unregistered instances
     *     are never in this state.
     *
     *     Enqueued: An element of the queue with which the instance was
     *     registered when it was created.  When an instance is removed from
     *     its ReferenceQueue, it is made Inactive.  Unregistered instances are
     *     never in this state.
     *
     *     Inactive: Nothing more to do.  Once an instance becomes Inactive its
     *     state will never change again.
     *
     * The state is encoded in the queue and next fields as follows:
     *
     *     Active: queue = ReferenceQueue with which instance is registered, or
     *     ReferenceQueue.NULL if it was not registered with a queue; next =
     *     null.
     *
     *     Pending: queue = ReferenceQueue with which instance is registered;
     *     next = this
     *
     *     Enqueued: queue = ReferenceQueue.ENQUEUED; next = Following instance
     *     in queue, or this if at end of list.
     *
     *     Inactive: queue = ReferenceQueue.NULL; next = this.
     *
     * With this scheme the collector need only examine the next field in order
     * to determine whether a Reference instance requires special treatment: If
     * the next field is null then the instance is active; if it is non-null,
     * then the collector should treat the instance normally.
     *
     * To ensure that a concurrent collector can discover active Reference
     * objects without interfering with application threads that may apply
     * the enqueue() method to those objects, collectors should link
     * discovered objects through the discovered field. The discovered
     * field is also used for linking Reference objects in the pending list.
     */
```

如果你没有耐心看完这段注释，请直接往后看：

- Active:活动状态,对象存在强引用状态,还没有被回收;
- Pending:垃圾回收器将没有强引用的Reference对象放入到pending队列中，等待ReferenceHander线程处理（前提是这个Reference对象创建的时候传入了ReferenceQueue，否则的话对象会直接进入Inactive状态）；
- Enqueued:ReferenceHander线程将pending队列中的对象取出来放到ReferenceQueue队列里；
- Inactive:处于此状态的Reference对象可以被回收,并且其内部封装的对象也可以被回收掉了，有两个路径可以进入此状态，

路径一：在创建时没有传入ReferenceQueue的Reference对象，被Reference封装的对象在没有强引用时，指向它的Reference对象会直接进入此状态；
路径二、此Reference对象经过前面三个状态后，已经由外部从ReferenceQueue中获取到,并且已经处理掉了。

Reference对象的状态只需要通过成员变量next和queue来判断：

- Active：   next=null
- Pending：  next = this ，queue = ReferenceQueue
- Enqueued:  queue =ReferenceQueue.ENQUEUED
- Inactive:    next = this  ,queue = ReferenceQueue.NULL; 

特例还是要拿出来单独说，上面的图不适用描述Cleaner对象，Cleaner对象是没有Enqueue状态的，它经过HandReference处理时执行其clean方法清理，然后就直接进入了inactive状态了；

下面简单看一下他们的源码实现，如果你理解了上面所有内容，下面的源码理解起来非常简单，我只做简单的注释说明：

### ReferenceHandler线程源码解析

ReferenceHandler线程是一个拥有最高优先级的守护线程，它是Reference类的一个内部类，在Reference类加载执行cinit的时候被初始化并启动；它的任务就是当pending队列不为空的时候，循环将pending队列里面的头部的Reference移除出来，如果这个对象是个Cleaner实例，那么就直接执行它的clean方法来执行清理工作；否则放入到它自己的ReferenceQueue里面；

所以这个线程是pending队列与ReferenceQueue的桥梁；

```java
/* High-priority thread to enqueue pending References*/
private static class ReferenceHandler extends Thread {

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }
 
    public void run() {
        for (;;) {
            //从pending中移除的Reference对象
            Reference<Object> r;
            //此处需要加全局锁，因为除了当前线程，gc线程也会操作pending队列
            synchronized (lock) {
                //如果pending队列不为空，则将第一个Reference对象取出
                if (pending != null) {
                    //缓存pending队列头节点
                    r = pending;
                    //将头节点指向discovered，discovered为pending队列中当前节点的下一个节点，这样就把第一个头结点出队了
                    pending = r.discovered;
                    //将当前节点的discovered设置为null；当前节点出队，不需要组成链表了；
                    r.discovered = null;
                } else {//如果pending队列为空，则等待
                    try {
                        try {
                            lock.wait();
                        } catch(OutOfMemoryError x) { }
                    } catch(InterruptedException x) { }
                    continue;
                }
            }
            // 如果从pending队列出队的r是一个Cleaner对象，那么直接执行其clean()方法执行清理操作；
            if (r instanceof Cleaner) {
                ((Cleaner)r).clean();
                //注意这里，这里已经不往下执行了，所以Cleaner对象是不会进入到队列里面的，给它设置ReferenceQueue的作用是为了让它能进入Pending队列后被ReferenceHander线程处理；
                continue;
            }
            //将对象放入到它自己的ReferenceQueue队列里
            ReferenceQueue<Object> q = r.queue;
            if (q != ReferenceQueue.NULL) q.enqueue(r);
            return true;
        }
    }
}

//以下是ReferenceHander线程初始化并启动的操作
static {
    ThreadGroup tg =Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
            tgn != null;
            tg = tgn, tgn = tg.getParent());
    //线程名称为Reference Handler
    Thread handler = newReferenceHandler(tg, "Reference Handler");
    /* If there were a special system-onlypriority greater than
        * MAX_PRIORITY, it would be used here
        */
    //线程有最高优先级
    handler.setPriority(Thread.MAX_PRIORITY);
    //设置线程为守护线程；
    handler.setDaemon(true);
    handler.start();
}
```

## ReferenceQueue源码解析

ReferenceQueue队列是一个单向链表，ReferenceQueue里面只有一个header成员变量持有队列的队头，Reference对象是从队头做出队入队操作，所以它是一个后进先出的队列

```java
public class ReferenceQueue<T> {

    public ReferenceQueue() { }
    //内部类，它是用来做状态识别的，重写了enqueue入队方法，永远返回false，所以它不会存储任何数据，见后面的NULL和ENQUEUED两个标识成员变量
    private static class Null<S> extendsReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }

    //当Reference对象创建时没有指定queue或Reference对象已经处于inactive状态
    staticReferenceQueue<Object> NULL = new Null<>();

    //当Reference已经被ReferenceHander线程从pending队列移到queue里面时
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    staticprivate class Lock { };

    //出队入队时对队列加锁
    privateLock lock = new Lock();

    //队列头
    privatevolatile Reference<? extends T> head = null;

    //队列长度
    private long queueLength = 0;

    //入队操作，ReferenceHander调用此方法将Reference放入到队列里
    boolean enqueue(Reference<? extends T> r) { /* Called only byReference class */

    //加锁操作队列
    synchronized(lock) {
        //如果Reference创建时没有指定队列或Reference对象已经在队列里面了，则直接返回
        ReferenceQueue<?> queue = r.queue;
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        //只有r的队列是当前队列才允许入队
        assert queue == this;
        //将r的queue设置为ENQUEUED状态，标识Reference已经入队
        r.queue = ENQUEUED;
        //从队列头部入队
        r.next = (head == null) ? r : head;
        head = r;
        //队列里面对象数量+1
        queueLength++;
        //如果r是一个FinalReference实例，那么将FinalReference数量也+1
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        //唤醒出队操作的等待线程
        lock.notifyAll();
        return true;
       }
    }

    //Reference对象出队操作，将头部第一个对象移出队列，并将队列长度-1
   @SuppressWarnings("unchecked")
   private Reference<? extends T> reallyPoll() {       /* Must hold lock */
       Reference<? extends T> r = head;
       if (r != null) {
           head = (r.next == r) ?
               null :
               r.next; // Unchecked due to the next field having a raw type inReference
           r.queue = NULL;
           r.next = r;
           queueLength--;
           if (r instanceof FinalReference) {
               sun.misc.VM.addFinalRefCount(-1);
           }
           return r;
       }
       return null;
    }

    //将队列头部第一个对象从队列中移除出来，如果队列为空则直接返回null（此方法不会被阻塞）
   public Reference<? extends T> poll() {
       if (head == null)
           return null;
       synchronized (lock) {
           return reallyPoll();
       }
    }

 
   //将头部第一个对象移出队列并返回，如果队列为空，则等待timeout时间后，返回null，这个方法会阻塞线程
   public Reference<? extends T> remove(long timeout)
       throws IllegalArgumentException, InterruptedException
    {
       if (timeout < 0) {
           throw new IllegalArgumentException("Negative timeout value");
       }
       synchronized (lock) {
           Reference<? extends T> r = reallyPoll();
           if (r != null) return r;
           for (;;) {
               lock.wait(timeout);
               r = reallyPoll();
               if (r != null) return r;
                if (timeout != 0) return null;
           }
       }
    }

   public Reference<? extends T> remove() throws InterruptedException{
       return remove(0);
    }
}

```

最后来一张图：

![image.png](https://i.loli.net/2020/12/11/MLrasIheZ1SUGRQ.png)