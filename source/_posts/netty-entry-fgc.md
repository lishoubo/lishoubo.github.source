---
title: netty-entry-fgc
date: 2017-03-21 15:20:27
tags: [netty,问题排查]
categories: 中间件
---

&#160; &#160; &#160; &#160;公司线上的MQ（公司自研的MQ，类似于rocketMQ）出现fgc，于是把内存dump下来后进行分析。


#### 第一步分析

&#160; &#160; &#160; &#160;通过MAT的leak suspects，快度定位到是netty ChannelOutboundBuffer$Entry有内存泄漏的问题：

![mat-01.png](/images/netty-entry-fgc-01.png)

将所有的Entry展现出来：

![mat-02.png](/images/netty-entry-fgc-02.png)

第一眼看过去发现两个问题：

* Entry占了好多内存
* Entry的量好大

进一步点开其中一个Entry看：

![mat-03.png](/images/netty-entry-fgc-03.png)

发现：

1. 每个Entry都有一个next指针，这么看，所有的entry是被串成了一个链表
2. Entry对象里面有一个3KB多的数据，把所有的entry排序后，发现，每个entry里面都有这3KB的数据，当所有entry的量上去之后，内存消耗就大了

第1点：查看Netty代码，发现ChannelOutboundBuffer$Entry使用了基于[Thread Local](http://localhost:4000)的cache策略，所有的Entry串成list，而这些Entry不会被释放掉，进入了old区。

第2点：查看MQ代码，发现消息实体在deallocate的时候，没有将里面的list释放掉，于是，这3KB的数据就会被累加起来

```
public void deallocate() {

        if (null != messages) {
            for (MessageWrapper messageWrapper : messages) {
                messageWrapper.onCompletion();
            }
            // messages没有被释放
        }
    }
```    

##### 第一步分析结论

&#160; &#160; &#160; &#160;我们的代码有内存泄漏，导致每次请求会有3KB左右的数据被累加起来；

&#160; &#160; &#160; &#160;措施：在回收消息对象的回调里面，将list清理s
