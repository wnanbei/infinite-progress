---
title: "Golang 第三方库 gjson"
description: 
date: 2021-08-05 00:00:00
categories:
  - Golang 第三方库
tags:
  - Golang
series:	
---

这是一个主要功能为从 Json 中提取值的包。

<!--more-->

安装：

```bash
go get -u github.com/tidwall/gjson
```

## 获取 Json 值

### 获取单个值

获取值最常用的是这两个方法，可以直接从 Json 数据中根据你提供的 `path` 提取结果。

`GetBytes` 需要的是 `[]byte` 数据。

方法将返回一个类型为 `Result` 的结果。

```go
func Get(json, path string) Result
func GetBytes(json []byte, path string) Result
```

### 获取多个值

如果要同时获取多个值，可以使用以下方法。

```go
func GetMany(json string, path ...string) []Result
func GetManyBytes(json []byte, path ...string) []Result
```

这两个方法可以传入多个 `path` 路径，返回的是一个以 `Result` 组成的切片。

### 解析

将 Json 数据直接解析成 Result。

```go
func Parse(json string) Result
func ParseBytes(json []byte) Result
```

## 路径语法

`Gjson` 的路径主要由 `.` 分割的字段名构成，其中还包含一些特殊的符号。

以下是一个示例 Json 数据：

```json
{
  "name": {"first": "Tom", "last": "Anderson"},
  "age":37,
  "children": ["Sara","Alex","Jack"],
  "fav.movie": "Deer Hunter",
  "friends": [
    {"first": "Dale", "last": "Murphy", "age": 44, "nets": ["ig", "fb", "tw"]},
    {"first": "Roger", "last": "Craig", "age": 68, "nets": ["fb", "tw"]},
    {"first": "Jane", "last": "Murphy", "age": 47, "nets": ["ig", "tw"]}
  ]
}
```

### 基础语法

以下是通过 `.` 代表层级递进的语法：

```go
name.last              "Anderson"
name.first             "Tom"
age                    37
children               ["Sara","Alex","Jack"]
```

Json 数组可以使用数字序号选取具体值：

```go
children.0             "Sara"
children.1             "Alex"
friends.1              {"first": "Roger", "last": "Craig", "age": 68}
friends.1.first        "Roger"
```

**注意：特殊符号 `*`, `?`, `.` 等符号如果出现在 Json 字段名中，则路径需要使用 `\` 转义。**

### 通配符

路径中可以使用通配符 `*` 和 `?`。

- `*` 代表任意多个任意字符
- `?` 代表一个任意字符

```go
child*.2               "Jack"
c?ildren.0             "Sara"
```

### 查询语法

Json 数组还可以使用 `#` 来进一步获取值：

```go
friends.#              3            // friends 数组元素的数量
friends.#.age         [44,68,47]    // 单独获取 friends 数组元素的 age 字段
```

除此以外，`Gjson` 还支持类似数据库的查询语法：

- `#(...)` 代表根据括号里的条件查询单个结果
- `#(...)#` 代表根据括号里的条件查询所有结果

例如：

```go
friends.#(last=="Murphy").first     "Dale"
friends.#(last=="Murphy")#.first    ["Dale","Jane"]
friends.#(age>45)#.last             ["Craig","Murphy"]
friends.#(first%"D*").last          "Murphy"
friends.#(first!%"D*").last         "Craig"
```

### 修饰符

使用修饰符可以实现一些特定的效果，目前有三个内置的修饰符：

- `@reverse` 反转一个数组内元素的顺序
- `@ugly` 移除 Json 中所有的空格
- `@pretty` 美化 Json 的显示

例子：

```go
children.@reverse                   ["Jack","Alex","Sara"]
children.@reverse.0                 "Jack"
```

其中 `@pretty` 还可以使用一些参数，例如：

```go
@pretty:{"sortKeys":true}
```

其可以使用的参数有：`sortKeys`, `indent`, `prefix`, and `width`.

## Result

`Gjson` 获取的内容都是 `Result` 类型的数据。

### 获取值数据

确定值的类型的获取方法

```go
func (t Result) String() string
func (t Result) Int() int64
func (t Result) Uint() uint64
func (t Result) Float() float64
func (t Result) Bool() bool
func (t Result) Array() []Result
func (t Result) Map() map[string]Result
func (t Result) Time() time.Time
```

还有一个方法，返回不确定类型的值，需要进行类型断言

```go
func (t Result) Value() interface{}
```

值的类型为以下其中之一

```go
boolean >> bool
number  >> float64
string  >> string
null    >> nil
array   >> []interface{}
object  >> map[string]interface{}
```

### 功能方法

```go
// 用于链式调用获取值
func (t Result) Get(path string) Result
// 判断值是否存在
func (t Result) Exists() bool
// 判断值是否是一个 Json 对象
func (t Result) IsObject() bool
// 用于遍历值，函数中返回 false 会停止遍历
func (t Result) ForEach(iterator func(key, value Result) bool)
func (t Result) Less(token Result, caseSensitive bool) bool
```

### 字段

`Result` 还含有一些有用的字段。

```go
result.Type    // Reault 类型 Number, True, False, Null, JSON
result.Str     // 获取字符串值
result.Num     // 获取数值
result.Raw     // 获取原始的 Json 文本
result.Index   // 在原始 Json 数据中的索引，0 表示 Gjson无法识别
```

## 检查 Json

在使用 `Get` 等方法获取值时，默认给予的 Json 数据为格式正确的，如果格式错误并不会报错，只会返回期望外的数据。

所以，如果希望验证 Json 格式的正确性，可以在获取值前先行验证 Json 数据的格式：

```go
if !gjson.Valid(json) {
	return errors.New("invalid json")
}
value := gjson.Get(json, "name.last")
```

## Bytes 数据

如果希望全程使用 `[]byte` 处理数据，而避免将 `result.Raw` 从字符串转换为 `[]byte`，可以使用以下方法：

```go
var json []byte = ...
result := gjson.GetBytes(json, path)
var raw []byte
if result.Index > 0 {
    raw = json[result.Index:result.Index+len(result.Raw)]
} else {
    raw = []byte(result.Raw)
}
```

## Jsoniter

将 Go 中数据转换成 Json 的方法，`Gjson` 中已经弃用了，建议使用滴滴开源的的 `jsoniter`。

安装：

```bash
go get github.com/json-iterator/go
```

使用：

```go
import "github.com/json-iterator/go"

jsoniter.Marshal(&data)
jsoniter.Unmarshal(input, &data)
```

### 兼容标准库

`jsoniter` 还提供了完全兼容标准库的使用方式。

```go
import "github.com/json-iterator/go"

var json = jsoniter.ConfigCompatibleWithStandardLibrary
json.Marshal(&data)
json.Unmarshal(input, &data)
```

