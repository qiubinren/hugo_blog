---
title: "Linux内核bug导致java进程僵死"
date: 2017-11-15T23:19:56+08:00
draft: false
tags: ["Java", "Linux", "Kernel"]
categories: ["Java", "Linux", "Kernel"]
---

# Linux内核bug导致java进程僵死

## 现象描述

云化服务开通进程与现网并行了有一段时间了，比对结果正确性基本和生产现网进程已经一致，但是遇到一个诡异的问题，进程运行一段时间就会不定期僵死，期间主机`CPU`资源占用率极高，资源监控告警疯狂告警`CPU`紧张。怀疑是`GC`问题，`jmap`和`jstack`都连不上`jvm`，只有加上-F参数才可以输出一些结果，观察输出以及走查代码都没有发现明显问题，十分诡异。

<!--more-->

## 排查过程

一开始以为是业务问题，走查了整个`java`业务侧的代码，没有发现有明显逻辑或者内存泄露问题，然后开始怀疑使用`swig`调用的`c++`动态库导致的问题，结果折腾很久仍然一无所获。接着有一次，启动分布式框架之后，没有启动`job`调度执行任何业务，结果两个小时后，进程又开始僵死，怀疑是云化框架内存泄露导致的`GC`问题，想用`jmap`和`jstack`分析内存和堆栈，结果发现`jmap`和`jstack`都无法连上`jvm`，必须加上`-F`参数才可以，观察输出也没看出个所以然。

当天晚上凌晨进程又一次僵死，第二天早上`jstack`和`jmap`一遍操作后，对着输出接着看，越看越费解，之后查看`GC`日志，发现一次`GC`一直从凌晨12点`GC`到早上第二天8点51，这段时间进程都处于僵死状态。

> **2017-11-11T00:01:51.210+0800:** 22324.612: [GC2017-11-11T00:01:51.211+0800: 22324.612: [ParNew: 1677844K->18K(1887488K), 204595.4025500 secs] 1739133K->61310K(3984640K), 204595.4026900 secs] [Times: **user=1523599.25 sys=18631.98, real=204564.30 secs**]
> **2017-11-13T08:51:46.613+0800:** 226920.015: **Total time for which application threads were stopped: 204595.4126760 seconds**, Stopping threads took: 0.0000440 seconds
> 2017-11-13T08:51:46.613+0800: 226920.015: Application time: 0.0000690 seconds
> 2017-11-13T08:51:46.624+0800: 226920.026: Total time for which application threads were stopped: 0.0107240 seconds, Stopping threads took: 0.0000440 seconds
> 2017-11-13T08:51:46.624+0800: 226920.026: Application time: 0.0000600 seconds

比较巧的是，测试环境的`activemq`也开始出现类似现象，至此开始怀疑是系统问题。

之后的并行过程又经历了两次这个问题，发现如果不对进程进行处理，它会无限期僵死下去，每次恢复的时间点，居然是我们输入`jstack -F`之后。

发现这个关键信息之后，在网上搜索后，发现了两个帖子遇到相同问题

[https://stackoverflow.com/questions/35165455/suspended-jvm-jstack-f-pid-only-fix](https://stackoverflow.com/questions/35165455/suspended-jvm-jstack-f-pid-only-fix)

[https://www.zhihu.com/question/39452240](https://www.zhihu.com/question/39452240)

比对内核版本发现一模一样，都是`2.6.32-504.el6.x86_64`。至此真相大白，反馈给领导，安排统一升级系统，升级至`redhat 6.8`，问题解决。

整个排查过程让人十分怀疑人生，一遍一遍的找代码里可能存在的问题，又一遍又一遍否认，结果最后却是系统内核问题，在这边记录下这次印象深刻的`debug`。。。
