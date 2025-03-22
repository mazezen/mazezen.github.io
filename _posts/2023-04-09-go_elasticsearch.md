---
layout: post
title: "Go操作Elasticsearch"
date:    2023-04-09
tags: [Go]
comments: true
author: mazezen
---


# Go操作Elasticsearch

## 一、elasticsearch是什么

elasticsearch是一个基于Lucene的搜索服务器，采用Java语言编写，使用Lucene构建索引、提供搜索功能，并作为Apache许可条款下的开发源码发布，是当前流行的企业级搜索引擎。其实Lucene的功能已经很强大了，为什么还要多此一举的开发elasticsearch呢？原因是因为Lucene只是一个由Java语言编写的库，对不适用Java语言的开发人员并不友好。所以elasticsearch在Lucene上做了很多改进，提供了多种语言的接口。Lucene之于elasticsearch堪比发动机之于汽车，elasticsearch底层使用的仍然是Lucene的api，Lucene专注于底层搜索的建设，elasticsearch专注于企业应用。elasticsearch的目标是让全文搜索变得简单，开发者可以通过简单明了的restful api轻松实现搜索功能，而不必去面对Lucene的复杂性。



## 二、 elasticsearch的优点

1. 分布式：Elasticsearch横向扩展非常方便灵活，当规模较小时可以使用小规模的集群，随着数据的增长，需要更大的容量和更高的性能，此时只需增加更多的节点，Elasticsearch的自动发现机制会识别新增的节点并重新平衡分配数据。
2. 全文检索：Apache Lucene是一个用Java编写的高性能的功能齐全的信息检索库，Elasticsearch在底层使用Lucene来提供强大的全文检索，提供任何开源产品的能力。自带多语言支持、强大的查询语言、地理位置支持，上下文感知的建议、自动完成和搜索片段。
3. 近实时搜索和分析：数据进入Elasticsearch后，可达到实时搜索。Elasticsearch还支持聚合分析操作
4. 高可用：高可用主要体现在容错方面，Elasticsearch集群会自动发现新的或失败的节点，重组和重新平衡数据，确保数据是安全和可访问的。
5. 模式自由：Elasticsearch的动态mapping机制可以自动的检测数据的结构和类型，创建索引，并使数据可搜索。
6. Restful Api：Elasticsearch是Api驱动，几乎任何操作都可以用一个简单的Restful Api使用JSON基于HTTP请求实现，客户端也可使用多种编程语言。
7. 应用场景丰富：站内搜索、nosql数据库（Elasticsearch在读写性能上由于MongoDB，同时也支持地理位置查询）、日志分析（日志分析平台ELK，能够对日志进行集中的收集、存储、搜索、分析、监控以及可视化）等。



## 三 、elasticsearch核心概念

来源： https://www.jianshu.com/p/cec1b8b3698d

### 集群（cluster）

代表一个集群，集群中有多个节点node，其中一个为主节点，这个主节点可以通过选举产生的，主节点是对于集群内部来说的。ES的一个概念就是去中心化，字面上理解就是无中心化节点，这是对于集群外部来说的，因为从外部来看ES集群，在逻辑上是一个整体，你与任何一个节点的通信和整个ES集群通信是等价的。一个集群就是由一个或者多个节点组织在一起，它们共同持有整个的数据，并在一起提供索引和搜索功能。一个集群由一个唯一的名字标识。一个节点只能通过指定某个集群的名字，来加入这个集群。

### 节点（node）

一个节点是集群中的一个服务器，作为集群的一部分，存储数据，参与集群的索引和搜索功能。一个节点也是由一个名字来标识的。默认情况下，这个名字是一个随机的漫威漫画角色的名字，这个名字会在启动的时候赋予节点，这个名字对于管理工作来说挺重要的，因为在这个管理过程中，要确定网络中的哪些服务器对应于Elasticsearch集群中的哪些节点。

### 索引（index）

ES将它的数据存储在一个或多个索引（index）中。类似sql中的数据库。可以向索引中写入文档或者读取文档，并通过ES内部使用Lucene将数据索引或从索引中检索数据

