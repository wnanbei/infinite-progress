---
title: "Go 并发安全的 sync.Map"
date: 2021-05-05 00:00:00
categories:
  - Go标准库
tags:
  - Go
  - Concurrency
  - sync
series:	
  - Go 面试大全
---

`sync.Map` 是标准库 `sync` 中实现的并发安全的 map。

<!--more-->

## 用法

### 使用方式

| 操作             | 普通 map       | sync.Map          |
| :--------------- | :------------- | :---------------- |
| map 获取某个 key | map[1]         | sync.Load(1)      |
| map 添加元素     | map[1] = 10    | sync.Store(1, 10) |
| map 删除一个 key | delete(map, 1) | sync.Delete(1)    |
| 遍历 map         | for...range    | sync.Range()      |

sync.Map 两个特有的函数:

- `LoadOrStore` - sync.Map 存在就返回，不存在就插入
- `LoadAndDelet` - sync.Map 获取某个 key，如果存在的话，同时删除这个 key

**例子：**

```go
var syncMap sync.Map
syncMap.Store("11", 11)
syncMap.Store("22", 22)

fmt.Println(syncMap.Load("11")) // 11
fmt.Println(syncMap.Load("33")) // 空

fmt.Println(syncMap.LoadOrStore("33", 33)) // 33
fmt.Println(syncMap.Load("33")) // 33
fmt.Println(syncMap.LoadAndDelete("33")) // 33
fmt.Println(syncMap.Load("33")) // 空

syncMap.Range(func(key, value interface{}) bool {
  fmt.Printf("key:%v value:%v\n", key, value)
  return true
})
```

### 数据结构

```go
type Map struct {
 mu Mutex
 read atomic.Value // readOnly  read map
 dirty map[interface{}]*entry  // dirty map
 misses int
}
```

**`read`**

是 `atomic.Value` 类型，主要负责并发读取。使用 lock free 的方式保证 load/store 的原子性。

- 如果需要更新 `read`，则需要加锁保护。对于 read 中存储的 entry 字段，可能会被并发地 CAS 更新。
- 如果要更新一个之前已被删除的 entry，则需要先将其状态从 expunged 改为 nil，再拷贝到 dirty 中，然后再更新。

**`dirty`**

是一个非线程安全的原始 map。使用 `mutex` 保证并发安全。

dirty 包含新写入的 key，并且包含 `read` 中的所有未被删除的 key。这样，可以快速地将 `dirty` 提升为 `read` 对外提供服务。

如果 `dirty` 为 nil，那么下一次写入时，会新建一个新的 `dirty`，这个初始的 `dirty` 是 `read` 的一个拷贝，但除掉了其中已被删除的 key。

- 当 dirty 为 nil 的时候，read 就代表 map 所有的数据。
- 当 dirty 不为 nil 的时候，dirty 代表 map 所有的数据。

**`misses`**

用于记录未命中 read 缓存的次数。

- 每次在 read 中没找到数据，而在 dirty 中找到，则这个数字加 1。
- 当 misses 大于 dirty 的数量时，会将 dirty 的数据整体复制到 read，并清空 dirty，此操作时间复杂度为 O(N)。

### 使用场景

sync.Map 里面有两个普通 map，read map 主要负责读，dirty map 负责读和写（加锁）。

在读多写少的场景下，read map 的值基本不发生变化，可以让 read map 做到无锁操作，就减少了使用 `Mutex + Map` 必须的加锁/解锁环节，因此也就提高了性能。

如果某些 key 写操作特别频繁，sync.Map 基本就退化成了 `Mutex + Map`，甚至有可能性能不如 Mutex + Map。

所以 sync.Map 适用于以下场景：

- 读多写少
- 写操作多，但是修改的 key 和读取的 key 特别不重合。

### 执行流程

**sync.Map.Load() 取出对象流程：**

![syncMapLoad](../../../assets/go/syncMapLoad.webp)

**sync.Map.Store() 插入对象流程：**

![syncMapStore](../../../assets/go/syncMapStore.webp)

**sync.Map.LoadAndDelete() 删除对象流程：**

![syncMapDelete](../../../assets/go/syncMapDelete.webp)

