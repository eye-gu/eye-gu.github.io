---
title: Opensearch
subtitle:
date: 2024-10-30T10:55:34+08:00
slug: f401d76
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



## docker


```shell
docker run -p 9200:9200 -p 9600:9600 \
-v /Users/guzemin/docker/opensearch/data:/usr/share/opensearch/data \
-v /Users/guzemin/docker/opensearch/config:/usr/share/opensearch/config \
-e OPENSEARCH_INITIAL_ADMIN_PASSWORD=Sorcara2024! \
-e "discovery.type=single-node" \
--name opensearch -d \
opensearchproject/opensearch:2.15.0



docker run -p 5601:5601 \
-v /Users/guzemin/docker/opensearch/opensearch_dashboards.yml:/usr/share/opensearch-dashboards/config/opensearch_dashboards.yml \
--name opensearch-dashboard -d \
opensearchproject/opensearch-dashboards:2.15.0
```

### opensearch.yml

plugins.security.ssl.http.enabled: false


### opensearch_dashboards.yml
opensearch.hosts: [http://host.docker.internal:9200]



## 兼容模式

version:7.10.2

```shell
curl -XPUT "http://192.168.10.18:9201/_cluster/settings" -u 'admin:admin' -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "compatibility.override_main_response_version": true
  }
}'
```





## 创建index

```shell
curl -X PUT 'http://192.168.10.18:9201/product_attribute_index?pretty' -u 'admin:admin' -H 'content-Type:application/json' -d '
{
    "mappings": {
        "properties": {
            "attr_name": {
                "type": "keyword"
            },
            "attr_values": {
                "type": "text",
                "fields": {
                    "keyword": {
                        "type": "keyword",
                        "ignore_above": 256
                    }
                }
            },
            "category_level1_name": {
                "type": "keyword"
            },
            "category_level2_name": {
                "type": "keyword"
            },
            "category_level3_name": {
                "type": "keyword"
            },
            "category_name": {
                "type": "keyword"
            },
            "cleaned_attr_values": {
                "type": "text",
                "fields": {
                    "keyword": {
                        "type": "keyword",
                        "ignore_above": 256
                    }
                }
            },
            "is_customizable": {
                "type": "long"
            },
            "properties_type": {
                "type": "keyword"
            },
            "tfidf_score": {
                "type": "float"
            }
        }
    }
}'
```

## count

```shell
curl 'http://192.168.10.18:9201/product_emb_v1/_count' -u 'admin:admin'
```
