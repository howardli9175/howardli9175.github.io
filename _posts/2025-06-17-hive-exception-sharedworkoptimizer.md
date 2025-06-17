---
layout: post
title: "[hive异常]SharedWorkOptimizer"
author: howard li
tags: [hive, 异常, SharedWorkOptimizer]
---

Hive on Tez，通过beeline提交一个查询，抛出异常UnsupportedOperationException。在hive server2上有完整日志：
```
FAILED: UnsupportedOperationException null
java.lang.UnsupportedOperationException
        at java.util.AbstractList.add(AbstractList.java:148)
        at java.util.AbstractList.add(AbstractList.java:108)
        at org.apache.hadoop.hive.ql.optimizer.SharedWorkOptimizer.pushFilterToTopOfTableScan(SharedWorkOptimizer.java:1794)
        at org.apache.hadoop.hive.ql.optimizer.SharedWorkOptimizer.sharedWorkOptimization(SharedWorkOptimizer.java:386)
        at org.apache.hadoop.hive.ql.optimizer.SharedWorkOptimizer.transform(SharedWorkOptimizer.java:163)
        ...
```
开源版本的hive，SharedWorkOptimizer是从3.0.0版本开始增加的，pushFilterToTopOfTableScan方法中并没有使用add方法。

我使用的是CDP版本hive3.1.3，通过反编译，发现SharedWorkOptimizer实现方式与开源版本不同。

通过远程debug发现，问题根源是向一个不可变的List添加元素造成的。这应该是CDP的实现不够健壮造成的。解决方式，只能是找Cloudera升级，或者直接关闭这个优化器。
```
set hive.optimize.shared.work=false;
```
