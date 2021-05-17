---
title: Spark连接mongodb异常
date: 2021-05-17 14:59:19
tags:
  - Spark
  - 大数据
categories:
  - Spark
  - 大数据
comments: true
---

## 1. 需求背景
客户要求spark和mongo整合一下，提供的截图是spark连接mongodb报错
![](http://zengy.cn-gd.ufileos.com/6a858c2d746dc1a7153c6029a0e429f7.png)  

## 2. 处理流程
查看报错信息是执行缺少相关的类，网上搜索了下找到下列网址
https://spark-packages.org/package/mongodb/mongo-spark
但是不确定该下哪个包，这时客户主动提供了
<span style='background: yellow'>mongo-spark-connector_2.11-2.1.0.jar</span>
将jar包放在/usr/local/src/spark-2.1.1-bin-hadoop2.6/jars/ ，服务重启

客户那测试还是未恢复，在机器上执行import包正常
![](http://zengy.cn-gd.ufileos.com/4cb1ad73028c66f864d0b30988d9fe5e.png)  

提供代码让我们自己spark-shell测试
```
import org.apache.spark.sql.Row
import org.apache.spark.sql.types.StructType
import org.apache.spark.sql.types.StructField
import org.apache.spark.sql.types.StringType
import org.apache.spark.sql.types.IntegerType
import com.mongodb.spark.sql.DefaultSource

val dm = spark.sparkContext.textFile("hdfs://GGRTF/dmkjbehaviordata/20190815/play/*").
      map(x => (x.split("\t")(20), 1)).
      reduceByKey(_ + _).
      sortBy(_._2, false).
      take(30).
      toList
    val dmkj = spark.sparkContext.parallelize(dm).cache()
    val row = dmkj.map(x => Row("20190815", "dmkj", x._1, x._2))
    val schema = StructType(
      List(
        StructField("date", StringType),
        StructField("scene_id", StringType),
        StructField("itemId", StringType),
        StructField("count", IntegerType)))
    spark.createDataFrame(row, schema).write.options(Map("spark.mongodb.output.uri" -> "mongodb://dn003:27017,dn004:27017,dn005:27017/res_dev.item_top_day")).
      mode("overwrite").
      format("com.mongodb.spark.sql").
      save()
    dmkj.unpersist(true)
```
执行脚本确实还报缺少某些类
![](http://zengy.cn-gd.ufileos.com/043e66c66294e8074bcbba5d6c47b4a6.png)  

搜索到以下页面，尝试添加 <span style='background: yellow'>mongo-java-driver-3.8.2.jar</span>
https://www.cnblogs.com/wwxbi/p/7170679.html

执行报错变成无法创建user
![](http://zengy.cn-gd.ufileos.com/3e59bd76c59fa39781e0fb1065abe034.png)  

查看给的代码连的是mongoc端口，改成mongos端口37017客户答复已正常
