title: 初探java内存管理机制-GC日志&GC参数设置
date: 2017-07-17 14:03:59
categories: programming
tags:
- java
- memory management
- GC
---

使用不同的垃圾回收器，打印出的GC日志格式会有所区别。
我们可以使用以下的JVM Flags来输出GC日志：

<!--more-->

* -Xloggc:/home/XX/gc/app_gc.log：可以把GC输出至文件，这对长时间服务器GC监控很有帮助。

* -verbose:gc：输出最基本的GC信息， 例如：**[GC 72104K->9650K(317952K), 0.0130635 secs]**

* -XX:+PrintGCDetails：输出详细的GC信息，例如： **[GC [PSYoungGen: 142826K->10751K(274944K)] 162800K->54759K(450048K), 0.0609952 secs] [Times: user=0.13 sys=0.02, real=0.06 secs]**

* -XX:+PrintGCDateStamps：输出包含日期的GC信息，例如：**2015-12-06T12:32:02.890+0800: [GC [PSYoungGen: 142833K->10728K(142848K)] 166113K->59145K(317952K), 0.0792023 secs] [Times: user=0.22 sys=0.00, real=0.08 secs] **

* -XX:+PrintGCTimeStamps：输出jvm从启动了之后，经过的时间秒数。例如：**0.089: [GC (Allocation Failure) 0.089: [DefNew: 7307K->331K(9216K), 0.0049114 secs] 7307K->6476K(19456K), 0.0049502 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]**

下面我们分析一下几个比较典型的GC日志：

```
2017-07-17T15:27:44.216-0800: 0.087: [GC (Allocation Failure) 2017-07-17T15:27:44.216-0800: 0.087: [DefNew: 7307K->331K(9216K), 0.0049123 secs] 7307K->6476K(19456K), 0.0049643 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
```

* 2017-07-17T15:27:44.216-0800：GC事件(GC event)开始的时间点。

* 0.087：GC时间的开始时间，相对于JVM的启动时间，单位是秒(Measured in seconds)。

