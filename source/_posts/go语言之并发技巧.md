---
title: go语言之chan struct{}
description: '一种特殊但非常实用的并发技巧'
tags: ['go']
toc: false
date: 2025-05-29 17:29:28
categories:
    - go
    - basic
---


## Go 语言并发利器：`chan struct{}` 与 `close()` 的妙用

在 Go 语言的并发世界里，goroutine 和 channel 是构建强大并发程序的基石。今天，我们来深入探讨一种特殊但非常实用的 channel 类型：`chan struct{}`，以及与之配合使用的 `close()` 函数。理解它们的机制和应用场景，能帮助我们写出更优雅、更高效的并发代码。

### `chan struct{}`：轻量级的信号通道

通常，channel 用于在 goroutine 之间传递数据。但有时，我们仅仅需要一个**信号**，通知某个或某些 goroutine 某个事件已经发生，而不需要传递任何具体的数据。这时，`chan struct{}` 就派上了用场。

`struct{}` 是一个零大小的类型，这意味着通过 `chan struct{}` 传递数据不会产生额外的内存开销。我们仅仅关心 channel 是否被发送或接收，以此作为事件发生的标志。

**声明和使用 `chan struct{}`:**

```go
package main

import "fmt"

func worker(done chan struct{}) {
	fmt.Println("Worker is starting...")
	// 模拟工作
	// ...
	fmt.Println("Worker is done!")
	done <- struct{}{} // 发送信号表示完成
}

func main() {
	done := make(chan struct{})
	go worker(done)
	<-done // 阻塞直到收到 worker 完成的信号
	fmt.Println("All done!")
}
```

在上面的例子中，`done` channel 的类型是 `chan struct{}`。`worker` goroutine 完成工作后，向 `done` channel 发送一个空结构体 `struct{}`。`main` goroutine 通过接收这个信号得知 `worker` 已经完成。

### `close()`：优雅地广播“结束”信号

`close()` 函数用于关闭一个 channel。**关闭一个 channel 会向所有正在等待从该 channel 接收数据的 goroutine 发送一个零值（对于 `chan struct{}` 来说就是零值，即可以接收到信号），并且后续不能再向已关闭的 channel 发送数据，否则会引发 panic。**

接收方可以通过接收操作的第二个返回值来判断 channel 是否已经关闭：

```go
value, ok := <-ch
```

如果 `ok` 是 `true`，表示接收到了 channel 中发送的值；如果 `ok` 是 `false`，表示 channel 已经关闭，并且 channel 中没有更多的数据可以接收。

**使用 `close()` 通知多个 goroutine 停止工作:**

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, quit chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Printf("Worker %d starting...\n", id)
	for {
		select {
		case <-quit:
			fmt.Printf("Worker %d received quit signal. Exiting.\n", id)
			return
		case <-time.After(time.Second): // 模拟工作
			fmt.Printf("Worker %d is working...\n", id)
		}
	}
}

func main() {
	numWorkers := 3
	var wg sync.WaitGroup
	quit := make(chan struct{})

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go worker(i+1, quit, &wg)
	}

	// 模拟一段时间后发出停止信号
	time.Sleep(3 * time.Second)
	fmt.Println("Sending quit signal to all workers...")
	close(quit) // 关闭 quit channel，广播停止信号

	wg.Wait()
	fmt.Println("All workers have exited.")
}
```

在这个例子中，我们创建了多个 `worker` goroutine，它们都监听同一个 `quit` channel。当 `main` goroutine 调用 `close(quit)` 时，所有正在阻塞等待从 `quit` 接收数据的 `worker` 都会收到一个零值，并通过 `select` 语句中的 `case <-quit:` 分支退出。`sync.WaitGroup` 用于等待所有 worker goroutine 优雅地退出。


## 实际应用场景


### 1. 事件通知：更细致的场景

在微服务架构中，一个服务完成某个关键操作后，可能需要通知其他依赖它的服务。使用 `chan struct{}` 可以实现这种轻量级的通知机制。

**场景：订单服务完成支付，通知库存服务扣减库存。**

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

// 库存服务
func inventoryService(orderPaid <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	for range orderPaid { // 接收到信号
		fmt.Println("库存服务：收到订单支付通知，正在扣减库存...")
		time.Sleep(1 * time.Second) // 模拟扣减库存
		fmt.Println("库存服务：库存扣减完成。")
	}
	fmt.Println("库存服务已停止。")
}

// 订单服务
func orderService(orderID string, paid chan<- struct{}) {
	fmt.Printf("订单服务：订单 %s 开始处理支付...\n", orderID)
	time.Sleep(2 * time.Second) // 模拟支付处理
	fmt.Printf("订单服务：订单 %s 支付完成。\n", orderID)
	paid <- struct{}{} // 发送支付完成的信号
}

func main() {
	var wg sync.WaitGroup
	orderPaid := make(chan struct{})

	wg.Add(1)
	go inventoryService(orderPaid, &wg)

	go orderService("order-123", orderPaid)

	time.Sleep(5 * time.Second) // 模拟主流程运行一段时间
	close(orderPaid)           // 关闭通知通道，库存服务会优雅退出
	wg.Wait()
	fmt.Println("主服务已停止。")
}
```

