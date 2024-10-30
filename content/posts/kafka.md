---
title: kafka
subtitle:
date: 2024-10-30T10:47:41+08:00
slug: 3bbcf35
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



## docker部署

貌似有点问题，还没有解决

```shell
docker run -d --name kafka \
    -e TZ=Asia/Shanghai \
    -e KAFKA_CFG_NODE_ID=0 \
    -e KAFKA_CFG_PROCESS_ROLES=controller,broker \
    -e KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
    -e KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
    -e KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@localhost:9093 \
    -e KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER \
    -v /Users/guzemin/docker/kafka/data:/bitnami/kafka \
    -p 9092:9092 \
    bitnami/kafka:3.7
```

```shell
docker run -d  \
  --name kafka \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.0.232:9092 \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1 \
  -e KAFKA_TRANSACTION_STATE_LOG_MIN_ISR=1 \
  -e KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=0 \
  -e KAFKA_NUM_PARTITIONS=1 \
  -p 9092:9092 \
  apache/kafka:3.8.0
```

## 本地部署

```shell
## zk
bin/kafka-server-start.sh config/server.properties

## kraft Ol32dbtpRTKtUJX2KKpRHQ
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
bin/kafka-storage.sh format -t Ol32dbtpRTKtUJX2KKpRHQ -c config/kraft/server.properties

bin/kafka-server-start.sh config/kraft/server.properties


## efak
JMX_PORT=9999 ./bin/kafka-server-start.sh ./config/server.properties
./bin/ke.sh start
./bin/ke.sh stop
```

```shell
./kafka-configs.sh --alter --bootstrap-server 192.168.0.232:9092 --entity-type topics  --entity-name error_log --add-config retention.ms=86400000

./kafka-configs.sh --describe --bootstrap-server 192.168.0.232:9092 --entity-type topics  --entity-name error_log
```
