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
weight: 0
categories:
  - bigdata
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
--conf spark.sql.catalog.mysql.url=jdbc:mysql://127.0.0.1:3306 \
--conf spark.sql.catalog.mysql.user=root \
--conf spark.sql.catalog.mysql.password=12345678
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

## iceberg

```shell
spark-sql \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
  --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.local.type=hadoop \
  --conf spark.sql.catalog.local.warehouse=/Users/guzemin/spark/iceberg \
  --packages org.apache.iceberg:iceberg-spark-runtime-4.1_2.13:1.11.0


spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-4.1_2.13:1.11.0\
    --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
    --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog \
    --conf spark.sql.catalog.spark_catalog.type=hive \
    --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
    --conf spark.sql.catalog.local.type=hadoop \
    --conf spark.sql.catalog.local.warehouse=/Users/guzemin/spark/iceberg \
    --conf spark.sql.defaultCatalog=local
```

## paimon

```shell
spark-sql \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.warehouse=file:/Users/guzemin/spark/paimon \
    --conf spark.sql.extensions=org.apache.paimon.spark.extensions.PaimonSparkSessionExtensions


spark-sql \
    --conf spark.sql.catalog.paimon=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon.warehouse=hdfs:///user/paimon/warehouse \
    --conf spark.sql.extensions=org.apache.paimon.spark.extensions.PaimonSparkSessionExtensions
```


```shell
spark-sql \
    --conf spark.sql.extensions=org.apache.paimon.spark.extensions.PaimonSparkSessionExtensions \
    --conf spark.sql.catalog.paimon_rest=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon_rest.metastore=rest \
    --conf spark.sql.catalog.paimon_rest.uri=http://localhost:56744/ \
    --conf spark.sql.catalog.paimon_rest.token.provider=bear \
    --conf spark.sql.catalog.paimon_rest.token=init_token \
    --conf spark.sql.catalog.paimon_rest.warehouse=file:/Users/guzemin/spark/paimon-warehouse

spark-sql \
    --conf spark.sql.extensions=org.apache.paimon.spark.extensions.PaimonSparkSessionExtensions \
    --conf spark.sql.catalog.paimon_jdbc=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon_jdbc.metastore=jdbc \
    --conf spark.sql.catalog.paimon_jdbc.uri=jdbc:mysql://127.0.0.1:3306/paimon \
    --conf spark.sql.catalog.paimon_jdbc.jdbc.user=root \
    --conf spark.sql.catalog.paimon_jdbc.jdbc.password=12345678 \
    --conf spark.sql.catalog.paimon_jdbc.catalog-key=jdbc \
    --conf spark.sql.catalog.paimon_jdbc.warehouse=file:/Users/guzemin/spark/paimon-warehouse

spark-sql \
    --conf spark.sql.extensions=org.apache.paimon.spark.extensions.PaimonSparkSessionExtensions \
    --conf spark.sql.catalog.paimon_hive=org.apache.paimon.spark.SparkCatalog \
    --conf spark.sql.catalog.paimon_hive.metastore=hive \
    --conf spark.sql.catalog.paimon_hive.uri=thrift://localhost:9083 \
    --conf spark.sql.catalog.paimon_hive.warehouse=/user/paimon/warehouse \
    --conf spark.sql.catalog.paimon_hive.hive-conf-dir=$HIVE_HOME/conf
```


```sql
use paimon_rest;
create database if not exists spark;
use spark;

CREATE TABLE app1_logs
TBLPROPERTIES (
  'type' = 'object-table',
  'path' = 'file:/Users/guzemin/spark/paimon'           -- 可选：不指定则用默认 warehouse 路径
);

CREATE TABLE app2_logs
TBLPROPERTIES (
  'type' = 'object-table'         -- 可选：不指定则用默认 warehouse 路径
);
```