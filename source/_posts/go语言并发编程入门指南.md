---
title: go语言并发编程入门指南
description: '快速熟悉go语言并发编程的基础知识，本文相当于一个目录性质的文章，想要学会go的并发编程还得下功夫'
tags: ['go']
toc: false
date: 2025-05-18 18:41:45
categories:
    - go
    - basic
---


# Go 并发编程指南

## 一、并发与并行的基本概念

* **并发（Concurrency）**：多个任务在同一时间段内交替执行（单核或多核皆可）。
* **并行（Parallelism）**：多个任务在同一时刻同时执行（仅在多核 CPU 中实现）。

## 二、Goroutine：轻量级线程

### 1. 基本使用

```go
go func() {
    fmt.Println("Hello from goroutine")
}()
```

* 每个 `goroutine` 约 2KB 栈空间，由 Go 调度器调度（非操作系统线程）。
* 创建开销小，可大规模使用（成千上万）。

### 2. 注意事项

* goroutine 是异步执行的，主协程退出会导致所有子 goroutine 被杀死。
* 需使用 `sync.WaitGroup` 或其他手段控制生命周期。

## 三、通道 Channel：Goroutine 间通信

### 1. 基本语法

```go
ch := make(chan int)     // 创建无缓冲通道
ch := make(chan int, 10) // 创建带缓冲通道

ch <- 1      // 发送数据
x := <-ch    // 接收数据
```

### 2. 特点

* 通道通信是**阻塞的**（无缓冲时发送/接收都阻塞）。
* 可用于 goroutine 之间安全地共享数据。

### 3. 关闭通道

```go
close(ch) // 通知接收方不再发送
```

### 4. 使用 `range` 读取所有数据

```go
for v := range ch {
    fmt.Println(v)
}
```

## 四、select：监听多个通道

```go
select {
case v := <-ch1:
    fmt.Println("From ch1:", v)
case ch2 <- 42:
    fmt.Println("Send to ch2")
default:
    fmt.Println("No channel ready")
}
```

* `select` 会阻塞直到某个分支可执行。
* 可用于实现超时、广播、负载均衡等逻辑。

## 五、WaitGroup：等待一组 goroutine 完成

```go
var wg sync.WaitGroup
wg.Add(2)

go func() {
    defer wg.Done()
    // 执行任务
}()

go func() {
    defer wg.Done()
    // 执行任务
}()

wg.Wait() // 阻塞直到计数器归零
```

## 六、Mutex：互斥锁

用于保护共享资源，避免数据竞争。

```go
var mu sync.Mutex

mu.Lock()
// 临界区
mu.Unlock()
```

## 七、Once：只执行一次（如单例）

```go
var once sync.Once

once.Do(func() {
    fmt.Println("Only once")
})
```

## 八、Cond：条件变量

适用于需要等待某条件成立再继续执行的情况。

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)

go func() {
    cond.L.Lock()
    cond.Wait() // 等待条件
    // 条件满足后执行
    cond.L.Unlock()
}()

cond.L.Lock()
cond.Signal() // 或 cond.Broadcast()
cond.L.Unlock()
```

## 九、Context：控制 goroutine 的生命周期

### 1. 取消 goroutine

```go
ctx, cancel := context.WithCancel(context.Background())
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // 正常执行
        }
    }
}(ctx)

cancel() // 通知 goroutine 退出
```

### 2. 超时 / 截止时间

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
```

## 十、定时器和 Ticker

```go
time.After(1 * time.Second)     // 一次性定时器
time.NewTicker(1 * time.Second) // 周期性定时器
```

## 十一、数据竞争与工具

* 使用 `go run -race` 启用数据竞争检测。
* Go 的 `race detector` 是诊断并发程序的强大工具。

## 十二、高级模式和最佳实践

* 使用工作池（Worker Pool）限制 goroutine 并发数。
* 避免共享内存，推荐通过 Channel 传递数据。
* 对共享状态使用原子操作或互斥锁。
* 理解 CSP 模型：“不要通过共享内存来通信，而应该通过通信来共享内存”。

---

