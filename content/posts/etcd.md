
---
title: Etcd
subtitle:
date: 2026-03-09T00:33:07+08:00
slug: 99e016a
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
  --name etcd-server \
  -p 2379:2379 \
  -v /Users/guzemin/docker/etcd:/etcd-data \
  quay.io/coreos/etcd:v3.5.25 \
  /usr/local/bin/etcd \
  --name s1 \
  --data-dir /etcd-data \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --initial-cluster-token tkn \
  --initial-cluster-state new
```

```yaml
services:
  etcd1:
    image: quay.io/coreos/etcd:v3.5.25
    container_name: etcd1
    command: >
      etcd
      --name etcd1
      --data-dir /etcd-data
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://127.0.0.1:2379
      --listen-peer-urls http://0.0.0.0:2380
      --initial-advertise-peer-urls http://etcd1:2380
      --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      --initial-cluster-token tkn
      --initial-cluster-state new
      --election-timeout 1000
      --heartbeat-interval 200
    ports:
      - "2379:2379"

  etcd2:
    image: quay.io/coreos/etcd:v3.5.25
    container_name: etcd2
    command: >
      etcd
      --name etcd2
      --data-dir /etcd-data
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://127.0.0.1:2380
      --listen-peer-urls http://0.0.0.0:2380
      --initial-advertise-peer-urls http://etcd2:2380
      --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      --initial-cluster-token tkn
      --initial-cluster-state new
      --election-timeout 1000
      --heartbeat-interval 200
    ports:
      - "2380:2379"

  etcd3:
    image: quay.io/coreos/etcd:v3.5.25
    container_name: etcd3
    command: >
      etcd
      --name etcd3
      --data-dir /etcd-data
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://127.0.0.1:2381
      --listen-peer-urls http://0.0.0.0:2380
      --initial-advertise-peer-urls http://etcd3:2380
      --initial-cluster etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      --initial-cluster-token tkn
      --initial-cluster-state new
      --election-timeout 1000
      --heartbeat-interval 200
    ports:
      - "2381:2379"
```