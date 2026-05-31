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


```shell
docker run -d --name minio \
    -e TZ=Asia/Shanghai \
    -e MINIO_ROOT_USER=minioadmin \
    -e MINIO_ROOT_PASSWORD=minioadmin \
    -v /Users/guzemin/docker/minio/data:/data \
    --publish 9000:9000 \
    --publish 9001:9001 \
    docker.m.daocloud.io/minio/minio:RELEASE.2024-12-18T13-15-44Z \
    server /data --console-address ":9001"
```

```shell
# 挂载目录的权限
chown -R 1001:1001 /data/minio
```
