---
title: Neo4j
subtitle:
date: 2025-08-26T17:20:03+08:00
slug: 97b0b2e
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
docker run -d --name neo4j \
    -e NEO4J_PASSWORD=12345678 \
    -v /Users/guzemin/docker/neo4j/data:/bitnami \
    -p 7474:7474 \
    -p 7687:7687 \
    bitnami/neo4j:5.26.10
```

