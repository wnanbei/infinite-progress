---
title: "Go 令牌桶限流器 golang.org/x/time/rate"
description: 
date: 2021-10-25 00:00:00
categories:
  - Go标准库
tags:
  - Go
  - Concurrency
series:	
  - Go 面试大全
---

`golang.org/x/time/rate` 提供了一个使用令牌桶 `Token Bucket` 算法实现的限流器。

<!--more-->

## 用法

### 创建限流器

```go
func NewLimiter(r Limit, b int) *Limiter
```

- r - 每秒令牌桶中产生的 Token，为 0 则不产生 Token。
- b - 令牌桶的最大容量。

`r` 的值类型为 `float64`，如果要设置超过秒以上的频率，可以使用 `Every()` 方法：

```go
limit := Every(100 * time.Second)
limiter := NewLimiter(limit, 1)
```

### 消费 Token

`golang.org/x/time/rate` 提供了三种方式消费 Token，这三种方式在令牌桶内令牌不足时，有不同的处理方式。

#### Allow/AllowN

```go
func (lim *Limiter) Allow() bool
func (lim *Limiter) AllowN(now time.Time, n int) bool
```

在某一时刻，如果桶中 Token 数量大于等于 n，则消费 n 个 Token 并返回 true。

如果不满足则不消费，返回 false。

`Allow` 等价于 `AllowN(time.Now(), 1)`。

**在超出频率限制时，希望丢弃或跳过事件的时候使用此方法。**

#### Wait/WaitN

```go
func (lim *Limiter) Wait(ctx context.Context) (err error)
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error)
```

从令牌桶中消费 n 个 Token，如果令牌桶中 Token 数量不足，那么将会阻塞，直到 Token 数量满足。

如果 Token 数量满足则直接返回。

Wait 方法可以使用 context 来控制超时时间。

`Wait` 等价于 `WaitN(ctx, 1)`。

**在超出频率时，如果希望有一个最长等待时间的，使用此方法。**

#### Reserve/ReserveN

```go
func (lim *Limiter) Reserve() *Reservation
func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation
```

调用 `Reserve` 方法后，无论 Token 是否充足，都会消费 N 个令牌并返回一个 `Reservation` 对象。

但是此时并不一定允许你执行相应逻辑，如果桶内 Token 不足，需要 `Delay()` 延迟一定时间执行。

示例：

```go
r := lim.ReserveN(time.Now(), 1)
if !r.OK() {
  // Not allowed to act! Did you remember to set lim.burst to be > 0 ?
  return
}
time.Sleep(r.Delay())
Act()
```

Reservation 对象拥有的方法：

```go
func (r *Reservation) Cancel() // 取消消费，并尝试返还 Token
func (r *Reservation) CancelAt(now time.Time)
func (r *Reservation) Delay() time.Duration // 返回需要延迟执行的时间
func (r *Reservation) DelayFrom(now time.Time) time.Duration
func (r *Reservation) OK() bool // 判断令牌桶是否能提供请求的令牌数
```

**在超出频率限制时，如果希望始终执行事件的，使用此方法。**

### 动态调整速率

获取限制频率和桶容量大小：

```go
func (lim *Limiter) Burst() int
func (lim *Limiter) Limit() Limit
```

调整桶的容量大小：

```go
func (lim *Limiter) SetBurst(newBurst int)
func (lim *Limiter) SetBurstAt(now time.Time, newBurst int)
```

调整限制频率：

```go
func (lim *Limiter) SetLimit(newLimit Limit)
func (lim *Limiter) SetLimitAt(now time.Time, newLimit Limit)
```

