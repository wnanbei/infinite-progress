---
title: "Golang 反射 reflect"
description: 
date: 2020-06-05 00:00:00
categories:
  - Golang 标准库
tags:
  - Golang
series:	
---

反射是程序在运行期间检查其自身结构的一种方式 。

反射三大法则：

- 反射可以将`接口类型变量`转换为`反射类型对象`
- 反射可以将`反射类型对象`转换为`接口类型变量`
- 如果要修改`反射类型对象`，其值必须是`可写的(settable)`

<!--more-->

## Type

`Type` 表示的是对象的具体类型。

从普通类型变量获取反射类型 `Type`：

```go
func TypeOf(i interface{}) Type
```

其他方法：

```go
func ChanOf(dir ChanDir, t Type) Type
func FuncOf(in, out []Type, variadic bool) Type
func MapOf(key, elem Type) Type
func SliceOf(t Type) Type
func ArrayOf(count int, elem Type) Type
func StructOf(fields []StructField) Type
```

`Type` 本身是一个 reflect 库暴露出的接口，定义了许多方法，但根据其本身的具体类型，可以使用的方法并不相同。

### 方法

```go
// 只返回类型名，不含包名
Name() string
// 返回类型名字，为包名 + Name()
String() string
// 返回导入路径，即 import 路径
PkgPath() string
// 返回 rtype.size 即类型大小，单位是字节数
Size() uintptr
// 返回 rtype.kind，描述一种基础类型
Kind() Kind
// 返回类型的位大小，但不是所有类型都能调这个方法，不能调的会 panic
Bits() int
// 检查当前类型能不能做比较运算，主要根据此类型底层有没有绑定 typeAlg 的 equal 方法。
Comparable() bool
// 变量的内存对齐，返回 rtype.align
Align() int
```

### Struct

```go
// 返回 struct 字段数量，不是 struct 会 panic
NumField() int
// 返回 struct 类型的第 i 个字段，不是 struct 会 panic，i 越界也会 panic
Field(i int) StructField
// 返回 struct 类型的字段，是嵌套调用，比如 [1, 2] 就是返回当前 struct 的第 1 个 struct 的第 2 个字段，适用于 struct 本身嵌套的类型
FieldByIndex(index []int) StructField
// 按名字找 struct 字段，第二个返回值 ok 表示有没有
FieldByName(name string) (StructField, bool)
// 按函数签名找 struct 字段，因为 struct 字段也可能是 func 类型
FieldByNameFunc(match func(string) bool) (StructField, bool)
// 根据传入的 i，返回方法实例，表示类型的第 i 个方法
Method(int) Method
// 根据名字返回方法实例，这个比较常用
MethodByName(string) (Method, bool)
// 返回类型方法集中可导出的方法的数量
NumMethod() int
// struct 字段的内存对齐，返回 rtype.fieldAlign
FieldAlign() int
```

### 函数

```go
// 返回函数的参数数量，不是 func 会 panic
NumIn() int
// 返回函数的返回值数量，不是 func 会 panic
NumOut() int
// 返回函数第 i 个参数的类型，不是 func 会 panic
In(i int) Type
// 返回函数第 i 个返回值的类型，不是 func 会 panic
Out(i int) Type
// 返回函数类型的最后一个参数是不是可变数量的，"..." 就这样的，同样，如果不是函数类型，会 panic
IsVariadic() bool
```

### Interface

```go
// 检查当前类型有没有实现接口 u
Implements(u Type) bool
// 检查当前类型能不能赋值给接口 u
AssignableTo(u Type) bool
// 检查当前类型能不能转换成接口 u 类型
ConvertibleTo(u Type) bool
```

### 基础数据类型

```go
// 返回 map 的 key 的类型，不是 map 会 panic
Key() Type
// 返回 array 的长度，不是 array 会 panic
Len() int
// 返回 channel 类型的方向，如果不是 channel，会 panic
ChanDir() ChanDir
// 返回所包含元素的类型，只有 Array, Chan, Map, Ptr, Slice 这些才能调，其他类型会 panic。
Elem() Type
```

## Value

`Value` 表示的是对象的值。

从普通类型变量获取反射值 `Value`：

