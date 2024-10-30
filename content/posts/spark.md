---
title: spark
subtitle:
date: 2024-10-30T10:01:04+08:00
slug: a86246a
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: false
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

## 命令

```shell
./bin/spark-sql

./bin/spark-shell
```



## 优化

```properties
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.initialExecutors=1
spark.dynamicAllocation.minExecutors=1
spark.dynamicAllocation.maxExecutors=8
spark.dynamicAllocation.executorAllocationRatio=0.5
spark.dynamicAllocation.executorIdleTimeout=60s
spark.dynamicAllocation.cachedExecutorIdleTimeout=30min
spark.dynamicAllocation.shuffleTracking.timeout=30min
spark.dynamicAllocation.shuffleTracking.enabled=true
spark.dynamicAllocation.schedulerBacklogTimeout=1s
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout=1s
spark.cleaner.periodicGC.interval=5min

spark.sql.adaptive.enabled=true
spark.sql.adaptive.forceApply=false
spark.sql.adaptive.logLevel=info
spark.sql.adaptive.advisoryPartitionSizeInBytes=256m
spark.sql.adaptive.coalescePartitions.enabled=true
spark.sql.adaptive.coalescePartitions.minPartitionNum=1
spark.sql.adaptive.coalescePartitions.initialPartitionNum=256
spark.sql.adaptive.fetchShuffleBlocksInBatch=true
spark.sql.adaptive.localShuffleReader.enabled=true
spark.sql.adaptive.skewJoin.enabled=true
spark.sql.adaptive.skewJoin.skewedPartitionFactor=5
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes=400m
spark.sql.adaptive.nonEmptyPartitionRatioForBroadcastJoin=0.2
spark.sql.adaptive.optimizer.excludedRules
spark.sql.autoBroadcastJoinThreshold=-1
```





## sql

```sql
show catalogs;

show schemas;
```

## mysql

驱动已放入jars

```shell
./bin/spark-sql \
--conf spark.sql.catalog.mysql=org.apache.spark.sql.execution.datasources.v2.jdbc.JDBCTableCatalog \
--conf spark.sql.catalog.mysql.driver=com.mysql.cj.jdbc.Driver \
--conf spark.sql.catalog.mysql.url=jdbc:mysql://192.168.10.17:3306 \
--conf spark.sql.catalog.mysql.user=sorcara \
--conf spark.sql.catalog.mysql.password=sorcara
```

## es

[doc](https://www.elastic.co/guide/en/elasticsearch/hadoop/current/spark.html)

包：`elasticsearch-spark-30_2.12-8.13.2.jar`

```scala
def main(args: Array[String]): Unit = {
val conf = new SparkConf()
.setMaster("local[*]")
.set("es.nodes", "192.168.10.18")
.set("es.port", "9200")
val sparkSession = SparkSession.builder().config(conf).getOrCreate()
val sqlContext = sparkSession.sqlContext

// sql
sqlContext.sql("CREATE TEMPORARY TABLE product " +
"USING org.elasticsearch.spark.sql " +
"OPTIONS (resource 'product', " +
"scroll_size '20'," +
"es.read.field.as.array.include 'images,priceList,productBasicProperties,productOtherProperties,productLightCustomizationList,skuList,skuList.attributeInstructList'" +
")")
val df = sqlContext.sql("select * from product limit 1")
df.take(10).foreach((t: Row) => {
println(t.json)
})

// 创建df
// val product = sparkSession.sparkContext.esRDD("product")
// product.take(10).foreach(println)

// 创建ds
// val ds = sql.read
// .format("org.elasticsearch.spark.sql")
// .option("scroll_size", 20)
// .option("es.read.field.as.array.include", "images,priceList,productBasicProperties,productOtherProperties,productLightCustomizationList,skuList")
// .load("product")
}
```

```shell
./bin/spark-sql \
--conf spark.es.nodes=192.168.10.18 \
--conf spark.es.port=9200
```

```sql
CREATE TEMPORARY VIEW product
USING org.elasticsearch.spark.sql
OPTIONS (resource 'product',
scroll_size '20',
es.read.field.as.array.include 'images,priceList,productBasicProperties,productOtherProperties,productLightCustomizationList,skuList,skuList.attributeInstructList'
);

select * from product limit 10;
```

```shell
./sbin/start-thriftserver.sh \
--conf spark.es.nodes=192.168.10.18 \
--conf spark.es.port=9200

./bin/beeline -u jdbc:hive2://127.0.0.1:10000 -n guzemin

:q
```
