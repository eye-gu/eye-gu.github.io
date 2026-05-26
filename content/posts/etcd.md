
---
title: Etcd
subtitle:
date: 2026-03-09T00:33:07+08:00
slug: 99e016a
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

<!--more-->



```shell
docker run -d \
  --name etcd-server \
  -p 2379:2379 \
  -v /Users/guzemin/docker/etcd:/etcd-data \
  quay.io/coreos/etcd:v3.5.25 \
  /usr/local/bin/etcd \
  --name s1 \
  --data-dir /etcd-data \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --initial-cluster-token tkn \
  --initial-cluster-state new
```

