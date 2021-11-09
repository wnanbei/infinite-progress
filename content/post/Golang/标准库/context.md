---
title: "Go 上下文 context"
description: 
date: 2020-06-05 00:00:00
categories:
  - Go标准库
tags:
  - Go
  - Concurrency
series:	
  - Go 面试大全
---

context 指的是上下文，以下是几种 ctx 类型:

- emptyCtx - 所有 ctx 类型的根，用 `context.TODO()`，或 `context.Background()` 来生成。
- valueCtx - 主要就是为了在 ctx 中嵌入上下文数据，一个简单的 k 和 v 结构，同一个 ctx 内只支持一对 kv，需要更多的 kv 的话，会形成一棵树形结构。
- cancelCtx - 用来取消程序的执行树，一般用 `WithCancel`，`WithTimeout`，`WithDeadline` 返回的取消函数本质上都是对应了 cancelCtx。
- timerCtx - 在 cancelCtx 上包了一层，支持基于时间的 cancel。

<!--more-->

## 用法

### 初始化 context

一般使用 `context.TODO()` 和 `context.Background()` 创建 context，是所有 context 的根，todo 和 background 两者本质上只有名字区别，在按 string 输出的时候会有区别。

### valueCtx

valueCtx 主要就是用来携带贯穿整个逻辑流程的数据，使用 `WithValue` 创建。

```go
type orderID int
var x = context.TODO()
x = context.WithValue(x, orderID(1), "1234")
x = context.WithValue(x, orderID(2), "2345")
x = context.WithValue(x, orderID(3), "3456")
```

key 必须为非空，且可比较。

在查找值，即执行 Value 操作时，会先判断当前节点的 k 是不是等于用户的输入 k，如果相等，返回结果，如果不等，会依次向上从子节点向父节点，一直查找到整个 ctx 的根。没有找到返回 nil。是一个递归流程：

```go
func (c *valueCtx) Value(key interface{}) interface{} {
 if c.key == key {
  return c.val
 }
 return c.Context.Value(key) // 这里发生了递归，c.Context 就是 c.parent
}
```

### cancelCtx

cancelCtx 主要用于协程的控制，例如关闭协程。使用 `WithCancel` 创建。

```go
ctx, cancelFn := context.WithCancel(context.TODO())
go func() {
    for {
        select {
            case <-jobChan:
            fmt.Println("do my job")
            case <-ctx.Done():
            fmt.Println("parent call me to quit")
            break jobLoop
        }
    }
}()
// 停止所有 worker
cancelFn()
```

### timerCtx

timerCtx 用于定时的取消任务。

```go
ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
defer cancel()

select {
    case <-time.After(1 * time.Second):
    fmt.Println("overslept")
    case <-ctx.Done():
    fmt.Println(ctx.Err()) // prints "context deadline exceeded"
}
```

