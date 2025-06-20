---
layout: post
title: "spark日志机制"
author: howard li
tags: [spark,yarn, 日志]
---

### 一个spark任务，在yarn上运行，driver和executor的日志存放在哪里？

首先，spark-core.jar中，org/apache/spark/log4j2.properties文件内容如下：
```
rootLogger.level = info
rootLogger.appenderRef.stdout.ref = console

appender.console.type = Console
appender.console.name = console
appender.console.target = SYSTEM_ERR
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n%ex
```
日志级别info，日志输出到标准错误流中。

接着，spark安装包解压后，conf目录下，会有一个模板文件，log4j2.properties.template，里面的内容跟spark-core.jar中一样。

再来看yarn相关的日志配置。
```
# 这个参数决定了，NM启动的container的日志根目录。
yarn.nodemanager.log-dirs=/mnt/sda1/container-logs
```
现在，我们来看一下spark_on_yarn任务运行时每个container的进程的参数。
```
java -server -Xmx1024m
...
org.apache.spark.executor.YarnCoarseGrainedExecutionBackend
--cores 1
--app-id application_174042693165_457469
...
1>/mnt/sda1/container-logs/application_174042693165_457469/container_e262_174042693165_457469_01_000069/stdout
2>/mnt/sda1/container-logs/application_174042693165_457469/container_e262_174042693165_457469_01_000069/stderr
```
当任务运行中，NM会在yarn.nodemanager.log-dirs指定的目录下，创建app-id子目录，再创建container-id子目录，里面的stdout/stderr，分别对应程序的标准输出流和错误流。结合开始时提到的log4j2配置，spark的log4j日志会到NM本机的stderr文件。

当任务完成后，日志文件还能查看吗？我们来看yarn的日志聚合配置：
```
yarn.log-aggregation.enabled=true
yarn.log-aggregation.remote-app-log-dir=/tmp/logs
yarn.log-aggregation.file-format=IFILE
yarn.log-aggregation.remote-app-log-dir-suffix=logs
yarn.log-aggregation.reatin-seconds=7 days
```
直接通过hdfs上聚合后的日志文件来理解上述配置：
```
/tmp/logs/foo/logs-ifile/application_174042693165_457469
                                                    |- dn-0001.example.com_8041/
                                                    |- dn-0002.example.com_8041/
                                                    |- ....
```
remote-app-log-dir指定的目录下，先是用户子目录(foo)，接着是文件格式子目录(logs-ifile)，接着是app-id子目录，最后是每个NM对应一个文件，其中包含NM上所有container的日志。

我们尝试一下cat命令看一下聚合后的日志文件，发现是乱码。用yarn logs命令查看聚合后的日志。
```
# 查看app-id下涉及哪些节点
yarn logs -applicationId application_174042693165_457469 -list_nodes

# 查看app-id涉及的所有节点的所有container 
yarn logs -applicationId application_174042693165_457469 -show_application_log_info

# 查看app-id涉及的所有节点的所有container涉及的所有日志文件
yarn logs -applicationId application_174042693165_457469 -show_container_log_info

# 展示app-id的某个container下所有的日志内容
yarn logs -applicationId application_174042693165_457469 -log_files_pattern ALL -containerId container_e262_174042693165_457469_01_000069
```

默认情况下，spark任务的日志运行时存放在NM本地，任务完成后，归集到hdfs，并保留一期限。这个机制满足绝大多数场景。

### yarn-client模式下，driver不受yarn管理，那么在任务完成后，driver日志就找不到了？
在spark3.0.0之前，只能靠用户自己保存driver日志。从spark3.0.0开始，增加了如下参数：
```
spark.driver.log.persistToDfs.enabled=true
spark.driver.log.dfsDir=/user/spark3/driverLogs
```
client模式下，spark.driver.log.persistToDfs.enabled=true并且spark.driver.log.dfsDir有值，会在client节点本地创建一个日志文件，同时为driver进程的RootLogger增加一个FileAppender，指向该日志文件。当任务开始后，client本地日志文件就会同步到spark.driver.log.dfsDir指向的远程目录。

### spark-streaming任务
实时任务产生的日志是一直增加的，并且当日志级别设置不当或者打印日志过多时，放在NM本地的日志文件会增长的很快，理论上总有一天会占满节点的磁盘。

所以，我们希望实时任务对应的日志输出为滚动日志文件。首先，定义新的log4j2.properties：
```
rootLogger.level = info
rootLogger.appenderRef.stdout.ref = file_appender

appender.file_appender.type = RollingFile
appender.file_appender.name = file_appender
appender.file_appender.fileName = ${sys:spark.yarn.app.container.log.dir}/spark.log
appender.file_appender.layout.type = PatternLayout
appender.file_appender.layout.pattern = %d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n%ex

# 触发策略：按天 & 按文件大小
appender.file_appender.policies.type = Policies
appender.file_appender.policies.time.type = TimeBasedTriggeringPolicy
appender.file_appender.policies.time.interval = 1
appender.file_appender.policies.time.modulate = true
appender.file_appender.policies.size.type = SizeBasedTriggeringPolicy
appender.file_appender.policies.size.size = 10MB

# 滚动策略：最多保留10个文件
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.max = 10
```
接着，我们在任务提交时使用新的log4j2.properties
```
--files /local/path/to/log4j2.properties
--conf "spark.driver.extraJavaOptions=-Dlog4j.configurationFile=file:log4j2.properties"
--conf "spark.executor.extraJavaOptions=-Dlog4j.configurationFile=file:log4j2.properties"
```
需要注意的，要使用cluster模式提交实时任务。client模式开启了driver日志持久化的情况下，driver日志会同时输出到标准错误流和日志文件，重要的是，我们无法控制日志文件的滚动模式，会造成日志文件不断增长。 

