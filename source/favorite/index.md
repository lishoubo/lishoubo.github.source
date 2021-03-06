---
title: 收藏的链接
date: 2017-07-04 17:25:50
---

* [因为delay ack和Nagle算法，导致小的tcp包延迟40ms左右的问题. 协议设计避免写-写-读的模式.](http://jm.taobao.org/2017/06/01/20170601/)
* 关于close_wait, time_wait的解释。[1](http://blog.oldboyedu.com/tcp-wait/),[2](https://typecodes.com/cseries/tcpdumpwiresharkclosewait2.html)
* [rocketMQ高性能的一些优化点，推荐](http://jm.taobao.org/2017/03/23/20170323/)
* [讲了JIT的依据；java server和java client的区别；栈上替换（OSR）；动态反优化；核心思想：因为JVM的一些优化，死代码清楚，预热等等，造成我们的一些测量并不是真正的测量.](https://www.ibm.com/developerworks/cn/java/j-jtp12214/index.html)
* [简单清晰的讲了MTU，MSS](http://blog.crhan.com/2014/05/mtu-and-mss/)
* [清晰的解释了TCP timestamp的作用；讲清楚了tcp－recycle在NAT网络环境，因为per-host的PAWS机制，timestamp无法保持同步的原因](http://perthcharles.github.io/2015/08/27/timestamp-intro/)
* [HTTP2.0介绍](https://developers.google.cn/web/fundamentals/performance/http2/?hl=zh-cn)，[HTTP2.0头部压缩](https://imququ.com/post/header-compression-in-http2.html)，[霍夫曼编码](http://coolshell.cn/articles/7459.html)
* [irqbalance在压力大的时候会导致中断自动漂移，对性能造成不稳定因素](http://blog.yufeng.info/archives/2422), [可以将不同的硬件中断，例如网卡，绑定到指定的cpu上，IRQ Affinity, 通过 cat /proc/interrupts 查看每个中断在各个cpu上的绑定情况](http://www.vpsee.com/2010/07/load-balancing-with-irq-smp-affinity/)
* [backlog参数的介绍，新的linux（2.2）后面，将半连接状态和全连接状态分开存储](http://www.cnblogs.com/Orgliny/p/5780796.html)