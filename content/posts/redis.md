---
title: Redis
subtitle:
date: 2025-09-09T09:57:42+08:00
slug: 59ba13b
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
podman run -d --name redis \
		-e ALLOW_EMPTY_PASSWORD=yes \
    -v /Users/guzemin/docker/redis:/bitnami/redis/data:U \
    -p 6379:6379 \
		bitnami/redis:8.0.3
```

