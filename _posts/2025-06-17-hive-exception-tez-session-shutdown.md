---
layout: post
title: "[hive异常]TezSession has already shutdown"
author: howard li
tags: [hive, tez]
---

生产环境，某一段时间，多个租户的hive on tez任务失败，异常提示如下：
```
ERROR: Failed to execute tez graph.
org.apache.tez.dag.api.SessionNotRunning: TezSession has already shutdown. No cluster diagnostics found.
        at org.apache.tez.client.TezClient.waitTillReady(TezClient.java:979)
        at org.apache.tez.client.TezClient.waitTillReady(TezClient.java:948)
        ...
```

排查hive server2的日志，只关注某个失败任务，根据query-id找到thread-id，把thread-id的日志提取出来。发现在如上异常出现之前，很长一段时间都没有日志。这是不正常的现象，异常出现之前，正在处理的事情是提交DAG，应该很快就会完成。

又查看hive server2的配置，发现参数tez.session.am.dag.submit.timeout.secs=30，默认配置应该是300。

所以，任务失败的原因是，hive server2收到一个查询，解析语句生成dag，并提交dag，但是耗时过长，超过了30秒，导致tez session关闭。

但是，有一个问题是，tez session超时30秒的配置已经存在很长时间了，之前并没有发生类似的异常，为何突然出现30秒内无法提交dag的情况呢？一种猜测为hive server2的负载过高，排查日志中的GC相关，果然发现有很多fullGC，最长的耗时为100秒以上。再查看集群管理页面中，hive server2的监控指标，发现个别实例的内存几乎打满。

所以，问题的根源是，hive server2内存占用过高，导致不断发生fullGC，不能正常服务。

本次异常还有一些遗留问题：
- 为什么部分hive server2实例内存使用很高？可以通过jvm dump结果来分析，但是在生产环境如何做到不影响存量的任务。
- 每个hive server2的堆内存设置为40GB，这值偏高。一方面会增长GC时间，另一个方面会增加jvm dump时的难度。推荐的方式为，每个实例的内存设置小一些，通过增加实例个数来提高服务能力。

