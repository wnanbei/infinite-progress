---
title: "Golang Gorm CRUD"
description: 
date: 2021-08-05
categories:
  - Golang 第三方库
tags:
  - Golang
  - Gorm
series:	
  - Golang Web 开发
---

Gorm 常见 CRUD 操作 API。

<!--more-->

## 通用操作

### 批量操作

最简单的方法是使用 `Table()` 方法，指定要操作的表名，这时候进行 `CRUD` 操作就是批量的。

```go
db.Table("users").Update(...)
```

或者使用 `Model()`，传入没有主键的模型实例，没有指定主键的话 Gorm 默认会使用批量操作，一般惯常用法是使用空的主键实例。

```go
db.Model(User{}).Update(...)
```

### 根据主键操作

使用 `Model()` 方法，如果传入的模型实例有主键值，那么会默认只操作符合主键值的单条数据。

```go
user := User{ID: 1}
db.Model(&user).Update(...)
```

### where

除了使用主键选定操作数据，还可以使用 `Where` 设定条件。

```go
// Get first matched record
// SELECT * FROM users WHERE name = 'jinzhu' limit 1;
db.Where("name = ?", "jinzhu").First(&user)

// Get all matched records
// SELECT * FROM users WHERE name = 'jinzhu';
db.Where("name = ?", "jinzhu").Find(&users)

// <>
// SELECT * FROM users WHERE name <> 'jinzhu';
db.Where("name <> ?", "jinzhu").Find(&users)

// IN
// SELECT * FROM users WHERE name in ('jinzhu','jinzhu 2');
db.Where("name IN (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)

// LIKE
// SELECT * FROM users WHERE name LIKE '%jin%';
db.Where("name LIKE ?", "%jin%").Find(&users)

// AND
// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)

// Time
// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';
db.Where("updated_at > ?", lastWeek).Find(&users)

// BETWEEN
// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)

// Struct
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)

// Map
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)

// 主键的切片
// SELECT * FROM users WHERE id IN (20, 21, 22);
db.Where([]int64{20, 21, 22}).Find(&users)
```

### not

与 `Where` 相反。

```go
// SELECT * FROM users WHERE name <> "jinzhu" LIMIT 1;
db.Not("name", "jinzhu").First(&user)

// Not In
// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");
db.Not("name", []string{"jinzhu", "jinzhu 2"}).Find(&users)

// Not In slice of primary keys
// SELECT * FROM users WHERE id NOT IN (1,2,3);
db.Not([]int64{1,2,3}).First(&user)

// SELECT * FROM users;
db.Not([]int64{}).First(&user)

// Plain SQL
// SELECT * FROM users WHERE NOT(name = "jinzhu");
db.Not("name = ?", "jinzhu").First(&user)

// Struct
// SELECT * FROM users WHERE name <> "jinzhu";
db.Not(User{Name: "jinzhu"}).First(&user)
```

### or

```go
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)

// Struct
// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2"}).Find(&users)

// Map
// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2"}).Find(&users)
```

### order

排序，设置第二个参数 `reorder` 为 `true` ，可以覆盖前面定义的排序条件。

```go
// SELECT * FROM users ORDER BY age desc, name;
db.Order("age desc, name").Find(&users)

// 多字段排序
// SELECT * FROM users ORDER BY age desc, name;
db.Order("age desc").Order("name").Find(&users)

// 覆盖排序
// SELECT * FROM users ORDER BY age desc; (users1)
// SELECT * FROM users ORDER BY age; (users2)
db.Order("age desc").Find(&users1).Order("age", true).Find(&users2)
```

### limit

```go
// SELECT * FROM users LIMIT 3;
db.Limit(3).Find(&users)

// -1 取消 Limit 条件
// SELECT * FROM users LIMIT 10; (users1)
// SELECT * FROM users; (users2)
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
```

### offset

```go
// SELECT * FROM users OFFSET 3;
db.Offset(3).Find(&users)

// -1 取消 Offset 条件
// SELECT * FROM users OFFSET 10; (users1)
// SELECT * FROM users; (users2)
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
```

### count

`Count` 必须是链式查询的最后一个操作 ，因为它会覆盖前面的 `SELECT`，但如果里面使用了 `count` 时不会覆盖。

```go
// SELECT * from USERS WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (users)
// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)
db.Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Find(&users).Count(&count)

// SELECT count(*) FROM users WHERE name = 'jinzhu'; (count)
db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)

// SELECT count(*) FROM deleted_users;
db.Table("deleted_users").Count(&count)

// SELECT count( distinct(name) ) FROM deleted_users;
db.Table("deleted_users").Select("count(distinct(name))").Count(&count())
```

