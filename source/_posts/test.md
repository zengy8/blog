---
title: mongo kafka hadoop配置记录
date: 2021-05-17 16:12:08
tags:
  - 大数据
  - mongodb
  - kafka
  - hadoop
categories:
  - 大数据
  - mongodb
  - kafka
  - hadoop
comments: true
---
### 客户需求
1. kafka滚动删除数据时间为3天
2. kafka数据目录扩展到3个（cache1，cache2，cache3）
3. yarn任务运行日志开启滚动删除，保留时间为7天。在hdfs上该日志的副本数设置为1个
4. datanode数据目录扩展到3个（cache1，cache2，cache3）
5. yarn的内存最大可调度96G

### 变更记录
#### mongo
mongodb默认启动占用总内存50%，yarn要求最大调度内存为96G，需先修改配置 /etc/mongod.conf 为24G
```
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  mmapv1:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 24
https://blog.csdn.net/qq_32523587/article/details/82219170
```

#### kafka /etc/kafka/server.properties
1. 修改日志保留时间
```
log.retention.hours=192 
 -> 
log.retention.hours=72
```
2. 新增数据存储目录
```
mkdir /cache{2..3}/kafka
log.dirs=/cache1/kafka 
 -> 
log.dirs=/cache1/kafka,/cache2/kafka,/cache3/kafka
```

#### yarn /etc/hadoop/conf/yarn-site.xml
#### hdfs /etc/hadoop/conf/hdfs-site.xml
3. 修改日志保留天数，hadoop-mapreduce-historyserver要重启
```
<name>yarn.log-aggregation.retain-seconds</name>
<value>1728000</value> 
 ->
<value>604800</value>
 ------
# spark新增配置
/usr/local/src/spark-2.1.1-bin-hadoop2.6/conf/spark-defaults.conf
spark.history.fs.cleaner.enabled true
spark.history.fs.cleaner.interval 1d
spark.history.fs.cleaner.maxAge 7d

https://blog.csdn.net/yu0_zhang0/article/details/80396080
```

4. 新增数据存储目录
```
mkdir -p /cache{2..3}/hdfs/yarn/local
mkdir -p /cache{2..3}/hdfs/dfs/data
chown -R hdfs:hadoop /cache{2..3}/hdfs/
chown -R yarn:hadoop /cache{2..3}/hdfs/yarn/
<name>yarn.nodemanager.local-dirs</name>
<value>file:///cache1/hdfs/yarn/local/</value>
 ->
<value>file:///cache1/hdfs/yarn/local/,file:///cache2/hdfs/yarn/local/,file:///cache3/hdfs/yarn/local/</value>
 ------
<name>dfs.datanode.data.dir</name>
<value>/cache1/hdfs/dfs/data/</value>
 ->
<value>/cache1/hdfs/dfs/data/,/cache2/hdfs/dfs/data/,/cache3/hdfs/dfs/data/</value>
# 有新增数据盘，原有数据盘使用率较大，新增配置使数据往新的磁盘写
<property>
  <name>dfs.datanode.fsdataset.volume.choosing.policy</name>
  <value>org.apache.hadoop.hdfs.server.datanode.fsdataset.AvailableSpaceVolumeChoosingPolicy</value>
</property>
```

5. 修改yarn最大可调度内存
```
<name>yarn.nodemanager.resource.memory-mb</name>
<value>49152</value>
 ->
<value>98304</value>
<name>yarn.scheduler.maximum-allocation-mb</name>
<value>49152</value>
 ->
<value>98304</value>
```
