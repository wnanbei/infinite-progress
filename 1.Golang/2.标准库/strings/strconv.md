## strconv

该包包含了一些变量用于获取程序运行的操作系统平台下 int 类型所占的位数，如：`strconv.IntSize`。

任何类型 **T** 转换为字符串总是成功的。

### 1. Int

```go
// int 转 string
strconv.Itoa(i int) string
// string 转 int
strconv.Atoi(s string) (i int, err error)
```

### 2. Float

```go
// float 转 string
strconv.FormatFloat(f float64, fmt byte, prec int, bitSize int) string
// string 转 float
strconv.ParseFloat(s string, bitSize int) (f float64, err error)
```

- `fmt` - 数据格式
- `prec` - 数据精度
- `bitSize` - 浮点型大小，32 表示 float32，64 表示 float64

