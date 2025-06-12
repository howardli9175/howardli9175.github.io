---
layout: post
title: "hive中的锁和事务"
author: howard li
tags: [hive, lock, transaction]
---

这个帖子的起因，是生产环境遇到了一个问题，一个任务报错，要读取hive表某个分区对应的hdfs文件不存在。

看到了这个问题，我的思考是：

- hive表是外表，但是auto.purge=true。生产环境中只清理文件但保留分区元信息，可能性不大。
- 比较可能原因是，在报错任务执行过程中，有其他任务执行了dro分区操作。
- hive表读取操作和drop partition操作应该会有锁机制来保护，drop分区操作应该会在读操作完成后才会执行。

带着这些思考，我查阅了一些资料：
- [hive-1293 Concurrency Model for Hive][hive-1293]

[hive-1293]:https://issues.apache.org/jira/browse/HIVE-1293
- [hive-5317 Implement insert, update, and delete in Hive with full ACID support][hive-5317]

[hive-5317]:https://issues.apache.org/jira/browse/HIVE-5317
- [hive-5843 Transaction manager for Hive][hive-5843]

[hive-5843]:https://issues.apache.org/jira/browse/HIVE-5843
- [hive-22158 HMS Translation layer - Disallow non-ACID MANAGED tables.][hive-22158]

[hive-22158]:https://issues.apache.org/jira/browse/HIVE-22158

hive-0.7.0之前，hive中的并发只能保证，不会读取到新旧版本混合的数据。

如果读写操作同时发生，会产生问题。hive-0.7.0开始，hive中加入了锁机制，可以通过锁机制让写操作和读操作串行执行。

hive-0.14.0开始，hive支持事务，支持update/delete操作。

我们使用的hive版本为CDP版本3.1.3，更接近于开源的4.0.0版本。

在创建表时，有这样几个配置项：
- hive.create.as.external.legacy - 为true时，创建的表是外表。
- hive.create.as.insert.only - 为true时，创建的表为内表，transactional_insert_only表。
- hive.create.as.acid - 为true时，创建的表为内表，transactional表。

3个配置项决定了默认情况下创建表的类型，优先级从高到低，第一个配置为false，第二配置项为true才有用。三个配置项都为false时，创建表的为外表。

还有几个配置项，是关于事务和锁的：
- hive.txn.manager
- hive.txn.ext.locking.enabled
- hive.lock.manager

hive.txn.manager可以是DummyTxnManager，也可以是DbTxnManager。DummyTxnManager可以认为是0.14.0之前的行为，所有表的读写都加锁。如果是DbTxnManager，并且hive.txn.ext.locking.enabled=false，则不对外表的读写加锁。

回到帖子开头描述的问题，生产环境大部分表都是默认的建表语句，即create table...并且不指定事务属性，3个建表相关的配置项都为false，所以大部分表都是外表。并且hive.txn.manager默认值设为DbTxnManager，hive.txn.ext.locking.enabled=false，因此大部分表的默认行为就是，读写都不加锁，当某个表被读取的过程中，同时有清理表分区的任务，就会产生并发冲突。

应该如何设置呢？我们大部分的表不需要update/delete等操作，所以我们可以选择以下之一，就可以让表的读写操作不会同时发生：
- 外表。hive.txn.manager设为DummyTxnManager。
- 外表。hive.txn.manager设为DbTxnManager，但hive.txn.ext.locking.enabled=true。
- 内表，transactional_insert_only表。hive.txn.manager设为DbTxnManager。

3种选择有什么区别呢？todo

还有一些参考资料：

- [Apache Hive : Hive Transactions][hive-acid]

[hive-acid]: https://hive.apache.org/docs/latest/hive-transactions_40509723/
- [Apache Hive : Locking][hive-locking]

[hive-locking]: https://hive.apache.org/docs/latest/locking_27362050/
- [Apache Hive 3 tables][hive3-tables]

[hive3-tables]: https://docs.cloudera.com/cdw-runtime/1.5.4/using-hiveql/topics/hive_hive_3_tables.html

