---
layout: post
title: "[spark异常]执行计划太长"
author: howard li
tags: [spark, 异常, 计划太长]
---

一个pyspark任务，接近2000行，使用spark版本是2.4.4，执行过程遇到这样的异常。
```
java.lang.StringIndexOutOfBoundsException: String index out of range: -47
	at java.lang.String.substring(String.java:1967)
	at org.apache.spark.sql.catalyst.util.StringUtils$StringConcat.append(StringUtils.scala:123)
	at org.apache.spark.sql.execution.QueryExecution.$anonfun$toString$1(QueryExecution.scala:207)
	at org.apache.spark.sql.execution.QueryExecution.$anonfun$toString$1$adapted(QueryExecution.scala:207)
	at org.apache.spark.sql.catalyst.trees.TreeNode.$anonfun$generateTreeString$1(TreeNode.scala:663)
	at org.apache.spark.sql.catalyst.trees.TreeNode.$anonfun$generateTreeString$1$adapted(TreeNode.scala:662)
	at scala.collection.immutable.List.foreach(List.scala:392)
```

找了一下资料，发现是已知问题。
- [spark-31916 StringConcat can overflow `length`, leads to StringIndexOutOfBoundsException][spark-31916]
- [spark-32157 Integer overflow when constructing large query plan string][spark-32157]

结论，这个问题影响的版本是3.0.0及以下，从3.0.1开始已修复。

再记录一下源代码的bug是如何产生，因为自己在初始分析过程中，没看到出为何会产生bug。
```
  class StringConcat(val maxLength: Int = ByteArrayMethods.MAX_ROUNDED_ARRAY_LENGTH) {
    protected val strings = new ArrayBuffer[String]
    protected var length: Int = 0

    def atLimit: Boolean = length >= maxLength

    def append(s: String): Unit = {
      if (s != null) {
        val sLen = s.length
        if (!atLimit) {
          val available = maxLength - length
          val stringToAppend = if (available >= sLen) s else s.substring(0, available)
          strings.append(stringToAppend)
        }
        length += sLen
      }
    }
 }
```
问题的根源在于，available为负值。available所在的分支要求length小于maxLength，那么maxLength-length不应该是正数吗？不可能为负数啊。

后来明白了原因，maxLength-length也有可能溢出。例如maxLength是Int.MaxValue，length是-1，maxLength-length就会溢出，available就成为负数。

### 到这里还没有结束

既然3.0.1版本已经修复上述问题，我决定在高版本上测试一下上述的任务。发现很快driver内存不够，频繁GC，虽然没有再出现上述的异常，但是一些dataframe的转换操作执行耗时很长。所以问题的根源在于，执行计划太大了，就像在5层楼上加一层很容易，但是在100层楼上加一层就会困难的多。

分析了这个pyspark脚本，处理逻辑应该是一点点增加而形成的，并没有进行整体逻辑的梳理和整合，一个复杂dataframe被多次引用，join或者union操作形成新的dataframe，新的dataframe又会被多次引用，如此进行下去，执行计划迅速膨胀，耗尽内存。

我觉得，对全局逻辑进行梳理和整合并重构代码，是正确的处理方式，无论从执行效率角度，还是从代码维护的角度。但是这个对开发者要求较高。

那个spark本身是如何应对超大的执行计划呢？

找到一篇很好的文章，[When the Spark Execution Plan Gets Too Big][big-plan]，可以仔细阅读一下。简单来说，有几种方式可以截断执行计划，包括checkpoint机制和临时写到磁盘再读回。很多人可能认为cache方法也能达到类似效果，在这篇文章中作者验证cache不会截断执行计划。



[big-plan]:https://engineering.statefarm.com/when-the-spark-execution-plan-gets-too-big-eb658872d603
[spark-31916]:https://issues.apache.org/jira/browse/SPARK-31916
[spark-32157]:https://issues.apache.org/jira/browse/SPARK-32157


