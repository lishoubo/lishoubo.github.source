---
title: MQ主备方案(续)
tags:
  - 架构
  - 高可用
categories: 技术
date: 2017-06-12 16:59:12
---


[前一篇](/2017/06/11/MQ主备方案/)写了下MQ HA方案的整体架构，下面介绍一下在实现过程面临的一些问题。

### HA主从同步协议

MQ主从同步借助于Netty实现，主从之间的通信协议主要包括两部分：心跳和数据同步。

心跳有Slave发起，每次汇报自己的最大物理偏移；数据同步由Master发起，包括本次数据的起始位点和消息数据。整体的结构如下：

![](/images/middleware/mq-ha-6.png)

#### 心跳

客户端定期发起心跳，心跳信息一个8字节的maxPyhOffset，表示自己的最大消息物理偏移。Master拿到这个便宜后，通知上层HAService，唤醒阻塞住的同步请求；另外，Master定期监测链路的Idle事件，当有Idle All事件触发后，关闭链接，客户端发起重连。

#### 数据传输

数据传输由Master发起，Master定期监测有没有数据更新，如果有，就会将数据推送至Slave，数据格式包括：

* 8字节的消息启示偏移。该偏移有服务端保存，每次传输的时候更新；Slave获取该值后和本地的消息最大偏移进行对比，如果不一致，则认为数据传输错乱，关闭连接重新发起重连
* 4字节的消息长度标识，表示本次数据传输的数据量。Master会对传输的数据量进行流控
* 消息数据。Master在传输数据的时候，不会对消息格式进行验证，只是简单的二进制数据传输。这种设计简化了Master的数据同步工作，但是需要Slave再回放消息的时候需要对消息进行验证

在实现上，借助于Netty的lengthfieldbasedframedecoder来解析tcp数据

### 遇到的问题

Netty用起来很爽，在写代码的时候，顺便看了Netty的代码以及网络上介绍的各种问题，结合HA的实现，总结一下

#### Slave failed to open a new selector

为了保证数据传输的正确性，Slave除了心跳机制外，每次验证位点不一致都会关闭连接发起重连；另外，实际中，Master可能处于各种状态，都要保证Slave会一直尝试连接Master，所以在实现时，Slave会死循环尝试连接Master，结果测试的时候就出现了下面的错误：

```
exception in thread "HAClient" java.lang.IllegalStateException: failed to create a child event loop
	at io.netty.util.concurrent.MultithreadEventExecutorGroup.<init>(MultithreadEventExecutorGroup.java:68)
	at io.netty.channel.MultithreadEventLoopGroup.<init>(MultithreadEventLoopGroup.java:49)
	at io.netty.channel.nio.NioEventLoopGroup.<init>(NioEventLoopGroup.java:61)
	at io.netty.channel.nio.NioEventLoopGroup.<init>(NioEventLoopGroup.java:52)
	at com.vdian.vdianmq.broker.ha.HAClient.connectedToMaster(HAClient.java:99)
	at com.vdian.vdianmq.broker.ha.HAClient.run(HAClient.java:60)
	at java.lang.Thread.run(Thread.java:745)
Caused by: io.netty.channel.ChannelException: failed to open a new selector
	at io.netty.channel.nio.NioEventLoop.openSelector(NioEventLoop.java:128)
	at io.netty.channel.nio.NioEventLoop.<init>(NioEventLoop.java:120)
	at io.netty.channel.nio.NioEventLoopGroup.newChild(NioEventLoopGroup.java:87)
	at io.netty.util.concurrent.MultithreadEventExecutorGroup.<init>(MultithreadEventExecutorGroup.java:64)
	... 6 more
Caused by: java.io.IOException: Too many open files in system
	at sun.nio.ch.IOUtil.makePipe(Native Method)
	at sun.nio.ch.KQueueSelectorImpl.<init>(KQueueSelectorImpl.java:84)
	at sun.nio.ch.KQueueSelectorProvider.openSelector(KQueueSelectorProvider.java:42)
	at io.netty.channel.nio.NioEventLoop.openSelector(NioEventLoop.java:126)
	... 9 more
```

看下代码，发现Netty在EventLoopGroup在创建的时候，会打开一个Selector，而Selector可以实现多路复用，因此，重连的时候，只需要关闭channel即可，不需要shutdown EventLoopGroup。

#### exceptionCaught

需要注意的是，Netty的Handler里面的exceptionCaught，它只是从inbound往上抛，而不会往下，这样造成的问题就是，我们必须在最后一个handler处理异常，否则，如果在中间的handler处理的话，会漏掉一些handler抛出的异常，出现下面的日志：

```
WARNING: An exceptionCaught() event was fired, and it reached at the tail of the pipeline. It usually means the last handler in the pipeline did not handle the exception
```

#### Netty4的线程模型


Netty4的线程模型如下：

![](/images/middleware/mq-ha-7.png)

inbound和outbound两条线都是在nio的线程里面执行，这个时候就要注意：

* ctx write是异步的，调用后返回，但数据还没有写入（本地buffer），因此，要避免write(flush也一样）返回后，重用数据对象造成数据错乱
* 不要再handler里面（outboud和inbound）处理一些耗时的操作，否则会降低netty的处理性能
* 我们一般在业务线程处理业务逻辑和一些耗时的操作，然后，调用channel来写回，这个时候主要bytebuf的跨线程释放的问题：pooled的bytebuf只有在同一个线程里面才会被复用,详细代码可以看看PoolArena这个类的free方法
* outboud的时候，只有当消息被flush的时候，才会调用写入对象的release（如果写入的对象实现了referencecount对象）

#### 文件结束符

在做主从同步的时候，好多MQ的细节设计很关键：

* 每隔消息文件固定大小，这就保证了，主从同步的时候，只要消息位点是对的，在slave节点上重放出来的消息文件就会保持一致
* 每隔消息文件的结束标识，虽然文件固定大小，但是每个消息大小不一样，这就会导致每个消息文件实际存储的消息数目是不一致的，那每个消息文件的结尾实际上也是不一致的，因此在设计的时候，需要在每个消息文件末尾插入一个结束标识。根据这个标识，对消息生成索引的时候，就可以实现按照实际的索引文件顺序来在slave上重现build 索引。
* 每个消息自带seq字段。这个的重要性上一篇以及讲了

