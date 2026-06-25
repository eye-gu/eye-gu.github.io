---
title: Docker
subtitle:
date: 2026-06-13T23:09:22+08:00
slug: c41c58c
draft: false
author:
  name:
  link:
  email:
  avatar:
description:
keywords:
license:
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
docker run -d \
  --name gitea \
  -p 3000:3000 \
  -p 2222:22 \
  docker.m.daocloud.io/gitea/gitea:1.26-rootless



docker run -d \
  --name gitlab \
  -p 8929:80 \
  -p 2289:22 \
  --shm-size=256m \
  --privileged=true \
  docker.m.daocloud.io/gitlab/gitlab-ce:19.0.2-ce.0


docker run --name nacos3 \
-e MODE=standalone \
-e NACOS_AUTH_TOKEN=U2VjcmV0S2V5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIzNDU2Nzg5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIzNA== \
-e NACOS_AUTH_IDENTITY_KEY=serverIdentity \
-e NACOS_AUTH_IDENTITY_VALUE=SecretKey012345678901234567890123456789012345678901234567890123456789 \
-p 8080:8080 -p 8848:8848 -p 9848:9848 -p 9080:9080 -d docker.m.daocloud.io/nacos/nacos-server:v3.2.2

docker run --name nacos \
-e MODE=standalone -e NACOS_AUTH_ENABLE=true \
-e NACOS_AUTH_TOKEN=U2VjcmV0S2V5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIzNDU2Nzg5MDEyMzQ1Njc4OTAxMjM0NTY3ODkwMTIzNA== \
-e NACOS_AUTH_IDENTITY_KEY=serverIdentity \
-e NACOS_AUTH_IDENTITY_VALUE=SecretKey012345678901234567890123456789012345678901234567890123456789 \
-p 8848:8848 -p 9848:9848 -d docker.m.daocloud.io/nacos/nacos-server:v2.5.0
```