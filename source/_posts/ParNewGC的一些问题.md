---
title: ParNewGC的一些问题
tags:
  - GC
  - 内存
categories: 技术
date: 2017-04-14 18:18:23
---



最近排查了一些FullGC的问题，就顺带着把Java的GC理一下, 本文捡一些ParNewGC的内容看看。

### 日志内容

```

2017-04-13T10:56:37.593+0800: 14084483.509: [GC 14084483.509: [ParNew: 1703042K->29303K(1887488K), 0.0293770 secs] 2070605K->397097K(5033216K), 0.0297680 secs] [Times: user=0.11 sys=0.00, real=0.03 secs]

```

ParNewGC的日志包括：

* GC开始时间
* GC算法类型
* GC前后young及整个Heap的使用量的变化
* GC耗时

观察ParNewGC的日志一般要注意：

1. 频率：我们网关的minor gc一般是每20s一次
2. 回收的内存变化：观察每次回收后young 区是不是基本被清理
3. 耗时：基本在ms级别

### JVM内存分配

我们基本上都知道eden，s0,s1,这些都是Java对堆区的分层划分。但是，Java在内存分配的时候还做了一些其他优化。

观察下面的代码：

```
private static class Foo {
    private int x;
    private static int counter;

    public Foo() {
        x = (++counter);
    }
}

//测试代码

for (int n = 0; n < nThreads; n++) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            waitBarrier(cyclicBarrier);

            for (int i = 0; i < total; ++i) {
                Foo foo = new Foo();
            }

            waitBarrier(cyclicBarrier);
        }
    }).start();
}    
    
```

通过两种方式进行测试, 方式一：开启<strong>逃逸分析</strong>,方法二：关闭逃逸分析. （通过打开或者关闭XX:+DoEscapeAnalysis）

```
java -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1 -Xloggc:/tmp/logs/gc.log -XX:+PrintGCDateStamps -verbose:gc -XX:+UnlockDiagnosticVMOptions -XX:-DisplayVMOutput -XX:+LogVMOutput -XX:LogFile=/tmp/logs/vm.log -server -Xms512m -Xmx512m -Xmn128m -XX:+UseConcMarkSweepGC  -XX:+DoEscapeAnalysis  -jar target/demo-1.0-SNAPSHOT.jar
```
观察结果：

>打开逃逸分析：time:601
>
>关闭逃逸分析：time:192199

差距大吧？背后的原理就是栈上分配

#### 逃逸分析和栈上分配

Java的JIT会对热点代码进行优化，优化的一个方面就是进行逃逸分析，如果代码不存在逃逸行为：

1. 方法逃逸：当一个对象在方法中定义之后，作为参数传递到其它方法中；
2. 线程逃逸：如类变量或实例变量，可能被其它线程访问到；

那么，JIT就会进行一系列的优化:

1. 同步消除: 消除无意义的加锁
2. 栈上分配：如果要分配的对象比较小，就会在栈空间分配

等等，关于JIT的文章可以[参考](https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html)

#### TLAB

接着上面的例子，在运行的时候添加参数：XX:-UseTLAB
```
java -XX:+PrintGCApplicationConcurrentTime -XX:+PrintGCApplicationStoppedTime -XX:+PrintGCDetails -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1 -Xloggc:/tmp/logs/gc.log -XX:+PrintGCDateStamps -verbose:gc -XX:+UnlockDiagnosticVMOptions -XX:-DisplayVMOutput -XX:+LogVMOutput -XX:LogFile=/tmp/logs/vm.log -server -Xms512m -Xmx512m -Xmn128m -XX:+UseConcMarkSweepGC  -XX:-DoEscapeAnalysis  -XX:-UseTLAB -jar target/demo-1.0-SNAPSHOT.jar
```

结果：
>time:2219433

再运行一遍，发现更慢了，那TLAB是干嘛的？

简单讲，TLAB就是jvm在eden区为每个线程拿出一块内存使用，因为每个线程单独一个，所以分配的时候不存在竞争，速度自然就快。要注意的是：TLAB适合小对象，如果对象比较大，就会在eden区申请，并且总体上有容量限制，默认为eden的1%。

### ParNewGC过程

大概过程如下：

1. 从root对象开始标记所有活的对象. GC Roots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）
2. 处理Card Table。Card Table记录了old区到young区的引用
3. 处理reference：java出了强引用之外还有好几种引用


可以作为GC root的有：

> Class - 由系统类加载器(system class loader)加载的对象，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象。我们需要注意的一点就是，通过用户自定义的类加载器加载的类，除非相应的java.lang.Class实例以其它的某种（或多种）方式成为roots，否则它们并不是roots，.
>
> Thread - 活着的线程
>
> Stack Local - Java方法的local变量或参数
>
> JNI Local - JNI方法的local变量或参数
>
> JNI Global - 全局JNI引用
> 
> Monitor Used - 用于同步的监控对象
> 
> Held by JVM - 用于JVM特殊目的由GC保留的对象，但实际上这个与JVM的实现是有关的。可能已知的一些类型是：系统类加载器、一些JVM知道的重要的异常类、一些用于处理异常的预分配对象以及一些自定义的类加载器等。然而，JVM并没有为这些对象提供其它的信息，因此就只有留给分析分员去确定哪些是属于"JVM持有"的了。

### 其他

#### PrintReferenceGC

统计younggc对reference的处理耗时，用于排查young gc拉长的场景

```
2017-07-14T17:47:45.276-0800: 5.617: [GC2017-07-14T17:47:45.276-0800: 5.617: [ParNew2017-07-14T17:47:45.278-0800: 5.619: [SoftReference, 0 refs, 0.0000930 secs]2017-07-14T17:47:45.278-0800: 5.619: [WeakReference, 0 refs, 0.0000110 secs]2017-07-14T17:47:45.278-0800: 5.619: [FinalReference, 0 refs, 0.0000090 secs]2017-07-14T17:47:45.278-0800: 5.619: [PhantomReference, 0 refs, 0 refs, 0.0000080 secs]2017-07-14T17:47:45.278-0800: 5.619: [JNI Weak Reference, 0.0000340 secs]: 104960K->2K(118016K), 0.0017400 secs] 105217K->259K(511232K), 0.0018410 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

#### StringTable造成young gc拉长

StringTable默认分配在heap里面而不是perm里，young gc不会对perm做回收，但会扫描StringTable，保证处于新生代的String不会被回收掉，所以，如果StringTable过大就会拉长young gc rt

#### SystemDictionary造成young gc拉长

SystemDictionary里面记录了class和classloader的对应关系，SystemDictionary作为GC root的一部分，而young gc并不会对类进行卸载，所以，如果SystemDictionary变大，young gc就会拉长。

