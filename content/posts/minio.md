---
title: Minio
subtitle:
date: 2024-10-30T10:56:23+08:00
slug: 7ca5514
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

<!--more-->

```shell
docker run -d --name minio \
    -e TZ=Asia/Shanghai \
    -e MINIO_ROOT_USER=root \
    -e MINIO_ROOT_PASSWORD=minio123456 \
    -v /Users/guzemin/docker/minio/data:/bitnami/minio/data \
    --publish 9000:9000 \
    --publish 9001:9001 \
    --restart=on-failure:5 \
    --cpus=0.5 \
    -m 1G \
    bitnami/minio:2024.5.7
```

```shell
# 挂载目录的权限
chown -R 1001:1001 /data/minio
```
