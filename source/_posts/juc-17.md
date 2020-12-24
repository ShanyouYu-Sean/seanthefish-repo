---
layout: post
title: juc之ThreadPoolExecutor
date: 2020-12-25 11:00:00
tags: 
- 并发
categories:
- 并发
---

## 类图

![image.png](https://i.loli.net/2020/11/25/2F5etwLdmrR9fTn.png)

## 线程池的主要处理流程

![image.png](https://i.loli.net/2020/11/25/i8es5gX9SJwD7GC.png)

根据返回的对象类型创建线程池可以分为三类：

- 创建返回ThreadPoolExecutor对象

- 创建返回ScheduleThreadPoolExecutor对象

- 创建返回ForkJoinPool对象
  
## ThreadPoolExecutor

在介绍Executors创建线程池方法前先介绍一下ThreadPoolExecutor，因为这些创建线程池的静态方法都是返回ThreadPoolExecutor对象，和我们手动创建ThreadPoolExecutor对象的区别就是我们不需要自己传构造函数的参数。

ThreadPoolExecutor的构造函数共有四个，但最终调用的都是同一个：

```java
public ThreadPoolExecutor(int corePoolSize, // 线程池核心线程数量
                          int maximumPoolSize, // 线程池最大数量
                          long keepAliveTime, // 空闲线程存活时间
                          TimeUnit unit, // 时间单位
                          BlockingQueue<Runnable> workQueue, // 线程池所使用的缓冲队列
                          ThreadFactory threadFactory, // 线程池创建线程使用的工厂
                          RejectedExecutionHandler handler // 线程池对拒绝任务的处理策略
                          );
```

### Executors#newCachedThreadPool方法

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

- corePoolSize => 0，核心线程池的数量为0
- maximumPoolSize => Integer.MAX_VALUE，可以认为最大线程数是无限的
- keepAliveTime => 60L
- unit => 秒
- workQueue => SynchronousQueue
  
当一个任务提交时，corePoolSize为0不创建核心线程，SynchronousQueue是一个不存储元素的队列，可以理解为队里永远是满的，因此最终会创建非核心线程来执行任务。
对于非核心线程空闲60s时将被回收。因为Integer.MAX_VALUE非常大，可以认为是可以无限创建线程的，在资源有限的情况下容易引起OOM异常

### Executors#newSingleThreadExecutor方法

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
- corePoolSize => 1，核心线程池的数量为1
- maximumPoolSize => 1，只可以创建一个非核心线程
- keepAliveTime => 0L
- unit => 秒
- workQueue => LinkedBlockingQueue

当一个任务提交时，首先会创建一个核心线程来执行任务，如果超过核心线程的数量，将会放入队列中，因为LinkedBlockingQueue是长度为Integer.MAX_VALUE的队列，可以认为是无界队列，因此往队列中可以插入无限多的任务，在资源有限的时候容易引起OOM异常，同时因为无界队列，maximumPoolSize和keepAliveTime参数将无效，压根就不会创建非核心线程

## Executors#newFixedThreadPool方法

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

- corePoolSize => 1，核心线程池的数量为1
- maximumPoolSize => 1，只可以创建一个非核心线程
- keepAliveTime => 0L
- unit => 秒
- workQueue => LinkedBlockingQueue

它和SingleThreadExecutor类似，唯一的区别就是核心线程数不同，并且由于使用的是LinkedBlockingQueue，在资源有限的时候容易引起OOM异常

总结：
FixedThreadPool和SingleThreadExecutor => 允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而引起OOM异常
CachedThreadPool => 允许创建的线程数为Integer.MAX_VALUE，可能会创建大量的线程，从而引起OOM异常
这就是为什么禁止使用Executors去创建线程池，而是推荐自己去创建ThreadPoolExecutor的原因

顺便说一下ScheduledThreadPoolExecutor，就不分析源码了：

ScheduledThreadPoolExecutor继承ThreadPoolExecutor来重用线程池的功能，它的实现方式如下：

- 将任务封装成ScheduledFutureTask对象，ScheduledFutureTask基于相对时间，不受系统时间的改变所影响；
- ScheduledFutureTask实现了java.lang.Comparable接口和java.util.concurrent.Delayed接口，所以有两个重要的方法：compareTo和getDelay。compareTo方法用于比较任务之间的优先级关系，如果距离下次执行的时间间隔较短，则优先级高；getDelay方法用于返回距离下次任务执行时间的时间间隔；
- ScheduledThreadPoolExecutor定义了一个DelayedWorkQueue，它是一个有序队列，会通过每个任务按照距离下次执行时间间隔的大小来排序；
- ScheduledFutureTask继承自FutureTask，可以通过返回Future对象来获取执行的结果。

如何定义线程池参数:

- CPU密集型 => 线程池的大小推荐为CPU数量 + 1，CPU数量可以根据Runtime.availableProcessors方法获取
- IO密集型 => CPU数量 * CPU利用率 * (1 + 线程等待时间/线程CPU时间)
- 混合型 => 将任务分为CPU密集型和IO密集型，然后分别使用不同的线程池去处理，从而使每个线程池可以根据各自的工作负载来调整
- 阻塞队列 => 推荐使用有界队列，有界队列有助于避免资源耗尽的情况发生
- 拒绝策略 => 
  - 直接丢弃（DiscardPolicy）
  - 丢弃队列中最老的任务(DiscardOldestPolicy)。
  - 抛异常(AbortPolicy)
  - 将任务分给调用线程来执行(CallerRunsPolicy)。
  默认采用的是AbortPolicy拒绝策略，直接在程序中抛出RejectedExecutionException异常【因为是运行时异常，不强制catch】，这种处理方式不够优雅。处理拒绝策略有以下几种比较推荐：
  - 在程序中捕获RejectedExecutionException异常，在捕获异常中对任务进行处理。针对默认拒绝策略
  - 使用CallerRunsPolicy拒绝策略，该策略会将任务交给调用execute的线程执行【一般为主线程】，此时主线程将在一段时间内不能提交任何任务，从而使工作线程处理正在执行的任务。此时提交的线程将被保存在TCP队列中，TCP队列满将会影响客户端，这是一种平缓的性能降低
  - 自定义拒绝策略，只需要实现RejectedExecutionHandler接口即可
  - 如果任务不是特别重要，使用DiscardPolicy和DiscardOldestPolicy拒绝策略将任务丢弃也是可以的
- 如果使用Executors的静态方法创建ThreadPoolExecutor对象，可以通过使用Semaphore对任务的执行进行限流也可以避免出现OOM异常

## 源码

### 核心属性

```java
// 状态控制属性：高3位表示线程池的运行状态，剩下的29位表示当前有效的线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

// 线程池的基本大小，当提交一个任务到线程池时，线程池会创建一个线程来执行任务，
// 即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于
// 线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法，
// 线程池会提前创建并启动所有基本线程。
private volatile int corePoolSize;

// 线程池线程最大数量，如果队列满了，并且已创建的线程数小于最大线程数，
// 则线程池会再创建新的线程执行任务。如果使用了无界的任务队列这个参数就没什么效果。
private volatile int maximumPoolSize;

// 用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设 置更有意义的名字。
private volatile ThreadFactory threadFactory;

// 饱和策略，默认情况下是AbortPolicy。
private volatile RejectedExecutionHandler handler;

// 线程池的工作线程空闲后，保持存活的时间。如果任务很多，并且每个任务执行的时间比较短，
// 可以调大时间，提高线程的利用率。
private volatile long keepAliveTime;

// 用于保存等待执行的任务的阻塞队列，具体可以参考[JAVA并发容器-阻塞队列](https://www.jianshu.com/p/5646fb5faee1)
private final BlockingQueue<Runnable> workQueue;

// 存放工作线程的容器，必须获取到锁才能访问
private final HashSet<Worker> workers = new HashSet<Worker>();

// ctl的拆包和包装
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }

// 阻塞队列参考之前的文章
// ctl状态控制属性，高3位表示线程池的运行状态（runState），剩下的29位表示当前有效的线程数量（workerCount）
// 线程池最大线程数是(1 << COUNT_BITS) - 1 = 536 870 911
```

### 线程池的运行状态runState

|状态|解释|
|-|-|
|RUNNING|运行态，可处理新任务并执行队列中的任务|
|SHUTDOW|关闭态，不接受新任务，但处理队列中的任务|
|STOP|停止态，不接受新任务，不处理队列中任务，且打断运行中任务|
|TIDYING|整理态，所有任务已经结束，workerCount = 0 ，将执行terminated()方法|
|TERMINATED|结束态，terminated() 方法已完成|

![image.png](https://i.loli.net/2020/11/25/kdPgAxHU8evOSsn.png)

### 核心内部类 Worker

```JAVA

```private final class Worker  extends AbstractQueuedSynchronizer  implements Runnable {
    // 正在执行任务的线程
    final Thread thread;
    // 线程创建时初始化的任务
    Runnable firstTask;
    // 完成任务计数器
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        // 在runWorker方法运行之前禁止中断，要中断线程必须先获取worker内部的互斥锁
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** delegates main run loop to outer runworker  */
    // 直接委托给外部runworker方法
    public void run() {
        runWorker(this);
    }
    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    // 重写的aqs方法，直接cas
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    // 重写方法，直接release
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
}
```

Worker 类将执行任务的线程封装到了内部，在初始化Worker 的时候，会调用ThreadFactory初始化新线程；Worker 继承了AbstractQueuedSynchronizer，在内部实现了一个互斥锁，主要目的是控制工作线程的中断状态。

线程的中断一般是由其他线程发起的，比如`ThreadPoolExecutor#interruptIdleWorkers(boolean)`方法，它在调用过程中会去中断worker内部的工作线程，Work的互斥锁可以保证正在执行的任务不被打断。它是怎么保证的呢？在线程真正执行任务的时候，也就是runWorker方法被调用时，它会先获取到Work的锁，当我们在其他线程需要中断当前线程时也需要获取到work的互斥锁，否则不能中断。

### 构造函数

前文说过了
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

### execute() 提交线程

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    // 获取控制的值
    int c = ctl.get();
    // 判断工作线程数是否小于corePoolSize
    if (workerCountOf(c) < corePoolSize) {
        // 新创建核心线程
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 工作线程数大于或等于corePoolSize
    // 判断线程池是否处于运行状态，如果是将任务command入队
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池的运行状态，如果不在运行中，那么将任务从队列里面删除，并尝试结束线程池
        if (!isRunning(recheck) && remove(command))
            // 调用驱逐策略
            reject(command);
        // 检查活跃线程总数是否为0
        else if (workerCountOf(recheck) == 0)
            // 新创建非核心线程
            addWorker(null, false);
    }
    // 队列满了，新创建非核心线程
    else if (!addWorker(command, false))
        // 调用驱逐策略
        reject(command);
}
```

excute()方法中添加任务的方式是使用addWorker（）方法。

### addWorker() 新创建线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层循环，用于判断线程池状态
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 仅在必要的时候检查队列是否为NULL
        // 检查队列是否处于非运行状态
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            // 内层的循环，任务是将worker数量加1
            // 获取活跃线程数
            int wc = workerCountOf(c);
            // 判断线程是否超过最大值，当队列满了则验证线程数是否大于maximumPoolSize，
            // 没有满则验证corePoolSize
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 增加活跃线程总数，否则重试
            if (compareAndIncrementWorkerCount(c))
                // 如果成功跳出外层循环
                break retry;
            c = ctl.get();  // Re-read ctl
            // 再次校验一下线程池运行状态
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
// worker加1后，接下来将woker添加到HashSet<Worker> workers中，并启动worker
    // 工作线程是否启动
    boolean workerStarted = false;
    // 工作线程是否创建
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 新创建线程
        w = new Worker(firstTask);
        // 获取新创建的线程
        final Thread t = w.thread;
        if (t != null) {
            // 创建线程要获得全局锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                // 检查线程池的运行状态
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 检查线程的状态
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 将新建工作线程存放到容器
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize) {
                        // 跟踪线程池最大的工作线程总数
                        largestPoolSize = s;
                    }
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动工作线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            // 启动新的工作线程失败，
            // 1. 将工作线程移除workers容器
            // 2. 还原工作线程总数（workerCount）
            // 3. 尝试结束线程
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

如果启动新线程失败那么addWorkerFailed()这个方法将做一下三件事：

- 将工作线程移除workers容器
- 还原工作线程总数（workerCount）
- 尝试结束线程

### execute() 执行过程

![image.png](https://i.loli.net/2020/11/25/ATZyz3X4kJv9wR1.png)

- 如果当前运行的线程少于corePoolSize，即使有空闲线程也会创建新线程来执行任务，（注意，执行这一步骤 需要获取全局锁）。如果调用了线程池的restartAllCoreThreads()方法， 线程池会提前创建并启动所有基本线程。
- 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。
- 如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执
行这一步骤需要获取全局锁）。
- 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用 RejectedExecutionHandler.rejectedExecution()方法。

### 线程任务的执行

线程的正在执行是`ThreadPoolExecutor.Worker#run()`方法，但是这个方法直接委托给了外部的runWorker()方法，源码如下：

```java
// 直接委托给外部runworker方法
public void run() {
    runWorker(this);
}
```

### runWorker() 执行任务

```java
final void runWorker(Worker w) {
    // 当前Work中的工作线程
    Thread wt = Thread.currentThread();
    // 获取初始任务
    Runnable task = w.firstTask;
    // 初始任务置NULL(表示不是建线程)
    w.firstTask = null;
    // 修改锁的状态，使需发起中断的线程可以获取到锁（使工作线程可以响应中断）
    // 因为worker初始化的时候setState(-1); 
    // 根据注释, 这样做的原因是为了抑制工作线程的 interrupt 信号, 直到此工作线程正是开始执行 task. 
    // 那么在 addWorker 中的 w.unlock(); 就是允许 Worker 的 interrupt 信号.
    w.unlock(); // allow interrupts
    // 工作线程是否是异常结束
    boolean completedAbruptly = true;
    try {
        // 循环的从队列里面获取任务
        while (task != null || (task = getTask()) != null) {
            // 每次执行任务时需要获取到内置的互斥锁
            w.lock();
            // 1. 当前工作线程不是中断状态，且线程池是STOP,TIDYING,TERMINATED状态，我们需要中断当前工作线程
            // 2. 当前工作线程是中断状态，且线程池是STOP,TIDYING,TERMINATED状态，我们需要中断当前工作线程
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)))
                    && !wt.isInterrupted())
                // 中断线程，中断标志位设置成true
                wt.interrupt();
            try {
                // 执行任务前置方法,扩展用
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行任务后置方法,扩展用
                    afterExecute(task, thrown);
                }
            } finally {
                // 任务NULL表示已经处理了
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 队列里没任务的时候
        // 将工作线程从容器中剔除
        processWorkerExit(w, completedAbruptly);
    }
}
```

正在执行线程的方法，执行流程：

- 获取到当前的工作线程
- 获取初始化的线程任务
- 修改锁的状态，使工作线程可以响应中断
- 获取工作线程的锁（保证在任务执行过程中工作线程不被外部线程中断），如果获取到的任务是NULL，则结束当前工作线程
- 判断先测试状态，看是否需要中断当前工作线程
- 执行任务前置方法beforeExecute(wt, task);
- 执行任务(执行提交到线程池的线程)task.run();
- 执行任务后置方法afterExecute(task, thrown);，处理异常信息
- 修改完成任务的总数
- 解除当前工作线程的锁
- 获取队列里面的任务，循环第4步
- 将工作线程从容器中剔除

```
- `wt.isInterrupted()`：获取中断状态，无副作用
- `Thread.interrupted()`：获取中断状态，并将中断状态恢重置成false(不中断)
- `beforeExecute(wt, task)`：执行任务前置方法，扩展用。如果这个方法在执行过程中抛出异常，那么会导致当前工作线程直接死亡而被回收，工作线程异常结束标记位completedAbruptly被设置成true，任务线程不能被执行
- `task.run()`： 执行任务
- `afterExecute(task, thrown)`：执行任务后置方法，扩展用。这个方法可以收集到任务运行的异常信息，这个方法如果有异常抛出，也会导致当前工作线程直接死亡而被回收，工作线程异常结束标记位completedAbruptly被设置成true
- 任务运行过程中的异常信息除了RuntimeException以外，其他全部封装成Error，然后被afterExecute方法收集
- `terminated()`这也是一个扩展方法，在线程池结束的时候调用
```

### getTask() 获取任务

主要从q里拿任务

```java
private Runnable getTask() {
    // 记录最后一次获取任务是不是超时了
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        // 获取线程池状态
        int rs = runStateOf(c);

        // 线程池是停止状态或者状态是关闭并且队列为空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 扣减工作线程总数
            decrementWorkerCount();
            return null;
        }
        // 获取工作线程总数
        int wc = workerCountOf(c);

        // 工作线程是否需要剔除
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 扣减工作线程总数
            if (compareAndDecrementWorkerCount(c))
                // 剔除工作线程，当返回为NULL的时候，runWorker方法的while循环会结束
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

getTask() 阻塞或定时获取任务。当该方法返回NULL时，当前工作线程会结束，最后被回收，下面是返回NULL的几种情况：

- 当前工作线程总数wc大于maximumPoolSize最大工作线程总数。maximumPoolSize可能被setMaximumPoolSize方法改变。
- 当线程池处于停止状态时。
- 当线程池处于关闭状态且阻塞队列为空。
- 当前工作线程超时等待任务，并且当前工作线程总数wc大于corePoolSize或者allowCoreThreadTimeOut=true允许核心线程超时被回收，默认是false。
- 线程池在运行过程中可以调用setMaximumPoolSize()方法来修改maximumPoolSize值，新的值必须大于corePoolSize，如果新的maximumPoolSize小于原来的值，那么在该方法会去中断当前的空闲线程(工作线程内置锁的是解锁状态的线程为空闲线程)。

### processWorkerExit() 工作线程结束

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 判断是否是异常情况导致工作线程被回收
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        // 如果是扣减工作线程总数，如果不是在getTask()方法就已经扣减了
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 将当前工作线程完成任务的总数加到completedTaskCount标志位上
        completedTaskCount += w.completedTasks;
        // 剔除当前工作线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 尝试结束线程池
    tryTerminate();

    // 判刑是否需要新实例化工程线程
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

剔除线程流程：

- 判断是否是异常情况导致工作线程被回收，如果是workerCount--
- 获取到全局锁
- 将当前工作线程完成任务的总数加到completedTaskCount标志位上
- 剔除工作线程
- 解锁
- 尝试结束线程池tryTerminate()
- 判刑是否需要重新实例化工程线程放到workers容器

### shutdown() 关闭线程池

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查权限
        checkShutdownAccess();
        // 设置线程池状态为关闭
        advanceRunState(SHUTDOWN);
        // 中断线程
        interruptIdleWorkers();
        // 扩展方法
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试结束线池
    tryTerminate();
}
```

- 通过遍历工作线程容器workers，然后逐个中断工作线程，如果无法响应中断的任务可能永远无法终止
- shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。
- 正在执行的任务依旧会执行

### shutdownNow() 关闭线程池

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查权限
        checkShutdownAccess();
        // 设置线程池状态为停止状态
        advanceRunState(STOP);
        // 中断线程
        interruptIdleWorkers();
        // 将所有任务移动到list容器
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    // 尝试结束线池
    tryTerminate();
    // 返回所有未执行的任务
    return tasks;
}
```

- 通过遍历工作线程容器workers，然后逐个中断工作线程，如果无法响应中断的任务可能永远无法终止
- shutdownNow首先将线程池的状态设置成 STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表

### tryTerminate() 尝试结束线程池

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        //  判断是否在运行中,如果是直接返回
        if (isRunning(c) ||
            // 判断是否进入整理状态，如果进入了直接返回
            runStateAtLeast(c, TIDYING) ||
            // 如果是状态是关闭并且队列非空，也直接返回（关闭状态需要等到队列里面的线程处理完）
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        // 判断工作线程是否都关闭了
        if (workerCountOf(c) != 0) { // Eligible to terminate
            // 中断空闲线程
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 将状态替换成整理状态
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 整理发放执行
                    terminated();
                } finally {
                    // 状态替换成结束状态
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

结束线程池大致流程为：

- 判断是否在运行中，如果是则不结束线程
- 判断是否进入整理状态，如果是也不用执行后面内容了
- 判断如果线程池是关闭状态并且队列非空，则不结束线程池（关闭状态需要等到队列里面的线程处理完）
- 判断工作线程是否都关闭了，如果没有就发起中断工作线程的请求
- 获取全局锁将线程池状态替换成整理状态
- 调用terminated();扩展方法（这也是一个扩展方法，在线程池结束的时候调用）
- 将线程池状态替换成结束状态
- 解除全局锁

注意：

- 我们可以通过的shutdown或shutdownNow方法来结束线程池。他们都是通过遍历工作线程容器，然后逐个中断工作线程，所以无法响应中断的任务 可能永远无法终止。
- shutdown和shutdownNow的区别在于：shutdownNow首先将线程池的状态设置成 STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表；而 shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线程。
- 只要调用了shutdown和shutdownNow那么isShutdown方法就会返回true
当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true
。

总结下，worker里面包着thread，thread执行task，线程池初始化是没有worker（可以通过prestartAllCoreThreads预处理），直到真正提交task的时候才开始初始化worker，task执行结束后，会再从queue里拿task，没有的话就会阻塞，以保持线程不会被销毁（waiting状态），保持了核心线程的活跃。特殊情况，例如task中抛出异常，worker会被回收，但如果回收后的线程数量少于核心线程时，就又会建立一个新的没有task的worker。

线程池需要关闭么？

局部线程池在确定不会使用的情况下需要关闭。

如何优雅的让线程池自动关闭？

- 核心线程数为 0 并指定线程存活时间
- 通过 allowCoreThreadTimeOut 控制核心线程存活时间

最后上张图

![image.png](https://i.loli.net/2020/11/25/1XPFJerVGqEv7ni.png)

最最后，线程池是可以执行runnable callable future的，三着区别详见：https://zhuanlan.zhihu.com/p/88933756