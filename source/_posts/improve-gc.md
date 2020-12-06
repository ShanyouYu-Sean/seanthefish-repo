---
layout: post
title: gc调优
date: 2020-11-13 10:00:00
tags: 
- jvm
categories:
- jvm
---

先贴个图

![DAwfZ6.png](https://s3.ax1x.com/2020/11/16/DAwfZ6.png)

## 如何阅读gclog：

以其中一行为例来解读下日志信息：

[GC (Allocation Failure) [ParNew: 367523K->1293K(410432K), 0.0023988 secs] 522739K->156516K(1322496K), 0.0025301 secs] [Times: user=0.04 sys=0.00, real=0.01 secs]

GC：
表明进行了一次垃圾回收，前面没有Full修饰，表明这是一次Minor GC ,注意它不表示只GC新生代，并且现有的不管是新生代还是老年代都会STW。

Allocation Failure：
表明本次引起GC的原因是因为在年轻代中没有足够的空间能够存储新的数据了。

ParNew：
表明本次GC发生在年轻代并且使用的是ParNew垃圾收集器。ParNew是一个Serial收集器的多线程版本，会使用多个CPU和线程完成垃圾收集工作（默认使用的线程数和CPU数相同，可以使用-XX：ParallelGCThreads参数限制）。该收集器采用复制算法回收内存，期间会停止其他工作线程，即Stop The World。

367523K->1293K(410432K)：单位是KB
三个参数分别为：GC前该内存区域(这里是年轻代)使用容量，GC后该内存区域使用容量，该内存区域总容量。

0.0023988 secs：
该内存区域GC耗时，单位是秒

522739K->156516K(1322496K)：
三个参数分别为：堆区垃圾回收前的大小，堆区垃圾回收后的大小，堆区总大小。

0.0025301 secs：
该内存区域GC耗时，单位是秒

[Times: user=0.04 sys=0.00, real=0.01 secs]：
分别表示用户态耗时，内核态耗时和总耗时

分析下可以得出结论：

该次GC新生代减少了367523-1293=366239K

Heap区总共减少了522739-156516=366223K

366239 – 366223 =16K，说明该次共有16K内存从年轻代移到了老年代，可以看出来数量并不多，说明都是生命周期短的对象，只是这种对象有很多。

我们需要的是尽量避免Full GC的发生，让对象尽可能的在年轻代就回收掉，所以这里可以稍微增加一点年轻代的大小，让那17K的数据也保存在年轻代中。

## 常见配置举例

### 堆大小设置

JVM 中最大堆大小有三方面限制：相关操作系统的数据模型（32-bt还是64-bit）限制；系统的可用虚拟内存限制；系统的可用物理内存限制。32位系统 下，一般限制在1.5G~2G；64为操作系统对内存无限制。我在Windows Server 2003 系统，3.5G物理内存，JDK5.0下测试，最大可设置为1478m。

典型设置：
`java -Xmx3550m -Xms3550m -Xmn2g -Xss128k`

-Xmx3550m：设置JVM最大可用内存为3550M。

-Xms3550m：设置JVM初始内存为3550m。此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存。

-Xmn2g：设置年轻代大小为2G。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。

-Xss128k： 设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内 存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。

`java -Xmx3550m -Xms3550m -Xss128k -XX:NewRatio=4 -XX:SurvivorRatio=4 -XX:MaxPermSize=16m -XX:MaxTenuringThreshold=0`

-XX:NewRatio=4:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5

-XX:SurvivorRatio=4：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6

-XX:MaxPermSize=16m:设置持久代大小为16m。

-XX:MaxTenuringThreshold=0：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概论。

### 回收器选择

JVM给了三种选择：串行收集器、并行收集器、并发收集器，但是串行收集器只适用于小数据量的情况，所以这里的选择主要针对并行收集器和并发收集器。默认情况下，JDK5.0以前都是使用串行收集器，如果想使用其他收集器需要在启动时加入相应参数。JDK5.0以后，JVM会根据当前系统配置进行判断。

#### 吞吐量优先的并行收集器

如上文所述，并行收集器主要以到达一定的吞吐量为目标，适用于科学技术和后台处理等。

典型配置：
`java -Xmx3800m -Xms3800m -Xmn2g -Xss128k -XX:+UseParallelGC -XX:ParallelGCThreads=20`

-XX:+UseParallelGC：选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下，年轻代使用并发收集，而年老代仍旧使用串行收集。

-XX:ParallelGCThreads=20：配置并行收集器的线程数，即：同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等。

`java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC -XX:ParallelGCThreads=20 -XX:+UseParallelOldGC`

-XX:+UseParallelOldGC：配置年老代垃圾收集方式为并行收集。JDK6.0支持对年老代并行收集。

java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100

-XX:MaxGCPauseMillis=100:设置每次年轻代垃圾回收的最长时间，如果无法满足此时间，JVM会自动调整年轻代大小，以满足此值。

`java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseParallelGC  -XX:MaxGCPauseMillis=100 -XX:+UseAdaptiveSizePolicy`

-XX:+UseAdaptiveSizePolicy：设置此选项后，并行收集器会自动选择年轻代区大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间或者收集频率等，此值建议使用并行收集器时，一直打开。

#### 响应时间优先的并发收集器

如上文所述，并发收集器主要是保证系统的响应时间，减少垃圾收集时的停顿时间。适用于应用服务器、电信领域等。

典型配置：
`java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:ParallelGCThreads=20 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC`

-XX:+UseConcMarkSweepGC：设置年老代为并发收集。测试中配置这个以后，-XX:NewRatio=4的配置失效了，原因不明。所以，此时年轻代大小最好用-Xmn设置。

-XX:+UseParNewGC:设置年轻代为并行收集。可与CMS收集同时使用。JDK5.0以上，JVM会根据系统配置自行设置，所以无需再设置此值。

`java -Xmx3550m -Xms3550m -Xmn2g -Xss128k -XX:+UseConcMarkSweepGC -XX:CMSFullGCsBeforeCompaction=5 -XX:+UseCMSCompactAtFullCollection`

-XX:CMSFullGCsBeforeCompaction：由于并发收集器不对内存空间进行压缩、整理，所以运行一段时间以后会产生“碎片”，使得运行效率降低。此值设置运行多少次GC以后对内存空间进行压缩、整理。

-XX:+UseCMSCompactAtFullCollection：打开对年老代的压缩。可能会影响性能，但是可以消除碎片。

### 常见配置汇总

#### 堆设置

-Xms:初始堆大小
-Xmx:最大堆大小
-XX:NewSize=n:设置年轻代大小
-XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-XX:MaxPermSize=n:设置持久代大小

#### 收集器设置

-XX:+UseSerialGC:设置串行收集器
-XX:+UseParallelGC:设置并行收集器
-XX:+UseParalledlOldGC:设置并行年老代收集器
-XX:+UseConcMarkSweepGC:设置并发收集器

##### 垃圾回收统计信息

-XX:+PrintGC
-XX:+Printetails
-XX:+PrintGCTimeStamps
-Xloggc:filename

##### 并行收集器设置

-XX:ParallelGCThreads=n:设置并行收集器收集时使用的CPU数。并行收集线程数。
-XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间
-XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)
并发收集器设置
-XX:+CMSIncrementalMode:设置为增量模式。适用于单CPU情况。
-XX:ParallelGCThreads=n:设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。

