---
title: "Go GMP 调度器"
description: 
date: 2021-08-01 00:00:00
categories:
  - Go设计与原理
tags:
  - Go
  - Concurrency
series:	
  - Go 面试大全
typora-root-url: ..\..\..\..\static
---



GMP Scheduler 是 Runtime 中几乎最重要的组件，它的作用是：

> For scheduling goroutines onto kernel threads.

GMP Scheduler 的核心思想是：

1. 重用线程。
2. 限制同时运行（不包含阻塞）的线程数为 N，N 为 CPU 逻辑核心数。

Go scheduler 的职责就是将所有处于 runnable 的 Goroutines 均匀分布到在 P 上运行的 M，利用多核并行，实现更强大的并发。

<!--more-->

![](/images/go/gmp.png)

## GMP 数据结构

### G

`Goroutine` 用户级别的协程，我们在程序里用 `go` 关键字创建的协程。

| 状态       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| _Gidle     | 空闲中，刚刚被分配并且还没有被初始化，值为 0，为创建 Goroutine 后的默认值 |
| _Grunnable | 待运行，G 在运行队列中, 等待 M 取出并运行，没有栈的所有权    |
| _Grunning  | 运行中，正在执行代码的 Goroutine，拥有栈的所有权             |
| _Gsyscall  | 系统调用中，正在执行系统调用，拥有栈的所有权，与 P 脱离，但是与某个 M 绑定，会在调用结束后被分配到运行队列 |
| _Gwaiting  | 等待中，此时为被阻塞的 G，阻塞在某个 channel 的发送或者接收队列 |
| _Gdead     | 已中止，当前 G 未被使用，没有执行代码，可能有分配的栈，分布在空闲列表 gFree，可能是一个刚初始化的 G，也可能是执行了 goexit 退出的 G |
| _Gcopystac | 栈复制中，栈正在被拷贝，没有执行代码，不在运行队列上         |
| _Gscan     | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

### M

`Machine` 指 Go 语言对一个关联的内核线程的封装。

M 并没有像 G 和 P 一样的状态标记, 但可以认为一个 M 有以下的状态

| 状态            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| spinning 自旋中 | M 正在从运行队列获取 G, 这时候 M 拥有一个 P                  |
| 执行 Go 代码中  | M 正在执行 Go 代码, 这时候 M 拥有一个P                       |
| 执行原生代码中  | M 正在执行原生代码或者阻塞的 syscall, 这时 M 不拥有 P        |
| 休眠中          | M 发现无待运行的 G 时会进入休眠, 并添加到空闲 M 链表中, 这时 M 不拥有 P |

### P

`Processor` 包含了 Goroutine 运行的资源，M 必须和一个 P 关联才能运行 G。

P 还包含自己的本地队列 `local runqueue` 来保存 G，这样可以避免竞争锁。

| 状态      | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| _Pidle    | 空闲中，当 M 发现无 runnable 的 G 时会进入休眠, 这时 M 拥有的 P 会变为空闲并放到空闲 P 链表中 |
| _Prunning | 运行中，当 M 拥有了一个 P 后, 这个 P 的状态就会变为运行中, M 会使用这个 P 中的资源运行 G |
| _Psyscall | 系统调用中，当 Go 调用原生代码, 原生代码又反过来调用 Go 代码时, 使用的 P 会变为此状态 |
| _Pgcstop  | GC 停止中，当 GC 停止了整个世界(STW)时, P 会变为此状态       |
| _Pdead    | 已中止，当 P 的数量在运行时改变, 且数量减少时多余的 P 会变为此状态 |

### P 和 M 的数量

P 的数量：

1. 由启动时环境变量 `$GOMAXPROCS` 或者是由 `runtime` 的方法 `GOMAXPROCS()` 决定。

   这意味着在程序执行的任意时刻都只有 `$GOMAXPROCS` 个goroutine在同时运行。

M 的数量：

1. GO 程序启动时，会设置 M 的最大数量，默认 10000。但是内核很难支持这么多的线程数，所以这个限制可以忽略。
2. `runtime/debug` 中的 `SetMaxThreads` 函数，设置 M 的最大数量。
3. 一个 M 阻塞了，会创建新的 M。

