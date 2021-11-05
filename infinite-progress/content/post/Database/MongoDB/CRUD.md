---
title: "MongoDB CRUD"
description: 
date: 2021-08-05
categories:
  - MongoDB
tags:
  - Database
  - MongoDB
series:	
---

## Query

### 查询所有数据

```javascript
db.inventory.find( {} )
```

### 条件查询

```javascript
db.inventory.find( { status: "D" } )
```

`IN` 查询：

```javascript
db.inventory.find( { status: { $in: [ "A", "D" ] } } )
```

`AND` 查询：

```javascript
db.inventory.find( {
  status: "A", 
  qty: { $lt: 30 }
} )
```

`OR` 查询：

```javascript
db.inventory.find( { 
	$or: [ 
		{ status: "A" }, 
		{ qty: { $lt: 30 } } 
	] 
} )
```

`AND` 和 `IN` 混用：

```javascript
db.inventory.find( {
     status: "A",
     $or: [ 
	     { qty: { $lt: 30 } }, 
	     { item: /^p/ } 
     ]
} )
```

### 嵌套数据查询

查询数据中嵌套的内容：

```javascript
db.inventory.find( { 
  size: { 
    h: 14, 
    w: 21, 
    uom: "cm" 
  } 
} )
```

**注：条件字段顺序必须完全与数据相同，否则匹配不到。**

也可以使用 `dot` 方式指定嵌套字段：

```javascript
db.inventory.find( { "size.h": { $lt: 15 } } )
db.inventory.find( { "size.h": { $lt: 15 }, "size.uom": "in", status: "D" } )
```

### 数组查询

数组完全匹配，包括顺序：

```javascript
db.inventory.find( { tags: ["red", "blank"] } )
```

查询所有数组内包含此条件的数据：

```javascript
db.inventory.find( { tags: "red" } )
db.inventory.find( { tags: { $all: ["red", "blank"] } } )
db.inventory.find( { dim_cm: { $gt: 25 } } )
```

多条件查询：

```javascript
db.inventory.find( { dim_cm: { $gt: 15, $lt: 20 } } )
```

根据数组中特定索引的值查询：

```javascript
db.inventory.find( { "dim_cm.1": { $gt: 25 } } )
```

根据数组长度查询：

```javascript
db.inventory.find( { "tags": { $size: 3 } } )
```

### 限制查询返回字段

仅返回指定的字段：

```javascript
db.inventory.find(
   { status: "A" },
   { item: 1, status: 1, "size.uom": 1 }
)
```

除了指定的字段，其他字段都返回：

```javascript
db.inventory.find( { status: "A" }, { status: 0, instock: 0 } )
```

不返回 `_id`:

```javascript
db.inventory.find( { status: "A" }, { item: 1, status: 1, _id: 0 } )
```

除了 `_id` 字段，其他字段不能进行组合。

### null 值处理

查询所有无此字段或字段值为 `null` 的数据：

```javascript
db.inventory.find( { item: null } )
```

仅查询字段存在且值为 `null` 的数据：

```javascript
db.inventory.find( { item : { $type: 10 } } )
```

仅查询字段不存在的数据：

```javascript
db.inventory.find( { item : { $exists: false } } )
```

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

