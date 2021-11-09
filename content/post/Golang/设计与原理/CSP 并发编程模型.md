---
title: "CSP 并发编程模型"
description: 
date: 2021-08-05 00:00:00
categories:
  - Go设计与原理
tags:
  - Go
  - Concurrency
series:	
  - Go 面试大全
---

`CSP - Communicating Sequential Process`，通信顺序进程，是一种并发编程模型，用于描述两个独立的并发实体通过共享的通讯 channel 进行通信。

<!--more-->

CSP 中 channel 是第一类对象，它不关注发送消息的实体，而关注发送消息时使用的 channel。



***

并发编程。

1. 并发程序经常出错的一个原因是人们认为自己所写代码的执行顺序是按书写的顺序来执行的，但在并发场景下，这显然是有问题的。
2. Atomicity，原子性。谈论原子性，必须要有一个 context。因为在一个 context 下是原子性的，但在另一个 context 下，就可能不是原子性的了。具体的 context 可能是：进程、操作系统、机器、集群……假想个例子，在一维空间中的 X 轴上，从坐标 1 到坐标 3 必须要经过坐标 2，这在一维空间中是绝对正确的。但作为活在三维空间里的人，我有很多种办法不经过 X 轴上的坐标 2 而到达坐标 3。仅管我的轨迹映射到 X 轴上还是会“经过”坐标 2，这也更像一个“降维打击”的例子。
3. 形成死锁的四个条件：Mutual Exclusion（并发实体任意时刻独占资源）、Wait For Condition（并发实体同时持有资源并都在等待其他资源）、No Preemption（资源只能被持有它的实体释放）、Circular Wait（循环等待，a 等 b，b 等 c，c 等 a……）。
4. 活锁是饥饿的一种，任何需要分享的资源都有可能发生饥饿，如 CPU、内存、文件句柄、数据库连接等。
5. 并发（Concurrency）说的是代码，并行（Parallelism）说的是正在运行的程序。我们无法写出并行的代码，只能写并发的代码，并且期望它能并行执行。想象一下，我们写的代码在单核 CPU 上运行，还能并行地起来吗？
6. 考察并发的代码是否是在并行执行，我们得看在哪一个抽象的层级上看：并发原语、程序的运行时、操作系统、操作系统所在的平台（容器、虚拟机……）、CPUs、机器、集群……
7. 和前面说的 Atomicity 一样，谈论 Parallelism 时，也要有一个 context。它决定是否将能将两个操作看成并行。例如，我们运行 2 个操作，每个操作花费 1 秒。如果 context 是 5 秒钟，那可以说这两个操作是在并行执行；但如果 context 是 1 秒钟，那我们认为，这两个操作是串行地在执行。注意，context 并不等同于时间，线程、进程、操作系统等都可以看成 context。
8. 给并发或者说并行定义什么样的 context 和并发程序是否正确运行有很大关系。例如，context 是两台电脑，我们分别在两台电脑上运行两个计算器程序，那理论上这两个计算器程序就是并行的，且不会相互影响。
9. 在上面的例子里，context 是两台电脑，operations 是两个进程。很明显，我在我的电脑上运行任何程序，都不会影响你的电脑。但是在同一台机器上，一个进程还能保证不影响另一个进程吗？回答是不一定，比如读写同一个文件……
10. 大部分程序的并发抽象层级是线程。Go 在抽象层级上又增加了一个 goroutine。按理说，层级层次越高，并发安全性越难保证。但实际上 goroutine 让事情变得更容易，因为它并不是在线程的抽象层级之上又加了一层，而是取代了线程。
11. Go channel 的设计思想来源于 Hoare 于 1978 年发表在 ACM 上的一篇关于 CSP（Communicating Sequential Processes）的论文。Go 是第一门吸收了 CSP 精华并且将其发扬光大的语言。
12. 大多数语言使用线程+并发同步访问控制作为并发模型，而 Go 的并发模型由 goroutine 和 channel 组成。线程类似于 goroutine，而并发同步访问控制则类似于 mutex。
13. Go 并发的理念是：简单，尽量使用 channel，尽情使用 goroutine。
14. 在 linux 上，简单测试线程切换成本：

```javascript
# 在 CPU0 上执行，在两个内核线程间发送、接收消息
taskset -c 0 perf bench sched pipe -T
```

