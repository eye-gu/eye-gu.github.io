---
title: Topn
subtitle:
date: 2026-03-16T21:50:55+08:00
slug: 418c478
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



## 查询测试

```shell
# standalone模式
go test ./test/integration/standalone/query/etcd... -v --ginkgo.focus="TopN Tests"
go test ./test/integration/standalone/query_ondisk/etcd/... -v --ginkgo.focus="TopN Tests"
go test ./test/integration/standalone/multi_segments/etcd... -v --ginkgo.focus="TopN Tests"
go test ./test/integration/standalone/cold_query/etcd... -v --ginkgo.focus="TopN Tests"

# distributed模式
go test ./test/integration/distributed/query/etcd... -v --ginkgo.focus="TopN Tests"
go test ./test/integration/distributed/multi_segments/etcd... -v --ginkgo.focus="TopN Tests"
```



