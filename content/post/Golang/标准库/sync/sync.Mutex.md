---
title: "Go 互斥锁 sync.Mutex"
description: 
date: 2021-05-05 00:00:00
categories:
  - Go标准库
tags:
  - Go
  - Concurrency
  - sync
series:	
  - Go 面试大全
typora-root-url: ..\..\..\..\..\static
---

`sync.Mutex` 是一个互斥锁，默认为零值时为开锁状态。

<!--more-->

## 用法

### 使用方式

Lock 方法锁住 m，如果 m 已经加锁，则阻塞直到 m 解锁。

```go
func (m *Mutex) Lock()
```

Unlock 方法解锁 m，如果 m 未加锁会导致运行时错误。

```go
func (m *Mutex) Unlock()
```

### 数据结构

```go
type Mutex struct {
    state int32
    sema  uint32
}
const (
   mutexLocked = 1 << iota // mutex is locked
   mutexWoken
   mutexStarving
   mutexWaiterShift = iota
)
```

- **`state`**

  是一个公用字段，共 32 位。其中低三位分别表示：

  - Mutex 是否已被加锁
  - 是否有某个唤醒的 G 要尝试获取锁
  - Mutex 是否处于饥饿状态

  高 29 位则表示等待锁的 G 数量。

- **`sema`**

  sema 是一个信号量，用来实现阻塞/唤醒申请锁的 G。

### 执行流程

**Mutex Lock 上锁流程：**

- 非饥饿模式下，新获取锁的 G 将会进入自旋，去竞争锁。为了避免自旋消耗太多 cpu，G 最多会自旋 4 次,每次空转 30 个 cpu 时钟周期；

![syncMutexLock](/images/go/syncMutexLock.webp)

**Mutex UnLock 解锁流程：**

![syncMutexUnlock](/images/go/syncMutexUnlock.webp)

### 饥饿状态

互斥锁有两种状态：正常状态和饥饿状态。

**正常状态：**

所有等待锁的 G 按照 FIFO 顺序等待。

- 刚唤醒的 G 不会直接拥有锁，而是会和新请求锁的 G 去竞争锁；
- 新请求锁的 G 具有一个优势：它正在 CPU 上执行；
- 可能有好几个 G 同时在新请求锁，所以刚唤醒的 G 有很大可能在锁竞争中失败；
- 在这种情况下，这个被唤醒的 G 在没有获得锁之后会加入到等待队列的最前面。

**饥饿状态：**

如果一个等待的 G 超过 `1ms` 没有获取锁，那么它将会把锁转变为饥饿模式。

- 饥饿模式下，锁的所有权将从执行 unlock 的 G 直接交给等待队列中的第一个 G;
- 新来的 G 将不能再去尝试竞争锁，即使锁是 unlock 状态，也不会去尝试自旋操作，而是放在等待队列的尾部;
- 如果一个等待的 G 获取了锁，并且满足以下其中一个条件,那么该 G 会将锁的状态转换为正常状态:
  1. 它是队列中的最后一个 G；
  2. 它等待的时间小于 1ms；

**总结：**

正常模式具有较好的性能，因为 G 可以连续多次尝试获取锁，即使还有其他的阻塞等待锁的 G，也不需要进入休眠阻塞。

饥饿模式的作用是阻止尾部延迟的现象。

### 总结

1. Mutex 不可被复制；
2. 就算在较低 QPS 下，Mutex 的锁竞争也会比较激烈。如果一定要使用 Mutex，一定要采用取模分片的方式去使用其中一个 Mutex 进行资源控制，降低锁粒度；
3. 不同 G 可以 Unlock 同一个 Mutex，但是 Unlock 一个无锁状态的 Mutex 会报错；
4. Mutex 不是可重入锁，如果连续两次 Lock 操作，会直接死锁。

