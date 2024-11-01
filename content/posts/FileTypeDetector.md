---
title: FileTypeDetector
subtitle:
date: 2024-11-01T17:43:49+08:00
slug: 8258677
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
categories:
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



## FileTypeDetector

```java
java.nio.file.Files#probeContentType
```

调用这个api获取content-type, webp类型的图片本地可以成功获取, 但是测试环境返回的是null

远程debug进去发现执行的代码和本地并不一样, 怀疑是环境问题:mac和linux, 或者jdk的问题

最后发现是jdk的问题, 测试环境是openjdk, 本地是azul.