## 经验

### 年轻代大小选择

响应时间优先的应用：设置接近系统的最低响应时间限制（根据实际情况选择）。在此种情况下，年轻代收集发生的频率也是最小的。同时，减少到达年老代的对象。ps：并非堆越大越好，大堆full gc时间就长，停顿时间变长。在full gc发生不可控的情况下，大内存的物理机考虑使用集群化环境。

吞吐量优先的应用：尽可能的设置大，可能到达Gbit的程度。因为对响应时间没有要求，垃圾收集可以并行进行，一般适合8CPU以上的应用。

### 年老代大小选择

响应时间优先的应用：年老代使用并发收集器，所以其大小需要小心设置，一般要考虑并发会话率和会话持续时间等一些参数。如果堆设置小了，可以会造成内存碎片、高回收频率以及应用暂停而使用传统的标记清除方式；如果堆大了，则需要较长的收集时间。最优化的方案，一般需要参考以下数据获得：

- 并发垃圾收集信息
- 持久代并发收集次数
- 传统GC信息
- 花在年轻代和年老代回收上的时间比例
- 减少年轻代和年老代花费的时间，一般会提高应用的效率

吞吐量优先的应用：一般吞吐量优先的应用都有一个很大的年轻代和一个较小的年老代。原因是，这样可以尽可能回收掉大部分短期对象，减少中期的对象，而年老代尽存放长期存活对象。

