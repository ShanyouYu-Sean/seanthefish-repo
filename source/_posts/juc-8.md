---
layout: post
title: juc之CyclicBarrier
date: 2020-11-22 15:00:00
tags: 
- 并发
categories:
- 并发
---

内容转载自：https://www.cnblogs.com/go2sea/tag/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E2%80%94%E2%80%94JUC%E5%8C%85/  稍有改动

## CyclicBarrier是什么？

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。
叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。
叫做栅栏，大概是描述所有线程被栅栏挡住了，当都达到时，一起跳过栅栏执行，也算形象。我们可以把这个状态就叫做barrier。

举个报旅行团旅行的例子。
出发时，导游会在机场收了护照和签证，办理集体出境手续，所以，要等大家都到齐才能出发，出发前再把护照和签证发到大家手里。
对应CyclicBarrier使用。
每个人到达后进入barrier状态。
都到达后，唤起大家一起出发去旅行。
旅行出发前，导游还会有个发护照和签证的动作。

```java
/**
 * 旅行线程
 * Created by jiapeng on 2018/1/7.
 */
public class TravelTask implements Runnable{

    private CyclicBarrier cyclicBarrier;
    private String name;
    private int arriveTime;//赶到的时间

    public TravelTask(CyclicBarrier cyclicBarrier,String name,int arriveTime){
        this.cyclicBarrier = cyclicBarrier;
        this.name = name;
        this.arriveTime = arriveTime;
    }

    @Override
    public void run() {
        try {
            //模拟达到需要花的时间
            Thread.sleep(arriveTime * 1000);
            System.out.println(name +"到达集合点");
            cyclicBarrier.await();
            System.out.println(name +"开始旅行啦～～");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}

/**
 * 导游线程，都到达目的地时，发放护照和签证
 * Created by jiapeng on 2018/1/7.
 */
public class TourGuideTask implements Runnable{

    @Override
    public void run() {
        System.out.println("****导游分发护照签证****");
        try {
            //模拟发护照签证需要2秒
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

/**
 * Created by jiapeng on 2018/1/7.
 */
public class Client {

    public static void main(String[] args) throws Exception{

        CyclicBarrier cyclicBarrier = new CyclicBarrier(3, new TourGuideTask());
        Executor executor = Executors.newFixedThreadPool(3);
        //登哥最大牌，到的最晚
        executor.execute(new TravelTask(cyclicBarrier,"哈登",5));
        executor.execute(new TravelTask(cyclicBarrier,"保罗",3));
        executor.execute(new TravelTask(cyclicBarrier,"戈登",1));
    }
}

```

通过CyclicBarrier我们可以实现n个线程相互等待。我们可以通过参数指定达到公共屏障点之后的行为。

我们先来看一下CyclicBarrier的成员变量：

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition trip = lock.newCondition();
private final int parties;
private final Runnable barrierCommand;
private Generation generation = new Generation();
private int count;
```

CyclicBarrier是通过独占锁lock和Condition对象trip来实现的，成员parties表示必须有parties个线程到达barrier，成员barrierCommand表示当parties个线程到达之后要执行的代码，成员count表示离触发barrierCommand还差count个线程（还有count个线程未到达barrier），成员generation表示当前的“代数”，“cyclic”表示可循环使用，generation是对一次循环的标识。注意：Generation是CyclicBarrier的一个私有内部类，他只有一个成员变量来标识当前的barrier是否已“损坏”：

```java
private static class Generation {
     boolean broken = false;
}
```

构造函数

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}
```

CyclicBarrier提供了两种构造函数，没有指定barrierCommand的构造函数是调用第二个构造函数实现的。第二个构造函数有两个参数：parties和barrierAction，分别用来初始化成员parties和barrierCommand。注意，parties必须大于0，否则会抛出IllegalArgumentException。