一个索引就是一个拥有几分相似特征的文档的集合（类似于我们在数据库中的库结构），一个索引由一个名字来标识，并且在我们要对对这个索引中的文档进行索引，搜索，更新和删除的时候，都要使用到这个名字。

### 文档（document）

文档是ES中主要的实体。对所有使用ES的案例来说，他们最终都可以终结为对文档的搜索。文档由字段构成。

### 映射（mapping）

所有文档写进索引之前都会先进行分析，如果将输入的文本分割为词条，哪些词条又会被过滤，这种行为叫做映射（mapping）。一般由用户自己定义规则

### 类型（type）

每个文档都有与之对应的类型定义。这允许用户在一个索引中存储多种文档类型，并为不同文档类型提供不同的映射

### 分片（shards）

shards代表索引分片，ES可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改。
分片有两个好处，一是可以水平扩展，另一个是可以并发提高性能。

### 副本（replicas）

代表索引副本，ES可以设置多个索引副本，副本的作用一是提高系统的容错性，实现高可用（HA），当某个节点某个分片损坏或丢失时可以从副本中恢复；二是提高ES的查询效率，ES会自动对搜索请求进行负载均衡。



## 四、elasticsearch单机版安装

````markdown
1. docker pull elasticsearch:7.7.0

2. 创建容器挂载目录
mkdir $PWD$/config
mkdir $PWD$/data
mkdir $PWD$/plugins
mkdir $PWD$/logs

3. 配置文件
vim %PWD%/elasticsearch/config/elasticsearch.yml
# 允许任意主机访问
http.host: 0.0.0.0
# es-head连接配置
http.cors.enabled: true
http.cors.allow-origin: "*"

4. 创建容器
docker run --name elasticsearch -p 9200:9200  -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
-v /Users/cc/docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /Users/cc/docker/elasticsearch/data:/usr/share/elasticsearch/data \
-v /Users/cc/docker/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /Users/cc/docker/elasticsearch/logs:/usr/share/elasticsearch/logs \
-d elasticsearch:7.7.0

5. 查看容器
docker ps
docker lgos "容器id"

6. 访问 (因为我是本机所以是127.0.0.1)
关闭防火墙
ip:9200

```json
{
  "name" : "cc39c9abff7e",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "B3wNsG1GTeSqj5bYixLqCw",
  "version" : {
    "number" : "7.7.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "81a1e9eda8e6183f5237786246f6dced26a10eaf",
    "build_date" : "2020-05-12T02:01:37.602180Z",
    "build_snapshot" : false,
    "lucene_version" : "8.5.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

7. 安装 ElasticSearch-Head
docker pull mobz/elasticsearch-head:5

8. 运行容器
docker run -di --name=elasticsearch-head -p 9100:9100 mobz/elasticsearch-head:5

9. 查看访问
docker ps
ip:9100


10. 安装IK分词器

10.1 ElasticSearch中自带的有分词器为什么还要使用IK分词器？

在ElasticSearch中的分词器会把中文分为一个一个的字，例如"今天是周五"，会被分成“今”、“天”、“是”，“周”、“五”，这里很明显是不合适的，在大多数场景下需要的是词而不是字。
所以就需要安装中文分词器IK来解决这个问题。
IK提供了两个分词算法：ik_smart和ik_max_word，其中ik_smart为最少切分，ik_max_word为最细力度。
这里需要注意安装的版本需要跟ElasticSearch版本一致。

10.2 安装

10.2.1 进入elasticsearch容器
docker exec -it 容器ID /bin/bash

10.2.2 进入plugins/ 创建文件夹
cd plugins 
mkdir ik
cd ik

10.2.3 下载
yum install wget -y
yum install zip unzip -y

wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.7.0/elasticsearch-analysis-ik-7.7.0.zip

10.2.4 解压
unzip elasticsearch-analysis-ik-7.7.0.zip
删除压缩包
rm -rf elasticsearch-analysis-ik-7.7.0.zip

10.2.5 退出 && 重启elasticsearch
exit
docker restart elasticsearchID
````



## 五、 elasticsearch 的 RESTful API 增删改查

### 增

```shell
curl --location --request POST 'http://127.0.0.1:9200/store/books/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "title": "Elasticsearch: The definitive Guide",
    "name": {
		"first": "Zachary",
		"last": "Tong"
	},
    "publish_date":"2005-02-06",
	"price": "49.99"
}'
```

### 查

```shell
curl --location --request GET 'http://127.0.0.1:9200/store/books/1'