### 较小堆引起的碎片问题

因为年老代的并发收集器使用标记、清除算法，所以不会对堆进行压缩。当收集器回收时，他会把相邻的空间进行合并，这样可以分配给较大的对象。但是，当堆空间较小时，运行一段时间以后，就会出现“碎片”，如果并发收集器找不到足够的空间，那么并发收集器将会停止，然后使用传统的标记、清除方式进行回收。如果出 现“碎片”，可能需要进行如下配置：
-XX:+UseCMSCompactAtFullCollection：使用并发收集器时，开启对年老代的压缩。
-XX:CMSFullGCsBeforeCompaction=0：上面配置开启的情况下，这里设置多少次Full GC后，对年老代进行压缩。

### 高分配速率(High Allocation Rate)

分配速率(Allocation rate)表示单位时间内分配的内存量。通常使用 MB/sec作为单位, 也可以使用 PB/year 等。

计算上一次垃圾收集之后,与下一次GC开始之前的年轻代使用量, 两者的差值除以时间,就是分配速率。

分配速率过高就会严重影响程序的性能。在JVM中会导致巨大的GC开销。

#### 分配速率的意义

分配速率的变化,会增加或降低GC暂停的频率, 从而影响吞吐量。但只有年轻代的 minor GC受分配速率的影响, 老年代GC的频率和持续时间不受分配速率(allocation rate)的直接影响, 而是受到提升速率(promotion rate)的影响, 请参见下文。

现在我们只关心Minor GC暂停, 查看年轻代的3个内存池。因为对象在Eden区分配, 所以我们一起来看Eden区的大小和分配速率的关系。看看增加Eden区的容量, 能不能减少Minor GC暂停次数, 从而使程序能够维持更高的分配速率。

经过我们的实验, 通过参数-XX:NewSize、 -XX:MaxNewSize以及 -XX:SurvivorRatio设置不同的Eden空间, 运行同一程序时, 可以发现:

- Eden 空间为 100 MB 时, 分配速率低于 100 MB/秒。
- 将 Eden 区增大为 1 GB, 分配速率也随之增长,大约等于 200 MB/秒。

为什么会这样? —— 因为减少GC暂停,就等价于减少了任务线程的停顿，就可以做更多工作, 也就创建了更多对象, 所以对同一应用来说, 分配速率越高越好。

在得出 “Eden区越大越好” 这个结论前, 我们注意到, 分配速率可能会,也可能不会影响程序的实际吞吐量。吞吐量和分配速率有一定关系, 因为分配速率会影响 minor GC 暂停, 但对于总体吞吐量的影响, 还要考虑 Major GC(大型GC)暂停, 而且吞吐量的单位不是 MB/秒，而是系统所处理的业务量。

#### 高分配速率对JVM的影响

