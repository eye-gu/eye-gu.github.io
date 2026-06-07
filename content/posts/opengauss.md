---
title: Opengauss
subtitle:
date: 2025-09-09T09:54:40+08:00
slug: 2534795
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
podman run -d \
--name opengauss \
--privileged=true \
-e GS_PASSWORD=Opengauss@123 \
-p 2881:5432 \
-v /Users/guzemin/docker/opengauss/data:/var/lib/opengauss:U \
-e GAUSSLOG=/var/lib/opengauss/log \
enmotech/opengauss-lite:5.0.3
```



使用opengauss/opengauss:7.0.0-RC2.B015或者opengauss/opengauss-server:7.0.0-RC2.B015的时候报错: Failed to parse cgroup config file. 需要修改cgroup配置. 改成opengauss-lite可以, 但是没有新版本的.



## navicat连接

| 选项       | 值       |
| ---------- | -------- |
| 初始数据库 | postgres |
| 用户名     | gaussdb  |



## jdbc连接

```yaml
spring:
  datasource:
    url: jdbc:opengauss://localhost:2881/shenyu
    username: gaussdb
    password: Opengauss@123
    driver-class-name: org.opengauss.Driver
```

