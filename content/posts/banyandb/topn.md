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



## 查询测试

```shell
# standalone模式
go test -a ./test/integration/standalone/query... -v --ginkgo.focus="TopN Tests"
go test -a ./test/integration/standalone/multi_segments... -v --ginkgo.focus="TopN Tests"

# distributed模式
go test -a ./test/integration/distributed/query... -v --ginkgo.focus="TopN Tests"
go test -a ./test/integration/distributed/multi_segments... -v --ginkgo.focus="TopN Tests"
```



## 数据分布

```go
shard_id = hash(sharding_key) % shard_num
node = (shard_id + replica_id) % node_count
```

