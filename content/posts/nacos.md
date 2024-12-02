---
title: Nacos
subtitle:
date: 2024-11-27T14:34:04+08:00
slug: c29d3a8
draft: true
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
  - draft
categories:
  - draft
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
docker run --name nacos \
-e MODE=standalone -e NACOS_AUTH_ENABLE=true \
-e NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789 \
-e NACOS_AUTH_IDENTITY_KEY=SecretKey012345678901234567890123456789012345678901234567890123456789 \
-e NACOS_AUTH_IDENTITY_VALUE=SecretKey012345678901234567890123456789012345678901234567890123456789 \
-p 8848:8848 -p 9848:9848 -d nacos/nacos-server:v2.3.2-slim
```

