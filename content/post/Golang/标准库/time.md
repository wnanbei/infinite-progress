---
title: "Go 时间处理库 time"
description: 
date: 2020-06-05 00:00:00
categories:
  - Go标准库
tags:
  - Go
series:	
---

time 是 Go 用于处理时间的标准库，包括格式化、计算、修改、定时、超时等功能。

<!--more-->

## 格式

### 时间占位符

常用：`2006-01-02 15:04:05`

- 年份：06，2006
- 月份：01，Jan，January
- 星期：Monday，Mon
- 日期：02，2，_2
- 小时：15
- 小时（12时制）：3，03
- 分钟：04，4
- 秒钟：05，5
- 上下午标志：PM，pm
- 时区：MST
- 时区偏移：-700

### 常量

```go
const (
    ANSIC       = "Mon Jan _2 15:04:05 2006"
    UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
    RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
    RFC822      = "02 Jan 06 15:04 MST"
    RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
    RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
    RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
    RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
    RFC3339     = "2006-01-02T15:04:05Z07:00"
    RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
    Kitchen     = "3:04PM"
    // Handy time stamps.
    Stamp      = "Jan _2 15:04:05"
    StampMilli = "Jan _2 15:04:05.000"
    StampMicro = "Jan _2 15:04:05.000000"
    StampNano  = "Jan _2 15:04:05.000000000"
)
```

## Time 时间点

### 初始化

当前时间

```go
func Now() Time
```

解析时间

```go
func Parse(layout, value string) (Time, error)
time.Parse("2016-01-02 15:04:05", "2018-04-23 12:24:51")
```

解析指定时区时间

```go
func ParseInLocation(layout, value string, loc *Location) (Time, error)
time.ParseInLocation("2006-01-02 15:04:05", "2017-05-11 14:06:06", time.Local)
```

生成指定时间

```go
func Date(year int, month Month, day, hour, min, sec, nsec int, loc *Location) Time
time.Date(2018, 1, 2, 15, 30, 10, 0, time.Local)
```

通过时间戳生成时间

```go
func Unix(sec int64, nsec int64) Time
time.Unix(1571818205, 67868768768)
```

### Time 方法

休眠，暂停当前协程

```go
func Sleep(d Duration)
```

格式化时间点

```go
func (t Time) Format(layout string) string
```

返回时间点字符串格式

```go
func (t Time) String() string
// "2006-01-02 15:04:05.999999999 -0700 MST"
```

获取时间戳

```go
// 获取10位长度时间戳
func (t Time) Unix() int64  
// 获取纳秒单位时间戳
func (t Time) UnixNano() int64 
```

#### 具体时间数据

```go
// 获取年份
func (t Time) Year() int  
// 获取月份
func (t Time) Month() Month  
// 获取星期
func (t Time) Weekday() Weekday  
// 获取日期
func (t Time) Day() int  
// 获取小时
func (t Time) Hour() int  
// 获取分钟
func (t Time) Minute() int  
// 获取秒钟
func (t Time) Second() int  
// 获取纳秒
func (t Time) Nanosecond() int  
// 获取时分秒
func (t Time) Clock() (hour, min, sec int)  
//获取年月日
func (t Time) Date() (year int, month Month, day int)  
// 获取时间点在一年中的天数
func (t Time) YearDay() int  
// 获取时间点年份和周数，周数范围为1-53
func (t Time) ISOWeek() (year, week int)  
```

#### 判断

```go
// 判断时间是否在此之后
func (t Time) After(u Time) bool  
// 判断时间是否在此之前
func (t Time) Before(u Time) bool  
// 判断时间是否相等
func (t Time) Equal(u Time) bool  
// 判断是否为0
func (t Time) IsZero() bool  
```

## Duartion 时间段

### 常量

```go
type Duration int64
```

时分秒

```go
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```

月份

```go
type Month int

const (
    January Month = 1 + iota
    February
    March
    April
    May
    June
    July
    August
    September
    October
    November
    December
)
```

星期

```go
type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```

### 方法

解析字符串生成时间段

```go
func ParseDuration(s string) (Duration, error)
// 可用单位"ns", "us" (or "µs"), "ms", "s", "m", "h".
```

获取特定单位的时间段

```go
func (d Duration) Hours() float64
func (d Duration) Minutes() float64
func (d Duration) Seconds() float64
func (d Duration) Milliseconds() int64  // 毫秒
func (d Duration) Microseconds() int64  // 微秒
func (d Duration) Nanoseconds() int64  // 纳秒
```

## 计算修改时间

### 修改时间

```go
// 加上时间
func (t Time) Add(d Duration) Time
// 加上日期
func (t Time) AddDate(years int, months int, days int) Time
```

### 计算时间

```go
// 减去时间
func (t Time) Sub(u Time) Duration
// 到当前时间过去了多久时间，等价 time.Now().Sub(t)
func Since(t Time) Duration
// 到指定时间还有多久，等价 t.Sub(time.Now())
func Until(t Time) Duration
```

### 时间点取整

```go
func (t Time) Round(d Duration) Time
func (t Time) Truncate(d Duration) Time
```

## 时区

```go
// 获取时间点的时区
func (t Time) Location() *Location  
// 获取时区名和偏移量
func (t Time) Zone() (name string, offset int)  
// 获取时间点在当前时区的副本
func (t Time) Local() Time  
// 获取时间点在UTC时区的副本
func (t Time) UTC() Time  
// 获取时间点在指定时区的副本
func (t Time) In(loc *Location) Time  
```

## 定时与超时

### Timer

此函数等待指定时间后，然后在返回的通道中发送当前时间。

```go
func After(d Duration) <-chan Time
```

可以与 select 配合实现超时功能：

```go
var c chan int

func handle(int) {}

func main() {
	select {
	case m := <-c:
		handle(m)
	case <-time.After(10 * time.Second):
		fmt.Println("timed out")
	}
}
```

Timer 方法：

```go
// 创建 Timer
func NewTimer(d Duration) *Timer
// 指定时间后调用 f 函数，可以使用返回的 Timer 取消
func AfterFunc(d Duration, f func()) *Timer
// 重置计时时间
func (t *Timer) Reset(d Duration) bool
// 停止 Timer
func (t *Timer) Stop() bool
```

### Ticker

此函数是 Ticker 的简单封装，可以返回一个通道，此通道间隔指定时间发送当前时间，注意的是，由于无法关闭底层 Ticker，此通道是默认泄露的。

```go
func Tick(d Duration) <-chan Time
```

实现定时效果

```go
func statusUpdate() string { return "" }

func main() {
	c := time.Tick(5 * time.Second)
	for now := range c {
		fmt.Printf("%v %s\n", now, statusUpdate())
	}
}
```

Ticker 方法

```go
// 创建 Ticker
func NewTicker(d Duration) *Ticker
// 停止 Ticker
func (t *Ticker) Stop()
```