demo：
```java
public class BoxingFailure {
  private static volatile Double sensorValue;
  private static void readSensor() {
    while(true) sensorValue = Math.random();
  }
  private static void processSensorValue(Double value) {
    if(value != null) {
      //...
    }
  }
}
```

如同类名所示, 这个Demo是模拟 boxing 的。为了 null 值判断, 使用的是包装类型 Double。 程序基于传感器的最新值进行计算, 但从传感器取值是一个重量级操作, 所以采用了异步方式： 一个线程不断获取新值, 计算线程则直接使用暂存的最新值, 从而避免同步等待。

Demo 程序在运行的过程中, 由于分配速率太大而受到GC的影响。

首先，我们应该检查程序的吞吐量是否降低。如果创建了过多的临时对象, minor GC的次数就会增加。如果并发较大, 则GC可能会严重影响吞吐量。

遇到这种情况时, GC日志将会像下面这样，当然这是上面的示例程序 产生的GC日志。 JVM启动参数为 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xmx32m:

```
2.808: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003076 secs]
2.819: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003079 secs]
2.830: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0002968 secs]
2.842: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
2.853: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0004672 secs]
2.864: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003371 secs]
2.875: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003214 secs]
2.886: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003374 secs]
2.896: [GC (Allocation Failure) 
        [PSYoungGen: 9760K->32K(10240K)], 0.0003588 secs]
```

minor GC的频率过高，就说明创建了大量的对象。而年轻代在GC之后的使用量又很低, 也没有full GC发生。 因此，GC对吞吐量造成了严重的影响。

解决方案
在某些情况下,只要增加年轻代的大小, 即可降低分配速率过高所造成的影响。增加年轻代空间并不会降低分配速率, 但是会减少GC的频率。如果每次GC后只有少量对象存活, minor GC 的暂停时间就不会明显增加。

运行 示例程序 时, 增加堆内存大小,(同时也就增大了年轻代的大小), 使用的JVM参数为 -Xmx64m:

```
2.808: [GC (Allocation Failure) 
        [PSYoungGen: 20512K->32K(20992K)], 0.0003748 secs]
2.831: [GC (Allocation Failure) 
        [PSYoungGen: 20512K->32K(20992K)], 0.0004538 secs]
2.855: [GC (Allocation Failure) 
        [PSYoungGen: 20512K->32K(20992K)], 0.0003355 secs]
2.879: [GC (Allocation Failure) 
        [PSYoungGen: 20512K->32K(20992K)], 0.0005592 secs]
```

但有时候增加堆内存的大小,并不能解决问题。通过前面学到的知识, 我们可以通过分配分析器找出大部分垃圾产生的位置。实际上在此示例中, 99%的对象属于 Double 包装类, 在readSensor 方法中创建。最简单的优化, 将创建的 Double 对象替换为原生类型 double, 而针对 null 值的检测, 可以使用 Double.NaN 来进行。由于原生类型不算是对象, 也就不会产生垃圾, 导致GC事件。优化之后, 不在堆中分配新对象, 而是直接覆盖一个属性域即可。

### 过早提升(Premature Promotion)

提升速率(promotion rate), 用于衡量单位时间内从年轻代提升到老年代的数据量。一般使用 MB/sec 作为单位, 和分配速率类似。

JVM会将长时间存活的对象从年轻代提升到老年代。根据分代假设, 可能存在一种情况, 老年代中不仅有存活时间长的对象,也可能有存活时间短的对象。这就是过早提升：对象存活时间还不够长的时候就被提升到了老年代。

major GC 不是为频繁回收而设计的, 但 major GC 现在也要清理这些生命短暂的对象, 就会导致GC暂停时间过长。这会严重影响系统的吞吐量。

GC之前和之后的 年轻代使用量以及堆内存使用量。这样就可以通过差值算出老年代的使用量。请注意, 只能根据 minor GC 计算提升速率。 Full GC 的日志不能用于计算提升速率, 因为 major GC 会清理掉老年代中的一部分对象。