## 插入

判断数据是否存在于数据库中：

```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}
db.NewRecord(user)
```

插入数据：

```go
db.Create(&user)
```

### 插入前处理

如果想在插入数据前做一定的处理，可以为模型设置 `BeforeCreate()` 方法：

```go
func (user *User) BeforeCreate(scope *gorm.Scope) error {
  scope.SetColumn("ID", uuid.New())
  return nil
}
```

## 查询

查询单条数据时可以使用单个 `user` 接收数据，如果查询到的是多条数据，需要使用切片 `users` 进行接收。

### 根据主键查询数据

```go
// 根据主键查询第一条记录
// SELECT * FROM users ORDER BY id LIMIT 1;
db.First(&user)

// 随机获取一条记录
// SELECT * FROM users LIMIT 1;
db.Take(&user)

// 根据主键查询最后一条记录
// SELECT * FROM users ORDER BY id DESC LIMIT 1;
db.Last(&user)

// 查询所有的记录
// SELECT * FROM users;
db.Find(&users)

// 查询指定的某条记录(仅当主键为整型时可用)
// SELECT * FROM users WHERE id = 10;
db.First(&user, 10)
```

### 内联条件

当多个立即执行方法一起使用时，会默认共享前一个执行方法的条件。如果不希望共享，可以使用内联条件。

```go
// 根据主键获取记录 (只适用于整形主键)
// SELECT * FROM users WHERE id = 23 LIMIT 1;
db.First(&user, 23)

// 根据主键获取记录, 如果它是一个非整形主键
// SELECT * FROM users WHERE id = 'string_primary_key' LIMIT 1;
db.First(&user, "id = ?", "string_primary_key")

// Plain SQL
// SELECT * FROM users WHERE name = "jinzhu";
db.Find(&user, "name = ?", "jinzhu")

// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;
db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)

// Struct
// SELECT * FROM users WHERE age = 20;
db.Find(&users, User{Age: 20})

// Map
// SELECT * FROM users WHERE age = 20;
db.Find(&users, map[string]interface{}{"age": 20})
```

## 更新

```go
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
```

`Save` 会更新所有字段，即使你没有更改这个字段的值。

### 更新指定字段

只更新指定的字段可以使用 `Update` 或 `Updates`。

- 更新单个字段

  ```go
  db.Model(&user).Update("name", "hello")
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;
  db.Model(&user).Where("active = ?", true).Update("name", "hello")
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
  ```

- 使用 `map` 更新多个字段

  ```go
  db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
  // UPDATE users SET name='hello', age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
  ```

- 使用 `struct` 更新多个字段，只会更新其中有变化且为非零值的字段

  ```go
  db.Model(&user).Updates(User{Name: "hello", Age: 18})
  // UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
  ```

  零值的字段不会发生任何更新，以下操作将没有任何更新

  ```go
  db.Model(&user).Updates(User{Name: "", Age: 0, Actived: false})
  ```

- 更新时选定或者排除某些字段

  ```go
  // Select 选定某些字段
  // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;
  db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
  
  // Omit 排除某些字段
  //// UPDATE users SET age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
  db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
  ```

### 忽略中间件

使用 `Update` 和 `Updates` 更新时会调用默认的 `BeforeUpdate()` 和 `AfterUpdate` 方法，这些默认方法会更新 `UpdateAt` 时间戳。

如果想不调用这些方法，可以使用 `UpdateColumn`， `UpdateColumns`。

```go
// 更新单个属性，类似于 `Update`
db.Model(&user).UpdateColumn("name", "hello")
//// UPDATE users SET name='hello' WHERE id = 111;

// 更新多个属性，类似于 `Updates`
db.Model(&user).UpdateColumns(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18 WHERE id = 111;
```

### 使用 SQL 表达式

```go
DB.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
//// UPDATE "products" SET "price" = price * '2' + '100', "updated_at" = '2013-11-17 21:34:10' WHERE "id" = '2';
```

## 删除

删除时确保主键字段有值，GORM 会通过主键去删除记录，如果主键为空，GORM 会删除该 model 的所有记录。

```go
db.Delete(&email)
```

批量删除

```go
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
//// DELETE from emails where email LIKE "%jinzhu%";

db.Delete(Email{}, "email LIKE ?", "%jinzhu%")
//// DELETE from emails where email LIKE "%jinzhu%";
```

