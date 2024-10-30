---
title: mongo
subtitle:
date: 2024-10-30T10:00:28+08:00
slug: 83d5f02
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
docker run -d --name mongo \
-e MONGO_INITDB_ROOT_USERNAME=root \
-e MONGO_INITDB_ROOT_PASSWORD=123456 \
-v /Users/guzemin/docker/mongo/data:/data/db \
-p 27017:27017 \
mongo:5.0.0
```



```json
db.getCollection("logistics_sub_quotation").find({_id: {'$eq': NumberLong("18508450327146497")}})
```