await（）方法

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
     throw new Error(toe); // cannot happen;
    }
}
```

await方法是由调用dowait方法实现的，两个参数分别代表是否定时等待和等待的时长。

doawait（）方法

```java
private int dowait(boolean timed, long nanos)
            throws InterruptedException, BrokenBarrierException, TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            //小概率事件：该线程在等待锁的过程中，barrier被破坏
            if (g.broken)
                throw new BrokenBarrierException();

            //小概率事件：该线程在等待锁的过程中被中断
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

           int index = --count;
           //当有parties个线程到达barrier
           if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                   final Runnable command = barrierCommand;
                   //如果设置了barrierCommand，令最后到达的barrier的线程执行它
                   if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
               } finally {
                    //注意：当执行barrierCommand出现异常时，ranAction派上用场
                    if (!ranAction)
                        breakBarrier();
               }
           }

            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        //注意：nanos值标识了是否超时，后续用这个nanos值判断是否breakBarrier
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        //小概率事件：该线程被中断，进入锁等待队列
                        //在等待过程中，另一个线程更新或破坏了generation
                        //当该线程获取锁之后，应重置interrupt标志而不是抛出异常
                        //原因在于：它中断的太晚了，generation已更新或破坏，它抛出InterruptedException的时机已经过去，
                        //两种情况：
                        //①g被破坏：已有一个线程抛出InterruptedException（只能由第一个抛），与它同时等待的都抛BrokenBarrierException（后续检查broken标志会抛）。
                        //②g被更新：此时抛异常没意义（后续检查g更新后会return index），这里重置interrupt标志，让线程继续执行，让这个标志由上层处理
                        Thread.currentThread().interrupt();
                    }
                }

                //barrier被破坏，抛出异常
                if (g.broken)
                    throw new BrokenBarrierException();

                //barrier正常进入下一循环，上一代await的线程继续执行
                if (g != generation)
                    return index;

                //只要有一个超时，就breakBarrier，后续线程抛的就是barrier损坏异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

dowait方法是CyclicBarrier的精华。应该重点来理解。

方法开头首先申请锁，然后做了两个判断：g.broken和Thread.interrupted()，这两个判断是分别处理两种小概率的事件：

- 该线程在等待锁的过程中，barrier被破坏
- 该线程在等待锁的过程中被中断。这两个事件应抛出相应的异常。
 
接下来dowait方法修改了令count减1，如果此时count减为0，说明已经有parties个线程到达barrier，这时由最后到达barrier的线程去执行barrierCommand。注意，这里设置了一个布尔值ranAction，作用是来标识barrierCommand是否被正确执行完毕，如果执行失败，finally中会执行breakBarrier操作。

如果count尚未减为0，则在Condition对象trip上执行await操作，注意：这里有一个InterruptedException的catch子句。当前线程在await中被中断时，会抛出InterruptedException，这时候如果g==generation&&!g.broken的话，我们执行breakBarrier操作，同时抛出这个异常；如果g!=generation或者g.broken==true的话，我们的操作是重置interrupt标志而不是抛出这个异常。这么做的原因我们分两种情况讨论：

- g被破坏，这也是一个小概率事件，当前线程被中断后进入锁等待队列，此时另一个线程由于某种原因（超时或者被中断）在他之前获取了锁并执行了breakBarrier方法，那么当前线程持有锁之后就不应再抛InterruptedException，逻辑上应该处理barrier被破坏事件，事实上在后续g.broken的检查中，他会抛出一个BrokenBarrierException。而当前的InterruptedException被我们捕获却没有做出处理，所以执行interrupt方法重置中断标志，交由上层程序处理。

- g被更新：说明当前线程在即将完成等待之际被中断，此时抛异常没意义（后续检查g更新后会return index），这里重置interrupt标志，让线程继续执行，让这个标志由上层处理。

后续对g.broken和g!=generation的判断，分表代表了被唤醒线程（非最后一个到达barrier的线程，也不是被中断或第一个超时的线程）的两种退出方法的方式：

- 第一种是以barrier被破坏告终（然后抛异常）
- 第二个是barrier等到parties个线程，寿终正寝（返回该线程的到达次序index）。

最后一个if是第一个超时线程执行breakBarrier操作并跑出异常。最后finally子句要释放锁。

至此，整个doawait方法流程就分析完毕了，我们可以发现，**在barrier上等待的线程，如果以抛异常结束的话，只有第一个线程会抛InterruptedException或TimeoutException并执行breakBarrier操作，其他等待线程只能抛BrokenBarrierException，逻辑上这也是合理的：一个barrier只能因超时或中断被破坏一次。**

```java
private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}

private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

doawait方法中用到的nextGeneration方法将所有等待线程唤醒，更新generation对象，复位count，进入下一轮任务。

breakBarrier方法将generation状态值为broken，复位count（这个复位看上去没有用，但实际上，在broken之后reset之前，如果调用getNumberWaiting方法查看等待线程数的话，复位count是合理的），并唤醒所有等待线程。在调用reset更新generation之前，barrier将处于不可用状态。

reset（）方法

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

reset方法先break当执行breakBarrier操作（如果有线程在barrier上等待，调用reset会导致BrokenBarrierException），再更新generation对象。

