---
title: "ElasticSearch CRUD 接口"
date: 2021-11-06 16:17:25
categories:
  - ElasticSearch
tags:
  - Database
  - ElasticSearch
series:	
---

ElasticSearch 使用 HTTP 协议的 Restful 接口，来对接不同的程序系统。

<!--more-->

## 查询

### Get

读取一条文档。

```json
GET index_name/_doc/id
```

### Mget

批量读取文档。

**在请求体中指定 index：**

```json
GET _mget
{
    "docs" : [
        {
            "_index" : "test",
            "_id" : "1"
        },
        {
            "_index" : "test",
            "_id" : "2"
        }
    ]
}
```

**URI 中指定 index：**

```json
GET index_name/_mget
{
    "docs" : [
        {
            "_id" : "1"
        },
        {
            "_id" : "2"
        }
    ]
}
```

### Msearch

批量搜索文档

```json
POST index_name/_msearch
{}
{"query" : {"match_all" : {}},"size":1}
{"index" : "kibana_sample_data_flights"}
{"query" : {"match_all" : {}},"size":2}
```

## 创建

### Create

创建一条文档，如果指定的 id 存在，则报错

```json
POST index_name/_create
PUT index_name/_create/id
PUT index_name/_doc/id?op_type=create
{
	"user" : "Mike",
    "post_date" : "2019-04-15T14:12:12",
    "message" : "trying out Kibana"
}
```

- 不指定 `id` 时，系统会自动生成 id
- 如果指定 `id`，则在 URI 中显式指定，如果指定的 id 存在，则报错

### Index

创建一条文档，如果指定的 id 存在，旧文档会被删除，插入新文档，文档版本信息 +1

```json
PUT index_name/_doc/id
```

## 更新

### Update

不会删除原文档，真正的数据更新

```json
POST index_name/_update/id
{
    "doc":{
       "post_date" : "2019-05-15T14:12:12",
       "message" : "trying out Elasticsearch"
    }
}
```

- Update 的内容必须放在 `doc` 字段中

## 删除

### Delete

删除一条文档

```json
DELETE index_name/_doc/id
```

## 批量操作

### Bulk

一次请求执行多条语句

```json
POST _bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "create" : { "_index" : "test2", "_id" : "3" } }
{ "field1" : "value3" }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

## 工具

### Analyze

分词接口

```json
GET _analyze
{
  "analyzer": "standard",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}
GET index_name/_analyze  // 根据某 index 某字段的分词方式分词
{
  "field": "title",
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}
GET _analyze // 自定义分词
{
  "tokenizer": "standard",
  "filter": ["lowercase"],
  "text": "2 running Quick brown-foxes leap over lazy dogs in the summer evening."
}
```

