---
title: Stream Float Tag
subtitle:
date: 2026-03-16T21:49:42+08:00
slug: 8c763d0
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

<!--more-->



## stream float tag

https://github.com/apache/skywalking/issues/10277 这个issue一开始理解成了为stream实现float64类型的tag了,后来才搞明白是要给topn支持float64的field.但是做的也差不多了,分支是:stream-tag-float.

## 改动

### 1. proto

为tag添加float的类型,以及tagValue添加float的值类型

### 2. 数据写入

banyand/stream/write_standalone.go中为float类型的tag进行encode编码,其实也可以为measure,trace也在类似的地方进行编码支持

### 3. 查询支持

`ComparableExpr`和`LiteralExpr`增加一个float的实现

### 4. skipping indexRule

主要就是要专门实现一个将float编码为可保序的字节数组. 然后在数据写入的时候,记录min/max, 最后在Range中进行过滤

### 5. inverted indexRule

在`appendField`中添加对float的支持.原来有对float的field的支持,其他基本不用改.

### 6. BanyanDB对float64的查询

目前是通过正则做的词法分析,添加float的正则,然后实现语法对float的转换

### 7. 前端

添加float的tag选项,以及对float值的展示

### 8. 集成测试

目前的集成测试,主要就是配置schema和data文件,然后配置input/want



## 测试

### 删除命令

```shell
./bydbctl-cli indexRuleBinding delete -g stream_group -n float1-index-rule-binding

./bydbctl-cli indexRule delete -g stream_group -n value

./bydbctl-cli group delete -g stream_group
```

### 数据准备

创建group, INVERTED和SKIPPING类型的indexRule.

```yaml
# group

./bydbctl-cli group create -f - <<EOF
metadata:
  name: stream_group
catalog: CATALOG_STREAM
resource_opts:
  shard_num: 1
  segment_interval:
    unit: UNIT_DAY
    num: 1
  ttl:
    unit: UNIT_DAY
    num: 7
EOF

# rule

./bydbctl-cli indexRule create -f - <<EOF
metadata:
  name: value_inverted
  group: stream_group
tags:
  - value
type: TYPE_INVERTED
EOF

./bydbctl-cli indexRule create -f - <<EOF
metadata:
  name: value_skipping
  group: stream_group
tags:
  - value
type: TYPE_SKIPPING
EOF

```



创建有float64类型的tag的stream,没有任何indexRule


```yaml
./bydbctl-cli stream create -f - <<EOF
metadata:
  name: float1
  group: stream_group
tagFamilies:
  - name: searchable
    tags:
      - name: stream_id
        type: TAG_TYPE_STRING
      - name: trace_id
        type: TAG_TYPE_STRING
      - name: value
        type: TAG_TYPE_FLOAT
entity:
  tagNames:
    - stream_id
EOF


# run insert float1
```



创建有float64类型的tag的stream,并且其和INVERTED类型的indexRule进行绑定

```yaml
./bydbctl-cli stream create -f - <<EOF
metadata:
  name: float2
  group: stream_group
tagFamilies:
  - name: searchable
    tags:
      - name: stream_id
        type: TAG_TYPE_STRING
      - name: trace_id
        type: TAG_TYPE_STRING
      - name: value
        type: TAG_TYPE_FLOAT
entity:
  tagNames:
    - stream_id
EOF


./bydbctl-cli indexRuleBinding create -f - <<EOF
metadata:
  name: float2-index-rule-binding
  group: stream_group
rules:
  - value_inverted
subject:
    catalog: CATALOG_STREAM
    name: float2
begin_at: "2021-04-15T01:30:15.01Z"
expire_at: "2121-04-15T01:30:15.01Z"
EOF


# run insert float2
```



创建有float64类型的tag的stream,并且其和SKIPPING类型的indexRule进行绑定


```yaml
./bydbctl-cli stream create -f - <<EOF
metadata:
  name: float3
  group: stream_group
tagFamilies:
  - name: searchable
    tags:
      - name: stream_id
        type: TAG_TYPE_STRING
      - name: trace_id
        type: TAG_TYPE_STRING
      - name: value
        type: TAG_TYPE_FLOAT
entity:
  tagNames:
    - stream_id
EOF


./bydbctl-cli indexRuleBinding create -f - <<EOF
metadata:
  name: float3-index-rule-binding
  group: stream_group
rules:
  - value_skipping
subject:
    catalog: CATALOG_STREAM
    name: float3
begin_at: "2021-04-15T01:30:15.01Z"
expire_at: "2121-04-15T01:30:15.01Z"
EOF


# run insert float3
```



创建有float64类型的tag的stream,并且该字段还是entity tag


```yaml
./bydbctl-cli stream create -f - <<EOF
metadata:
  name: float4
  group: stream_group
tagFamilies:
  - name: searchable
    tags:
      - name: stream_id
        type: TAG_TYPE_STRING
      - name: trace_id
        type: TAG_TYPE_STRING
      - name: value
        type: TAG_TYPE_FLOAT
entity:
  tagNames:
    - stream_id
    - value
EOF


# run insert float4

```

