---
layout: post
title: 总结下jvm gc的知识
date: 2020-11-12 10:00:00
tags: 
- jvm
categories:
- jvm
---

## jvm的内存模型

jdk8之前

![Dkodu6.png](https://s3.ax1x.com/2020/11/16/Dkodu6.png)

jdk8之后

![DABmHP.png](https://s3.ax1x.com/2020/11/16/DABmHP.png)

程序计数器，本地方法栈和虚拟机栈都是线程私有的，不需要回收。

堆是线程共享的，所有对象实例都应该在堆上分配内存。

方法区，用于存储已被虚拟机加载的类型信息、常量、静态变量等。在jdk8之前为永久带，8之后改为metaspace，（原因是永久带有最大限制，更容易出现oom），metaspace直接使用堆外内存（即本地内存）进行储存。

堆外内存，可以通过nio中的DirectByteBuffer.allocateDirect()直接申请内存空间。

## 对象

对象的分配会先检查类的加载，确定类加载后为对象从堆中分配出一块连续空间，然后再设置对象头信息，最后执行构造函数。

对象包含三部分：

对象头：包括两类信息。第一类是用于存储对象自身的运行时数据，如哈 希码(HashCode)、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。另外一部分是类型指针，即对象指向它的类型元数据的指针，Java虚拟机通过这个指针 来确定该对象是哪个类的实例。

实例数据：对象真正存储的有效信息。

对齐填充：不是必须的，对象为8字节的整数倍，所以需要占位符。

## 回收原则

### 对象的回收

引用计数：给对象添加一个引用计数器，引用时+1，引用失效时-1。java并没有选择这种方法，因为无法判断循环引用。

可达性分析：通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过 程所走过的路径称为“引用链”(Reference Chain)，如果某个对象到GC Roots间没有任何引用链相连， 或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。

![DAp7jO.png](https://s3.ax1x.com/2020/11/16/DAp7jO.png)

固定可作为GC Roots的对象包括以下几种:

- 在虚拟机栈(栈帧中的本地变量表)中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。
- 在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量。
- 在方法区中常量引用的对象，譬如字符串常量池(String Table)里的引用。
- 在本地方法栈中JNI(即通常所说的Native方法)引用的对象。
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象(比如NullPointExcepiton、OutOfMemoryError)等，还有系统类加载器。
- 所有被同步锁(synchronized关键字)持有的对象。
- 反映Java虚拟机内部情况的JM XBean、JVM TI中注册的回调、本地代码缓存等。

要真正宣告一个对象死亡，至少要经历两次标记过程:

如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记，随后进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用 过，那么虚拟机将这两种情况都视为“没有必要执行”。

如果这个对象被判定为确有必要执行finalize()方法，那么该对象将会被放置在一个名为F-Queue的队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize()方法。这里所说的“执行”是指虚拟机会触发这个方法开始运行，但并不承诺一定会等待它运行结束。finalize()方法是对象逃脱死亡命运的最后一次机会，如果对象要在finalize()中成功拯救自己————只要重新与引用链上的任何一个对象建立关联即可，譬如把自己 (this关键字)赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出“即将回收”的 合;如果对象这时候还没有逃脱，那基本上它就真的要被回收了。

### 方法区的回收

方法区的垃圾收集主要回收两部分内容:废弃的常量和不再使用的类型。

常量不再被引用跟对象的判定相似。

类的回收判定比较苛刻：

- 该类所有的实例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例。
- 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换类加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。
- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

### 堆外内存的回收

ps：此处涉及到虚引用知识，最好对着下一篇文章关于虚引用的知识一起食用。

JDK的ByteBuffer类提供了一个接口allocateDirect(int capacity)进行堆外内存的申请

先说一下堆外内存的好处：

实际上在网络io读写的时候，堆内内存会先复制一份到堆外内存，操作完了后再去finanlly去回收这块内存。这样做的原因是在io读写的时候，需要一块连续不变的内存地址，但jvm gc的时候会移动堆内的内存地址，导致io读写的异常。因此使用堆外内存可以避免内存的复制，反而效率更高。

那么堆外内存何时回收呢？

我们先梳理一下回收堆外内存的调用逻辑：

在DirectByteBuffer#new的时候，会调用Cleaner.create(this, new Deallocator(base, size, cap))去创建一个Cleaner对象。Cleaner是PhantomReference的子类，并通过自身的next和prev字段维护的一个双向链表。PhantomReference的作用在于跟踪垃圾回收过程，并不会对对象的垃圾回收过程造成任何的影响。 所以cleaner = Cleaner.create(this, new Deallocator(base, size, cap)); 用于对当前构造的DirectByteBuffer对象的垃圾回收过程进行跟踪。当DirectByteBuffer对象从pending状态 ——> enqueue状态时，会触发Cleaner的clean()，而Cleaner的clean()的方法会实现通过unsafe对堆外内存的释放。

我们看到PhantomReference#new会直接调用其父类Reference#new，而Reference中的静态代码块里，会启动ReferenceHandler线程，该线程会调用Reference#tryHandlePending方法，进而调用Cleaner的clean()方法，把cleaner对象自己从双向链表中移出，并通过启动DirectByteBuffer$Deallocator线程调用Unsafe#freeMemory完成对外内存的释放。

```java
DirectByteBuffer(int cap) {                   // package-private
       // 省略若干代码
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    }
```

```java
public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null ? null : add(new Cleaner(var0, var1));
    }
private static synchronized Cleaner add(Cleaner var0) {
    if (first != null) {
        var0.next = first;
        first.prev = var0;
    }

    first = var0;
    return var0;
}
private Cleaner(Object var1, Runnable var2) {
  // 这个super调用PhantomReference构造器
        super(var1, dummyQueue);
        this.thunk = var2;
    }
```

```java
public PhantomReference(T referent, ReferenceQueue<? super T> q) {
  // 这个super调用Reference构造器
        super(referent, q);
    }
```

```java
Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
static {
        // 省略若干代码
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```

```java
// 这块代码留着吧
    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }
// 主动调用clean方法回收
        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
```

因此DirectByteBuffer对象实际上只包含了堆外内存的引用指针，在任何gc的情况下，只要确定可以被gc，他都是可以被回收的。

但是Cleaner对象不会，因为他是一个虚引用，还保存着自己双向链表中的引用，只有当jvm开始下一次gc时，虚引用对象才会被回收。

通常情况下，虚引用不会被gc回收，因为与软引用和弱引用不同，加入ReferenceQueue中的虚引用对象并不会被gc，因为其get方法永远返回null，只有在虚引用被清除或者自身变得不可访问时（手动置为null），才会被gc。

但是Cleaner对象作为一种特殊的虚引用对象，会在从pending队列移出后，主动调用自身的clean方法，将自己标记为不可达状态，并主动回收其监控的堆外内存。因此，我们关心的堆外内存就会在这个时候被回收。

所以，什么是对外内存的自动回收呢？实际上HotSpot VM只会在old gen GC（full GC/major GC或者concurrent GC都算）的时候才会对old gen中的对象做reference processing，而在young GC/minor GC时只会对young gen里的对象做reference processing。

也就是说，做full GC的话会对old gen做reference processing，进而能触发Cleaner对已死的DirectByteBuffer对象做清理工作。而如果很长一段时间里没做过GC或者只做了young GC的话则不会在old gen触发Cleaner的工作，那么就可能让本来已经死了的、但已经晋升到old gen的DirectByteBuffer关联的native memory得不到及时释放。

因此，我们要尽量的在使用完对外内存后，手动的调用cleaner的clean方法对其进行手动回收，来避免full gc一直没有发生所造成的堆外内存溢出。

此外，其实在初始化DirectByteBuffer对象时，如果当前堆外内存的条件很苛刻时，会主动调用System.gc()强制执行FGC，不过很多线上环境的JVM参数有-XX:+DisableExplicitGC，导致了System.gc()等于一个空函数，根本不会触发FGC，这一点也需要注意。

参考：https://hllvm-group.iteye.com/group/topic/27945

## 回收算法

### 分带与跨带

设计者一般至少会把Java堆划分为新生代(Young Generation)和老年代(Old Generation)两个区域。顾名思义，在新生代中，对象是朝生夕灭的，每次垃圾收集时都发现有大批对象死去，而每次回收后存活的少量对象，将会逐步晋升到老年代中存放。

但是，对象不是孤立的，对象之间会存在跨代引用。如果老年代中引用了新生代的对象，在新生代中做可达性分析时就会带上老年代，反而降低效率。

所以有了 跨代引用假说(Intergenerational Reference Hypothesis):跨代引用相对于同代引用来说仅占极少数。

依据这条假说，我们就不应再为了少量的跨代引用去扫描整个老年代，也不必浪费空间专门记录 每一个对象是否存在及存在哪些跨代引用，只需在新生代上建立一个全局的数据结构(该结构被称 为“记忆集”，Remembered Set)，这个结构把老年代划分成若干小块，标识出老年代的哪一块内存会 存在跨代引用。此后当发生Minor GC时，只有包含了跨代引用的小块内存里的对象才会被加入到GC Roots进行扫描。

一些常见的定义：
部分收集(Partial GC):指目标不是完整收集整个Java堆的垃圾收集，其中又分为:

- 新生代收集(Minor GC/Young GC):指目标只是新生代的垃圾收集。
- 老年代收集(Major GC/Old GC):指目标只是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为。
- 混合收集(Mixed GC):指目标是收集整个新生代以及部分老年代的垃圾收集。目前只有G1收集器会有这种行为。
- 整堆收集(Full GC):收集整个Java堆和方法区的垃圾收集。

Minor GC和Full GC的区别以及触发条件：

Minor GC:

对于复制算法来说，当年轻代Eden区域满的时候会触发一次Minor GC，将Eden和From Survivor的对象复制到另外一块To Survivor上。

注意：如果某个对象存活的时间超过一定Minor gc次数会直接进入老年代，不在分配到To Survivor上(默认15次，对应虚拟机参数 -XX:+MaxTenuringThreshold)。另外，如果单个 Survivor 区已经被占用了 50% (对应虚拟机参数: -XX:TargetSurvivorRatio)，那么较高复制次数的对象也会被晋升至老年代。

Full GC:

用于清理整个堆空间。它的触发条件主要有以下几种：

- 显式调用System.gc方法(建议JVM触发)。
- 方法区空间不足(JDK8及之后不会有这种情况了)
- 老年代空间不足，引起Full GC。这种情况比较复杂，有以下几种：
  - 大对象直接进入老年代引起，由-XX:PretenureSizeThreshold参数定义
  - Minor GC时，经历过多次Minor GC仍存在的对象进入老年代。上面提过，由-XX:MaxTenuringThreashold参数定义
  - Minor GC时，动态对象年龄判定机制会将对象提前转移老年代。年龄从小到大进行累加，当加入某个年龄段后，累加和超过survivor区域 * -XX:TargetSurvivorRatio的时候，从这个年龄段往上的年龄的对象进入老年代
  - Minor GC时，Eden和From Space区向To Space区复制时，大于To Space区可用内存，会直接把对象转移到老年代

JVM的空间分配担保机制可能会触发Full GC：

空间担保分配是指在发生Minor GC之前，虚拟机会检查老年代最大可用的连续空间是否大于新生代所有对象的总空间。

如果大于，则此次Minor GC是安全的。如果小于，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果HandlePromotionFailure=true，那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代的对象的平均大小，如果大于，则尝试进行一次Minor GC，但这次Minor GC依然是有风险的，失败后会重新发起一次Full gc；如果小于或者HandlePromotionFailure=false，则改为直接进行一次Full GC。

所有才会说一次Full GC很有可能是由一次Minor GC触发的。

HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC

### 标记清除算法

首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回 收所有未被标记的对象。标记过程就是对象是否属于垃圾的判定过程。

它的主要缺点有两个:

第一个是执行效率不稳定，如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低;

第二个是内存空间的碎片化问题，标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后在程序运行过程中需要分配较大对象时无法找 到足够的连续内存而不得不提前触发另一次垃圾收集动作。

![DAAXsx.png](https://s3.ax1x.com/2020/11/16/DAAXsx.png)

### 标记-复制算法

它将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。如果内存中多数对象都是存 活的，这种算法将会产生大量的内存间复制的开销，但对于多数对象都是可回收的情况，算法需要复 制的就是占少数的存活对象，而且每次都是针对整个半区进行内存回收，分配内存时也就不用考虑有空间碎片的复杂情况，只要移动堆顶指针，按顺序分配即可。这样实现简单，运行高效，不过其缺陷也显而易见，这种复制回收算法的代价是将可用内存缩小为了原来的一半，空间浪费未免太多了一点。

![DAE1ln.png](https://s3.ax1x.com/2020/11/16/DAE1ln.png)

Appel式回收: HotSpot虚拟机的Serial、ParNew等新生代收集器均采用了这种策略来设计新生代的内存布局。Appel式回收的具体做法是把新生代分为一块较大的Eden空间和两块较小的 Survivor空间，每次分配内存只使用Eden和其中一块Survivor。发生垃圾搜集时，将Eden和Survivor中仍 然存活的对象一次性复制到另外一块Survivor空间上，然后直接清理掉Eden和已用过的那块Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8∶1，也即每次新生代中可用内存空间为整个新 生代容量的90%(Eden的80%加上一个Survivor的10%)，只有一个Survivor空间，即10%的新生代是会 被“浪费”的。当Survivor空间不足以容纳一次Minor GC之后存活的对象时，就需要依赖其他内存区域(实际上大多就是老年代)进行分配担保(Handle Promotion)。

新生代上对象朝生夕灭的特点决定了它更适用复制算法，而不用担心空间的损耗。

Survivor区的意义：

如果没有survivor,Eden每进行一次minor gc，存活的对象就会进入老年代，老年代很快被填满就会进入major gc。由于老年代空间一般很大，所以进行一次gc耗时要长的多！尤其是频繁进行full GC，对程序的响应和连接都会有影响！

Survivor存在就是减少被送到老年代的对象，进而减少Full gc的发生。默认设置是经历了16次minor gc还在新生代中存活的对象才会被送到老年代。

为什么要有两个Survivor：

主要是为了解决内存碎片化和效率问题。如果只有一个Survivor时，每触发一次minor gc都会有数据从Eden放到Survivor，一直这样循环下去。注意的是，Survivor区也会进行垃圾回收，这样就会出现内存碎片化问题。

### 标记-整理算法

针对老年代对象的长时间存在的特征，“标记-整理”(Mark-Compact)算法诞生，其中的标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可 回收对象进行清理，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存。

移动存活对象，尤其是在老年代这种每次回收都有大量对象存活区域，移动存活对象并更新所有引用这些对象的地方将会是一种极为负重的操作，而且这种对象移动操作必须全程暂停用户应用程序才能进行。

HotSpot虚拟机里面关注吞吐量的Parallel Scavenge收集器是基于标记-整理算法的，而关注延迟的CMS收集器则是基于标记-清除算法的。

![DAm2DI.png](https://s3.ax1x.com/2020/11/16/DAm2DI.png)

### 根节点枚举

根节点枚举，所有收集器在这一步骤时都是必须暂停用户线程的（stop the world）。当用户线程停顿下来之后，hotspot使用一组称为OopMap的数据结构来得到哪些地方存放着对象引用。

一旦类加载动作完成的时候， HotSpot就会把对象内什么偏移量上是什么类型的数据计算出来，在即时编译过程中，也会在特定的位置记录下栈里和寄存器里哪些位置是引用。这样收集器在扫描时就可以直接得知这些信息了，并不需要真正一个不漏地从方法区等GC Roots开始查找。

### 安全点

如果为每一条指令都生成对应的OopMap，那将会需要大量的额外存储空间。HotSpot只是在“特定的位置”记录了这些信息，这些位置被称为安全点(Safepoint)。

有了安全点的设定，也就决定了用户程序执行时并非在代码指令流的任意位置都能够停顿下来开始垃圾收集，而是强制要求必须执行到达安全点后才能够暂停。

因此，安全点的选定既不能太少以至于让收集器等待时间过长，也不能太过频繁以至于过分增大运行时的内存负荷。具备“长时间执行”的特征的指令序列的复用，例如方法调用、循环跳转、异常跳转等才会产生安全点。

### 安全区域

安全区域是指能够确保在某一段代码片段之中，引用关系不会发生变化，因此，在这个区域中任 意地方开始垃圾收集都是安全的。我们也可以把安全区域看作被扩展拉伸了的安全点。

### 记忆集与卡表

记忆集是一种用于记录从非收集区域指向收集区域的指针集合的抽象数据结构。

为解决对象跨代引用所带来的问题，垃圾收集器在新生代中建立了名为记忆集(Remembered Set)的数据结构，用以避免把整个老年代加进GC Roots扫描范围。

在垃圾收集的场景中，收集器只需要通过记忆集判断出某一块非收集区域是否存在有指向了收集区域的指针。

HotSpot使用“卡精度”:每个记录精确到一块内存区域，该区域内有对象含有跨代指针。也称为称为“卡表”(Card Table)。

字节数组CARD_TABLE的每一个元素都对应着其标识的内存区域中一块特定大小的内存块，这个内存块被称作“卡页”(Card Page)。

一个卡页的内存中通常包含不止一个对象，只要卡页内有一个(或更多)对象的字段存在着跨代指针，那就将对应卡表的数组元素的值标识为1，称为这个元素变脏(Dirty)，没有则标识为0。在垃圾收集发生时，只要筛选出卡表中变脏的元素，就能轻易得出哪些卡页内存块中包含跨代指针，把它们加入GC Roots中一并扫描。

![DAuY01.png](https://s3.ax1x.com/2020/11/16/DAuY01.png)

### 写屏障

卡表状态的维护：有其他分代区域中对象引用了本区域对象时，其对应的卡表元素就应该变脏，变脏时间点原则上应该发生在引用类型字段赋值的那一刻。

在HotSpot虚拟机里是通过写屏障(Write Barrier)技术维护卡表状态的。写屏障可以看作在虚拟机层面对“引用类型字段赋值”这个动作的AOP切面，在引用对象赋值时会产生一个环形(Around)通知，供程序执行额外的动作，也就是说赋值的 前后都在写屏障的覆盖范畴内。

应用写屏障后，虚拟机就会为所有赋值操作生成相应的指令，一旦收集器在写屏障中增加了更新 卡表操作，无论更新的是不是老年代对新生代对象的引用，每次只要对引用进行更新，就会产生额外的开销。

### 并发的可达性分析

可达性分析算法理论上要求全过程都基于一个能保障一致性的快照中才能够进行分析， 这意味着必须全程冻结用户线程的运行。

我们引入三色标记(Tri-color Marking)作为工具来辅
助推导，把遍历对象图过程中遇到的对象，按照“是否访问过”这个条件标记成以下三种颜色:

- 白色:表示对象尚未被垃圾收集器访问过。显然在可达性分析刚刚开始的阶段，所有的对象都是白色的，若在分析结束的阶段，仍然是白色的对象，即代表不可达。
- 黑色:表示对象已经被垃圾收集器访问过，且这个对象的所有引用都已经扫描过。黑色的对象代表已经扫描过，它是安全存活的，如果有其他对象引用指向了黑色对象，无须重新扫描一遍。黑色对象不可能直接(不经过灰色对象)指向某个白色对象。
- 灰色:表示对象已经被垃圾收集器访问过，但这个对象上至少存在一个引用还没有被扫描过。

如果用户线程与收集器是并发工作，收集器在对象图上标记颜色，同时用户线程在修改引用关系————即修改对象图的结构，这样可能出现两种后果。一种是把原本消亡的对象错误标记为存活。另一种是把原本存活的对象错误标记为已消亡，后者十分严重，会产生对象消失。

对象消失，即原本应该是黑色的对象被误标为白色，其产生条件：

- 赋值器插入了一条或多条从黑色对象到白色对象的新引用;
- 赋值器删除了全部从灰色对象到该白色对象的直接或间接引用。

![DlGbon.png](https://s3.ax1x.com/2020/11/20/DlGbon.png)

有两种解决方法：

- 增量更新，增量更新要破坏的是第一个条件，当黑色对象插入新的指向白色对象的引用关系时，就将这个新插入的引用记录下来，等并发扫描结束之后，再将这些记录过的引用关系中的黑色对象为根，重新扫 描一次。这可以简化理解为，黑色对象一旦新插入了指向白色对象的引用之后，它就变回灰色对象了。（插入的白变灰）
- 原始快照，原始快照要破坏的是第二个条件，当灰色对象要删除指向白色对象的引用关系时，就将这个要删 除的引用记录下来，在并发扫描结束之后，再将这些记录过的引用关系中的灰色对象为根，重新扫描一次。这也可以简化理解为，无论引用关系删除与否，都会按照刚刚开始扫描那一刻的对象图快照来进行搜索。（不会删除引用）

以上无论是对引用关系记录的插入还是删除，虚拟机的记录操作都是通过写屏障实现的。在 HotSpot虚拟机中，增量更新和原始快照这两种解决方案都有实际应用，CMS是基于增量更新 来做并发标记的，G1、Shenandoah则是用原始快照来实现。

## 垃圾收集器

![DAQ36g.png](https://s3.ax1x.com/2020/11/16/DAQ36g.png)

### Serial收集器

单线程工作的收集器，它进行垃圾收集时，必须暂停其他所有工作线程，直到它收集结束。

![DAQB1U.png](https://s3.ax1x.com/2020/11/16/DAQB1U.png)

优于其他收集器的地方，那就是简单而高效(与其他收集器的单线程相比)，对于内
存资源受限的环境，它是所有收集器里额外内存消耗(Memory Footprint)最小的;对于单核处理器或处理器核心数较少的环境来说，Serial收集器没有线程交互的开销。

### Serial Old收集器

Serial Old是Serial收集器的老年代版本，使用标记整理算法。

### Parallel Scavenge收集器

吞吐量优先收集器。（吞吐量：每秒钟接待多少人；响应时间：每个人要等多长时间才能得到服务。）Parallel Scavenge收集器提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间 的-XX:MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX:GCTimeRatio参数。

垃圾收集停顿时间缩短是以牺牲吞吐量和新生代空间为代价换取的: 系统把新生代调得小一些，收集300MB新生代肯定比收集500MB快，但这也直接导致垃圾收集发生得更频繁，原来10秒收集一次、每次停顿100毫秒，现在变成5秒收集一次、每次停顿70毫秒。停顿时间 的确在下降，但吞吐量也降下来了。

### Parallel Old收集器

Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。

### ParNew收集器

ParNew收集器实质上是Serial收集器的多线程并行版本，除了同时使用多条线程进行垃圾收集之外，其余的行为包括Serial收集器可用的所有控制参数 、收集算法、Stop The World、对象分配规则、回收策略等都与Serial收集器完全一致。

除了Serial收集器外，目前只有它能与CMS收集器配合工作。自JDK 9开始，ParNew合并入CMS，成为它专门处理新生代的组成部分。

![DAlNKe.png](https://s3.ax1x.com/2020/11/16/DAlNKe.png)

### CMS收集器

CMS(Concurrent Mark Sweep)收集器是一种以获取最短回收停顿时间为目标的收集器。基于标记-清除算法实现的。整个过程分为四个步骤：

- 初始标记(CMS initial mark) ————“Stop The World”，标记GC Roots
- 并发标记(CMS concurrent mark) ————并发进行可达性分析
- 重新标记(CMS remark) ————“Stop The World”，增量更新
- 并发清除(CMS concurrent sweep)

![DANS7F.png](https://s3.ax1x.com/2020/11/16/DANS7F.png)

缺点：

- CMS收集器对处理器资源非常敏感。事实上，面向并发设计的程序都对处理器资源比较敏感。在并发阶段，它虽然不会导致用户线程停顿，但却会因为占用了一部分线程(或者说处理器的计算能力)而导致应用程序变慢，降低总吞吐量。
- 由于CMS收集器无法处理“浮动垃圾”(FloatingGarbage)。在CMS的并发标记和并发清理阶段，用户线程是还在继续运行的，程序在运行自然就还会伴随有新的垃圾对象不断产生，但这一部分垃圾对象是出现在标记过程结束以后，CMS无法在当次收集中处理掉它们，只好留待下一次垃圾收集时再清理掉。这一部分垃圾就称为“浮动垃圾”。
- 可能出现“Concurrent Mode Failure”失败进而导致另一次完全“Stop The World”的Full GC的产生。由于在垃圾收集阶段用户线程还需要持续运行，那就还需要预留足够内存空间提供给用户线程使用，因此CMS收集器不能像其他收集器那样等待到老年代几乎完全被填满了再进行收集，必须预留一部分空间供并发收集时的程序运作使用。要是CMS运行期间预留的内存无法满足程序分配新对象的需要，就会出现一次“并发失败”(Concurrent Mode Failure)，这时候虚拟机将不得不启动后备预案:冻结用户线程的执行，临时启用Serial Old收集器来重新进行老年代的垃圾收集，但这样停顿时间就很长了。
- CMS是一款基于“标记-清除”算法实现的收集器，这意味着收集结束时会有大量空间碎片产生。空间碎片过多时，将会给大对象分配带来很大麻烦，往往会出现老年代还有很多剩余空间，但就是无法找到足够大的连续空间来分配当前对象，而不得不提前触发一次Full GC的情况。

### g1收集器

全堆收集器，衡量标准不再是它属于哪个分代，而是哪块内存中存放的垃圾数量最多，回收收益最大，这就是G1收集器的Mixed GC模式。

基于Region的堆内存布局，连续的Java堆划分为多个大小相等的独立区域(Region)，每一个Region都可以根据需要，扮演新生代的Eden空间、Survivor空间，或者老年代空间。虽然G1仍然保留新生代和老年代的概念，但新生代和老年代不再是固定的了，它们都是一系列区 域(不需要连续)的动态集合。收集器能够对扮演不同角色的Region采用不同的策略去处理。

每个Region的大小可以通过参数-XX:G1Heap RegionSize设定，取值范围为1M B~32MB，且应为2的N次幂。而对于那些超过了整个Region容量的超级大对象，将会被存放在N个连续的Humongous Region之中，G1的大多数行为都把Humongous Region作为老年代 的一部分来进行看待。

g1将Region作为单次回收的最小单元，即每次收集到的内存空间都是Region大小的整数倍，这样可以有计划地避免在整个Java堆中进行全区域的垃圾收集。更具体的处理思路是让G1收集器去跟踪各个Region里面的垃圾堆积的“价值”大小，价值即回收所获得的空间大小以及回收所需时间的经验值，然后在后台维护一 个优先级列表，每次根据用户设定允许的收集停顿时间(使用参数-XX:MaxGCPauseMillis指定，默认值是200毫秒)，优先处理回收价值收益最大的那些Region。

跨Region使用特殊的记忆集（双向卡表，Key是别的Region的起始地址，Value是一个集合，里面存储的元素是卡表的索引号。）通过写屏障来维护记忆集能处理跨代指针，得以实现Region的增量回收。

![DAajXt.png](https://s3.ax1x.com/2020/11/16/DAajXt.png)

G1收集器的运作过程大致可划分为以下四个步骤:

- 初始标记(Initial Marking):仅仅只是标记一下GC Roots能直接关联到的对象（stop the world）
- 并发标记(Concurrent Marking):从GC Root开始对堆中对象进行可达性分析，通过原始快照来处理并发问题。
- 最终标记(Final Marking):对用户线程做另一个短暂的暂停，用于处理并发阶段结束后仍遗留 下来的最后那少量的原始快照记录。（stop the world）
- 筛选回收(Live Data Counting and Evacuation):负责更新Region的统计数据，对各个Region的回收价值和成本进行排序，根据用户所期望的停顿时间来制定回收计划，可以自由选择任意多个Region构成回收集（整理），然后把决定回收的那一部分Region的存活对象复制到空的Region中（复制），再清理掉整个旧Region的全部空间。这里的操作涉及存活对象的移动，是必须暂停用户线程，由多条收集器线程并行完成的。（stop the world）

![DAdYB6.png](https://s3.ax1x.com/2020/11/16/DAdYB6.png)

G1和CMS的比较:

- G1从整体上看是“标记-整理”算法，从局部（两个Region之间）上看是“标记-复制”算法，不会产生内存碎片，而CMS基于“标记-清除”算法会产生内存碎片。
- G1在垃圾收集时产生的内存占用和程勋运行时的额外负载都比CMS高
- G1支持动态指定停顿时间，而CMS无法指定
- 两者都利用了并发标记这个技术

### ZGC收集器

ZGC收集器是一款基于Region内存布局的，(暂时) 不设分代的，使用了读屏障、染色指针和内存多重映射等技术来实现可并发的标记-整理算法的，以低延迟为首要目标的一款垃圾收集器。接下来，笔者将逐项来介绍ZGC的这些技术特点。

（有点复杂，不展开了，记住Region分为大中小，整个收集过程都全程可并发，短暂停顿也只与GC Roots大小相关而与堆内存大小无关，完全没有使用记忆集，它甚至连分代都没有，不需要维护记忆集的写屏障，而用读屏障）

## gc调优

单开一篇文章说吧，这一篇太长了