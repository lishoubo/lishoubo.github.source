---
title: 高并发统计qps
date: 2017-07-04 10:04:15
tags: [Java,性能]
categories: 技术
---

最近在写一个dubbo组件，用来统计服务的qps等指标，考虑到高并发的场景，就去研究了下[Netflix的RollingNumber](https://github.com/Netflix/Hystrix/blob/master/hystrix-core/src/main/java/com/netflix/hystrix/util/HystrixRollingNumber.java)，收获还不少（也有点杂）。

#### 关于longadder

为什么longadder的性能会优于AtomicLong，这里有比较[详细的解释](http://coolshell.cn/articles/11454.html),基本有两个原因：

1. 避免多线程冲突。AtomicLong在多线程冲突的时候，会让冲突的线程自旋等待，而longadder则通过线程hash，让每个线程更新不同的cell，这样就避免了冲突
2. 通过padding，避免了[内存的伪共享](http://ifeve.com/falsesharing/)

另外，要提的一点是java8里面提供了@Contended注解，被这个注解修饰的字段会和其他字段使用不同的cache line。可以通过下面的命令来查看：

```
java -cp target/demo-1.0-SNAPSHOT.jar -XX:+PrintFieldLayout ContentedObj
```

(ps: PrintFieldLayout需要debug版本的jdk)

#### longadder压测

看起来这么牛逼，自然想压测下看看，然后又了解了[jmh](http://openjdk.java.net/projects/code-tools/jmh/),一个微基准测试框架，用起来很简单。

例如，编写一个对longadder的测试如下：

```
/**
 * Created by lishoubo on 17/7/3.
 */
@State(Scope.Benchmark)
@Threads(4)
@Warmup(iterations = 5)
@Measurement(iterations = 5, time = 20, timeUnit = TimeUnit.SECONDS)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@BenchmarkMode(Mode.Throughput)
public class LongAdderJMHBench {
    private LongAdder longAdder = new LongAdder();

    @Benchmark
    public void longAddderBenchmark() {
        longAdder.increment();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(LongAdderJMHBench.class.getSimpleName())
                .build();
        new Runner(opt).run();
    }
}
```

同样的，编写一个AtomicLong的测试。在我的mac的运行结果如下：

> 环境：2.7 GHz Intel Core i5，2核

> AtomicLong: 33946.890 ±(99.9%) 1511.374 ops/ms Average (min, avg, max) = (27910.393, 33946.890, 40546.125), stdev = 3053.051

> LongAdder: 232074.921 ±(99.9%) 6109.323 ops/ms Average (min, avg, max) = (190614.773, 232074.921, 242731.652), stdev = 12341.135


这是并发数为4的时候，当并发数更高的时候，LongAdder的优势更大。

#### 关于jmh

刚开始压测的是，我自己简单写了一个压测的代码，虽然也能证明longadder确实比AtomicLong要好，但是没有jmh做的这么规范。浏览下jmg的代码，可以借鉴的有：

1. 预热。执行测试之前，先进行预热，消除环境的影响，例如[jit](https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html)的影响 

2. gc一次，通过system.gc显示触发一次fgc，保证内存没有收到影响
3. 同步线程进度，等待所有的压测线程都准备好（而不是都new出来）。例如，可以通过下面的办法（worker线程）：

	```
	public void run() {
        controller.waitForBarrier();
        for (int i = 0; i < controller.addCount; i++) {
            controller.incrementCounter();
        }
        controller.waitForBarrier();
    }
	```
	在run方法里面，通过CyclicBarrier让所有的压测线程和当前主线程到达同样的状态，主线程代码：
	
	```
	/**
     * 同步开始，主线程等待所有的worker线程就绪
     */
    waitForBarrier();

    /**
     * 等待结束，主线程等待所有的worker线程完成
     */
    waitForBarrier();
	```
4. 不能只统计平均值，最小，最大，方差也要统计，这个时候我们可以使用common-math3里面的[DescriptiveStatistics](http://commons.apache.org/proper/commons-math/javadocs/api-3.3/org/apache/commons/math3/stat/descriptive/DescriptiveStatistics.html)来计算。

#### 关于RollingNumber

RollingNumber的高性能关键有两点：

1. 采用了LongAdder，见前面的压测
2. 实现了一个无锁（基本没有）的环形队列。环形队列的位置固定，可以在同一个位置上填上不同的bucket，而不同的bucket是根据当前的时间计算出来的。这样实现上就有个好处：不会出现时间覆盖后，复用了bucket里面老的值；因为随着时间的增长，新创建了一批bucket，这些bucket被放到了环形队列的对应位置上

RollingNumber怎么避免多线程冲突的呢？说白了就是牺牲一下精度，从而提高了速度。

```
/***如果当前时间没有对应的bucket，那么走到这里，进行创建***/
//1. 只有一个线程获取锁成功
if (newBucketLock.tryLock()) {
	//根据时间，在环形队列上补齐bucket
}//2. 其他线程没有获取成功，走到这一个分支 
else {
    //3. 直接返回最后一个，而不是按照时间上严格对应的一个bucket
    currentBucket = buckets.peekLast();
    if (currentBucket != null) {
        return currentBucket;
    } else {
        //4. 几率很小，只出现在第一个bucket的创建上
        try {
            Thread.sleep(5);
        } catch (Exception e) {
            // ignore
        }
        return getCurrentBucket();
    }
    }
```

可以看到，再第3步，没有让所有的线程再等待当前时间对应的bucket创建完毕，而是让那些没有锁的线程尽快拿到一个bucket就返回，这个bucket对应的时间窗口可能不准确，但是可以接受。
