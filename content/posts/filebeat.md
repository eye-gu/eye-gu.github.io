---
title: Filebeat
subtitle:
date: 2024-10-30T14:08:43+08:00
slug: 6798df1
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



```yaml
filebeat.inputs:
- type: log
  id: error
  enabled: true
  paths:
    - /usr/share/filebeat/data/reptileProduct/error.log
  # multiline.pattern: '^[[:space:]]+[(at|\.{3})][[:space:]]+\b|^[[:space:]]+Caused by:'
  multiline.pattern: '^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d+'
  multiline.negate: true
  multiline.match: after

  processors:
    - dissect:
        tokenizer: "%{logDate} %{logTime} [%{thread}] %{logLevel} %{logger} - %{message}"



# ============================== Filebeat modules ==============================

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false


# output.console:
#   enabled: true
#   pretty: true


output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["192.168.10.232:9092"]

  # message topic selection + partitioning
  topic: error_log
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
```


```shell
docker run -d \
  --name=filebeat \
  --volume="/home/sorcara/merlin/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/home/sorcara/logs/reptileProduct/error.log:/usr/share/filebeat/data/reptileProduct/error.log:ro" \
  docker.elastic.co/beats/filebeat:8.15.0
```
