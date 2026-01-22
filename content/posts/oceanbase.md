---
title: Oceanbase
subtitle:
date: 2025-09-17T15:40:06+08:00
slug: d02a821
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
podman network create --subnet=172.20.0.0/16 ob-net

podman run -p 2881:2881 \
  --name obstandalone \
  --net ob-net --ip 172.20.0.10 \
  -e MODE=MINI \
  -e OB_TENANT_PASSWORD=12345678 \
  -d quay.io/oceanbase/oceanbase-ce:4.3.5-lts
```

