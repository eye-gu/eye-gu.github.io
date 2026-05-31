---
title: Zookeeper
subtitle:
date: 2025-09-09T09:58:10+08:00
slug: 55b9ea9
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
docker run -d --name zookeeper \
		-v /Users/guzemin/docker/zookeeper/data:/data \
		-p 2181:2181 \
		-e ZOO_4LW_COMMANDS_WHITELIST="*" \
    docker.m.daocloud.io/zookeeper:3.9.3
```

