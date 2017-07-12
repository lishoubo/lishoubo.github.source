---
title: 性能分析工具TProfiler
date: 2017-07-04 14:14:07
tags: [工具,性能]
categories: 技术
---


在写一个统计qps的组件，收获了不少，在[上一篇](https://lishoubo.github.io/2017/07/04/%E9%AB%98%E5%B9%B6%E5%8F%91%E7%BB%9F%E8%AE%A1qps/)里面有讲。写完之后想压测下看看，统计下每个方法的耗时。以前用过JProfile，但是，JProfle在linux部署比较麻烦，当然，更关键的是这个是收费的；找了半天，发现阿里出了一个TProfiler，看起来简单好用，还免费，试了试确实还不错。

[TProfile的详细介绍](https://github.com/alibaba/TProfiler/wiki/TProfiler%E4%BB%8B%E7%BB%8D%E6%96%87%E6%A1%A3)

使用上还比较简单，主要有下面几步：

1. 启动的时候，添加javaagent，TProfile通过Java的instrument来修改目标类的字节码：
> java -javaagent:/Users/lishoubo/p/projects/TProfiler/dist/TProfiler_1.0.1/lib/tprofiler-1.0.1.jar -Dprofile.properties=/Users/lishoubo/p/projects/TProfiler/dist/TProfiler_1.0.1/profile.properties -jar target/demo-1.0-SNAPSHOT.jar 10

2. 启动后，可以通过下面几个命令来控制：
>查看状态：./bin/tprofiler-client 127.0.0.1 50000 status
>
>停止：./bin/tprofiler-client 127.0.0.1 50000 stop
>
>启动（停止之后可以再启动）：./bin/tprofiler-client 127.0.0.1 50000 start
>
>刷新方法(输出统计的方法列表）: ./tprofiler-client 127.0.0.1 50000 flushmethod

3. 分析日志：TProfile会生成三个文件：tsampler.log、tprofiler.log、tmethod.log。得到以上三个日志文件后就可以进行分析了。

* 分析sampler log。命令: ./bin/sampler-log-analysis  ~/logs/tsampler.log ~/logs/tmethod.log ~/logs/thread.log

* 分析profile log。命令：./bin/tprofiler-log-analysis  ~/logs/tprofiler.log ~/logs/tmethod.log ~/logs/topmethod.log ~/logs/topobject.log

具体的日志格式在[这里](https://github.com/alibaba/TProfiler/wiki/TProfiler%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90).

看TProfile的介绍，基本可以放到线上运行，后面找个应用试试。