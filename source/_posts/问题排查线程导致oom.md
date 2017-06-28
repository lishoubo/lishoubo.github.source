---
title: 问题排查线程导致oom
date: 2017-06-28 10:15:53
tags:
---

有个项目发现mq消息堆积，上去看了下，发现日志中出现：
```
Caused by: java.lang.OutOfMemoryError: unable to create new native thread
```

OOM了，立马jstack了一把，发现好多下面的线程：

```
"pool-63-thread-4" prio=10 tid=0x00007f81601ee800 nid=0x1e79 waiting on condition [0x00007f8145353000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000007167b7490> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2043)
        at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1043)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1103)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:603)
        at java.lang.Thread.run(Thread.java:722)

```

好吧，一看线程名字就知道创建线程池的时候没有定制名字，浏览了下jstack文件，没有发现runnable的这样的线程，没办法定位到谁创建的，只好dump了一把

#### dump文件

dump下来之后，首先发现ThreadPool实例有3000多个，肯定有问题

![](/images/middleware/threadpool-oom-01.png)

然后，找到这样的线程池看了下：

![](/images/middleware/threadpool-oom-02.png)

注意到一个细节，就这这样的线程池基本都被Java referent引用的，说明这些线程池都已经进入finalizer队列，快被销毁了。

> 因为这些线程池都已经进入到了finalizer队列，很可能是在一个方法内部，或者一个临时对象里面，临时创建了这样的线程池。Java的ThreadPoolExecutor实现了finalize方法，因此当ThreadPoolExecutor对象没有被引用时，不会马上释放，而是进入到finalizer队列。

这样就比较难查了，因为没有业务代码引用着线程池，没法定位到问题

### Btrace

一个方案，可以使用btrace，来定位是谁在使用默认的Factory来创建线程池。

```
@BTrace
public class ThreadPoolCreateBtrace {
    @Export
    static long counter;

    @OnMethod(
            clazz = "java.util.concurrent.Executors",
            method = "defaultThreadFactory"
    )
    public static void onDefaultThreadFactory(@Self Object self) {
        if (counter < 10) {
            jstack();
            counter++;
        }
    }
}
```

执行btrace命令：
```
./btrace 1845 ThreadPoolCreateBtrace.java
```

就可以看到整个堆栈了(下面的写的demo，不是实际的业务代码）:

```
java.util.concurrent.Executors.defaultThreadFactory(Executors.java)
java.util.concurrent.ThreadPoolExecutor.<init>(ThreadPoolExecutor.java:1198)
java.util.concurrent.Executors.newFixedThreadPool(Executors.java:89)
com.vdian.lishoubo.demo.threadpool.ThreadPoolCreator.createNewPool(ThreadPoolCreator.java:25)
com.vdian.lishoubo.demo.threadpool.ThreadPoolCreator$1.run(ThreadPoolCreator.java:17)
```

这样就可以很快定位问题

### Greys

[https://github.com/oldmanpushcart/greys-anatomy](https://github.com/oldmanpushcart/greys-anatomy)

如果使用btrace麻烦的话，还可以使用greys。

> 注意，greys默认不会增强系统的类

1. 打开系统类增强：

```
options unsafe true
```

2. 跟踪堆栈

```
ga?>stack java.util.concurrent.Executors defaultThreadFactory
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 50 ms.
thread_name="Thread-0" thread_id=0x8;is_daemon=false;priority=5;
    @java.util.concurrent.Executors.defaultThreadFactory(Executors.java:-1)
        at java.util.concurrent.ThreadPoolExecutor.<init>(ThreadPoolExecutor.java:1198)
        at java.util.concurrent.Executors.newFixedThreadPool(Executors.java:89)
        at com.vdian.lishoubo.demo.threadpool.ThreadPoolCreator.createNewPool(ThreadPoolCreator.java:25)
        at com.vdian.lishoubo.demo.threadpool.ThreadPoolCreator$1.run(ThreadPoolCreator.java:17)
```
3. 别忘了reset回来，取消增强

```
reset
```