M 与 P 的数量没有绝对关系，一个 M 阻塞，P 就会去创建或者切换另一个 M，所以即使 P 的默认数量是 1，也有可能会创建很多个 M 出来。

### P 和 M 何时被创建

1. P：在确定了 P 的最大数量 n 后，运行时系统会根据这个数量创建 n 个 P。

2. M：没有足够的 M 来关联 P 并运行其中的可运行的 G。

   比如所有的 M 此时都阻塞住了，而 P 中还有很多就绪任务，就会去寻找空闲的 M，而没有空闲的，就会去创建新的 M。

### GRQ

`Global Runnable Queue` 全局队列。存放等待运行的 G。

- 此队列优先度较低。

- GRQ 入队和出队需要使用线程锁。

- 全局运行队列的数据结构是链表, 由两个指针 `head`, `tail` 组成。

### LRQ

`Local Runnable Queue` P 的本地队列。

- 同 GRQ 类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。

- 新建 G 时，G 优先加入到 P 的 LRQ，如果队列满了，则会把 LRQ 中随机一半的 G 移动到 GRQ。
- LRQ 入队和出队不需要使用线程锁。
- LRQ 的数据结构是环形队列, 由一个 256 长度的数组和两个序号 `head`, `tail` 组成。

## 概念

### Goroutine 和 Thread 区别

1. 内存占用

   - 创建一个 Goroutine 的栈内存消耗为 2 KB，如果栈空间不够用，会自动进行扩容。
   - 创建一个 Thread 需要消耗 1 MB 栈内存，还需要一个 `a guard page` 的区域用于和其他 Thread 的栈空间进行隔离。

2. 创建和销毁

   - Goroutine 由 `Go runtime` 负责管理，创建和销毁的消耗非常小，是用户级。
   - Thread 由于是内核级的，创建和销毀都会有巨大的消耗，通常解决的办法是线程池。

3. 切换

   - Goroutines 切换只需保存三个寄存器：`Program Counter`, `Stack Pointer` and `BP`。

     Goroutine 的切换约为 200 ns，相当于 2400-3600 条指令。

   - Threads 切换时，需要保存各种寄存器，以便将来恢复：

     > 16 general purpose registers, PC (Program Counter), SP (Stack Pointer), segment registers, 16 XMM registers, FP coprocessor state, 16 AVX registers, all MSRs etc.

     一般而言，线程切换会消耗 1000-1500 纳秒，执行指令的条数会减少 12000-18000。

### M:N 模型

`Runtime` 在程序启动的时候，会创建 M 个系统线程，之后创建的 N 个 `Goroutine` 都会在这 M 个线程上执行。这就是 `M:N` 模型。

- 同一时刻，一个线程上只能跑一个 Goroutine。

- `sysmon` 会检测长时间（超过 10 ms）运行的 Goroutine，将其调度到 `global runqueues`，让其他 Goroutine 执行。

  `global runqueues` 是一个全局的 runqueue，优先级比较低，以示惩罚。

### Work Stealing

当一个 G 执行结束，P 会去 LRQ 获取下一个 G 来执行。

如果 LRQ 已经空了，这时，P 会随机选择一个 P 的 LRQ “偷”过来一半的 G。

这样就有更多的 P 可以一起工作，加速执行完所有的 G。

## 调度流程

![Go调度器生命周期](/images/go/schedule.png)

### M0

`M0` 是启动程序后的编号为 0 的主线程。

- 这个 M 对应的实例会在全局变量 `runtime.m0` 中，不需要在 heap 上分配
- M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。

### G0

`G0` 是每次启动一个 M 都会第一个创建的 gourtine。

- G0 是仅用于负责调度的 G，不指向任何可执行的函数。
- 每个 M 都会有一个自己的 G0。
- 在调度或系统调用时会使用 G0 的栈空间, 全局变量的 G0 是 M0 的 G0。

### 基于信号的抢占式调度