```go
func ValueOf(i interface{}) Value
```

### 创建 Value

以下这些方法可以创建指定类型的 `Value`：

```go
func MakeChan(typ Type, buffer int) Value
func MakeFunc(typ Type, fn func(args []Value) (results []Value)) Value
func MakeMap(typ Type) Value
func MakeMapWithSize(typ Type, n int) Value
func MakeSlice(typ Type, len, cap int) Value
// 创建值并返回指向它的指针
func New(typ Type) Value
func NewAt(typ Type, p unsafe.Pointer) Value
```

### 操作 Value

```go
// 将值添加到 Slice 中
func Append(s Value, x ...Value) Value
// 将一个 Slice 内的内容添加到 Slice 中
func AppendSlice(s, t Value) Value
// 解引用，获取指针 v 指向的值
func Indirect(v Value) Value
// Select
func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool)
// 返回一个类型的零值，不可更改不可寻址
func Zero(typ Type) Value
```

## Value 方法

`Value` 是一个定义的结构体，其有许多定义好的方法，根据其原始类型的不同，能使用的方法也不同。

```go
type Value struct {
    typ *rtype           // 反射出来此值的类型
    ptr unsafe.Pointer  // 数据形式的指针值
    flag                // 保存元数据
}
```

### 获取具体类型值

使用这些方法可以获取到 `Value` 对应类型的具体的值。

```go
func (v Value) String() string
func (v Value) Int() int64
func (v Value) Uint() uint64 
func (v Value) Float() float64
func (v Value) Bool() bool
func (v Value) Slice(i, j int) Value
func (v Value) Slice3(i, j, k int) Value  // 用 v[i:j:k] 的方式获取切片
func (v Value) Bytes() []byte
func (v Value) Complex() complex128
func (v Value) Interface() (i interface{})
func (v Value) Pointer() uintptr  // 获取指针
func (v Value) UnsafeAddr() uintptr  // 获取 unsafe 包中的指针
```

### 通用方法

这些通用方法基本上是所有类型都可以使用的方法。

```go
// 获取 Value 的 Type
func (v Value) Type() Type  
// 获取 Value 的 Kind
func (v Value) Kind() Kind
// 转换 v 的类型，返回转换以后的值
func (v Value) Convert(t Type) Value
// 判断 v 是否合法，如果返回 false，那么除了 String() 以外的其他方法调用都会 panic，事前检查是必要的
func (v Value) IsValid() bool
// 判断 Value 是否为零值
func (v Value) IsZero() bool
// 获取表示 v 的地址的指针值
func (v Value) Addr() Value
// 判断 v 是否是可寻址的
func (v Value) CanAddr() bool
// 判断 v 的值是否可以被更改
func (v Value) CanSet() bool
// 这几个方法判断值是否超出了它的类型能表示的范围
func (v Value) OverflowInt(x int64) bool
func (v Value) OverflowFloat(x float64) bool
func (v Value) OverflowUint(x uint64) bool
func (v Value) OverflowComplex(x complex128) bool
```

### 更改值

以下这些方法可以更改 `Value` 的值，但必须符合底层类型。

```go
func (v Value) Set(x Value)
func (v Value) SetBool(x bool)
func (v Value) SetBytes(x []byte)
func (v Value) SetCap(n int)
func (v Value) SetComplex(x complex128)
func (v Value) SetFloat(x float64)
func (v Value) SetInt(x int64)
func (v Value) SetLen(n int)
func (v Value) SetMapIndex(key, elem Value)
func (v Value) SetPointer(x unsafe.Pointer)
func (v Value) SetString(x string)
func (v Value) SetUint(x uint64)
```

### Struct

```go
// 获取 struct 字段数量
func (v Value) NumField() int
// 获取 struct 方法数量
func (v Value) NumMethod() int
// 获取 sturct 第 i 个字段，主要用于遍历
func (v Value) Field(i int) Value
// 获取 struct 名为 name 的字段
func (v Value) FieldByName(name string) Value
// 获取 Struct 内嵌套的方法，[]int 为嵌套的路径
func (v Value) FieldByIndex(index []int) Value
// 根据函数签名获取 struct 字段中的函数
func (v Value) FieldByNameFunc(match func(string) bool) Value
// 获取 struct 第 i 个方法
func (v Value) Method(i int) Value
// 获取 struct 名为 name 的方法
func (v Value) MethodByName(name string) Value
```

