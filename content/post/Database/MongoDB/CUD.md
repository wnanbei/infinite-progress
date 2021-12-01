---
title: "MongoDB 增删改操作"
date: 2021-11-05 00:00:00
categories:
  - MongoDB
tags:
  - Database
  - MongoDB
series:	
---

MongoDB 的 Insert、Update、Delete 操作。

MongoDB 官方中文文档：[MongoDB中文手册|官方文档中文版](https://docs.mongoing.com/)，基于 4.2 版本。

<!--more-->

## Insert

**特性：**

- MongoDB 所有对单条数据的写操作都是原子操作；
- 不指定 `_id` 会自动生成；
- 插入数据会返回对应 `_id`

### 单条插入

```javascript
db.inventory.insertOne(
   { item: "canvas", qty: 100, tags: ["cotton"], size: { h: 28, w: 35.5, uom: "cm" } }
)
```

### 批量插入

```javascript
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
   { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
   { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
])
```

## Update

**特性：**

- MongoDB 所有对单条数据的写操作都是原子操作；
- 一条数据插入之后，`_id` 字段将不能再更改和替换；
- 对于写操作，mongo 会保留字段的顺序，除非以下情况：
  1. `_id` 字段始终排在第一位。
  2. 字段重命名可能会导致文档字段重新排序。

### 单条更新

```javascript
db.inventory.updateOne(
   { item: "paper" },
   {
     $set: { "size.uom": "cm", status: "P" },
     $currentDate: { lastModified: true }
   }
)
```

### 批量更新

```javascript
db.inventory.updateMany(
   { "qty": { $lt: 50 } },
   {
     $set: { "size.uom": "in", status: "P" },
     $currentDate: { lastModified: true }
   }
)
```

### 替换数据

完全替换此条数据。

```javascript
db.inventory.replaceOne(
   { item: "paper" },
   { item: "paper", instock: [ { warehouse: "A", qty: 60 }, { warehouse: "B", qty: 40 } ] }
)
```

## Delete

**特性：**

- MongoDB 所有对单条数据的写操作都是原子操作；
- 就算删除了全部数据，也不会删除索引。

### 单条删除

单条删除会删除匹配到的第一条数据：

```javascript
db.inventory.deleteOne( { status: "D" } )
```

### 批量删除

```javascript
db.inventory.deleteMany({})
db.inventory.deleteMany({ status : "A" })
```