#### 提升速率的意义

和分配速率一样, 提升速率也会影响GC暂停的频率。但分配速率主要影响 minor GC, 而提升速率则影响 major GC 的频率。有大量的对象提升,自然很快将老年代填满。 老年代填充的越快, 则 major GC 事件的频率就会越高。

#### 过早提升的影响

demo:
```java
public class PrematurePromotion {
   private static final Collection<byte[]> accumulatedChunks 
                = new ArrayList<>();
   private static void onNewChunk(byte[] bytes) {
       accumulatedChunks.add(bytes);
       if(accumulatedChunks.size() > MAX_CHUNKS) {
           processBatch(accumulatedChunks);
           accumulatedChunks.clear();
       }
   }
}
```

一般来说,过早提升的症状表现为以下形式:

- 短时间内频繁地执行 full GC。
- 每次 full GC 后老年代的使用率都很低, 在10-20%或以下。
- 提升速率接近于分配速率。

要演示这种情况稍微有点麻烦, 所以我们使用特殊手段, 让对象提升到老年代的年龄比默认情况小很多。指定GC参数 -Xmx24m -XX:NewSize=16m -XX:MaxTenuringThreshold=1, 运行程序之后,可以看到下面的GC日志:

```
2.176: [Full GC (Ergonomics) 
        [PSYoungGen: 9216K->0K(10752K)] 
        [ParOldGen: 10020K->9042K(12288K)] 
        19236K->9042K(23040K), 0.0036840 secs]
2.394: [Full GC (Ergonomics) 
        [PSYoungGen: 9216K->0K(10752K)] 
        [ParOldGen: 9042K->8064K(12288K)] 
        18258K->8064K(23040K), 0.0032855 secs]
2.611: [Full GC (Ergonomics) 
        [PSYoungGen: 9216K->0K(10752K)] 
        [ParOldGen: 8064K->7085K(12288K)] 
        17280K->7085K(23040K), 0.0031675 secs]
2.817: [Full GC (Ergonomics) 
        [PSYoungGen: 9216K->0K(10752K)] 
        [ParOldGen: 7085K->6107K(12288K)] 
        16301K->6107K(23040K), 0.0030652 secs]
```

乍一看似乎不是过早提升的问题。事实上,在每次GC之后老年代的使用率似乎在减少。但反过来想, 要是没有对象提升或者提升率很小, 也就不会看到这么多的Full GC了。

简单解释一下这里的GC行为: 有很多对象提升到老年代, 同时老年代中也有很多对象被回收了, 这就造成了老年代使用量减少的假象. 但事实是大量的对象不断地被提升到老年代, 并触发full GC。

#### 解决方案

简单来说, 要解决这类问题, 需要让年轻代存放得下暂存的数据。有两种简单的方法:

一是增加年轻代的大小, 设置JVM启动参数, 类似这样: -Xmx64m -XX:NewSize=32m, 程序在执行时, Full GC 的次数自然会减少很多, 只会对 minor GC的持续时间产生影响:

二是减少每次批处理的数量, 也能得到类似的结果. 至于选用哪个方案, 要根据业务需求决定。在某些情况下, 业务逻辑不允许减少批处理的数量, 那就只能增加堆内存,或者重新指定年轻代的大小。

如果都不可行, 就只能优化数据结构, 减少内存消耗。但总体目标依然是一致的: 让临时数据能够在年轻代存放得下。

### Weak, Soft 及 Phantom 引用

https://blog.csdn.net/qq_40955824/article/details/90035050

