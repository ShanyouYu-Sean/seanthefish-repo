---
layout: post
title: jvm与并发
date: 2020-11-18 10:00:00
tags: 
- 并发
categories:
- 并发
---

## Java的线程实现

Java的线程是Linux中的内核线程，即轻量级进程。

程序一般不会直接使用内核线程，而是使用内核线程的一种高级接口——轻量级进程(Light Weight Process，LWP)，轻量级进程就是我们通常意义上所讲的线程，由于每个轻量级进程都由一个 内核线程支持，因此只有先支持内核线程，才能有轻量级进程。这种轻量级进程与内核线程之间1:1 的关系称为一对一的线程模型。

浅谈Linux线程模型
https://zhuanlan.zhihu.com/p/57349087

## Java中的内存并发模型

“内存模型”，它可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象。

Java内存模型规定了所有的变量都存储在主内存(Main Memory)中。每条线程还有自己的工作内存(Working Memory ，可与前面讲的处理器高速缓存类比)，线程的工作内存中保存了被该线程使用的变量的主内存副本，线程对变量的所有操作(读取、赋值等)都必须在工作内存中进行，而不能直接读写主内存中的数据。

此处的变量(Variables)与Java编程中所说的变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但是不包括局部变量与方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。

![DQQT2Q.png](https://s3.ax1x.com/2020/11/20/DQQT2Q.png)

Java内存模型中定义了以下8种操作来完成。Java虚拟机实 现时必须保证下面提及的每一种操作都是原子的、不可再分的：

- lock(锁定):作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
- unlock(解锁):作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read(读取):作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用。
- load(载入):作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中。
- use(使用):作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作。
- assign(赋值):作用于工作内存的变量，它把一个从执行引擎接收的值赋给工作内存的变量， 每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store(存储):作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随 后的write操作使用。
- write(写入):作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。

## 并发的三个特性

- 原子性（注意 “long和double的非原子性协定”）
- 可见性
- 有序性

## volatile

当一个变量被定义成volatile之后，它将具备两项特性:

#### 保证可见性

第一项是保证此变量对所有线程的可见性，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量并不能做到这一点，普通变量的值在线程间传递时均需要通过主内存来完成。比如，线程A修改一个普通变量的值，然后向主内存进行回写，另外一条线程B在线程A回写完成了之后再对主内存进行读取操作，新变量值才会对线程B可见。（volatile变量在各个线程的工作内存中是不存在一致性问题的，但是Java里面的运算操作符并非原子操作， 这导致volatile变量的运算在并发下一样是不安全的）

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁 (使用synchronized、java.util.concurrent中的锁或原子类)来保证原子性:

- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 变量不需要与其他的状态变量共同参与不变约束。

#### 通过内存屏障避免指令重排

什么是指令重排

在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待3。通过乱序执行的技术，处理器可以大大提高执行效率。

普通的变量仅会保证，在该方法的执行过程中，所有依赖赋值结果的地方都能获取到正确的结果，而不能保证，变量赋值操作的顺序与程序代码中的执行顺序一致。这就是Java内存模型中描述的所谓“线程内表现为串行的语义”(As-If-Serial)，即所有的动作都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。

Java内存模型关于重排序的规定，总结后如下表所示：

![DQYX6S.png](https://s3.ax1x.com/2020/11/20/DQYX6S.png)

什么是内存屏障

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序。

内存屏障可以被分为以下几种类型：

- LoadLoad屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。
- StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。
- LoadStore屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。
- StoreLoad屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

对于Java编译器而言，Intel 64/IA-32架构下处理器不需要LoadLoad、LoadStore、StoreStore屏障，因为不会发生需要这三种屏障的重排序。

Java编译器会这样使用内存屏障：

![DQYzwj.png](https://s3.ax1x.com/2020/11/20/DQYzwj.png)

## synchronized

在Java里面，最基本的互斥同步手段就是synchronized关键字，这是一种块结构(Block Structured)的同步语法。synchronized关键字经过Javac编译之后，会在同步块的前后分别形成
monitorenter和monitorexit这两个字节码指令。这两个字节码指令都需要一个reference类型的参数来指明要锁定和解锁的对象。如果Java源码中的synchronized明确指定了对象参数，那就以这个对象的引用作为reference;如果没有明确指定，那将根据synchronized修饰的方法类型，来决定是取代码所在的对象实例还是取类型对应的Class对象来作为线程要持有的锁。

在执行monitorenter指令时，首先要去尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经持有了那个对象的锁，就把锁的计数器的值增加一，而在执行monitorexit指令时会将锁计数器的值减一。一旦计数器的值为零，锁随即就被释放了。如果获取对象锁失败，那当前线程就应当被阻塞等待，直到请求锁定的对象被持有它的线程释放为止。

- 被synchronized修饰的同步块对同一条线程来说是可重入的。这意味着同一线程反复进入同步块也不会出现自己把自己锁死的情况。
- 被synchronized修饰的同步块在持有锁的线程执行完毕并释放锁之前，会无条件地阻塞后面其他线程的进入。

对象锁和类锁

- 对象锁锁住的是非静态变量，this 对象或非静态方法，使用对象锁的情况，只有使用同一实例的线程才会受锁的影响，多个实例调用同一方法也不会受影响。
- 类锁锁住的是类中的静态变量，静态方法或xxx.class，类锁是所有线程共享的锁，所以同一时刻，只能有一个线程使用加了锁的方法或方法体，不管是不是同一个实例。

使用synchronized是，请将锁住的变量设为final，why：

```java
private String color = "red";

private void doSomething(){
  synchronized(color) {  // lock is actually on object instance "red" referred to by the color variable
    //...
    color = "green"; // other threads now allowed into this block
    // ...
  }
}
```

加一点解释（自己的理解，不一定准确），在Linux中其实没有线程的，只有进程，我们也知道Java的线程是有进程产生的，但问题是Linux进程只有waiting这种状态，没有block，也就是说，我们在Linux层面讨论，阻塞，挂起，其实都是进入了waiting状态，Java实际上是在线程等待锁的时候自己设置了一个block的状态，这个状态实际上是有Linux的mutex lock实现的，关于mutex lock详见 https://www.cnblogs.com/huangchaosong/p/7127532.html 。简单来说，进程会自旋进行计数器的cas来试图获取锁，获取锁的进程就进入临界区了，没有获取到锁的进程会停止自旋，进入waiting状态，并将进程标识为不可中断（这也是Synchronized不能被中断的原因），并进入waiting队列，等待锁释放的信号。在得到锁释放的信号后，会重新进行上面的操作试图获取锁。这个过程就是Java线程的block状态。

## happens-before

- 程序次序规则: 在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。注意，这里说的是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- 管程锁定规则: 一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是“同一个锁”，而“后面”是指时间上的先后。
- volatile变量规则: 对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后。
- 线程启动规则: Thread对象的start()方法先行发生于此线程的每一个动作。
- 线程终止规则: 线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过`Thread::join()`方法是否结束、`Thread::isAlive()`的返回值等手段检测线程是否已经终止执行。
- 线程中断规则: 对线程`interrupt()`方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过`Thread::interrupted()`方法检测到是否有中断发生。
- 对象终结规则:一个对象的初始化完成(构造函数执行结束)先行发生于它的`finalize()`方法的开始。
- 传递性: 如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。

## java线程

java是使用内核态线程的抢占式模型。

Java线程实现：继承thread类和实现runnable接口，有啥区别吗？没啥区别，Thread也是实现了Runnable接口，提供了更多线程状态控制转换的方法罢了。

#### 状态转换

- 新建(New): 创建后尚未启动的线程处于这种状态。
- 运行(Runnable): 包括操作系统线程状态中的Running和Ready也就是处于此状态的线程有可能正在执行，也有可能正在等待着操作系统为它分配执行时间。
- 无限期等待(Waiting): 处于这种状态的线程不会被分配处理器执行时间，它们要等待被其他线程显式唤醒。以下方法会让线程陷入无限期的等待状态:
  - 没有设置Timeout参数的`Object::wait()`方法;
  - 没有设置Timeout参数的`Thread::join()`方法;
  - `LockSupport::park()`方法。
- 限期等待(Timed Waiting): 处于这种状态的线程也不会被分配处理器执行时间，不过无须等待被其他线程显式唤醒，在一定时间之后它们会由系统自动唤醒。以下方法会让线程进入限期等待状态:
  - `Thread::sleep()`方法;
  - 设置了Timeout参数的`Object::wait()`方法;
  - 设置了Timeout参数的`Thread::join()`方法;
  - `LockSupport::parkNanos()`方法;
  - `LockSupport::parkUntil()`方法。
- 阻塞(Blocked):线程被阻塞了，“阻塞状态”与“等待状态”的区别是“阻塞状态”在等待着获取到一个锁，这个事件将在另外一个线程放弃这个锁的时候发生;而“等待状态”则是在等待一段时间，或者唤醒动作的发生。在程序等待进入同步区域的时候，线程将进入这种状态。
- 结束(Terminated):已终止线程的线程状态，线程已经结束执行。

![DQg8wq.png](https://s3.ax1x.com/2020/11/20/DQg8wq.png)

![DJTGa6.png](https://s3.ax1x.com/2020/11/23/DJTGa6.png)

#### 线程中断

感觉必须要加一段线程中断和响应中断的详解，不然后续对aqs理解会有问题：

Java没有提供任何机制来安全地终止线程，但提供了中断机制，即thread.interrupt()方法。线程中断是一种协作式的机制，并不是说调用了中断方法之后目标线程一定会立即中断，而是发送了一个中断请求给目标线程，目标线程会自行在某个取消点中断自己。这种设定很有必要，因为如果不论线程执行到何种情况都立即响应中断的话，很容易造成某些对象状态不一致的情况出现。

每个线程对象里都有一个 boolean 类型的标识（不一定就要是 Thread 类的字段，实际上也的确不是，这几个方法最终都是通过 native 方法来完成的），代表着是否有中断请求（该请求可以来自所有线程，包括被中断的线程本身）。例如，当线程 t1 想中断线程 t2，只需要在线程 t1 中将线程 t2 对象的中断标识置为 true，然后线程 2 可以选择在合适的时候处理该中断请求，甚至可以不理会该请求，就像这个线程没有被中断一样。

涉及到中断的线程基础方法有三个：interrupt()、isInterrupted()、interrupted()，它们都位于Thread类下。

- interrupt()方法：对目标线程发送中断请求，看其源码会发现最终是调用了一个本地方法实现的线程中断；

- interrupted()方法：返回目标线程是否中断的布尔值（通过本地方法实现），且返回后会重置中断状态为未中断；换句话说，如果连续两次调用该方法，则第二次调用将返回 false（在第一次调用已清除了其中断状态之后，且第二次调用检验完中断状态前，当前线程再次中断的情况除外）。

- isInterrupted()方法：该方法返回的是线程中断与否的布尔值（通过本地方法实现），不会重置中断状态；

其中，interrupt 方法是唯一能将中断状态设置为 true 的方法。静态方法 interrupted 会将当前线程的中断状态清除，但这个方法的命名极不直观，很容易造成误解，需要特别注意。

Thread.interrupt()方法仅仅是在当前线程中打了一个停止的标识将中断标志修改为true，并没有真正的停止线程。如果在此基础上进入waiting状态（sleep(),wait(),join()）,马上就会抛出一个InterruptedException，且中断标志被清除，重新设置为false。记住一点，当调用Thread.interrupt()，还没有进行中断时，此时的中断标志位是true，当发生中断之后（执行到sleep(),wait(),join()），这个时候的中断标志位就是false了。

中断是通过调用Thread.interrupt()方法来做的. 这个方法通过修改了被调用线程的中断状态来告知那个线程, 说它被中断了. 对于非waiting中的线程, 只是改变了中断状态, 即Thread.isInterrupted()将返回true; 对于可取消的waiting状态中的线程, 比如等待在这些函数上的线程, Thread.sleep(), Object.wait(), Thread.join(), 这个线程收到中断信号后, 会抛出InterruptedException, 同时会把中断状态置回为false.但调用Thread.interrupted()会对中断状态进行复位。

#### 响应中断的方法和不响应中断的方法

响应中断的方法： 线程进入等待或是超时等待的状态后，调用interrupt方法都是会响应中断的，所以响应中断的方法：Object.wait()、Thread.join、Thread.sleep、LockSupport.park的有参和无参方法。

不响应中断的方法：线程进入阻塞状态后，是不响应中断的，等待进入synchronized的方法或是代码块，都是会被阻塞的，此时不会响应中断，另外还有一个不响应中断的，那就是阻塞在ReentrantLock.lock方法里面的线程，也是不响应中断的，如果想要响应中断，可以使用ReentrantLock.lockInterruptibly方法。

其ReentrantLock底层是使用LockSupport.park方法进行等待的，前面说了LockSupport.park是响应中断的。

当调用interrupt方法时，会把中断状态设置为true，然后park方法会去判断中断状态，如果为true，就直接返回，然后往下继续执行，并不会抛出异常。注意，这里并不会清除中断标志。（Object.wait()、Thread.join、Thread.sleep方法则会清除中断状态）

当线程进入ReentrantLock.lock方法里面进行阻塞后，此时调用Thread.interrupt()方法之后，该线程是会被中断被唤醒的，但是唤醒之后，会调用LockSupport.park再次进入等待状态，所以仅从宏观（表面）上面看ReentrantLock.lock是不支持响应中断的，从微观（原理）上面讲ReentrantLock.lock内部确实中断了响应，但是还是会被迫进行等待状态。（当然，如果这期间抢到锁了，会在aqs里调用selfInterrupt补一下中断状态）

![image.png](https://i.loli.net/2020/11/25/osDpLKSBgYuPt4C.png)

## java锁分类

![DQg6k6.png](https://s3.ax1x.com/2020/11/20/DQg6k6.png)

synchronized是一个重量级的，阻塞的，非公平的，可重入的，排他的，悲观锁。

无锁即使cas。Java中的CAS操作，由sun.misc.Unsafe类里面的`compareAndSwapInt()`和`compareAndSwapLong()`等几个方法包装提供。HotSpot虚拟机在内部对这些方法做了特殊处理，即时编译出来的结果就是一条平台相关的处理器CAS指令，没有方法调用的过程，或者可以认为是无条件内联进去了。

当cas与自旋锁结合，便成为了原子类和aqs的核心，cas自旋做形成了不阻塞的锁。

自适应意味着自旋的时间（次数）不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋等待持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略掉自旋过程，直接阻塞线程，避免浪费处理器资源。

synchronized已经详细说过，cas自旋锁我会在接下来的文章里详细说，这里主要再仔细说一下偏向锁 和轻量级锁。

#### 轻量级锁

Java对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。

Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。

Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

![DQfW5t.png](https://s3.ax1x.com/2020/11/20/DQfW5t.png)

在代码即将进入同步块的时候，如果此同步对象没有被锁定(锁标志位为“01”状态)，虚拟机首先将在当前线程的栈帧中建立一个名为锁记录(Lock Record)的空间，用于存储锁对象目前的Mark Word的拷贝。然后，虚拟机将使用CAS操作尝试把对象的Mark Word更新为指向Lock Record的指针。

如果这个更新动作成功了，即代表该线程拥有了这个对象的锁，并且对象Mark Word的锁标志位(Mark Word的最后两个比特)将转变为“00”，表示此对象处于轻量级锁定状态。

如果这个更新操作失败了，那就意味着至少存在一条线程与当前线程竞争获取该对象的锁。虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了这个对象的锁，那直接进入同步块继续执行就可以了，否则就说明这个锁对象已经被其他线程抢占了。如果 出现两条以上的线程争用同一个锁的情况，那轻量级锁就不再有效，必须要膨胀为重量级锁，锁标志的状态值变为“10”，此时Mark Word中存储的就是指向重量级锁(互斥量)的指针，后面等待锁的线程也必须进入阻塞状态。

轻量级锁能提升程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”这一经验法则。如果没有竞争，轻量级锁便通过CAS操作成功避免了使用互斥量的开销;但如果确实存在锁竞争，除了互斥量的本身开销外，还额外发生了CAS操作的开销。因此在有竞争的情况下，轻量级锁反而会比传统的重量级锁更慢。

#### 偏向锁

偏向锁中的“偏”，就是偏心的“偏”、偏袒的“偏”。它的意思是这个锁会偏向于第一个获得它的线 程，如果在接下来的执行过程中，该锁一直没有被其他的线程获取，则持有偏向锁的线程将永远不需要再进行同步。

假设当前虚拟机启用了偏向锁，那么当锁对象第一次被线程获取的时候，虚拟机将会把对象头中的标志位设置为“01”、把偏向模式设置为“1”，表示进入偏向模式。同时使用CAS操作把获取到这个锁的线程的ID记录在对象的Mark Word之中。如果CAS操作成功，持有偏向锁的线程以后每次进入这个锁相关的同步块时，虚拟机都可以不再进行任何同步操作(例如加锁、解锁及对Mark Word的更新操作等)。

一旦出现另外一个线程去尝试获取这个锁的情况，偏向模式就马上宣告结束。根据锁对象目前是否处于被锁定的状态决定是否撤销偏向(偏向模式设置为“ 0”)，撤销后标志位恢复到未锁定(标志位 为“01”)或轻量级锁定(标志位为“00”)的状态，后续的同步操作就按照上面介绍的轻量级锁那样去执行。

![DQoPl6.png](https://s3.ax1x.com/2020/11/20/DQoPl6.png)
