---
title: Docker机YGC拉长问题
date: 2018-09-02 21:28:19
tags:
  - GC
categories: 技术

---

最近准备在云上进行扩容演练，上云统计几个业务的YGC情况：

|模块|日期|YGC平均耗时MS（Docker/物理机）|YGC次数（Docker/物理机）|
|---|---|---|---|
|模块一| 0820 |0.0230139/0.0144614|3162/7668
|| 0823 |0.0292437/0.017284|476/1134
|| 0824 |0.0243489/0.0112218|4093/9854
|模块二| 0814 |0.0188349/0.0152121|3459/2569
|| 0823 |0.0233527/0.0187653|2064/2049
|| 0828 |0.0271809/0.018225|564/569

可以看出来，YGC的次数没有什么规律，但是YGC的时间确实比较明显：docker机器比物理机基本拉长了1倍，这就比较奇怪了。

#### Java的GC线程

看了下我们线上的java启动参数，没有指定ParallelGCThreads和ConcGCThreads参数，于是到线上统计了两个模块的线程数量：

* jinfo -flag ParallelGCThreads
* jinfo -flag ConcGCThreads

|模块| ParallelGCThreads线程数（Docker/物理机）| ConcGCThreads线程数（Docker/物理机）| CPU核数（Docker/物理机）
|---|---|---|---|
|模块一| 38/28 |10/7|56/40
|模块一| 33/28|9/7|48/40

([一些资料](https://www.ibm.com/developerworks/cn/java/j-lo-JVMGarbageCollection/index.html))


GC线程数量的计算方式：

```
ParallelGCThreads = (ncpus <= 8) ? ncpus : 3 + ((ncpus * 5) / 8)

CMS默认启动的并发线程数是（ParallelGCThreads+3）/4。
```
docker的GC线程比物理机高是因为docker的宿主机高，但是，这个地方如果使用宿主机就不合理，会导致docker的GC线程过高，进而因为线程竞争线程切换导致GC性能降低

#### 解决方案

理想的方案肯定是根据docker分配的CPU核数来动态计算，所以，需要在启动脚本里面做环境识别，这个要根据公司的运维场景了，每个公司有不同的环境变量，但思路是一致的：不能只根据宿主机CPU核数来计算GC线程数。