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

- [Apache Hive : Hive Transactions][hive-acid]

[hive-acid]: https://hive.apache.org/docs/latest/hive-transactions_40509723/
- [Apache Hive : Locking][hive-locking]

[hive-locking]: https://hive.apache.org/docs/latest/locking_27362050/
- [Apache Hive 3 tables][hive3-tables]

[hive3-tables]: https://docs.cloudera.com/cdw-runtime/1.5.4/using-hiveql/topics/hive_hive_3_tables.html


hive-0.7.0之前，hive中的并发只能保证，不会读取到新旧版本混合的数据。

如果读写操作同时发生，会产生问题。hive-0.7.0开始，hive中加入了锁机制，可以通过锁机制让写操作和读操作串行执行。
- hive.support.concurrency=false
- hive.lock.manager=org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager

hive-0.14.0开始，hive支持事务，支持update/delete操作。

