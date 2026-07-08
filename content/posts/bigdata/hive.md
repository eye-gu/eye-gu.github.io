---
title: Hive
subtitle:
date: 2026-07-08T12:24:10+08:00
slug: e04e7e5
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
tags:
  -
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

## 配置

### mysql

驱动: https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar

放入hive的lib目录

```sql
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8mb4;
```

### TEZ

```shell
curl -L -O https://archive.apache.org/dist/tez/0.10.5/apache-tez-0.10.5-bin.tar.gz
tar -xzvf apache-tez-0.10.5-bin.tar.gz

# 下载 
wget https://repo1.maven.org/maven2/commons-collections/commons-collections/3.2.2/commons-collections-3.2.2.jar
cp commons-collections-3.2.2.jar /Users/guzemin/tez/apache-tez-0.10.5-bin/lib/
cp commons-collections-3.2.2.jar /Users/guzemin/hadoop/hadoop-3.4.3/share/hadoop/common/lib/


# 重新打成 tar：Tez 要求 tar 包内部直接是文件，不能多一层目录
cd apache-tez-0.10.5-bin
tar zcvf ../tez.tar.gz *

# 上传到hdfs
hdfs dfs -mkdir -p /apps/tez
hdfs dfs -put tez.tar.gz /apps/tez/
hdfs dfs -ls /apps/tez
```

`tez-site.xml`

```xml
<?xml version="1.0"?>

<configuration>
  <property>
    <name>tez.lib.uris</name>
    <value>hdfs://localhost:9000/apps/tez/tez.tar.gz</value>
  </property>
  <property>
    <name>tez.use.cluster.hadoop-libs</name>
    <value>true</value>
  </property>
</configuration>
```

```shell
cp /Users/guzemin/tez/apache-tez-0.10.5-bin/conf/tez-site.xml $HADOOP_HOME/etc/hadoop
cp /Users/guzemin/tez/apache-tez-0.10.5-bin/conf/tez-site.xml $HIVE_HOME/conf/
```



`hadoop-env.sh`

```shell
export TEZ_CONF_DIR=/Users/guzemin/tez/apache-tez-0.10.5-bin/conf
export TEZ_JARS=/Users/guzemin/tez/apache-tez-0.10.5-bin
export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*:${HADOOP_CLASSPATH}
```



### hive

`.zprofile`

```shell
export HIVE_HOME=/Users/guzemin/hive/apache-hive-4.2.0-bin
export PATH=$HIVE_HOME/bin:$PATH
```

`hive-site.xml`

```xml
<configuration>
  <!-- 执行引擎：Tez -->
  <property>
    <name>hive.execution.engine</name>
    <value>tez</value>
  </property>
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>

  <!-- 元数据库：MySQL（嵌入式 metastore，跑在 HS2 进程内，无 9083） -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/metastore?createDatabaseIfNotExist=true&amp;useSSL=false&amp;characterEncoding=UTF-8</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.cj.jdbc.Driver</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>12345678</value>
  </property>

  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://localhost:9083</value>
  </property>
</configuration>
```



## 命令

```shell
schematool -dbType mysql -initSchema --verbose #第一次执行
```

```shell
hive --service metastore

hiveserver2 # Web UI：http://localhost:10002/

beeline -u 'jdbc:hive2://localhost:10000/' -n $USER
```

