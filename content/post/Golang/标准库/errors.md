---
title: "Golang 错误处理 errors"
description: 
date: 2020-06-05 00:00:00
categories:
  - Golang 标准库
tags:
  - Golang
series:	
---

`errors` 用于增强 Golang 的错误处理能力。

<!--more-->

## Errors

`error` 类型为一个接口，其定义为：

```go
type error interface {
	Error() string
}
```

### Error

`Error()` 函数返回一个字符串，用以表示这个 `error` 类型。

```go
func (e *error) Error() string (
	return e.msg
)
```

### Wrap

Go 1.13 中新增，可以将 `error` 嵌套起来，形成多层结构。

简单的嵌套方法，用 `Errorf()` 和 `%w`：

```go
e := errors.New("原始错误")
w := fmt.Errorf("Wrap了一个错误%w", e)
```

复杂的自定义类型需要拥有 `Unwrap()` 方法的 `error` 类型。

`Unwrap()` 的定义举例：

```go
type NewError struct {
	err error
	msg string
}

func (e *NewError) Error() string {
	return e.err.Error() + e.msg
}

func (e *NewError) Unwrap() error (
	return e.err
)
```

用法：

```go
e := errors.New("原始错误")
w := &NewError{err: e, msg: "wrap了一个错误"}
```

### Unwrap

Go 1.13 新增，这个函数可以把嵌套在 `error` 中的 `error` 取出。

```go
func Unwrap(err error) error
```

### Is

用来判断 `err` 或者其嵌套链中，是否有 `target` 类型的异常，只能判断已经生成的特定类型 `error`，也就是所谓的哨兵异常。

```go
func Is(err error, target error) bool
```

用法：

```go
if errors.Is(err, os.ErrExist)
```

### As

用来判断 `err` 或者其嵌套链中，是否有 `target` 的异常，如果有，就将符合类型的 `err` 赋值给 `target`。

这种方式只能判断指定的自定义异常类型，也就是 `struct`。

```go
func As(err error, target interface{}) bool
```

用法：

```go
var tar *os.PathError
if errors.As(err, &tar) {
	fmt.Println(perr.Path)
}
```

