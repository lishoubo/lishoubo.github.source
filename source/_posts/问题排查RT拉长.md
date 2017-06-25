---
title: 问题排查RT拉长
date: 2017-06-25 19:58:54
tags: [问题排查]
categories: 性能
---

负责公司的DUBBO，业务在压一个DUBBO服务的时候，发现RT逐渐拉长,从最开始的4ms，到后面的18ms，业务觉得比较诡异，怀疑是DUBBO的问题，于是就到了我这

### 现象

接到问题，马上到业务压测现场，看了下基本的数据：

1. 刚开始比较正常，压测端rt在3～4ms
2. 压测端rt慢慢拉长，涨到了18～20ms，整体qps降了下来
3. 服务端的性能统计发现，服务端的时间基本稳定
4. 我大概看了下机子的基本指标：cpu在30%，网络没有到瓶颈（才几百K）
5. 没有io瓶颈
6. gc没有问题，没有cms gc，ygc时间正常

大概看了下来，没有发现大的问题。

### 排查

在业务那边大概扫了下没有发现问题，就看了下业务的代码，业务的线程模型基本是这样的：

1.tomcat worker线程－－－2. 提交到业务线程池----3. 业务线程池里面访问dubbo服务

业务在第二步会提及多个任务，然后通过future机制在等待并发请求的返回结果。

看起来，也没有问题。不过业务自己为了排查问题，把第二步的线程池去掉了，然后只访问一个DUBBO服务（让代码简单些），在压测，发现rt降了下来；业务就开始怀疑dubbo服务是不是跟其他中间件冲突了。。。


看压测机器没有发现问题，看业务的代码也没有发现问题，没有办法，我就申请了压测机器的权限，然后jstack了一把。***（业务自己也jstack过了，没有发现问题）***

```
"pool-8-thread-1" prio=10 tid=0x00007f7c30097000 nid=0x5043 waiting for monitor entry [0x00007f7c21a67000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.weidian.wdp.slf4j.utils.CommonUtil.addInvokeTime(CommonUtil.java:162)
        - waiting to lock <0x00000006f0408428> (a java.lang.Class for com.weidian.wdp.slf4j.utils.CommonUtil)
        at com.vdian.tradeserver.web.DubboInvokeTimeFilter.invoke(DubboInvokeTimeFilter.java:23)

```

### Jstack文件

通过jstack文件，明显可以看到有很多线程被block了，都在等 monitor entry，这说明业务再等一个临界区资源，类似于syncronized资源，结合堆栈信息的这个代码：

```
com.weidian.wdp.slf4j.utils.CommonUtil.addInvokeTime
```

我怀疑这个代码不合理了使用syncronized，就跟业务说了下，业务看了代码，果然是每次请求都在syncronized(CommonUtil.class),于是当流量上来的时候，其他的线程就被blocked了。


### 还有一点

刚开始业务同学也有jstack，却没有发现问题，事后讨论下来，可能是当时rt还没有上去的时候dump的，我的建议是：

>jstack是一个静态的快照，一次dump很可能发现不了问题，要在程序的各个状态都jstack一把，进行对比




