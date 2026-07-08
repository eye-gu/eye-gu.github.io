---
title: Hadoop
subtitle:
date: 2026-07-08T11:38:01+08:00
slug: 6f86d3b
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

ssh

```shell
cat ~/.ssh/id_ed25519.pub > ~/.ssh/authorized_keys
```

`hadoop-env.sh`

```shell
export JAVA_HOME=/Users/guzemin/.sdkman/candidates/java/21.0.11-zulu
```

`core-site.xml`

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
  </property>
  <!-- 数据落盘根目录：默认是 /tmp/hadoop-<用户>，会被系统清理导致丢数据，必须改到稳定目录 -->
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/Users/guzemin/hadoop/hadoop-3.4.3/data</value>
  </property>
  <!-- 把 guzemin 换成你的用户名（whoami） -->
  <property>
    <name>hadoop.proxyuser.guzemin.groups</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.guzemin.hosts</name>
    <value>*</value>
  </property>
</configuration>
```

`hdfs-site.xml`

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
```

`mapred-site.xml`

```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>yarn.app.mapreduce.am.env</name>
    <value>JAVA_HOME=/Users/guzemin/.sdkman/candidates/java/21.0.11-zulu</value>  <!-- 换成你实际的 JAVA_HOME -->
</property>
</configuration>
```

`yarn-site.xml`

```xml
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>4096</value>
  </property>
  <property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>1024</value>
  </property>
  <property>
    <name>yarn.nodemanager.vmem-pmem-ratio</name>
    <value>2.1</value>
  </property>
</configuration>
```

`.zprofile`

```shell
export HADOOP_HOME=/Users/guzemin/hadoop/hadoop-3.4.3
export HADOOP_USER_NAME=guzemin
export PATH=$HADOOP_HOME/bin:$PATH
```

## 命令

```shell
hdfs namenode -format # 首次必做，只做一次
```

```shell
./sbin/start-all.sh
./sbin/stop-all.sh
```

```shell
# start-all.sh 不会启动 JobHistory Server，需单独启动
mapred --daemon start historyserver
mapred --daemon stop historyserver
```

```shell
hdfs dfs -mkdir -p /tmp

hdfs dfs -ls /
```

## Web UI

| 端口  | 服务                          | 地址                      |
| ----- | ----------------------------- | ------------------------- |
| 9870  | HDFS NameNode                 | http://localhost:9870     |
| 9868  | Secondary NameNode            | http://localhost:9868     |
| 9864  | HDFS DataNode                 | http://localhost:9864     |
| 8088  | YARN ResourceManager          | http://localhost:8088     |
| 8042  | YARN NodeManager              | http://localhost:8042     |
| 19888 | MapReduce JobHistory Server   | http://localhost:19888    |

> 注：以上为 Hadoop 3.x 端口，2.x 不同（NameNode 为 50070、Secondary NameNode 为 50090、DataNode 为 50075）。
> JobHistory Server 需单独启动，`start-all.sh` 不包含它。

