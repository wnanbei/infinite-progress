# New 和 Make 区别

Go 语言中 `new` 和 `make` 是内建的两个函数，主要用来创建分配类型内存。

### 1. 变量声明

通常的声明方式

```go
var i int
var s string
```

- 声明值类型变量时，默认值是他们的零值
- 声明引用类型时，类型的零值是 `nil`

而直接声明引用类型，会造成引用类型的内存空间没有分配，无法存储数据，例如以下代码会报错：

```go
var i *int
*i = 10
fmt.Println(*i)
```

### 2. New 声明

`new()` 函数主要用于为变量分配内存，并初始化为零值。

它只接受一个类型作为参数，分配好内存后，返回一个指向该类型内存地址的指针。同时把分配的内存值置为零，也就是类型的零值。

```go
var i *int
i = new(int)
*i = 10
fmt.Println(*i)
```

struct 可以用此方法将 struct 内部的字段初始化，例如下面这个 user 类型的 lock 字段，不需要额为的初始化了。

`new()` 也等价于 `u := user{}`

```go
type user struct {
	lock sync.Mutex
	name string
	age int
}

u:=new(user)
u.lock.Lock()
u.name = "张三"
u.lock.Unlock()
```

### 3. Make 声明

`make()` 函数主要用于为特定类型分配内存：`Slice`、`Map`、`Chan`。

- 因为这三种类型就是引用类型，所以没有必要返回他们的指针。

- 由于是引用类型，所以必须得初始化，但不是置为零值，这是和 `new` 不一样的。

### 4. 异同

相同：

- 二者都是内存的分配（堆上）

差异：

- `make` 只用于 slice、map、chan 的初始化（非零值）
- `new` 用于类型的内存分配，并且内存置为零值