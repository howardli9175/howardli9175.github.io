---
layout: post
title: "hive中的锁和事务"
author: howard li
tags: [hive, lock, transaction]
---

这个帖子的起因，是生产环境遇到了一个问题，一个任务报错，要读取hive表某个分区对应的hdfs文件不存在。
```
org.Apache.hadoop.mapred.InvalidInputException: Input path does not exist .... 
```
看到了这个问题，我的思考是：
- hive表是外表，但是auto.purge=true。生产环境中只清理文件但保留分区元信息，可能性不大。
- 比较可能原因是，在报错任务执行过程中，有其他任务执行了drop分区操作。

但是问题来了，hive表读取操作和drop partition操作应该会有锁机制来保护，drop分区操作应该会在读操作完成后才会执行。
## 那么为什么锁机制没有生效呢？

带着这些思考，我查阅了一些资料：
- [hive-1293 Concurrency Model for Hive][hive-1293]
- [hive-5317 Implement insert, update, and delete in Hive with full ACID support][hive-5317]
- [hive-5843 Transaction manager for Hive][hive-5843]
- [hive-22158 HMS Translation layer - Disallow non-ACID MANAGED tables.][hive-22158]
- [Apache Hive : Hive Transactions][hive-acid]
- [Apache Hive : Locking][hive-locking]
- [Apache Hive 3 tables][hive3-tables]
- [Hive Insert-Only Transactional Tables][insert-only]

锁和事务机制的演化为：
- hive-0.7.0之前，hive中的并发只能保证，不会读取到新旧版本混合的数据。如果读写操作同时发生，会产生问题。
- hive-0.7.0开始，hive中加入了锁机制，可以通过锁机制让写操作和读操作串行执行。
- hive-0.14.0开始，hive支持事务，支持update/delete操作。

文档还是不能完全解答我的疑问，最后还是阅读和调试源码比较直观。我们使用的hive版本为CDP版本3.1.3，更接近于开源的4.0.0版本。

在创建表时，有这样几个配置项：
- hive.create.as.external.legacy - 为true时，创建的表是外表。
- hive.create.as.insert.only - 为true时，创建的表为内表，transactional_insert_only表。
- hive.create.as.acid - 为true时，创建的表为内表，transactional表。

3个配置项决定了默认情况下创建表的类型，优先级从高到低，第一个配置为false，第二配置项为true才有用。三个配置项都为false时，创建表的为外表。

还有几个配置项，是关于事务和锁的：
- hive.support.concurrency - 为true时，才会在执行语句之前尝试加锁。
- hive.txn.manager - 事务管理器。决定哪些情况下需要加锁。
- hive.txn.ext.locking.enabled - 外表是否加锁。只对DbTxnManager生效。
- hive.lock.manager - 锁管理器。决定加锁和解锁等的具体实现。

hive.txn.manager有两种配置：
- DummyTxnManager - 可以认为是0.14.0之前的行为，所有表的读写都加锁。
- DbTxnManager - 遍历SQL语句的输入和输出，并根据一些配置参数来决定是否加锁。hive.txn.ext.locking.enabled=false，表示不对外表的读写加锁。

回到帖子开头描述的问题：
- 生产环境大部分表都是默认的建表语句，即create table...并且不指定事务属性，3个建表相关的配置项都为false，所以大部分表都是外表。
- HiveServer2中配置，hive.txn.manager=DbTxnManager / hive.txn.ext.locking.enabled=false，因此大部分表的默认行为就是，读写都不加锁。
- 这意味着，在某个表被读取的过程中，同时有清理表分区的任务，就可能会产生并发冲突。

## 应该如何设置呢？
我们大部分的表不需要update/delete等操作，所以我们可以选择以下之一，就可以让默认方式创建的表的读写操作受锁机制保护：
| config        | Opt1  | Opt2  | Opt3  |
| ------------- |:-------------:| -----:| -----:|
| hive.create.as.external.legacy   | false           | false        | false        |
| hive.create.as.insert.only       | false           | false        | true         |
| hive.create.as.acid              | false           | false        | -            |
| hive.txn.manager                 | DummyTxnManager | DbTxnManager | DbTxnManager |
| hive.txn.ext.locking.enabled     | -               | true         | -            |


## 3种选择有什么区别呢？
Opt3中，意味着表为insert_only_transactional类型，意味着insert执行中，会先把数据写入到临时目录，事务提交时才会把文件移动到临时文件中。我印象中，hive中所有insert都应该是这个机制。什么场景下使用这种表，还有什么样的特性，后续再研究。

Opt1和Opt2，两种选择都默认为外表，区别在于是DummyTxnManager还是DbTxnManager来加锁。目前的思路是，对大部分批量任务（只有insert overwrite和drop操作），只需要锁，并不需要事务，本着最简化原则，使用DummyTxnManager为默认值比较好。

[hive-1293]:https://issues.apache.org/jira/browse/HIVE-1293
[hive-5317]:https://issues.apache.org/jira/browse/HIVE-5317
[hive-5843]:https://issues.apache.org/jira/browse/HIVE-5843
[hive-22158]:https://issues.apache.org/jira/browse/HIVE-22158
[hive-acid]: https://hive.apache.org/docs/latest/hive-transactions_40509723/
[hive-locking]:https://hive.apache.org/docs/latest/locking_27362050/
[hive3-tables]:https://docs.cloudera.com/cdw-runtime/1.5.4/using-hiveql/topics/hive_hive_3_tables.html
[insert-only]:https://stackoverflow.com/questions/58718650/hive-insert-only-transactional-tables

