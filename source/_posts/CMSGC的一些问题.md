---
title: CMSGC的一些问题
tags:
  - GC
  - 内存
categories: 技术
date: 2017-04-15 11:18:23
---


最近排查了一次业务的FullGC，顺便理一下各种GC的日志和问题，记录下来。

### 正常的CMSGC的日志分析

#### 初始标记(STW initial mark)

根据GC Roots，标记出直接可达的活跃对象，这个阶段STW

```
2017-04-13T20:17:43.636+0800: 94618.250: [GC [1 CMS-initial-mark: 3483806K(4194304K)] 3601223K(5138048K), 0.0750220 secs] [Times: user=0.05 sys=0.02, real=0.08 secs]
```
日志显示了前后old区的所有对象占内存大小和old的capacity，整个JavaHeap（不包括perm）所有对象占内存总的大小和JavaHeap的capacity

#### 并发标记(Concurrent marking)

根据初始标记标记出的活跃对象，并发迭代便利所有活跃一下，该过程和用户线程并发执行

```
//正常的并发标记：
2013 - 11 -27T04: 00 : 12.958 + 0800 :  38892.883 : [CMS-concurrent-mark-start]
2013 - 11 -27T04: 00 : 19.231 + 0800 :  38899.155 : [CMS-concurrent-mark:  6.255 / 6.272  secs] [Times: user= 8.49  sys= 1.57 , real= 6.27  secs]
```
并发标记阶段，占用 6.255秒CPU时间, 6.272秒墙钟时间(也包含线程让出CPU给其他线程执行的时间)

但是看我们业务的日志里面并发标记失败，出现：concurrent mode failure，这个后面会讲到。

#### 并发预清理(Concurrent precleaning)

并发预清理阶段仍然是并发的。在这个阶段，虚拟机查找在执行并发标记阶段新进入老年代的对象(可能会有一些对象从新生代晋升到老年代， 或者有一些对象被分配到老年代)。通过重新扫描，减少下一个阶段”重新标记”的工作，因为下一个阶段会Stop The World

```
2013 - 11 -27T04: 00 : 19.231 + 0800 :  38899.155 : [CMS-concurrent-preclean-start]
2013 - 11 -27T04: 00 : 19.250 + 0800 :  38899.175 : [CMS-concurrent-preclean:  0.018 / 0.019  secs] [Times: user= 0.02  sys= 0.00 , real= 0.02  secs]
```

时间的含义跟并发标记一样

#### 重新标记(STW remark)

由于并发标记阶段是和用户线程并发执行的，该过程中可能有用户线程修改某些活跃对象的字段，指向了一个非标记过的对象，在这个阶段需要重新标记出这些遗漏的对象，防止在下一阶段被清理掉，这个过程也是需要stw

```
40.704: [GC40.704: [Rescan (parallel) , 0.1790103 secs]40.883: [weak refs processing, 0.0100966 secs] [1 CMS-remark: 26386K(786432K)] 52644K(1048384K), 0.1897792 secs]
```

日志显示重新扫描费时0.1790103秒，处理弱引用对象费时0.0100966秒，本阶段费时0.1897792 秒。后面是old区以及整个堆的内存变化。

#### 并发清理(Concurrent sweeping)

并发清理阶段，在清理阶段，应用线程还在运行。

```
4392.609: [CMS-concurrent-sweep-start]  
4394.310: [CMS-concurrent-sweep: 1.595/1.701 secs] [Times: user=4.78 sys=1.05, real=1.70 secs]  
```
并发清理阶段费时1.595秒

#### 并发重置(Concurrent reset)

这个阶段，重置CMS收集器的数据结构，等待下一次垃圾回收

```
2013 - 11 -27T04: 00 : 26.436 + 0800 :  38906.360 : [CMS-concurrent-reset-start]
2013 - 11 -27T04: 00 : 26.441 + 0800 :  38906.365 : [CMS-concurrent-reset:  0.005 / 0.005  secs] [Times: user= 0.00  sys= 0.00 , real= 0.00  secs]
```

### CMSGC其他日志

#### 并发可中止预清理(concurrent abortable preclean)阶段

这个阶段主要是在重新标记之前确保eden区达到期望的空间占用率。<strong>CMSGC会把新生代的对象作为GC Root</strong>，就像在介绍ParNewGC的时候，youngc 也会扫描card table 。因为重新标记阶段是STW的，所以要尽可能的缩短这个过程。

```
7688.150: [CMS-concurrent-preclean-start]
7688.186: [CMS-concurrent-preclean: 0.034/0.035 secs]
7688.186: [CMS-concurrent-abortable-preclean-start]
7688.465: [GC 7688.465: [ParNew: 1040940K->1464K(1044544K), 0.0165840 secs] 1343593K->304365K(2093120K), 0.0167509 secs]
7690.093: [CMS-concurrent-abortable-preclean: 1.012/1.907 secs]
7690.095: [GC[YG occupancy: 522484 K (1044544 K)]7690.095: [Rescan (parallel) , 0.3665541 secs]7690.462: [weak refs processing, 0.0003850 secs] [1 CMS-remark: 302901K(1048576K)] 825385K(2093120K), 0.3670690 secs]
```

可以通过参数：CMSScheduleRemarkEdenSizeThreshold来控制期望的eden区占比，当eden区占比大于这个值后，就会在preclean和remark阶段插入一个abortable preclean

#### promotion fail & concurrent mode failure

像下面的日志：

```
197.976: [GC 197.976: [ParNew: 260872K->260872K(261952K), 0.0000688 secs]197.976: [CMS197.981: [CMS-concurrent-sweep: 0.516/0.531 secs]
(concurrent mode failure): 402978K->248977K(786432K), 2.3728734 secs] 663850K->248977K(1048384K), 2.3733725 secs]
```

出现这个问题的根本原因就是old区无法满足需求，有可能是“真”的满足不了：有大对象，或者对象产生的速度太频繁，或者CMSInitiatingOccupancyFraction设置的太高导致old区来不及回首；也有可能是old区有内存，但是碎片太多，导致没有连续的内存在存放对象.

### 其他

#### CMS默认不会对永久代进行垃圾收集

所以，要防止对永久代的过度使用，否则，只能有full gc来做压缩了。或者设置XX:+CMSClassUnloadingEnabled来启用类卸载功能。


