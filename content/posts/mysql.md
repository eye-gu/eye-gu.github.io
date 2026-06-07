---
title: Mysql
subtitle:
date: 2024-10-30T14:06:52+08:00
slug: 64667e3
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



## docker

```shell
podman run -d --name mysql \
    -e MYSQL_ROOT_PASSWORD=12345678 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/mysql/data:/bitnami/mysql/data:U \
    -v /Users/guzemin/docker/mysql/my.cnf:/opt/bitnami/mysql/conf/my.cnf \
    -p 3306:3306 \
    bitnami/mysql:8.2.0
```

### vector

> MySQL 社区版仅支持 VECTOR 类型存储，DISTANCE 函数和向量索引仅限 HeatWave/MySQL AI。
> MariaDB 11.7+ 原生支持向量索引（HNSW）和距离函数。

```shell
docker run -d --name mariadb \
    -e MARIADB_ROOT_PASSWORD=12345678 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/mariadb/data:/var/lib/mysql:Z \
    -p 3306:3306 \
    docker.m.daocloud.io/mariadb:11.8
```

## 导入导出

```shell
mysqldump -h 127.0.0.1 -u root -p123456 product reptile_product > /bitnami/mysql/data/reptile_product.sql

mysql -h localhost -u sorcara -psorcara -D product
source /home/sorcara/data/reptile_product.sql
```