### 混合类型方法

这些混合类型方式是限定为一部分类型可用的。

```go
// 判断 v 是不是 nil，类型为：Chan, Func, Interface, Map, Pointer, Slice
func (v Value) IsNil() bool
// 获取 v 的 len 值，类型为：Array, Chan, Map, Slice, String.
func (v Value) Len() int
// 返回 v 的 cap 值，类型为：Array, Slice, Chan
func (v Value) Cap() int
// 返回 v 的第 i 个元素，类型为：Array, Slice, String
func (v Value) Index(i int) Value
```

### Channel

```go
// Recv 和 Send 在没有完成操作时会阻塞
func (v Value) Recv() (x Value, ok bool)
func (v Value) Send(x Value)
// TryRecv 和 TrySend 不会阻塞，如果操作没有完成，会返回零值和 false
func (v Value) TryRecv() (x Value, ok bool)
func (v Value) TrySend(x Value) bool
// 关闭通道
func (v Value) Close()
```

### 7. Map

```go
// 用 key 获取 map 的值
func (v Value) MapIndex(key Value) Value
// 获取 map 所有的key
func (v Value) MapKeys() []Value
// 可以用来遍历 map 的方法
func (v Value) MapRange() *MapIter
```

遍历例子：

```go
iter := reflect.ValueOf(m).MapRange()
for iter.Next() {
	k := iter.Key()
	v := iter.Value()
	...
}
```

### Func

```go
// 调用函数 v，并将参数作为 Value 切片传入
func (v Value) Call(in []Value) []Value
// 调用函数 v，并将参数作为 Value 切片传入，CallSlice 会将 in 中最后一个参数作为可变参数传入
func (v Value) CallSlice(in []Value) []Value
```

### Interface

```go
// 返回 v 的接口值或者指针
func (v Value) Elem() Value
func (v Value) CanInterface() bool
func (v Value) InterfaceData() [2]uintptr
```

## Kind

`Kind` 表示的是对象的原生底层类型。有以下这些类型：

```go
const (
    Invalid Kind = iota
    Bool
    Int
    Int8
    Int16
    Int32
    Int64
    Uint
    Uint8
    Uint16
    Uint32
    Uint64
    Uintptr
    Float32
    Float64
    Complex64
    Complex128
    Array
    Chan
    Func
    Interface
    Map
    Ptr
    Slice
    String
    Struct
    UnsafePointer
)
```

还有一个 `func (k Kind) String() string` 方法可以返回 Kind 的名称。

## StructField

此结构体用来表示 Struct 的字段：

```go
type StructField struct {
    Name      string    // 字段名
    PkgPath   string    // PkgPath 是未导出字段的程序包路径，可导出字段此值为空
    Type      Type      // 字段类型
    Tag       StructTag // 字段的 Tag
    Offset    uintptr   // 在 struct 中的偏移量，单位 bytes
    Index     []int     // 字段在 struct 中的数字索引
    Anonymous bool      // 是否是一个嵌入式字段
}
```

## StructTag

此类型用来表示 Struct 字段的 Tag 字符串，其有两个方法可以获取 Tag 中设定的值。

```go
func (tag StructTag) Get(key string) string
func (tag StructTag) Lookup(key string) (value string, ok bool)
```

- `Get` 在没有这个字段和字段为空字符时，都会返回空字符。

- 所以如果需要检查是否设置了这个 Tag，需要使用 `Lookup`。例子：

  ```go
  type S struct {
      F0 string `alias:"field_0"`
      F1 string `alias:""`
      F2 string
  }
  
  s := S{}
  st := reflect.TypeOf(s)
  for i := 0; i < st.NumField(); i++ {
      field := st.Field(i)
      if alias, ok := field.Tag.Lookup("alias"); ok {
          if alias == "" {
              fmt.Println("(blank)")
          } else {
              fmt.Println(alias)
          }
      } else {
          fmt.Println("(not specified)")
      }
  }
  ```

  