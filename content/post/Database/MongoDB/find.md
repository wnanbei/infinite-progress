---
title: "MongoDB find 查询"
date: 2021-11-05 00:00:00
categories:
  - MongoDB
tags:
  - Database
  - MongoDB
series:	
---

MongoDB find 数据查询命令。

MongoDB 官方中文文档：[MongoDB中文手册|官方文档中文版](https://docs.mongoing.com/)，基于 4.2 版本。

<!--more-->

## 查询所有数据

```javascript
db.inventory.find( {} )
```

## 条件查询

```javascript
db.inventory.find( { status: "D" } )
```

### IN

```javascript
db.inventory.find( { status: { $in: [ "A", "D" ] } } )
```

### AND

```javascript
db.inventory.find( {
  status: "A", 
  qty: { $lt: 30 }
} )
```

### OR

```javascript
db.inventory.find( { 
	$or: [ 
		{ status: "A" }, 
		{ qty: { $lt: 30 } } 
	] 
} )
```

### AND 和 IN 混用

```javascript
db.inventory.find( {
     status: "A",
     $or: [ 
	     { qty: { $lt: 30 } }, 
	     { item: /^p/ } 
     ]
} )
```

### 范围查询

```javascript
db.bios.find( { birth: { $gt: new Date('1940-01-01'), $lt: new Date('1960-01-01') } } )
```

## 返回结果筛选

`sort`、`limit`、`skip` 三个命令的顺序不会影响执行的效果。

### sort

```javascript
db.bios.find().sort( { name: 1, age: -1 } )
```

### limit

```javascript
db.bios.find().limit( 5 )
```

### skip

跳过指定条数数据：

```javascript
db.bios.find().skip( 5 )
```

### countDocuments

`count` 方法在没有设置 query 条件时，只能返回 collection 在 meta 中的不确切的数量，在 MongoDB 4.0 已被弃用。

使用新的 `countDocuments` 方法返回确切的数据数量：

```javascript
db.orders.countDocuments( { ord_dt: { $gt: new Date('01/01/2012') } }, { limit: 100 } )
```



## 嵌套数据查询

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

## 数组查询

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

## 限制查询返回字段

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

## null 值处理

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