- 强引用：代码中普遍存在的类似`object obj = new object()`的引用，只要强引用存在，垃圾处理器就不会回收。
- 软引用：描述有些还有用但非必须的对象。在系统将要发生内存溢出时，会将这些对象列为回收范围进行二次回收，如果这次回收后还没有足够的内存，才会oom。Java中SoftReference类表示软引用。
- 弱引用：描述非必须对象，被弱引用关联的对象只能生存到下一次回收之前，垃圾收集器工作之后，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。Java中WeakReference类表示弱引用。
- 虚引用：这个引用存在的唯一目的，就是在这个对象被收集器回收时得到一个系统通知，被虚引用关联的对象，和其生存时间完全没有关系。Java中PhantomReference类表示虚引用。

#### 弱引用的缺点

首先, 弱引用(weak reference) 是可以被GC强制回收的。当垃圾收集器发现一个弱可达对象(weakly reachable,即指向该对象的引用只剩下弱引用) 时, 就会将其置入相应的ReferenceQueue 中, 变成可终结的对象. 之后可能会遍历这个 reference queue, 并执行相应的清理。典型的示例是清除缓存中不再引用的KEY。

当然, 在这个时候, 我们还可以将该对象赋值给新的强引用, 在最后终结和回收前, GC会再次确认该对象是否可以安全回收。因此, 弱引用对象的回收过程是横跨多个GC周期的。

实际上弱引用使用的很多。大部分缓存框架(caching solution)都是基于弱引用实现的, 所以虽然业务代码中没有直接使用弱引用, 但程序中依然会大量存在。

其次, 软引用(soft reference) 比弱引用更难被垃圾收集器回收. 回收软引用没有确切的时间点, 由JVM自己决定. 一般只会在即将耗尽可用内存时, 才会回收软引用,以作最后手段。这意味着, 可能会有更频繁的 full GC, 暂停时间也比预期更长, 因为老年代中的存活对象会很多。

最后, 使用虚引用(phantom reference)时, 必须手动进行内存管理, 以标识这些对象是否可以安全地回收。表面上看起来很正常, 但实际上并不是这样。 javadoc 中写道:

为了防止可回收对象的残留, 虚引用对象不应该被获取: phantom reference 的 get 方法返回值永远是 null。

令人惊讶的是, 很多开发者忽略了下一段内容(这才是重点):

与软引用和弱引用不同, 虚引用不会被 GC 自动清除, 因为他们被存放到队列中. 通过虚引用可达的对象会继续留在内存中, 直到引用自身变为不可达。（如果虚引用为clearer对象，会主动调用其clean方法进行引用自身变为不可达的操作）

也就是说,我们必须手动使虚引用变为不可达状态（把引用置为null）, 否则可能会造成 OutOfMemoryError 而导致 JVM 挂掉. 使用虚引用的理由是, 对于用编程手段来跟踪某个对象何时变为不可达对象, 这是唯一的常规手段。 和软引用/弱引用不同的是, 我们不能复活虚可达(phantom-reachable)对象。

建议使用JVM参数 -XX:+PrintReferenceGC 来看看各种引用对GC的影响。

弱引用的典型代表使threadlocal。
虚引用的典型代表使堆外内存。

#### 解决方案

如果程序确实碰到了 mis-, ab- 问题或者滥用 weak, soft, phantom 引用, 一般都要修改程序的实现逻辑。每个系统不一样, 因此很难提供通用的指导建议, 但有一些常用的办法:

- 弱引用(Weak references) —— 如果某个内存池的使用量增大, 造成了性能问题, 那么增加这个内存池的大小(可能也要增加堆内存的最大容量)。如, 增加堆内存的大小, 以及年轻代的大小, 可以减轻症状。
- 虚引用(Phantom references) —— 请确保在程序中调用了虚引用的 clear 方法。编程中很容易忽略某些虚引用, 或者清理的速度跟不上生产的速度, 又或者清除引用队列的线程挂了, 就会对GC 造成很大压力, 最终可能引起 OutOfMemoryError。
- 软引用(Soft references) —— 如果确定问题的根源是软引用, 唯一的解决办法是修改程序源码, 改变内部实现逻辑。