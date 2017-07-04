---
title: 性能分析工具TProfiler
date: 2017-07-03 14:14:07
tags: [工具]
categories: 性能
---


java -javaagent:/Users/lishoubo/p/projects/TProfiler/dist/TProfiler_1.0.1/lib/tprofiler-1.0.1.jar -Dprofile.properties=/Users/lishoubo/p/projects/TProfiler/dist/TProfiler_1.0.1/profile.properties -jar target/demo-1.0-SNAPSHOT.jar 10

./tprofiler-client 127.0.0.1 50000 status

./tprofiler-client 127.0.0.1 50000 stop

./tprofiler-client 127.0.0.1 50000 flushmethod


运行采样后会生成三个文件：tsampler.log、tprofiler.log、tmethod.log。endProfTime时间到了后才会输出tmethod.log。

得到以上三个日志文件后就可以进行分析了。

分析sampler log命令: java -cp tprofiler.jar com.taobao.profile.analysis.SamplerLogAnalysis d:/tsampler.log d:/method.log d:/thread.log,会生成method.log和thread.log

分析profiler log命令: java -cp tprofiler.jar com.taobao.profile.analysis.ProfilerLogAnalysis d:/tprofiler.log d:/tmethod.log d:/topmethod.log d:/topobject.log,会生成topmethod.log和topobject.log