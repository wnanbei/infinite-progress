---
title: "Golang 原子操作 atomic"
description: 
date: 2021-05-05 00:00:00
categories:
  - Golang 标准库
tags:
  - Golang
  - Concurrency
  - sync
series:	
  - Golang 面试大全
---

`atomic` 包封装了系统底层的原子操作。官方建议尽量少使用此包的原子操作，尽量遵循通过通信分享内存，而不是通过分享内存来通信的原则。

这个包的方法有以下特点：

- 方法操作的都是 `int` 系列类型或指针。
- 操作的数据需要其地址。

<!--more-->

## 用法

### 加法

原子性的将 `delta` 与 `addr` 相加，并返回新值。

```go
func AddInt32(addr *int32, delta int32) (new int32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
```

使用 `int` 时传入负数就意味着减法，但 `uint` 类型限制了数据的符号，所以如果要减法需要利用二进制补码机制：

```go
AddUint32(&x, ^uint32(c-1))
AddUint64(&x, ^uint64(c-1))
```

每次递减 `1` 可以这样：

```go
AddUint32(&x, ^uint32(0))
AddUint64(&x, ^uint64(0))
```

### 读取

原子性的读取 `addr` 的值。

```go
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)
func LoadUintptr(addr *uintptr) (val uintptr)
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
```

### 存储

原子性的将值 `val` 存储到 `addr`。

```go
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)
func StoreUintptr(addr *uintptr, val uintptr)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
```

### 交换

原子性的将值 `new` 交换给 `addr`，并返回旧值。

```go
func SwapInt32(addr *int32, new int32) (old int32)
func SwapInt64(addr *int64, new int64) (old int64)
func SwapUint32(addr *uint32, new uint32) (old uint32)
func SwapUint64(addr *uint64, new uint64) (old uint64)
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
```

先比较再交换，原子性的先进行比较，如果 `addr` 与 `old` 值相同，则将 `new` 交换给 `addr`，返回的 `swapped` 表示是否进行了交换。

```go
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
```

### Value

`Value` 是 `atomic` 包中用来存储任意类型值的容器。主要有两个方法：

```go
func (v *Value) Load() (x interface{})  // 获取 v 的值
func (v *Value) Store(x interface{})  // 存储 v 的值
```

例1，此例用于一个经常读取，但很少写入的数据结构：

```go
type Map map[string]string
var m atomic.Value
m.Store(make(Map))
var mu sync.Mutex // 仅仅用于写入

// 不需要进一步同步的读取函数
func read(key string) (val string) {
    m1 := m.Load().(Map)
    return m1[key]
}

// 不需要进一步同步的写入函数
func insert(key, val string) {
    mu.Lock() // 保证与其他潜在写入者同步
    defer mu.Unlock()
    m1 := m.Load().(Map) // 读取数据
    m2 := make(Map)      // 创建一个新 map
    for k, v := range m1 {
        m2[k] = v // 复制所有数据到新的 map 中
    }
    m2[key] = val // 写入更新数据
    m.Store(m2)   // 原子性的将这个对象替换为更新以后的 map
    // 这一点开始后所有新的读取者都会读取新版的数据
    // 旧版本将在读取者（如果存在）读取完毕后被垃圾回收
}
```

例2，此例用于周期性的更新数据，并传播给使用者：

```go
func loadConfig() map[string]string {  // 加载 config 数据
	return make(map[string]string)
}

func requests() chan int {  // 进来的请求
	return make(chan int)
}

func main() {
	var config atomic.Value
	config.Store(loadConfig())  // 存储初始 config 数据
	go func() {
		// 每十秒加载一次 config 数据
		for {
			time.Sleep(10 * time.Second)
			config.Store(loadConfig())
		}
	}()
	// 使用最新的 config 数据处理进来的请求
	for i := 0; i < 10; i++ {
		go func() {
			for r := range requests() {
				c := config.Load()
				// 用 c 处理请求
				_, _ = r, c
			}
		}()
	}
}
```

