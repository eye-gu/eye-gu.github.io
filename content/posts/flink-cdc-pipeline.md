---
title: Flink Cdc Pipeline
subtitle:
date: 2024-10-30T10:52:58+08:00
slug: 46fc631
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



```yaml
source:
  type: mysql
  hostname: 192.168.0.17
  port: 3306
  username: sorcara
  password: sorcara
  tables: product.product, product.priority_reptile_category, product.aiqicha_company, product.qichacha_company, product.reptile_\.*, product.company_category_rating_top_match, product.category, product.category_1688
  # tables: product.product
  server-id: 5415
  server-time-zone: UTC
  scan.incremental.snapshot.chunk.size: 4096
  scan.snapshot.fetch.size: 10240
  scan.newly-added-table.enabled: true

sink:
  type: starrocks
  name: StarRocks Sink
  jdbc-url: jdbc:mysql://192.168.10.211:9030
  load-url: 192.168.10.211:8085
  username: root
  password: ""
  table.create.properties.replication_num: 1
  table.create.properties.fast_schema_evolution: true
  table.schema-change.timeout: 30min

pipeline:
  name: Sync MySQL Database to StarRocks
  parallelism: 2
```

```shell
./bin/flink-cdc.sh ./conf/mysql-to-starrocks.yaml --flink-home /home/sorcara/merlin/flink-1.19.1 -s /home/sorcara/merlin/checkpoint/eea9cca12ef1c45b847952e936fe6ed0/chk-37464
```
