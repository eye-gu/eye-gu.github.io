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


docker run -d --name zookeeper \
		-v /Users/guzemin/docker/zookeeper/data:/data \
		-p 2181:2181 \
		-e ZOO_4LW_COMMANDS_WHITELIST="*" \
    docker.m.daocloud.io/zookeeper:3.9.3


docker run -d --name redis \
    -v /Users/guzemin/docker/redis/data:/data \
    -p 6379:6379 \
    -p 8001:8001 \
    docker.m.daocloud.io/redis/redis-stack:7.4.0-v8


docker run -d --name postgresql \
    -e POSTGRES_PASSWORD=123456 \
    -e TZ=Asia/Shanghai \
    -v /Users/guzemin/docker/postgres/data:/var/lib/postgresql/data \
    -p 5432:5432 \
    pgvector/pgvector:0.8.2-pg17-trixie


docker run -d --name neo4j \
		-e NEO4J_AUTH=neo4j/12345678 \
    -e NEO4J_apoc_export_file_enabled=true \
    -e NEO4J_apoc_import_file_enabled=true \
    -e NEO4J_apoc_import_file_use__neo4j__config=true \
    -e NEO4JLABS_PLUGINS='["apoc"]' \
    -v /Users/guzemin/docker/neo4j/data:/data \
    -p 7474:7474 \
    -p 7687:7687 \
    docker.m.daocloud.io/neo4j:2026.03.1


docker run -d \
  --name=filebeat \
  --volume="/home/sorcara/merlin/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/home/sorcara/logs/reptileProduct/error.log:/usr/share/filebeat/data/reptileProduct/error.log:ro" \
  docker.elastic.co/beats/filebeat:8.15.0
```