在这个例子中，`orderPaid` 是一个 `chan struct{}`。当订单服务完成支付后，会向 `orderPaid` 发送一个信号。库存服务监听这个信号，一旦收到，就开始扣减库存。使用 `chan struct{}` 的好处是，我们只关心“支付完成”这个事件的发生，不需要传递订单的具体信息给库存服务（订单信息应该通过其他更适合的方式传递）。最后通过 `close(orderPaid)` 通知库存服务可以停止监听了。

### 2. 服务优雅关闭：更全面的考量

在构建长时间运行的服务（如 Web 服务器、后台任务处理程序）时，优雅关闭至关重要。这涉及到通知所有正在运行的 goroutine 停止工作，并等待它们清理资源后退出。

**场景：一个简单的 Web 服务器，需要能够优雅地停止所有请求处理 goroutine。**

```go
package main

import (
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

func requestHandler(w http.ResponseWriter, r *http.Request, quit <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	select {
	case <-quit:
		fmt.Println("请求处理 goroutine 收到停止信号，正在退出...")
		return
	case <-time.After(5 * time.Second): // 模拟请求处理
		fmt.Fprintf(w, "请求处理完成 at %s\n", time.Now().Format(time.RFC3339))
		fmt.Println("请求处理完成。")
	}
}

func main() {
	var wg sync.WaitGroup
	quit := make(chan struct{})

	http.HandleFunc("/process", func(w http.ResponseWriter, r *http.Request) {
		wg.Add(1)
		go requestHandler(w, r, quit, &wg)
	})

	server := &http.Server{Addr: ":8080", Handler: nil}

	go func() {
		fmt.Println("Web 服务器正在监听 :8080")
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			fmt.Fatalf("HTTP server ListenAndServe: %v", err)
		}
		fmt.Println("Web 服务器已停止监听。")
	}()

	// 监听操作系统的中断信号 (Ctrl+C) 或终止信号
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)

	<-sig // 阻塞直到收到信号
	fmt.Println("\n收到停止信号，开始关闭服务器...")

	// 通知所有请求处理 goroutine 停止
	close(quit)

	// 给正在处理的请求一些时间完成
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(shutdownCtx); err != nil {
		fmt.Printf("HTTP server shutdown error: %v\n", err)
	}

	wg.Wait() // 等待所有请求处理 goroutine 退出
	fmt.Println("所有 goroutine 已退出，服务器已优雅关闭。")
}
```

在这个更复杂的例子中：

* 我们创建了一个 `quit` channel (`chan struct{}`) 用于通知请求处理的 goroutine 停止。
* 当收到操作系统的停止信号时，我们首先 `close(quit)`，向所有正在处理请求的 `requestHandler` goroutine 发送停止信号。
* 然后，我们使用 `http.Server.Shutdown` 来优雅地关闭 HTTP 服务器，这会阻止新的连接，并等待正在处理的连接完成（在超时时间内）。
* 最后，我们使用 `wg.Wait()` 等待所有正在处理的 `requestHandler` goroutine 退出。

`close(quit)` 在这里起到了广播停止信号的关键作用，使得多个并发的请求处理 goroutine 能够收到通知并开始清理退出。

### 3. 限流：令牌桶的简单实现

虽然 Go 语言有更专业的限流库，但使用带缓冲的 `chan struct{}` 可以实现一个简单的令牌桶限流器。

**场景：限制某个操作的并发执行数量。**

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, rateLimiter <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	<-rateLimiter // 从限流器获取一个“令牌”
	fmt.Printf("Worker %d started at %s\n", id, time.Now().Format(time.RFC3339))
	time.Sleep(2 * time.Second) // 模拟工作
	fmt.Printf("Worker %d finished at %s\n", id, time.Now().Format(time.RFC3339))
}

func main() {
	numWorkers := 5
	concurrencyLimit := 2
	var wg sync.WaitGroup
	rateLimiter := make(chan struct{}, concurrencyLimit)

	// 初始化令牌桶
	for i := 0; i < concurrencyLimit; i++ {
		rateLimiter <- struct{}{}
	}

	for i := 0; i < numWorkers; i++ {
		wg.Add(1)
		go worker(i+1, rateLimiter, &wg)
		time.Sleep(500 * time.Millisecond) // 稍微延迟启动 worker
		// 模拟令牌的补充 (实际场景可能更复杂)
		select {
		case rateLimiter <- struct{}{}:
		default:
			// 如果令牌桶已满，则不补充
		}
	}

	wg.Wait()
	fmt.Println("All workers finished.")
}
```

在这个例子中，`rateLimiter` 是一个容量为 `concurrencyLimit` 的 `chan struct{}`。每个 `worker` 在开始工作前需要从 `rateLimiter` 接收一个值（相当于获取一个“令牌”）。只有当 `rateLimiter` 中有可用的“令牌”时，`worker` 才能继续执行。虽然这个例子中的令牌补充比较简单，但它展示了如何使用带缓冲的 `chan struct{}` 来控制并发数量。
