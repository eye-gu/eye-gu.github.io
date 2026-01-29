---
title: Oracle
subtitle:
date: 2025-09-09T09:56:40+08:00
slug: 049699f
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
podman run -d \
  --name oracle-db \
  -p 1521:1521 \
  -e ORACLE_PWD=123456 \
  -v /Users/guzemin/docker/oracle/data:/opt/oracle/oradata:U \
  container-registry.oracle.com/database/free:23.9.0.0-lite-arm64
```



## navicat连接

| 选项     | 值       |
| -------- | -------- |
| 连接类型 | 基本     |
| 服务名称 | FREEPDB1 |
| 用户名   | SYSTEM   |
| 角色     | 默认     |

## jdbc连接

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@localhost:1521/FREEPDB1
    username: SYSTEM
    password: 123456
    driver-class-name: oracle.jdbc.OracleDriver
```

