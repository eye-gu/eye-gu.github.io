---
title: Postgres
subtitle:
date: 2025-09-09T09:57:08+08:00
slug: a3181f4
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
tags:
  -
categories:
  -
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
podman run -d --name postgresql \
    -e POSTGRESQL_PASSWORD=123456 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/postgres/data:/bitnami/postgresql:U \
    -p 5432:5432 \
    bitnami/postgresql:17.5.0
```

