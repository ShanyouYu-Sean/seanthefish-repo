---
layout: post
title: thread，threadlocal，InheritableThreadLocal源码详解
date: 2020-11-20 12:14:47
tags: 
- 并发
categories:
- 并发
---

## Thread 源码解析

构造方法：

所有构造方法都调用了init（），直接看此方法：

```java
private void init(ThreadGroup g, Runnable target, String name,
                    long stackSize, AccessControlContext acc) {
      if (name == null) {
          throw new NullPointerException("name cannot be null");
      }
      this.name = name.toCharArray();
      // 父线程
      Thread parent = currentThread();
      SecurityManager security = System.getSecurityManager();
      if (g == null) {
          // 实际是Thread.currentThread().getThreadGroup();
          // 拿到threadgroup以后会对其进行操作
          // threadgroup里面有nUnstartedThreads代表未启动线程数
          // nthreads启动线程数
          // threads[]线程的数组
          /* Determine if it's an applet or not */
          /* If there is a security manager, ask the security manager
             what to do. */
          if (security != null) {
              g = security.getThreadGroup();
          }
          /* If the security doesn't have a strong opinion of the matter
             use the parent thread group. */
          if (g == null) {
              g = parent.getThreadGroup();
          }
      }
      /* checkAccess regardless of whether or not threadgroup is
         explicitly passed in. */
      g.checkAccess();
      /*
       * Do we have the required permissions?
       */
      if (security != null) {
          if (isCCLOverridden(getClass())) {
              security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
          }
      }
      // 在ThreadGroup中标记增加了一个未启动的线程，里面操作很简单，nUnstartedThreads++;
      g.addUnstarted();
      this.group = g;
      // 继承父线程的属性
      this.daemon = parent.isDaemon();
      this.priority = parent.getPriority();
      if (security == null || isCCLOverridden(parent.getClass()))
          this.contextClassLoader = parent.getContextClassLoader();
      else
          this.contextClassLoader = parent.contextClassLoader;
      this.inheritedAccessControlContext =
              acc != null ? acc : AccessController.getContext();
      this.target = target;
      // 再设置线程优先级
      setPriority(priority);
      // 设置inheritableThreadLocals用于父子线程变量共享
      if (parent.inheritableThreadLocals != null)
          this.inheritableThreadLocals =
              ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
      /* Stash the specified stack size in case the VM cares */
      // 给这个线程设置的栈的大小,默认为0 
      this.stackSize = stackSize;
      // 设置线程id
      tid = nextThreadID();
  }
```

start（）：

```java
 public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        // 一个新建的未启动的线程的threadStatus必须为0
        // 0 就代表“new”
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        // 将当前线程加入group
        // group中的启动线程数++ 未启动线程数--
        group.add(this);

        boolean started = false;
        try {
            // 调用native方法
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

exit（）：
```java
/**
 * This method is called by the system to give a Thread
 * a chance to clean up before it actually exits.
 */
private void exit() {
    if (group != null) {
        group.threadTerminated(this);
        group = null;
    }
    /* Aggressively null out all reference fields: see bug 4006245 */
    target = null;
    /* Speed the release of some of these resources */
    threadLocals = null;
    inheritableThreadLocals = null;
    inheritedAccessControlContext = null;
    blocker = null;
    uncaughtExceptionHandler = null;
}
```

exit（）方法没啥可说的，更重要的是其中调用的group.threadTerminated(this);

```java
 /**
     * Notifies the group that the thread {@code t} has terminated.
     *
     * <p> Destroy the group if all of the following conditions are
     * true: this is a daemon thread group; there are no more alive
     * or unstarted threads in the group; there are no subgroups in
     * this thread group.
     *
     * @param  t
     *         the Thread that has terminated
     */
void threadTerminated(Thread t) {
        synchronized (this) {
            // 从group中删除当前线程
            remove(t);

            if (nthreads == 0) {
                // 唤醒其他线程
                // 这就是为什么被join挂起的线程会被唤醒
                notifyAll();
            }
            if (daemon && (nthreads == 0) &&
                (nUnstartedThreads == 0) && (ngroups == 0))
            {
                destroy();
            }
        }
    }
```

sleep（）：
```java
public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }
        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }
        // 直接调用native方法
        sleep(millis);
    }
