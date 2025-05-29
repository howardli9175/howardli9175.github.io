---
layout: post
title: "spark能否同时访问多个hive?"
author: howard li
tags: [spark, hive]
---

准确的说，问题是spark能否同时访问多个hive metastore？

先说结论，原生的spark是不支持的，或者说至少在3.5.6版本是不支持的。

但是从3.0.0开始，Spark增加了CatalogPlugin接口，支持通过CatalogManager管理多个CatalogPlugin，让spark同时访问多个hive有了理论上可能性。

## 1、Catalog起什么作用？

一个重要作用就是被Analyzer使用，把UnresolvedRelation变成ResolvedRelation，例如，当Analyzer看到一个my_db.my_table名字时，它需要清楚这个名字对应什么数据，是否存在。
![behind_spark_sql](/images/behind_spark_sql.png)

spark中，Catalog的实现一直在演进。

## 2、spark2.0中的Catalog
从2.0.0开始，SparkSession是所有API的统一入口，如果想在spark中使用hive，需要在创建实例时调用enableHiveSupport。

下面是一个SparkSession对象持有的属性及子属性，缩进代表了对象之间的持有关系。

- SparkSession
  - (源)[Hive]SharedState - a class that holds all state shared across sessions. session间共享的状态
    - (源)[Hive]ExternalCatalog - Interface for the system catalog (of tables, databases, etc). 库表清单
      - (源)HiveClient[Impl] - An externally visible interface to the Hive client. 对外暴露Hive
        - (源)Hive - Hive包中的类，用于与HiveMetaStore交互
  - (源)[Hive]SessionState - a class that holds all session-specific state in a given SparkSession. 某个session独有的状态。
    - HiveClient[Impl]
    - (源)[Hive]SessionCatalog - An internal catalog that is used by a Spark Session. SparkSession使用的Catalog
      - [Hive]ExternalCatalog - 来自HiveSharedState
      - HiveClient[Impl]
    - (源)Analyzer
      - [Hive]SessionCatalog - 来自HiveSessionState

解释一下上面的层级结构：

- (源)[Hive]SharedState表示，SharedState是接口或者基类，HiveSharedState是实现类，[源]是指首次创建的地方。
- 在创建SparkSession的时候，调用了enableHiveSupport方法后，SparkSession持有的SharedState就是HiveSharedState实例，SessoinState也是类似机制。
- 可以看到，SparkSession.SessionState.Analyzer，即是用来解析库名表名的。
- Analyzer通过持有的HiveSessionCatalog来解析库和表，解析请求继而经过HiveExternalCatalog、HiveClientImpl、Hive，最终到达HiveMetaStore.

我们可以得出，Analyzer只能有一种上述的解析路径，因此它不能指向多个HiveMetaStore。

我们可能还想问，我们创建两个SparkSession可以指向不同的HiveMetaStore吗？答案是不可以。

SparkSession是private类，获取方法有两种，Builder.getOrCreate()和sparkSession.newInstance()。Builder.getOrCreate()只有在首次创建的时候才会new，后续调用只会获取首次创建的实例。在已经存在的sparkSession实例上调用newSession方法，可以获取一个新SparkSessoin实例，但是SharedState是共用的，因此HiveExternalCatalog实例也是共用的，所以Analyzer到HiveMetaStore的解析路径也是一样的。

## 3、spark3.0中的Catalog
在spark3.0.0中，Catalog的实现有较大变化，增加了CatalogPlugin接口和CatalogManager，可以有多个CatalogPlugin实现，由CatalogManager通过一个Map来管理。对象的层级关系如下：

- SparkSession
  - (源)SharedState - A class that holds all state shared across sessions. session间共享的状态
    - (源)[Hive]ExternalCatalog - Interface for the system catalog (of tables, databases, etc). 库表清单
      - (源)HiveClient[Impl] - An externally visible interface to the Hive client. 对外暴露Hive
        - (源)Hive - Hive包中的类，用于与HiveMetaStore交互
  - (源)SessionState - a class that holds all session-specific state in a given SparkSession. 某个session独有的状态。
    - (源)[Hive]SessionCatalog - An internal catalog that is used by a Spark Session. SparkSession使用的Catalog
      - [Hive]ExternalCatalog - 来自SharedState
    - (源)Analyzer
      - (源)CatalogManager - tracks all the registered catalogs. 管理所有CatalogPlugin
        - (源)V2SessionCatalog extends CatalogPlugin - 
          - [Hive]SessionCatalog - 来自SessionState
        - [Hive]SessionCatalog - 来自SessionState
        - (源)HashMap - 所有CatalogPlugin的名字到实例的映射。
      - [Hive]SessionCatalog - 来自SessionState

跟2.0.0最大区别在于，新增CatalogPlugin接口，针对hive的场景的一个实现类是V2SessionCatalog，将[Hive]SessionCatalog封装在内部。

针对其它场景，例如数据湖产品，Hudi/Iceberg/DeltaLake，是在自己的jar包中提供了CatalogPlugin的实现类。

同时，在配置参数中也增加如下的配置项，使得spark可以加载和创建相应的CatalogPlugin实现类。
```
spark.sql.catalog.[catalog_name]=org.example.XXXCatalogPlugin
spark.sql.catalog.[catalog_name].prop1=value1
spark.sql.catalog.[catalog_name].prop2=value2
```
V2SessionCatalog虽然是一个支持Hive的CatalogPlugin实现，但是不能通过配置文件的方式来加载。

理论上是可以自己实现一个CatalogPlugin，可以通过在配置文件里指定hive相关的配置项，来达到访问多个hive的目的。

从上面的Catalog演进和现状来看，Hive在Spark中是一个比较特殊的组件，enableHiveSupport以及各种Hive相关的实现类，可以说hive和spark是深度耦合的，至少曾经是。
目前还不能把hive完全看作是一种CatalogPlugin，猜测原因可能是历史原因导致修改难度大、社区可能认为没有迫切的需求等。

## 4、结论
目前原生的spark3版本，无法做到访问多个hive。

那么，确实有这个需求怎么办？
- [Waggle Dance][waggle-dance]。提供多个hive的联邦服务。

[waggle-dance]: https://github.com/ExpediaGroup/waggle-dance
- 自己实现CatalogPlugin。目前可以找到的，[Kyuubi Spark Hive connector(KSHC)][kshc]。

[kshc]: https://kyuubi.readthedocs.io/en/v1.10.1/connector/spark/hive.html
