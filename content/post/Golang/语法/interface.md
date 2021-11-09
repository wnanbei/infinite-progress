---
title: "Golang interface 接口"
date: 2019-10-01 00:00:00
categories:
  - Golang 语法
tags:
  - Golang
series:
---

interface 的一些特性：

- 类型不需要显式声明它实现了某个接口：接口被隐式地实现。
- 多个类型可以实现同一个接口。
- 实现某个接口的类型，除了实现接口方法外，可以有其他的方法。
- 一个类型可以实现多个接口。
- 接口是动态类型，可以包含一个实例的引用，该实例的类型实现了此接口。
- 即使接口在类型之后才定义，二者处于不同的包中，被单独编译，只要类型实现了接口中的方法，它就实现了此接口。
- 接口只能访问接口内声明的方法。

<!--more-->

## 定义

```go
type Namer interface {
    Method1(param_list) return_type
    Method2(param_list) return_type
}
```

- 按照约定，只包含一个方法的接口的名字由方法名加 `[e]r` 后缀组成，例如 `Printer`、`Reader`、`Writer`、`Logger`、`Converter` 等等。
- 当后缀 `er` 不合适时，比如 `Recoverable`，此时接口名以 `able` 结尾，或者以 `I` 开头。

- Go 语言中的接口都很简短，通常它们会包含 0 个、最多 3 个方法。

### 空接口

空接口 `interface{}` 不包含任何方法，它对实现不做任何要求：

```go
type Any interface{}
```

- 空接口类型的变量可以赋任何类型的值。
- 任何其他类型都实现了空接口，`any` 或 `Any` 是空接口一个很好的别名或缩写。

- 每个 `interface {}` 变量在内存中占据两个字长：一个用来存储它包含的类型，另一个用来存储它包含的数据或者指向数据的指针。

### 嵌套接口

一个接口可以包含一个或多个其他的接口，这相当于直接将这些内嵌接口的方法列举在外层接口中一样。

比如接口 `File` 包含了 `ReadWrite` 和 `Lock` 的所有方法，它还额外有一个 `Close()` 方法。

```go
type ReadWrite interface {
    Read(b Buffer) bool
    Write(b Buffer) bool
}
type Lock interface {
    Lock()
    Unlock()
}
type File interface {
    ReadWrite
    Lock
    Close()
}
```

### 接口继承

当一个 Struct 包含另一个 Struct（实现了一个或多个接口）的指针时，这个 Struct 就可以使用另一个 Struct 所有的接口方法。例如：

```go
type Task struct {
    Command string
    *log.Logger
}
```

这个 Struct 的工厂方法像这样：

```go
func NewTask(command string, logger *log.Logger) *Task {
    return &Task{command, logger}
}
```

当 `log.Logger` 实现了 `Log()` 方法后，Task 的实例 task 就可以调用该方法：

```go
task.Log()
```

Struct 可以通过继承多个接口来提供像 `多重继承` 一样的特性：

```go
type ReaderWriter struct {
    *io.Reader
    *io.Writer
}
```

## 实现

当一个类型实现了某一个接口所包含的所有方法时，则认为这个类型实现了这个接口。

```go
type Handler interface {
	Do(k, v interface{})
}

type HandlerFunc func(k, v interface{})

func (f HandlerFunc) Do(k, v interface{}) {
	f(k, v)
}
```

类型可以是函数，也可以是 Struct。

### 接收者问题

对于一个 Struct，作用于变量上的方法不区分变量到底是指针还是值。

但当碰到接口类型值时，由于接口变量中存储的具体值是不可寻址的，所以有一定区别。

```go
type List []int

func (l List) Len() int {
    return len(l)
}

func (l *List) Append(val int) {
    *l = append(*l, val)
}

type Appender interface {
    Append(int)
}

type Lener interface {
    Len() int
}
```

- 由于 Appender 接口的方法 Append 实现时使用的是 Struct 的指针，此时使用 `var lst List` 方式声明的值类型，将不被认为是实现了 Appender 接口。
- 此时必须使用 `plst := new(List)` 方式，得到的指针类型，才会被认定为实现了 Appender 接口。
- 而无论哪种类型，都会被认定为实现了 `Lener` 接口，因为如果是指针类型，会自动解引用。

**总结：**

- 指针方法可以通过指针调用。
- 值方法可以通过值调用。
- 接收者是值的方法可以通过指针调用，因为指针会首先被解引用。
- 接收者是指针的方法不可以通过值调用，因为存储在接口中的值没有地址。


## 断言

一个接口类型的变量 `varI` 中可以包含任何类型的值，必须有一种方式来检测它的动态类型，即运行时在变量中存储的值的实际类型。

在执行过程中动态类型可能会有所不同，但是它总是可以分配给接口变量本身的类型。通常我们可以使用类型断言来测试在某个时刻 `varI` 是否包含类型 `T` 的值。

```go
v := varI.(T)
```

- `varI` 必须是一个接口变量，否则编译器会报错。

类型断言失败后会导致异常退出，如果不希望退出，可以使用以下方法。

```go
if v, ok := varI.(T); ok {
    Process(v)
    return
}
```

- 如果类型符合，`varI` 是类型 `T` 的值，`ok` 会是 `true`。
- 否则 `v` 是类型 `T` 的零值，`ok` 是 `false`，没有运行时错误发生。

此方法也可以检测变量是否实现了某个接口

```go
type Stringer interface {
    String() string
}

if sv, ok := v.(Stringer); ok {
    fmt.Printf("v implements String(): %s\n", sv.String())
}
```
