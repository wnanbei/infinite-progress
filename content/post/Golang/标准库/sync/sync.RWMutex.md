---
title: "Golang 读写锁 sync.RWMutex"
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

`sync.RWMutex` 是一个读写锁，在读多写少的场景中，比 Mutex 的并发能力有很大的提升。

<!--more-->

## 用法

### 使用方式

读写锁的读锁与写锁、写锁与写锁互斥，读锁与读锁之间互不影响。

```go
func (rw *RWMutex) Lock         // 写锁加锁
func (rw *RWMutex) Unlock       // 写锁解锁
func (rw *RWMutex) RLock        // 读锁加锁
func (rw *RWMutex) RUnlock      // 读锁解锁
```

### 数据结构

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```

**`w`** - 用互斥锁解决多个 writer 的竞争。

- 当有 G 获取写锁后，会阻塞其他 G 的写操作。

**`writerSem`** - 写操作的信号量。

**`readerSem`** - 读操作的信号量。

**`readerCount`** - 当前读操作的数量，以及是否有写操作在等待。

- 每一次获取读锁，都会将此数量加 1，如果此数量为负数，说明有 G 获取了写锁，当前 G 会陷入休眠等锁释放。
- 每一次释放读锁，都会将此数量减 1。
- 获取写锁时，会阻塞后续的读操作，并休眠等待当前正在进行的读操作执行完毕。
- 释放写锁时，会将此数量变回正数，释放读锁。

**`readerWait`** - 写操作请求锁时，需要等待完成的读操作数量。

### 总结

读写互斥锁在互斥锁之上提供了额外的更细粒度的控制，能够在读操作远远多于写操作时提升性能。

