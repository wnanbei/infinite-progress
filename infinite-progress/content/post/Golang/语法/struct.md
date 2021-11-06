---
title: "Golang struct"
date: 2019-11-01 00:00:00
categories:
  - Golang 语法
tags:
  - Golang
series:
---

Go 通过类型别名 `alias types` 和结构体的形式支持用户自定义类型。

## 定义和声明

### 定义

结构体定义的一般方式如下

```go
type identifier struct {
    field1 type1
    field2 type2
    ...
}
type T struct {a, b int}
```

- 结构体的字段可以是任何类型，甚至是结构体本身，也可以是函数或者接口

### 声明与赋值

```go
var s T
s.a = 5
s.b = 8
```

简短初始化方式

```go
intr := Interval{0, 3}            // 需要按顺序赋值
intr := Interval{end:5, start:1}  // 不需要按顺序
intr := Interval{end:5}           // 甚至可以忽略某些字段
```

- 结构体未赋值的字段的值是它们所属类型的零值

### new 声明

使用 `new` 函数给一个新的结构体变量分配内存，它会返回指向已分配内存的指针

```go
var t *T = new(T)
t := new(T)
var t *T  // 也可以分离定义和声明
t = new(T) 
```

- 变量 `t` 是一个指向 `T` 的指针，此时结构体字段的值是它们所属类型的零值

- 声明 `var t T` 也会给 `t` 分配内存，并零值化内存，但是这个时候 `t` 是类型 `T`

初始化一个结构体实例更简短和惯用的方式如下

```go
ms := &struct1{10, 15.5, "Chris"}
// 此时ms的类型是 *struct1
```

这是一种简写，底层仍然会调用 `new ()`，这里值的顺序必须按照字段顺序来写。表达式 `new(Type)` 和 `&Type{}` 是等价的。

### struct 字段

使用点号符给字段赋值取值

```go
structname.fieldname = value
v := structname.fieldname
```

无论变量是一个结构体类型还是一个结构体类型指针，都使用同样的方式来使用结构体的字段

```go
type myStruct struct { i int }
var v myStruct    // v是结构体类型变量
var p *myStruct   // p是指向一个结构体类型变量的指针
v.i
p.i
```

也可以通过解引用的方式来设置值：`(*pers2).lastName = "Woodward"`

### struct 转换

Go 中的类型转换遵循严格的规则。当为结构体定义了一个 alias 类型时，此结构体类型和它的 alias 类型都有相同的底层类型，它们可以互相转换。

同时需要注意其中非法赋值或转换引起的编译错误。

```go
type number struct {
    f float32
}
type nr number   // 别名

a := number{5.0}
b := nr{5.0}
var c = number(b)
```

## struct 嵌套

### 匿名字段

结构体可以包含一个或多个匿名字段，即这些字段没有显式的名字，只有字段的类型是必须的，此时类型就是字段的名字。

- 在一个结构体中对于每一种数据类型只能有一个匿名字段

```go
type outerS struct {
    b    int
    c    float32
    int
}

outer := new(outerS)
outer.b = 6
outer.c = 7.5
outer.int = 60

fmt.Printf("outer.b is: %d\n", outer.b)
fmt.Printf("outer.c is: %f\n", outer.c)
fmt.Printf("outer.int is: %d\n", outer.int)
```

### 内嵌 struct

同样地，结构体也是一种数据类型，所以它也可以作为一个匿名字段来使用。

当一个匿名类型被内嵌在结构体中时，匿名类型的可见方法也同样被内嵌，这在效果上等同于外层类型继承了这些方法：

- 将父类型放在子类型中来实现亚型。

这个机制提供了一种简单的方式来模拟经典面向对象语言中的子类和继承相关的效果，类似 Ruby 中的 `mixin`。

```go
type A struct {
    ax, ay int
}

type B struct {
    A
    bx, by float32
}

func main() {
    b := B{A{1, 2}, 3.0, 4.0}
    fmt.Println(b.ax, b.ay, b.bx, b.by)
    fmt.Println(b.A)
}
```

- 内嵌将一个已存在类型的字段和方法注入到了另一个类型里，匿名字段上的方法晋升成为了外层类型的方法。

- 结构体内嵌和自己在同一个包中的结构体时，可以彼此访问对方所有的字段和方法。

### 字段名冲突

当 struct 嵌套时有字段名冲突，按以下规则：

1. 外层字段名会覆盖内层字段名，但是两者的内存空间都保留，这提供了一种重载字段或方法的方式

2. 如果相同的名字在同一级别出现了两次，如果这个名字被程序使用了，将会引发一个异常，不使用则没关系。

   没有办法来解决这种问题引起的二义性，必须由程序员自己修正。

例子：

```go
type A struct {a int}
type B struct {a, b int}

type C struct {A; B}
var c C
```

在字段名冲突时，如果依然需要使用被覆盖的字段，则可以使用 `c.b.a` 这种方式。

## method

Go 的方法是作用在接收者 `receiver` 上的一个函数，接收者是某种类型的变量。因此方法是一种特殊类型的函数。

接收者类型可以是几乎任何类型，不仅仅是结构体类型，任何类型都可以有方法，甚至可以是函数类型，可以是 int、bool、string 或数组的别名类型。

- 接收者不能是一个接口类型，因为接口是一个抽象定义，但是方法却是具体实现。

- 接收者不能是一个原始的指针类型，但是它可以是任何其他允许类型的指针。

### 定义

定义方法的一般格式如下

```go
func (recv recv_type) method(param_list) (return_list) { ... }
```

如果 `recv` 是一个指针，Go 会自动解引用。

```go
type TwoInts struct {
    a int
    b int
}

func (tn *TwoInts) AddToParam(param int) int {
    return tn.a + tn.b + param
}
```

### 接收者

由于性能的原因，`recv` 最常见的值是一个指向 `recv_type` 的指针，特别是在 `recv` 类型是结构体时。

- 当接收者是值类型时，方法无法修改接收者本身的数据，因为是值传递
- 当接收者是指针类型时，方法内可以修改接收者本身的数据

无论是值还是指针，通过接收者调用方法时，Go 为我们做了探测工作，方法都支持运行。

### 给包外类型定义方法

类型和作用在它上面定义的方法必须在同一个包里定义，这就是为什么不能在 int、float 或类似这些的类型上定义方法。试图在 int 类型上定义方法会编译失败。

但是有一个间接的方式：可以先定义该类型的别名类型，然后再为别名类型定义方法。

或者将它作为匿名类型嵌入在一个新的结构体中。当然方法只在这个别名类型上有效。

## Tag

## 底层

### struct 的内存布局

Go 语言中，结构体和它所包含的数据在内存中是以连续块的形式存在的，即使结构体中嵌套有其他的结构体，这在性能上带来了很大的优势。

下面的例子清晰地说明了这些情况：

```go
type Rect1 struct {Min, Max Point }
type Rect2 struct {Min, Max *Point }
```