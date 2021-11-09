---
title: "Go 字符串处理 strings"
description: 
date: 2021-08-05
categories:
  - Go标准库
tags:
  - Go
series:	
---

作为一种基本数据结构，每种语言都有一些对于字符串的预定义处理函数。Go 中使用 `strings` 包来完成对字符串的主要操作。

<!--more-->

### 判断前后缀

```go
// 判断字符串 s 是否以 prefix 开头
strings.HasPrefix(s, prefix string) bool
// 判断字符串 s 是否以 prefix 结尾
strings.HasSuffix(s, suffix string) bool
```

### 判断包含关系

```go
// 判断字符串 s 是否包含 substr
strings.Contains(s, substr string) bool
```

### 判断位置

`Index` 返回字符串 `str` 在字符串 `s` 中的索引（`str` 的第一个字符的索引），-1 表示字符串 `s` 不包含字符串 `str`：

```go
strings.Index(s, str string) int
```

`LastIndex` 返回字符串 `str` 在字符串 `s` 中最后出现位置的索引（`str` 的第一个字符的索引），-1 表示字符串 `s` 不包含字符串 `str`：

```go
strings.LastIndex(s, str string) int
```

如果 `ch` 是非 ASCII 编码的字符，建议使用以下函数来对字符进行定位：

```go
strings.IndexRune(s string, r rune) int
```

### 替换

`Replace` 用于将字符串 `str` 中的前 `n` 个字符串 `old` 替换为字符串 `new`，并返回一个新的字符串，如果 `n = -1` 则替换所有字符串 `old` 为字符串 `new`：

```go
strings.Replace(str, old, new, n) string
```

### 统计出现次数

```go
// 计算字符串 str 在字符串 s 中出现的非重叠次数
strings.Count(s, str string) int
```

### 重复字符串

```go
// 用于重复 count 次字符串 s 并返回一个新的字符串：
strings.Repeat(s, count int) string
```

### 大小写

```go
// 将字符串中的 Unicode 字符全部转换为相应的小写字符
strings.ToLower(s) string
// 将字符串中的 Unicode 字符全部转换为相应的大写字符
strings.ToUpper(s) string
```

### 裁剪前后

修剪掉 s 字符串前后，由 unicode 指定的空格字符，包括 `\n\t\r\n`

```go
func TrimSpace(s string) string
```

修剪掉 s 字符串前后的 cutset 字符，可以直接指定多个字符

```go
func Trim(s string, cutset string) string
func TrimLeft(s string, cutset string) string
func TrimRight(s string, cutset string) string
```

传入一个函数来依次判断字符是否需要被剪裁

```go
func TrimFunc(s string, f func(rune) bool) string
func TrimLeftFunc(s string, f func(rune) bool) string
func TrimRightFunc(s string, f func(rune) bool) string
```

修剪掉 s 字符串前后特定的 prefix 字符串前后缀

```go
func TrimPrefix(s, prefix string) string
func TrimSuffix(s, suffix string) string
```

### 分割字符串

`strings.Fields(s)` 将会利用 1 个或多个空白符号来作为动态长度的分隔符将字符串分割成若干小块，并返回一个 slice，如果字符串只包含空白符号，则返回一个长度为 0 的 slice。

`strings.Split(s, sep)` 用于自定义分割符号来对指定字符串进行分割，同样返回 slice。

因为这 2 个函数都会返回 slice，所以习惯使用 for-range 循环来对其进行处理。

### 拼接

```go
strings.Join(sl []string, sep string) string
```

### 从字符串中读取内容

函数 `strings.NewReader(str)` 用于生成一个 `Reader` 并读取字符串中的内容，然后返回指向该 `Reader`的指针，从其它类型读取内容的函数还有：

- `Read()` 从 []byte 中读取内容。
- `ReadByte()` 和 `ReadRune()` 从字符串中读取下一个 byte 或者 rune。



