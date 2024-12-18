---
title: Nebula
subtitle:
date: 2024-12-18T10:51:24+08:00
slug: 0fcabe4
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



## docker 部署

按照官方教程, 首先clone nebula-docker-compose 这个仓库

```shell
git clone -b release-3.8 https://github.com/vesoft-inc/nebula-docker-compose.git
```

然后docker-compose启动

```shell
docker-compose up -d
```

这个配置文件meta, storaged, graphd都是三个进程, 一下子启动的有点多, 本地有点受不了, 看到还有一个`docker-compose-lite.yaml`文件, 里面都是只有一个, 本地测试也足够了, 于是切换成这个

```shell
docker-compose -f docker-compose-lite.yaml up -d
```

nebula有一个studia的web应用, 索性也添加到``docker-compose-lite.yaml``中, 方便查看

```yml
studia:
  image: docker.io/vesoft/nebula-graph-studio:v3.10
  environment:
    USER: root
    TZ:   "${TZ}"
  depends_on:
    - graphd
  ports:
    - 7001:7001
  networks:
    - nebula-net
  restart: on-failure
```

注意studia连接的时候ip需要填graphd, 或者host.docker.internal