* [GC："[GC"和"[Full GC"说明了这次垃圾收集的停顿类型，而不是用来区分新生代GC还是老年代GC的。如果有"Full"，说明这次GC是发生了Stop-The-World的。 如果是调用System.gc()方法所触发的收集，那么在这里将显示"[Full GC（System）"。

* Allocation Failure：引起垃圾回收的原因。本次GC是因为年轻代中没有任何合适的区域能够存放需要分配的数据结构而触发的。

* [DefNew：表示发生GC的内存区域。其中使用的垃圾回收器与这里的内存区域名称的对应关系为：

| 简写内存区域名称 | 完整内存区域名称   |  对应的垃圾回收器  |
| :-----:   | :-----:  | :----:  |
| DefNew    | Default New Generation |   Serial收集器 |
| ParNew    | Parallel New Generation |   ParNew收集器   |
| PSYoungGen |    Parallel Scavenge Young Generation    |  Parallel Scavenge收集器  |
| Tenured (老年代)   | N/A |   未知   |
| Perm（永久代）    | N/A |   未知   |

* 7307K->331K：在本次垃圾收集之前和之后的年轻代内存使用情况。

* (9216K)：年轻代的内存总大小。

* 7307K->6476K：在本次垃圾收集之前和之后整个堆内存的使用情况(Total used heap)。

* (19456K)：总的可用的堆内存(Total available heap)。

* 0.0049643 secs：GC持续的总时间。

* [Times: user=0.00 sys=0.00, real=0.01 secs]：user – 此次垃圾回收, 垃圾收集线程消耗的所有CPU时间(Total CPU time)；sys – 操作系统调用(OS call) 以及等待系统事件的时间(waiting for system event)；real – 应用程序暂停的时间(Clock time). 由于串行垃圾收集器(Serial Garbage Collector)只会使用单个线程, 所以 real time 等于 user 以及 system time 的总和。

# JVM参数与垃圾回收器的对应关系
---

| 参数 | 新生代使用的垃圾回收器 | 老年代使用的垃圾回收器 | 备注 |
| :-----:   | :-----: | :-----: | :-----: |
| **-XX:+UseSerialGC**    |  **Serial** | **Serial Old** | 虚拟机运行在client模式下的默认值 |
| **-XX:+UseParallelGC** **-XX:+UseParallelOldGC**    |  **Parallel Scavenge** | **Parallel Old** | 无 |
| **-XX:+UseParNewGC** **-XX:+UseConcMarkSweepGC**    |  **Parallel New** | **CMS** | 无 |
| **-XX:+UseG1GC** |  **G1** | **G1** | G1里没有老年代和新生代的概念，而是region |
| -XX:+UseParallelGC |  Parallel Scavenge | Serial Old | 虚拟机运行在server模式下的默认值 |
| -Xincgc |  Incremental | Incremental | 无 |
| -XX:-UseParNewGC -XX:+UseConcMarkSweepGC |  Serial | CMS | 无 |
| N/A |  Parallel New | Serial | 无 |
| N/A |  Serial | Parallel Old | 无 |
| N/A |  Parallel New | Parallel Old | 无 |
| N/A |  Parallel Scavenge | CMS  | 无 |

在生产环境中, 一般只会用到前四种垃圾回收器的组合，其余的组合要么不实用(例如中间的三种)，要么根本无法设置(标记为N/A的)这些组合。

# 常用的关于GC参数设置:
---
以下的参数都是适用-XX:+xxx的方式设置，例如：

```
-XX:+UseSerialGC
```

| 参数 | 描述 | 
| :-----:   | :-----: |
| SurvivorRatio    | 新生代中Eden和Survivor区域的内存容量比值，默认值为8，表示Eden : Survivor = 8 : 1 |
| PretenureSizeThreshold    | 直接晋升到老年代对象的大小，设置这个参数了之后，大于这个参数的对象直接在老年代当中分配。这样做的目的是避免在Eden区和两个Survivor区之间发生大量的内存拷贝。默认值为0。 |
| MaxTenuringThreshold  | 晋升到老年代对象年龄。在每一次Minor GC之后，存活的对象的代龄将加1，当超过这个值之后将进入老年代。默认值为15。 |
| UseAdaptiveSizePolicy  | 开启了该参数之后将动态调整Java堆中各个区域的大小，以及进入老年代的年龄。默认值为开启状态。 |
| HandlePromotionFailure  | 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden和Survivor区的所有对象都存活的极端情况。每次进行Minor GC时，JVM会计算Survivor区移至老年区对象的平均大小，如果这个值大于老年区的剩余值大小则进行一次Full GC，如果小于检查HandlePromotionFailure设置，如果true则只进行Minor GC，如果false则进行Full GC。 |
| ParallelGCThreads    | 设置并行GC时进行内存回收的线程数。默认值为CPU的核数。 |
| GCTimeRatio    | GC时间占总时间的百分比。默认值为99，即只允许1%的GC时间，**仅在使用Parallel Scavenge收集器时有效**。 |
| MaxGCPauseMillis    | 设置GC最大停顿时间。仅在使用Parallel Scavenge收集器时有效。 |
| UseCMSCompactAtFullCollection    | 设置CMS收集器在完成垃圾回收了之后是否进行一次内存碎片整理。仅在使用CMS收集器时有效。 |
| CMSFullGCsBeforeCompaction    | 设置CMS收集器在进行了若干次垃圾回收了之后是否再进行一次内存碎片整理。仅在使用CMS收集器时有效。|

# 参考文章：
---

* [GC性能优化](http://blog.csdn.net/column/details/14851.html)：对各个垃圾回收算法，垃圾回收器的实现，调优策略（吞吐量、延迟时间）等有较为详细的介绍，英文版本参考[这里](https://plumbr.eu/handbook/what-is-garbage-collection)

* [关键业务系统的JVM参数推荐(2016热冬版)](http://calvin1978.blogcn.com/articles/jvmoption-2.html)

* [深入理解Java虚拟机](https://read.douban.com/ebook/15233695/)