```

stop（）：
stop( )方法是停止线程的执行，这个方法是一个不推荐使用的方法，已经被废弃了，因为使用该方法会出现异常情况。
因为它是天生不安全的。停止一个线程会导致它解锁它所锁定的所有monitor（当一个ThreadDeath Exception沿着栈向上传播时会解锁monitor），如果这些被释放的锁所保护的objects有任何一个进入一个不一致的状态，其他将要访问该objects的线程也会以一种不一致的状态来访问这些objects。这种objects称为“被损坏了”。当线程对被损坏的objects上做操作时，可能会产生意想不到的结果，这些行为可能是很严重的，并且难以探测到，
```java
@Deprecated
  public final void stop() {
      SecurityManager security = System.getSecurityManager();
      if (security != null) {
        //检查是否有权限
          checkAccess();
          if (this != Thread.currentThread()) {
              security.checkPermission(SecurityConstants.STOP_THREAD_PERMISSION);
          }
      }
      // A zero status value corresponds to "NEW", it can't change to
      // not-NEW because we hold the lock.
      if (threadStatus != 0) {
          resume(); // Wake up thread if it was suspended; no-op otherwise
      }
      // The VM can handle all thread states
      stop0(new ThreadDeath());
  }
```

join（）：

join方法是等待该线程执行，直到超时或者终止，可以作为线程通信的一种方式，A线程调用B线程的join（阻塞），等待B完成后再往下执行。 join（）方法中重载了多个方法，但是主要的方法是下面的方法。

```java
public final synchronized void join(long millis)
   throws InterruptedException {
       //得到当前的系统给时间
       long base = System.currentTimeMillis();
       long now = 0;
       if (millis < 0) {
           throw new IllegalArgumentException("timeout value is negative");
       }
       if (millis == 0) {
           //如果是活跃的的
           while (isAlive()) {
           //无限期的等待
               wait(0);
           }
       } else {
           while (isAlive()) {
               long delay = millis - now;
               if (delay <= 0) {
                   break;
               }
               //有限期的进行等待
               wait(delay);
               now = System.currentTimeMillis() - base;
           }
       }
   }
```

interrupt():

```java
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();
    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```

interrupt()方法是中断当前的线程， 其实Thread类中有三个方法，比较容易混淆，在这里解释一下。

- interrupt: 置为中断状态
- isInterrupt: 线程是否中断
- interrupted: 返回线程的上次的中断状态，并清除中断状态。

一般来说，阻塞函数：如Thread.sleep、Thread.join、Object.wait等在检查到线程的中断状态的时候，会抛出InteruptedExeption, 同时会清除线程的中断状态。

对于InterruptedException的处理，可以有两种情况：

外层代码可以处理这个异常，直接抛出这个异常即可
如果不能抛出这个异常，比如在run()方法内，因为在得到这个异常的同时，线程的中断状态已经被清除了，需要保留线程的中断状态，则需要调用`Thread.currentThread().interrupt()`

## threadlocal

以下内容参考： https://juejin.cn/post/6844903552477822989  https://www.jianshu.com/p/9d289d33f696

threadLocal是每线程独有的

![DlmQnP.png](https://s3.ax1x.com/2020/11/20/DlmQnP.png)

每个Thread维护一个ThreadLocalMap，存储在ThreadLocalMap内的就是一个以Entry为元素的数组，Entry就是一个key-value结构，key为ThreadLocal，value为存储的值。类比HashMap的实现，其实就是每个线程借助于一个哈希表，存储线程独立的值。

这里ThreadLocal和key之间的线是虚线，因为Entry是继承了WeakReference实现的，当ThreadLocalRef销毁时，指向堆中ThreadLocal实例的唯一一条强引用消失了，只有Entry有一条指向ThreadLocal实例的弱引用，那么这里ThreadLocal实例是可以被GC掉的。这时Entry里的key为null了，但是enrty本身还是有一个强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value，如果线程迟迟没有死亡，那么永远无法回收，造成内存泄漏。下面会讲。

ThreadLocalMap用来存储用户的value，这个map的引用在Thread类里，是全线程唯一的。

```java
static class ThreadLocalMap {
    // 注意虚引用
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    // 主要数据结构就是一个Entry的数组table；
    private Entry[] table;