// 指定查询字段 _source
curl --location --request GET 'http://127.0.0.1:9200/store/books/1?_source=title,name'
```

### 修改

```shell
curl --location --request PUT 'http://127.0.0.1:9200/store/books/3' \
--header 'Content-Type: application/json' \
--data-raw '{
    "title": "Elasticsearch Blueprints",
    "name": {
		"first": "Vineeth",
		"last": "Mohan"
	},
    "publish_date":"2005-06-06",
	"price": "1999.99"
}'

// 更新指定的字段 _update
curl --location --request POST 'http://127.0.0.1:9200/store/books/3/_update' \
--header 'Content-Type: application/json' \
--data-raw '{
    "doc": {
        "price": 1000
    }
}'
```



### 删除

```shell
curl --location --request DELETE 'http://127.0.0.1:9200/store/books/3'
```

### 过滤查询

_search: 查询

query: 查询条件 

bool： 组合查询

term: 不分词

match_all: 匹配全部

filter: 过滤条件

must: 条件必须满足 相当于and

should: 条件可以满足也可以不满足，相当于 or

must_not: 条件不需要满足，相当于not

```shell
curl --location --request GET 'http://127.0.0.1:9200/store/books/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
    "query": {
        "bool": {
            "must": {
                "match_all":{}
            },
            "filter": {
                "term": {
                    "price": 49.99
                }
            }
        }
    }
}'



curl --location --request GET 'http://127.0.0.1:9200/store/books/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
    "query": {
        "constant_score": {
            "filter": {
                "term": {
                    "price": 49.99
                }
            }
        }
    }
}'


curl --location --request GET 'http://127.0.0.1:9200/store/books/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
    "query": {
        "bool": {
            "filter": {
                "term": {
                    "price": 49.99
                }
            }
        }
    }
}'
```

## 六、elasticsearch实操

```shell
curl --location --request PUT 'http://127.0.0.1:9200/news2'
```



elasticsearch7.0 之后不支持type导致,需要加上 include_type_name=true 

```shell
curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/_mapping&include_type_name=true' \
--header 'Content-Type: application/json' \
--data-raw '{
    "properties": {
        "content": {
            "type": "text",
            "analyzer": "ik_max_word",
            "search_analyzer": "ik_max_word"
        }
    }
}'
```



```shell
curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/1' \
--header 'Content-Type: application/json' \
--data-raw '{
    "content": "美国留给伊拉克的是个烂摊子"
}'


curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/' \
--header 'Content-Type: application/json' \
--data-raw '{
    "content": "公安部： 各地校车将享受最高路权"
}'

curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/3' \
--header 'Content-Type: application/json' \
--data-raw '{
    "content": "中韩渔警冲突调查：韩警平均每天扣1搜中国渔船"
}'

curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/4' \
--header 'Content-Type: application/json' \
--data-raw '{
    "content": "中国驻洛杉矶领事馆遭亚裔男子抢劫，嫌犯自己自首"
}'
```

### 查找高亮显示

```shell
curl --location --request POST 'http://127.0.0.1:9200/news2/fulltext/_search' \
--header 'Content-Type: application/json' \
--data-raw '{
    "query": {"match":{"content":"中国"}},
    "highlight":{
        "pre_tags":["<font color='\''red'\''>","<tag2>"],
        "post_tags":["</font>","</tag2>"],
        "fields":{
            "content":{}
        }
    }
}'
```



## 七、go操作elasticsearch
