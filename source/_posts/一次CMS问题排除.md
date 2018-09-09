---
title: 一次CMS问题排除
date: 2018-08-25 16:11:58
tags: [问题排查]
categories: 技术
---

#### 定位问题

最近在做扩容演练，注意到一个问题，每次docker启动都会触发一次CMS GC，刚开始以为是docker性能不好，后来看了下线上物理机分配给每个应用的内存也比较小（4G），所以感觉不是这个问题，就认真看了一下。

接着查看了下其他的docker机器发现也有CMS GC，于是对比了docker机器和物理机的CMS GC次数，发现都是每个3天左右触发一次，docker机器没有明显的增多。

既然docker机器没有明显的CMS GC增多，只好看看每次触发的时机，发现到了一个有意思的现象：

	* 查看发布时间点，发现每次发布的时候，都会触发CMS GC。

初步的结论是每次发布都会触发CMS GC。于是，进一步查看GC 日志：

```
Java HotSpot(TM) 64-Bit Server VM (25.65-b01) for linux-amd64 JRE (1.8.0_65-b17), built on Oct  6 2015 17:16:12 by "java_re" with gcc 4.3.0 20080428 (Red Hat 4.3.0-8)
Memory: 4k page, physical 131779324k(4738628k free), swap 0k(0k free)
CommandLine flags: -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:+DisableExplicitGC -XX:InitialHeapSize=4294967296 -XX:LargePageSizeInBytes=134217728 -XX:MaxHeapSize=4294967296 -XX:MaxNewSize=2684354560 -XX:MaxTenuringThreshold=6 -XX:NewSize=2684354560 -XX:OldPLABSize=16 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:ThreadStackSize=256 -XX:+UseCMSCompactAtFullCollection -XX:+UseCMSInitiatingOccupancyOnly -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseFastAccessorMethods -XX:+UseParNewGC
2018-08-20T11:38:36.800+0800: 2.133: [GC (CMS Initial Mark) [1 CMS-initial-mark: 0K(1572864K)] 796919K(3932160K), 0.0577900 secs] [Times: user=0.08 sys=0.00, real=0.05 secs]
2018-08-20T11:38:36.858+0800: 2.191: [CMS-concurrent-mark-start]
2018-08-20T11:38:36.858+0800: 2.191: [CMS-concurrent-mark: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2018-08-20T11:38:36.858+0800: 2.191: [CMS-concurrent-preclean-start]
2018-08-20T11:38:36.861+0800: 2.195: [CMS-concurrent-preclean: 0.003/0.003 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
2018-08-20T11:38:36.861+0800: 2.195: [CMS-concurrent-abortable-preclean-start]
2018-08-20T11:38:38.811+0800: 4.144: [GC (GCLocker Initiated GC) 2018-08-20T11:38:38.811+0800: 4.145: [ParNew: 2097152K->30361K(2359296K), 0.0289636 secs] 2097152K->30361K(3932160K), 0.0296327 secs] [Times: user=0.36 sys=0.11, real=0.03 secs]
2018-08-20T11:38:40.025+0800: 5.358: [CMS-concurrent-abortable-preclean: 0.892/3.164 secs] [Times: user=8.26 sys=0.57, real=3.16 secs]
2018-08-20T11:38:40.025+0800: 5.359: [GC (CMS Final Remark) [YG occupancy: 1218768 K (2359296 K)]2018-08-20T11:38:40.025+0800: 5.359: [Rescan (parallel) , 0.0704570 secs]2018-08-20T11:38:40.096+0800: 5.429: [weak refs processing, 0.0000365 secs]2018-08-20T11:38:40.096+0800: 5.429: [class unloading, 0.0069690 secs]2018-08-20T11:38:40.103+0800: 5.436: [scrub symbol table, 0.0036416 secs]2018-08-20T11:38:40.106+0800: 5.440: [scrub string table, 0.0006395 secs][1 CMS-remark: 0K(1572864K)] 1218768K(3932160K), 0.0839617 secs] [Times: user=1.94 sys=0.01, real=0.08 secs]
2018-08-20T11:38:40.109+0800: 5.443: [CMS-concurrent-sweep-start]
2018-08-20T11:38:40.109+0800: 5.443: [CMS-concurrent-sweep: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2018-08-20T11:38:40.109+0800: 5.443: [CMS-concurrent-reset-start]
2018-08-20T11:38:40.116+0800: 5.450: [CMS-concurrent-reset: 0.007/0.007 secs] [Times: user=0.02 sys=0.01, real=0.00 secs]
```

跟业务经过确认，发布时间能够对的起来。疑问点：

* 每次发布，应用还没有启动完毕，会触发一次CMS
* CMS的时候，old区使用量为0:

```
2018-08-20T11:38:36.800+0800: 2.133: [GC (CMS Initial Mark) [1 CMS-initial-mark: 0K(1572864K)] 796919K(3932160K), 0.0577900 secs] [Times: user=0.08 sys=0.00, real=0.05 secs]

```

刚开始想到的是，我们的代码启动的时候主动触发了一次GC，可以，查看启动参数：

```
-server -Xmx4g -Xms4g -Xmn2g -XX:PermSize=128m -Xss256k -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+PrintGCDetails -XX:+PrintGCDateStamps
```

已经禁止了程序触发GC。如果不是主动触发，那么肯定是内存不够了，但是，old的使用量又是0，这个时候，只能考虑perm区了。

又看了下perm大小，我们基本的配置都是128M，应该足够，只好试着区排查一下：

```
jstat -gc 6798 1000
S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
262144.0 262144.0  0.0   1601.7 2097152.0 687465.5 1572864.0   522538.5  47724.0 46235.0 5288.0 4900.6   3141   45.946   2      0.142   46.087
```

有几个有意思的信息：

* MC，MU：显示perm才使用不到50M
* 为什么是MC，MU，而不是显示perm大小？

然后查看了jdk版本，我们线上使用的jdk8，而jdk8里面perm区做了修改，可以查看[资料](http://www.importnew.com/14933.html).

所以，我们的jvm参数配置：permSize，maxPermSize就会失效。

#### 发现问题

基本上可以确定是perm没有配置的原因，于是了解了下jdk8的metaspace机制，以及需要配置的参数，大家可以查看资料详细了解下，这里总结下：

|参数|含义|默认值|
|---|---|---|
|-XX:MetaspaceSize|Metaspace扩容时触发FullGC的初始化阈值，也是最小的阈值。|依照系统而区别，可以运行进行查看，我们的线上机子大概在20M.|
|-XX:MaxMetaspaceSize|jdk8 MaxMetaspace上限，配置太小触发OOM|默认是几乎无穷大，如果meta区内存泄漏，可以把系统内存吃掉|

可以通过下面的方式查看系统默认的metaspace初始阈值

```
java -XX:+PrintFlagsFinal -version |grep JVMParamName
```
因为我们是jdk8，而配置的XX:PermSize无效，所以，应用启动之后，meta区很快到达默认值（20M），然后触发FullGC（这个依赖于Old区配置，我们配置的是CMS），FullGC之后，meta区被扩容。

#### 建议

* -XX:MetaspaceSize和-XX:MaxMetaspaceSize设置一样大；我们的应用里面，check下如果jdk8，原来的-XX:PermSize=128m无效。
* 具体设置多大，建议稳定运行一段时间后通过jstat -gc pid确认且这个值大一些，对于大部分项目128m，或者256m即可。
