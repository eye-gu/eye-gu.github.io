---
title: es
subtitle:
date: 2024-10-30T09:36:47+08:00
slug: 27add30
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

包含kibana

```shell
docker network create elasticsearch_network

docker run -d --name elasticsearch \
-e ELASTICSEARCH_PASSWORD=123456 \
-e TZ=Asia/Shanghai \
-v /Users/guzemin/docker/elasticsearch/data:/bitnami/elasticsearch/data \
-p 9200:9200 \
--net=elasticsearch_network \
bitnami/elasticsearch:8.13.2

docker run -d -p 5601:5601 --name kibana --net=elasticsearch_network \
-e KIBANA_ELASTICSEARCH_URL=elasticsearch \
bitnami/kibana:8.13.2
```



```JSON
{"query":{ 
"match":{"rowKey":"11158546311053316"} 
}} 

{"query":{"match_all":{}}} 
```



## 创建index

```
PUT /product4
{
  "mappings": {
      "properties":{
        "name":{
          "type":"keyword"
        },
        "age":{
          "type": "long"
        },
        "address":{
          "type":"text"
        },
        "birthday":{
           "type": "date",
           "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis" 
        }
      }
   
  },
  "settings":{
    "index":{
      "number_of_shards":1,
      "number_of_replicas":1
    }
  }
}
```



## 删除index

```shell
curl -XDELETE 'http://127.0.0.1:9200/product' --user elastic:123456
```

## es7.10

```shell
docker run -d --name elasticsearch7 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.10.1
```




https://www.cnblogs.com/nhdlb/p/16551485.html