因为是单核，所以在两个线程间发送、接收消息，需要进行上下文切换。在我的乞丐版阿里云主机上得到结果：

```javascript
# Running 'sched/pipe' benchmark:
# Executed 1000000 pipe operations between two threads

     Total time: 69.171 [sec]

      69.171280 usecs/op
          14456 ops/sec
```

计算出大致的线程切换成本：69.171280/2 = 34.58564 us。

1. 使用 sync.WaitGroup 时要注意，sync.Add 要在新起 goroutine 语句的外层调用，否则执行到 sync.Wait 时，可能新起的 goroutine 还没调度到，sync.Add 自然没执行，最终导致逻辑出错。
2. mutex 是 mutual exclusion 的简写，翻译一下：互相排斥。
3. sync.cond 有两个比较有意思的方法：sync.Cond.Signal 和 sync.Cond.Broadcast。前者会唤醒等待时间最长的 goroutine，后者会唤醒所有等待的 goroutine。另外，要注意 sync.Cond.Wait 方法内部，隐藏了一些副作用，会先解锁：`c.L.Unlock()`，然后再加锁：`c.L.Lock()`。
4. 查询 Go 源码使用了多少次 sync.Once：

```javascript
grep -ir sync.Once $(go env GOROOT)/src | wc -l
```

1. channel 是粘合 goroutine 的胶水，select 则是粘合 channel 的胶水。
2. 关于 runtime.GOMAXPROCS(n) 函数的一个可能的使用场景：代码中可能存在 data race 的情况，增加 n 值可以让 data race 更快地发生，从而可以更快地调试错误。
3. 为了避免 goroutine 泄露，请注意：生成子 goroutine 的父 goroutine 需要负责停止子 gotoutine，即谁创建谁销毁。
4. 可以将一个“无序、耗时长”的 stage 转成 fan-out。fan-in 是多转一，fan-out 则是一转多。
5. 设计系统的时候，应该一开始就考虑 timeout 和 cancel。
6. 分布式系统需要支持 timeout 的几个理由：

- 饱和 系统饱和时，最后到达的请求需要直接超时返回，否则可能引发雪崩；
- 数据过期 数据其实有一定的时间窗口，过了窗口，就是无效数据了。例如前端一个请求过来，假设用户可以容忍 2s，那这个窗口就是 2s，分布式系统需要支持 2s 的超时设置，超过 2s 后数据无效；
- 防止死锁 当然，触发 timeout，有可能使死锁变成活锁。系统设计的目标应该是在不触发 timeout 的情况下不发生死锁。

1. 与上一条对应的，分布式系统应该支持 cancel 操作的几个理由：

- 超时 超时需要取消；
- 用户干预 当有用户驱动的并发操作时，用户可取消他发起的操作；
- 父节点取消 就像 context 一样，父 context 取消了，子 context 也要跟着取消；
- 重复的请求 为了得到更快的响应，同时向几个系统发起请求，当得到了最快的系统响应后，取消其他系统的请求。

1. 可以将多个 ratelimiter 组合在一起，提供更有表达力的 ratelimiter。例如我可以限制每秒 1 个请求，同时每分钟限制 10 个请求。具体见第五章 Rate Limiting 小节。
2. Go 使用 fork-join 模型。fork 即 go func(){}(), 而 join 则一般是指 sync.WaitGroup 或 channels。
3. 在一个函数里（位于某个 goroutine）不断地执行 go func(){}() 语句时，会不断地产生相应的 goroutine，并被添加到当前 goroutine 所在的 P 上的 LRQ 中，LRQ 可以看作是一个双端队列，越靠近队列尾的 goroutine 和当前 goroutine 的空间局部性越紧密，越需要优先执行。基于这点考虑，新产生的 goroutine 并不是直接放到 LRQ，而是会先放到 P 的 runnext 字段，执行完当前 goroutine 或当前 goroutine 被 park 后，首先执行的就是这个 runnext。如果之后又有新创建的 goroutine，它又会把当前挂在 runnext 上的 goroutine 顶到 LRQ 中。P 执行的时候从队列头的 goroutine 开始执行，而当 steal-working 发生时，也总是先从 LRQ 的头部偷，其实就是 FIFO。

