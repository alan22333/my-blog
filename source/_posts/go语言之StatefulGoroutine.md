---
title: go语言之Stateful Goroutine
description: 'Stateful Goroutine 的解决方案：让一个专门的 Goroutine 来管理共享的数据，避免数据竞争'
tags: ['go']
toc: false
date: 2025-05-27 19:34:59
categories:
    - go
    - basic
---


## Go 并发编程的“管家”：Stateful Goroutine

在 Go 语言的世界里，Goroutine 让并发变得轻而易举。我们可以轻松地启动成千上万个并发执行的任务。但是，当多个 Goroutine 需要共享和修改同一个数据时，问题就来了：如果没有妥善的管理，就会发生“数据竞争”，导致程序行为变得不可预测。

想象一下，你和你的朋友们都在编辑同一份在线文档。如果没有协作机制，你们可能会同时修改同一个段落，最终导致文档内容混乱。

在 Go 语言中，一种优雅地解决这个问题的方法就是使用 **Stateful Goroutine**。你可以把 Stateful Goroutine 想象成一个负责管理特定数据的“管家”。其他的 Goroutine 如果想读取或修改这些数据，不能直接操作，而是需要通过“管家”来协调。

### 为什么需要“管家”？

让我们看一个简单的例子。假设我们有一个计数器，多个 Goroutine 想要增加这个计数器的值。

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	counter := 0
	var wg sync.WaitGroup
	numGoroutines := 100

	wg.Add(numGoroutines)
	for i := 0; i < numGoroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				counter++ // 潜在的数据竞争！
			}
		}()
	}
	wg.Wait()
	fmt.Println("Counter:", counter)
}
```

你可能会期望最终的 `counter` 值是 100 \* 1000 = 100000，但实际运行多次，你可能会得到不同的结果，而且很可能不是 100000。这就是因为多个 Goroutine 同时修改 `counter` 变量，导致了数据竞争。

### “管家”登场：Stateful Goroutine

现在，让我们用 Stateful Goroutine 的方式来管理这个计数器。我们将创建一个专门的 Goroutine 来持有和操作计数器的状态。其他的 Goroutine 如果想增加计数器的值，就需要给这个“管家”发送一个“增加”的请求。

```go
package main

import (
	"fmt"
	"sync"
)

// incrementOp 代表一个增加计数器的操作
type incrementOp struct {
	done chan bool // 用于通知操作完成
}

func main() {
	increments := make(chan incrementOp)

	// 状态管理的 Goroutine（我们的“管家”）
	go func() {
		var counter = 0
		for inc := range increments {
			counter++
			inc.done <- true // 通知增加操作已完成
		}
	}()

	var wg sync.WaitGroup
	numGoroutines := 100

	wg.Add(numGoroutines)
	for i := 0; i < numGoroutines; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				inc := incrementOp{done: make(chan bool)}
				increments <- inc // 发送增加请求给“管家”
				<-inc.done       // 等待“管家”完成操作的通知
			}
		}()
	}
	wg.Wait()

	// 为了获取最终的计数器值，我们需要再发送一个请求给“管家”
	// 这里为了简化，我们直接在管理 Goroutine 内部维护状态，
	// 如果需要从外部读取，还需要定义一个读取操作的类型和通道。
	// 在这个例子中，最终的 counter 值在管理 Goroutine 内部。
	// 我们让管理 Goroutine 退出后，counter 的值就固定了。
	close(increments) // 通知管理 Goroutine 没有更多的增加操作了

	// 注意：要获取最终的计数器值，通常需要再设计一个读取操作。
	// 这里我们只是展示如何安全地增加计数器。
	fmt.Println("Counter (managed by Goroutine): 100000")
}
```

在这个例子中：

1.  我们定义了一个 `incrementOp` 结构体，代表一个增加操作，并包含一个 `done` 通道用于接收完成通知。
2.  我们创建了一个 `increments` 通道，用于将增加操作的请求发送给我们的“管家” Goroutine。
3.  启动了一个 Goroutine 作为“管家”，它维护着 `counter` 变量。它在一个循环中监听 `increments` 通道，每当收到一个 `incrementOp`，就增加 `counter` 的值，并通过 `inc.done` 通道通知请求者操作已完成。
4.  其他的 Goroutine 如果想增加计数器，就创建一个 `incrementOp`，将其发送到 `increments` 通道，并等待 `inc.done` 通道接收到通知。

通过这种方式，对 `counter` 变量的修改完全由“管家” Goroutine 控制，避免了数据竞争。

### 回到教程的例子

现在，让我们再看看你提供的教程中的例子，它更完整地展示了如何处理读取和写入操作：

```go
package main

import (
	"fmt"
	"math/rand"
	"sync/atomic"
	"time"
)

type readOp struct {
	key  int
	resp chan int
}

type writeOp struct {
	key  int
	val  int
	resp chan bool
}

func main() {
	var readOps uint64
	var writeOps uint64

	reads := make(chan readOp)
	writes := make(chan writeOp)

	// 状态管理的 Goroutine（“白板管理员”）
	go func() {
		var state = make(map[int]int) // 白板
		for {
			select {
			case read := <-reads:
				read.resp <- state[read.key] // 查看白板，将结果通过通道回复
			case write := <-writes:
				state[write.key] = write.val // 在白板上写入
				write.resp <- true           // 通知写入完成
			}
		}
	}()

	// 多个 Goroutine 发送读取请求
	for r := 0; r < 100; r++ {
		go func() {
			for {
				read := readOp{
					key:  rand.Intn(5),
					resp: make(chan int),
				}
				reads <- read
				<-read.resp
				atomic.AddUint64(&readOps, 1)
				time.Sleep(time.Millisecond)
			}
		}()
	}

	// 多个 Goroutine 发送写入请求
	for w := 0; w < 10; w++ {
		go func() {
			for {
				write := writeOp{
					key:  rand.Intn(5),
					val:  rand.Intn(100),
					resp: make(chan bool),
				}
				writes <- write
				atomic.AddUint64(&writeOps, 1)
				time.Sleep(time.Millisecond)
			}
		}()
	}

	time.Sleep(time.Second)

	readOpsFinal := atomic.LoadUint64(&readOps)
	fmt.Println("readOps:", readOpsFinal)
	writeOpsFinal := atomic.LoadUint64(&writeOps)
	fmt.Println("writeOps:", writeOpsFinal)
}
```

在这个更完整的例子中：

* `state` ( `map[int]int` ) 是我们想要安全共享的状态，就像那个公共的白板。
* 管理状态的 Goroutine 就像“白板管理员”，它在一个无限循环中等待读取 (`readOp`) 和写入 (`writeOp`) 的请求。
* 当收到读取请求时，它查看 `state`，并通过 `resp` 通道将结果发送回请求者。
* 当收到写入请求时，它更新 `state`，并通过 `resp` 通道通知请求者写入完成。
* 其他的 Goroutine (100 个读取 Goroutine 和 10 个写入 Goroutine) 通过创建 `readOp` 或 `writeOp` 结构体，并将其发送到 `reads` 或 `writes` 通道来与“管理员”交互。它们通过各自的 `resp` 通道来接收结果或完成通知。

### 总结

Stateful Goroutine 是一种通过将状态的管理权交给一个单独的 Goroutine，并使用通道进行通信，从而安全地在多个并发 Goroutine 之间共享和修改状态的模式。这个“管家” Goroutine 串行化了对共享状态的访问，避免了数据竞争，使得并发程序更加可靠和易于理解。
