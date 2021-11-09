---
title: "Go 并发消息队列 channel"
date: 2019-11-01 00:00:00
categories:
  - Go语法
tags:
  - Go
  - Concurrency
series:
  - Go 面试大全
---

`Channel` 实际上是类型化消息的队列，它有以下特性：

- 只能传输一种类型的数据。
- 所有的类型都可以用于通道，空接口 `interface{}` 也可以。
- 先进先出 `FIFO` 的结构。
- 引用类型，所以使用 `make()` 函数来给它分配内存。

<!--more-->

## 用法

### 声明

声明一个无缓冲的 `Channel`：

```go
var ch1 chan int
ch2 := make(chan int)
```

有缓冲 `Channel`：

```go
ch := make(chan int, 1)
```

### 发送数据

```go
ch := make(chan int)
defer close(ch)

ch <- 5
```

**阻塞：**

- 向 `nil` 通道发送数据会被阻塞。
- 向无缓冲 channel 写数据，如果读协程没有准备好会阻塞。
- 向有缓冲 channel 写数据，如果缓冲已满会阻塞。

**panic：**

- 向已 closed 的 channel 写数据会 panic。

有缓冲的 channel，如果发送的时候，刚好有等待接收的协程，那么会直接交换数据。

### 接收数据

```go
v, ok := <- ch
v := <-ch
```

`ok` 可以用来判断 channel 是否已经关闭，如果关闭该值为 true ，此时 v 接收到的是 channel 类型的零值。

`<- ch` 可以单独调用获取通道的（下一个）值，当前值会被丢弃，但是可以用来验证，所以以下代码是合法的：

```go
if <- ch != 1000{
    ...
}
```

**阻塞：**

- 从 `nil` 通道接收数据会被阻塞。

- 从无缓冲 channel 读数据，如果写协程没有准备好，会阻塞。
- 从有缓冲 channel 读数据，如果缓冲为空，会阻塞。

读取的 channel 如果被关闭，并不会影响正在读的数据，它会将所有数据读取完毕，并不会立即就失败或者返回零值。

### 通道方向

通道在创建时都是双向的，但是我们可以声明单向的通道，在传递参数时用来限制使用者对通道的操作：

```go
func source(ch chan<- int){ // 单向入队列
    for { ch <- 1 }
}

func sink(ch <-chan int) { // 单向出队列
    for { <-ch }
}
```

**特性：**

- 只接收的通道 `<-chan T` 是无法关闭的。

### 遍历读取

`for-range` 可以用来便利读取 channel 中的数据：

```go
ch := make(chan int, 1)

for val := range ch {
    fmt.Println(val)
}
```

- 如果 channel 已经被关闭，它还是会继续执行，直到所有值被取完，然后退出执行。

- 如果通道没有关闭，但是 channel 没有可读取的数据，它则会阻塞在 range 这句位置，直到被唤醒。
- 如果 channel 是 nil，那么同样符合我们上面说的的原则，读取会被阻塞，也就是会一直阻塞在 `range` 位置。

### select

`select case` 语句可以方便的对通道进行操作，每次执行 select 语句，将会执行所有的 case 语句，判断是否有表达式能够执行。

也就是说，select case 语句能同时监控多个通道，当有通道离开堵塞状态，就可以获知并执行相应的代码。

```go
ch := make(chan int)
q := make(chan int)

for {
    select {
    case ch <- x:
        x, y = y, x+y
        break
    case <-q:
        fmt.Println("quit")
        return
    }
}
```

1. select 只要有默认语句，就不会被阻塞，换句话说，如果没有 default，然后 case 又都不能读或者写，则会被阻塞。
2. nil 的 channel，不管读写都会被阻塞。
3. select 不能够像 `for-range` 一样发现 channel 被关闭而终止执行，所以需要结合 `multi-valued assignment` 来处理。
4. 如果同时有多个 case 满足了条件，会使用伪随机选择一个 case 来执行。
5. select 语句如果不配合 for 语句使用，只会对 case 表达式求值一次。
6. 每次 select 语句的执行，是会扫码完所有的 case 后才确定如何执行，而不是说遇到合适的 case 就直接执行了。

### 关闭通道

channel 在使用完毕以后需要关闭，一般的建议是谁写入，谁负责关闭。

如果有多个写协程的 channel 需要关闭，可以使用额外的 channel 来标记，也可以使用 `sync.Once` 或者 `sync.Mutex` 来处理。

```go
ch := make(chan int, 1)
defer close(ch)
```

**注：closed 的 channel，再次关闭会 panic。**

### 空 struct

有时我们需要将 channel 作为一个信号传输的管道，此时管道中的内容并不重要，只需要知道有信号传输过来即可。

此时可以使用空 struct 作为传输的数据类型，因为空 struct 类型的内存占用大小为 0。即：

```go
var c1 chan struct{}
c1 <- struct{}{}
```

