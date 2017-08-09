---
title: 问题排查一次诡异的rt拉高现象
date: 2017-08-09 14:23:21
tags: [问题排查, Java, 性能]
categories: 技术
---

升级了公司的Dubbo到3.0.0版本，然后推动业务升级，结果，业务反馈升级后，rt拉高了2～3ms，由于业务的整体rt才在7ms左右，所以，这次升级对业务的影响还是很大的，业务决定回滚，而我这边就开始了问题的排查


#### 现象

根据我们的监控，可以很明显看到升级新的dubbo后的rt变化:

![](/images/middleware/rt-more-02.png)

同时，观察了升级后的cpu变化：

![](/images/middleware/rt-more-01.png)

现在看，升级新的dubbo后，直接的结果就是：cpu和rt都被拉高了

#### 压测

首先就是先看了代码的变更，梳理了提交历史，发现这次dubbo发布主要是添加了StatisticFilter，用于统计每次服务调用的rt，然后，进行汇总输出到日志文件，实现是借助于[Netflix的RollingNumber](https://github.com/Netflix/Hystrix/blob/master/hystrix-core/src/main/java/com/netflix/hystrix/util/HystrixRollingNumber.java)，具体介绍可以看我的[另一篇文章](/2017/07/04/高并发统计qps/)。 当时通过代码分析觉得这个实现确实应该很快，但是没有压测，难道问题出在这里吗？

接下来写了一个简单的demo，用Thread.sleep模拟业务耗时（5ms），对不同对dubbo版本进行了压测，结果如下：


##### 新版的DUBBO（带有statistic）
```
Result "com.learn.lishoubo.demo.bench.dubbo.StatisticBench.doRune":
  5.510 ±(99.9%) 0.116 ms/op [Average]
  (min, avg, max) = (5.190, 5.510, 5.800), stdev = 0.155
  CI (99.9%): [5.394, 5.627] (assumes normal distribution)


# Run complete. Total time: 00:13:00

Benchmark              Mode  Cnt  Score   Error  Units
StatisticBench.doRune  avgt   25  5.510 ± 0.116  ms/op
```

##### 业务的DUBBO（没有statistic）
```
Result "com.learn.lishoubo.demo.bench.dubbo.StatisticBench.doRune":
  5.239 ±(99.9%) 0.083 ms/op [Average]
  (min, avg, max) = (5.035, 5.239, 5.412), stdev = 0.110
  CI (99.9%): [5.156, 5.321] (assumes normal distribution)


# Run complete. Total time: 00:13:00

Benchmark              Mode  Cnt  Score   Error  Units
StatisticBench.doRune  avgt   25  5.239 ± 0.083  ms/o
```

期间还进行了更复杂的压测，模拟业务的场景：服务调用另外一个服务，但是结果一样：新版带statistic版本的dubbo和老的没有statistic斑驳的dubbo，服务的rt相差很少，不可能想线上反馈的那样：rt拉高了有2ms左右


#### 思维转换

通过压测没有发现问题，至少说明不应该是StatisticFilter引入的问题，接下来应该观察实际的业务场景来寻找问题。

<strong>一个好的思路就是，直接对比业务接了老的Dubbo和新的Dubbo后的CPU耗时变化，rt变化等各项指标，来定位问题。</strong>

但这样有点理想：因为业务只有一个性能环境，并且业务团队是在北京，配合起来是个问题，没法让他们快速的进行切换dubbo版本（人家也很忙），这就导致了我这边一直没有好的办法去定位问题，只能先让他们把dubbo升级到最新版本，一直在性能环境跑着，我时不时上去看看top啥的，不过除了cpu占比比较高之外，没有发现别的问题。

这期间一直研究了linux的perf，来看看到底哪儿占cpu比较严重，花了一段时间后，突然觉得，我可以换个思路：

<strong>因为我一直的想法是，通过对比来定位问题，但这样实际的操作性不高，那么，我为什么不当作一次性能优化呢？不去关心是不是引入了新的DUBBO引起的问题，就是去解决提高性能，把rt降低2ms的一个优化工作</strong>

我觉得，思路一旦转变了之后，立马有了行动的动力。

#### Linux Perf

要进行CPU调优，首先就要找出耗CPU的操作来，可以先通过linux的perf来试试：

> perf top -p 22175
> 
```
   4.68%  libjvm.so           [.] ContiguousSpace::prepare_for_compaction
   3.92%  [kernel]            [k] iowrite16
   3.85%  libjvm.so           [.] StringTable::intern
   2.84%  [kernel]            [k] _spin_unlock_irqrestore
   2.63%  libjvm.so           [.] InstanceKlass::oop_adjust_pointers
   1.97%  libjvm.so           [.] java_lang_String::equals
   1.94%  libjvm.so           [.] Symbol::as_klass_external_name
   1.86%  libjvm.so           [.] Method::line_number_from_bci
   1.62%  libjvm.so           [.] UTF8::convert_to_unicode
```

但是，linux的perf有个不好的地方，就是没法看Java的名字，于是找了mapping工具，可以将Java的类名映射出来的工具：[https://github.com/jvm-profiling-tools/perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent)

使用这个工具，可以比较好的查看Java方法的cpu耗时

#### String.intern

有了perf map agent之后，我到性能环境运行了看看，下面是cpu的耗时：

```
./perf-java-top 3472
   6.09%  libjvm.so           [.] StringTable::intern
   3.55%  libjvm.so           [.] java_lang_String::equals
   3.54%  libjvm.so           [.] Symbol::as_klass_external_name
   3.36%  [kernel]            [k] iowrite16
   3.17%  [kernel]            [k] _spin_unlock_irqrestore
   2.72%  libjvm.so           [.] UTF8::convert_to_unicode
   2.32%  libjvm.so           [.] Method::line_number_from_bci
   1.91%  libjvm.so           [.] java_lang_Throwable::fill_in_stack_trace
   1.74%  libjvm.so           [.] java_lang_StackTraceElement::create
   1.59%  libjvm.so           [.] UTF8::unicode_length
   1.43%  libjvm.so           [.] InstanceKlass::method_with_orig_idnum
   1.42%  libjvm.so           [.] BacktraceBuilder::push
   1.35%  libpthread-2.12.so  [.] pthread_getspecific
   1.19%  libjvm.so           [.] oopDesc::obj_field_put
   1.17%  [kernel]            [k] __do_softirq
   1.16%  [kernel]            [k] retint_careful
   1.11%  [kernel]            [k] finish_task_switch
   1.09%  libjvm.so           [.] UTF8::unicode_length
   0.99%  libjvm.so           [.] CodeHeap::find_start
   0.95%  libjvm.so           [.] Symbol::as_unicode
   0.85%  libjvm.so           [.] objArrayOopDesc::obj_at
   0.81%  perf-21271.map      [.] Ljava/util/Hashtable;::put
Press '?' for help o
```

发现很奇怪的就是，排名第一个竟然是StringTable.intern, 以前看过因为intern造成ygc拉长的问题，当时就是因为fastjson启用了intern的feature，导致fastjson每次操作symbol的时候，都会转换为intern。看到这心里很开心，以为立马找到问题了。

然后去翻FastJSON的源码，发现，后面的版本（至少我们使用的版本）已经解决了这个问题：

```
public final SymbolTable  symbolTable = new SymbolTable(4096);
public SymbolTable(int tableSize){
        this.indexMask = tableSize - 1;
        this.symbols = new String[tableSize];
        
        this.addSymbol("$ref", 0, 4, "$ref".hashCode());
        this.addSymbol(JSON.DEFAULT_TYPE_KEY, 0, JSON.DEFAULT_TYPE_KEY.length(), JSON.DEFAULT_TYPE_KEY.hashCode());
    }
```

FastJSON对SymbolTable做了大小限制，不会超过4096个。看来不是FastJSON的问题。。。。


#### Throwable.fillInStackTrace

没办法，只好继续看。往下看:

```
   1.91%  libjvm.so           [.] java_lang_Throwable::fill_in_stack_trace
   1.74%  libjvm.so           [.] java_lang_StackTraceElement::create
```
为什么会一直在填充throwable的stackTrace？难倒程序一直在抛异常？首先看了下业务日志，发现业务OK，那什么地方在不停的构造Throwable？

接下来，就该greys（或者btrace）上场了，用greys将Throwable的创建堆栈打了出来：

```
stack -n 10 java.lang.Throwable fillInStackTrace

@java.lang.Throwable.fillInStackTrace(Throwable.java:-1)
        at java.lang.Throwable.<init>(Throwable.java:250)
        at java.lang.Exception.<init>(Exception.java:54)
        at java.lang.Thread.getStackTrace(Thread.java:1556)
        at com.weidian.wdp.slf4j.utils.CommonUtil.getClassName(CommonUtil.java:70)
        at com.weidian.wdp.slf4j.WrapLogger.getMsg(WrapLogger.java:254)
        at com.weidian.wdp.slf4j.WrapLogger.debug(WrapLogger.java:90)
        at com.weidian.wdp.cache.jedis.RedisCacheClient.mget(RedisCacheClient.java:110)
        at com.weidian.wdp.cache.CacheManager.getBulkFromCache(CacheManager.java:165)
        at com.weidian.wdp.cache.CacheManageAspect.tryWithCache(CacheManageAspect.java:94)
        at sun.reflect.GeneratedMethodAccessor200.invoke(null:-1)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:629)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:618)
        at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:70)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:168)
        at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
        at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:656)
        at com.weidian.wdp.item.data.dao.impl.ItemCacheDAOImpl$$EnhancerBySpringCGLIB$$e84c9158.getItemBaseByItemIds(<generated>:-1)
        at sun.reflect.GeneratedMethodAccessor266.invoke(null:-1)
```

发现业务在打日志，看了下业务的实现:

```
com.weidian.wdp.slf4j.utils.CommonUtil.getClassName

public static String getClassName(){
        try {
            return Thread.currentThread().getStackTrace()[Constants.JVM_STACK_METHOD_DEEP].getClassName();
        }catch (Exception e){
            return "";
        }
    }
    
public StackTraceElement[] getStackTrace() {
        if (this != Thread.currentThread()) {
			.......
        } else {
            // Don't need JVM help for current thread
            return (new Exception()).getStackTrace();
        }
    }    
```
可以看到，业务打的日志操作确实会触发fillStackTrace，马上联系了业务，让他们把性能环境的日志去掉。

```
./perf-java-top 3472
  399: 5.91%  libjvm.so           [.] StringTable::intern                     
  400:    3.24%  libjvm.so           [.] java_lang_String::equals                
  401:    3.17%  libjvm.so           [.] Symbol::as_klass_external_name          
  402:    3.06%  [kernel]            [k] iowrite16                               
  403:    2.94%  [kernel]            [k] _spin_unlock_irqrestore                 
  404:    2.49%  libjvm.so           [.] UTF8::convert_to_unicode                
  405:    2.22%  libjvm.so           [.] Method::line_number_from_bci            
  406:    1.75%  libjvm.so           [.] java_lang_Throwable::fill_in_stack_trace
  407:    1.65%  libjvm.so           [.] java_lang_StackTraceElement::create     
  408:    1.47%  libjvm.so           [.] UTF8::unicode_length                    
  409:    1.34%  libjvm.so           [.] BacktraceBuilder::push                  
  410:    1.30%  libjvm.so           [.] InstanceKlass::method_with_orig_idnum   
  411:    1.06%  [kernel]            [k] __do_softirq                            
  412:    1.06%  libjvm.so           [.] oopDesc::obj_field_put                  
  413:    1.02%  libjvm.so           [.] UTF8::unicode_length                    
  414:    1.01%  libpthread-2.12.so  [.] pthread_getspecific                     
  415:    0.97%  [kernel]            [k] finish_task_switch                      
  416:    0.97%  [kernel]            [k] retint_careful                          
  417:    0.95%  libjvm.so           [.] PhaseIdealLoop::get_late_ctrl           
  418:    0.88%  libjvm.so           [.] Symbol::as_unicode                      
  419:    0.74%  libjvm.so           [.] Symbol::as_unicode                      
  420     0.70%  perf-29940.map      [.] Lorg/springframework/asm/ClassReader;::readConst     
  421:    0.64%  libjvm.so           [.] objArrayOopDesc::obj_at        
```

业务去掉日志后，再统计了下, 发现排第一个仍然是StringTable.intern，好吧，继续greys了一把：

```
java.lang.Throwable.fillInStackTrace(Throwable.java)
java.lang.Throwable.<init>(Throwable.java:250)
ch.qos.logback.classic.spi.LoggingEvent.getCallerData(LoggingEvent.java:259)
ch.qos.logback.classic.AsyncAppender.preprocess(AsyncAppender.java:45)
ch.qos.logback.classic.AsyncAppender.preprocess(AsyncAppender.java:1)
ch.qos.logback.core.AsyncAppenderBase.append(AsyncAppenderBase.java:147)
ch.qos.logback.core.UnsynchronizedAppenderBase.doAppend(UnsynchronizedAppenderBase.java:84)
ch.qos.logback.core.spi.AppenderAttachableImpl.appendLoopOnAppenders(AppenderAttachableImpl.java:48)
ch.qos.logback.classic.Logger.appendLoopOnAppenders(Logger.java:270)
ch.qos.logback.classic.Logger.callAppenders(Logger.java:257)
ch.qos.logback.classic.Logger.buildLoggingEventAndAppend(Logger.java:421)
ch.qos.logback.classic.Logger.filterAndLog_0_Or3Plus(Logger.java:383)
ch.qos.logback.classic.Logger.log(Logger.java:765)
org.apache.commons.logging.impl.SLF4JLocationAwareLog.debug(SLF4JLocationAwareLog.java:131)
org.apache.ibatis.logging.commons.JakartaCommonsLoggingImpl.debug(JakartaCommonsLoggingImpl.java:54)
org.apache.ibatis.logging.jdbc.BaseJdbcLogger.debug(BaseJdbcLogger.java:159)
org.apache.ibatis.logging.jdbc.PreparedStatementLogger.invoke(PreparedStatementLogger.java:52)
com.sun.proxy.$Proxy58.execute(Unknown Source)
org.apache.ibatis.executor.statement.PreparedStatementHandler.query(PreparedStatementHandler.java:63)
org.apache.ibatis.executor.statement.RoutingStatementHandler.query(RoutingStatementHandler.java:79)
org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:63)
org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:324)
org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:156)
org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:109)
org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:83)
org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:148)
org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141)
sun.reflect.GeneratedMethodAccessor198.invoke(Unknown Source)
sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.lang.reflect.Method.invoke(Method.java:498)
org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:434)
com.sun.proxy.$Proxy35.selectList(Unknown Source)
org.mybatis.spring.SqlSessionTemplate.selectList(SqlSessionTemplate.java:231)
org.apache.ibatis.binding.MapperMethod.executeForMany(MapperMethod.java:137)
org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:75)
org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:59)
com.sun.proxy.$Proxy41.getByItemIds(Unknown Source)
com.weidian.wdp.item.data.dao.impl.ItemCacheDAOImpl.getItemDescInfoByItemIds(ItemCacheDAOImpl.java:233)
com.weidian.wdp.item.data.dao.impl.ItemCacheDAOImpl$$FastClassBySpringCGLIB$$a0c45433.invoke(<generated>)
org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:721)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:656)
com.weidian.wdp.item.data.dao.impl.ItemCacheDAOImpl$$EnhancerBySpringCGLIB$$39e532f5.getItemDescInfoByItemIds(<generated>)
```

终于发现了点有意思的现象：业务调用mybatis，然后，mybatis会触发debug和trace的日志，而logback的LogginEvent获取callerData的方法如下：

```
public StackTraceElement[] getCallerData() {
        if (callerDataArray == null) {
            callerDataArray = CallerData
                            .extract(new Throwable(), fqnOfLoggerClass, loggerContext.getMaxCallerDataDepth(), loggerContext.getFrameworkPackages());
        }
        return callerDataArray;
    }
```

new了一个Throwable，在Throwable里面调用了fillStackTrace：

```
public Throwable() {
        fillInStackTrace();
}
```
奇怪，为什么业务的mybatis日志默认都是isDebugEnabled和isTraceEnabled呢？这样的话，mybatis会打印好多日志啊？跟业务沟通了一下，他们没有做什么特别的配置。没办法，只好去找mybatis看看了。

看了官网的mybatis的[日志介绍](http://www.mybatis.org/mybatis-3/zh/logging.html), 介绍了mybatis选日志的顺序，突然想到，新版的dubbo还有另外一个修改：使用logback，难道logback有问题？怎么都感觉不太可能。

#### 问题的定位

马上联系北京的业务，把dubbo版本倒退回去：回到老的dubbo版本，然后，greys了一把, 发现Throwable的fillStackTrace的调用堆栈里面，竟然没有了因为mybatis的日志输出造成的链路。这么奇怪？

现在，能够看出新版的直接问题就是：升级了新版的问题后，mybatis会打开logback的日志，级别还是trace级别。然后，每次都会构造Throwable，导致cpu利用率上升。而使用老版的dubbo的时候，mybatis就不会打开日志。

别急，这期间，我还查了一个问题，就是Java的Throwable在填充堆栈的时候，确实会使用String.intern,具体介绍见[这里](http://www.importnew.com/14142.html)。可以看到，在构造Throwable的时候，jdk7会导致StringTable被填满，这也就解释了我们的perf结果。

下一步，我又对比了新版的变动，发现这么一条改变：

```
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>log4j-over-slf4j</artifactId>
    <version>${log4j_slf4j_version}</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>${jcl_slf4j_version}</version>
</dependency>
```
当时修改dubbo的时候，觉得这么多日志太乱了，就想大家都用logback就好了。于是，把所有其他的日志都桥接到了logback, 现在问题就是出在这里:

> 业务配置了logback.xml，但是没有配置log4j.xml， 在老的dubbo版本里面，因为log4j找不到log4j.xml，于是就不会触发构造日志的过程，而新版的里面，因为让我去掉了log4j而都桥接到了logback, 于是，mybatis就会构造loggingEvent，进而构造Throwable，结果导致cpu拉高，rt拉长。

#### 问题的修复

找到直接问题后，马上改了下dubbo，把桥接的log都去掉，把log4j加了回来，然业务升级后，cpu没有升高，rt也没有拉长那么严重：

![](/images/middleware/rt-more-03.png)

![](/images/middleware/rt-more-04.png)

#### 关于日志

虽然问题修复，但是，日志这块的处理流程还是没有完全理顺：

1. 为什么mybatis默认的日志级别是trace？
2. 既然使用了logback后，mybatis会构造日志，那日志打到哪儿了?没有在业务日志里面找到mybatis的日志
3. 看mybatis的日志模块的介绍，会优先使用slf4j, 那为什么最后使用到了log4j而不是logback？

后面有时间需要继续看一下。

#### 总结

这次问题的排查持续了一周多，算是印象深刻的一次排查过程。主要的收获有：

1. 转换看问题的方式：如果我一直以定位问题来排查，估计还真不好找，因为如果老是想着定位问题，就会侧重于对比，而perf工具显示的一些耗时操作跟dubbo都没有什么直接关系，很容易被漏掉
2. 一些工具的学习：linux perf和perf-map-agent, 继greys，tcpdump后，又一定位问题的利器。