    ...

```

几个方法一起说了：

```java

public void set(T value) {
    // 获取当前的Thread对象，通过getMap获取Thread内的ThreadLocalMap
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }


public T get() {
    // 获取当前的Thread对象，通过getMap获取Thread内的ThreadLocalMap
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 如果map已经存在，以当前的ThreadLocal为键，获取Entry对象，并从从Entry中取出值
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 否则，调用setInitialValue进行初始化。
        return setInitialValue();
    }

ThreadLocalMap getMap(Thread t) {
        // ThreadLocalMap在当前线程被所有ThreadLocal共享
        return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    // 初始化map,构建table与Enrty
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

private T setInitialValue() {
    // 首先是调用initialValue生成一个初始的value值，深入initialValue函数，我们可知它就是返回一个null；
    T value = initialValue();
    Thread t = Thread.currentThread();
    // 然后还是在get以下Map
    ThreadLocalMap map = getMap(t);
    if (map != null)
    // 如果map存在，则直接map.set
        map.set(this, value);
    else
    // 如果不存在则会调用createMap创建ThreadLocalMap
        createMap(t, value);
    return value;
}
```

每一个ThreadLocal对象是如何区分的：

```java
    //java提供的,可以用原子方式更新的 int值的类。
    private static AtomicInteger nextHashCode = new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
    private final int threadLocalHashCode = nextHashCode();

    private static int nextHashCode() {
        //原子性加一
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
对于每一个ThreadLocal对象，都有一个final的int值threadLocalHashCode；AtomicInteger 是static修饰的，全局唯一，每一次加一之后的值仍然可用，并且保证原子性。所以，每一个线程的ThreadLocal对象都有唯一的threadLocalHashCode值。

为什么不使用当前线程作为key？

上面知道，每一个线程周期，都有一个全线程唯一的map用于存储value，如果线程内多个ThreadLocal对象set了value，那么以当前线程作为键是不能保证key的唯一性的；而每一个ThreadLocal对象都可以由threadLocalHashCode进行唯一区分，所以key使用为ThreadLocal方便存取。

解决内存泄漏

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 首先是计算索引位置i
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
    // 根据获取Entry，如果Entry存在且Entry的key恰巧等于ThreadLocal，那么直接返回Entry对象；
        return e;
    else
    // 否则，也就是在此位置上找不到对应的Entry，那么就调用getEntryAfterMiss。
        return getEntryAfterMiss(key, i, e);
}


private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
        // 如果k==key,那么代表找到了这个所需要的Entry，直接返回；
            return e;
        if (k == null)
        // 如果k==null，那么证明这个Entry中key已经为null,那么这个Entry就是一个过期对象，这里调用expungeStaleEntry清理该Entry。
        // 这里解决了内存泄漏问题
        // ThreadLocal实例由于只有Entry中的一条弱引用指着，那么就会被GC掉，Entry的key没了，value可能会内存泄露的
        // 其实在每一个get，set操作时都会不断清理掉这种key为null的Entry的。
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    // 这段主要是将i位置上的Entry的value设为null，Entry的引用也设为null，那么系统GC的时候自然会清理掉这块内存；
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    // 这段就是扫描位置staleSlot之后，null之前的Entry数组，清除每一个key为null的Entry，同时若是key不为空，做rehash，调整其位置。
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

我们发现无论是set,get还是remove方法，过程中key为null的Entry都会被擦除，那么Entry内的value也就没有强引用链，GC时就会被回收。那么怎么会存在内存泄露呢？但是以上的思路是假设你调用get或者set方法了，很多时候我们都没有调用过，所以最佳实践就是*
1 .使用者需要手动调用remove函数，删除不再使用的ThreadLocal.
2 .还有尽量将ThreadLocal设置成private static的，这样ThreadLocal会尽量和线程本身一起消亡。

## InheritableThreadLocal父子线程共享变量

InheritableThreadLocal的源码非常简单，继承自ThreadLocal，重写其中三个方法。

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    public InheritableThreadLocal() {
    }

    protected T childValue(T var1) {
        return var1;
    }

    ThreadLocalMap getMap(Thread var1) {
        return var1.inheritableThreadLocals;
    }

    void createMap(Thread var1, T var2) {
        var1.inheritableThreadLocals = new ThreadLocalMap(this, var2);
    }
}
```

InheritableThreadLocal获取的也是ThreadLocalMap, 解决父子线程共享变量的地方实际在thread类里，上文说过：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        //..........
        Thread parent = currentThread();
        //..........
        // 如果父线程的inheritableThreadLocals != null
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        // 新线程：`this.inheritableThreadLocals=ThreadLocal.createInheritedMap(parent.inheritableThreadLocals)`
        // 传入父线程的inheritableThreadLocals 
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
```
```java
    static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
    }
    
    // 实际上就是构造了一个新的包含所有parentMap元素的新map
    private ThreadLocalMap(ThreadLocalMap parentMap) {
        // 得到parent的数组
        Entry[] parentTable = parentMap.table;
        int len = parentTable.length;
        setThreshold(len);
        table = new Entry[len];

        for (int j = 0; j < len; j++) {
            Entry e = parentTable[j];
            if (e != null) {
                @SuppressWarnings("unchecked")
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                if (key != null) {
                    // InheritableThreadLocal重写了这个方法返回原值
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    //计算hash位置，与ThreadLocal的set方法一样
                    int h = key.threadLocalHashCode & (len - 1);
                    while (table[h] != null)
                        h = nextIndex(h, len);
                    table[h] = c;
                    size++;
                }
            }   
        }
    }   
```
