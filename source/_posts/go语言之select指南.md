---
title: go语言之select指南
description: '通俗化学习select语法的用法'
tags: ['go']
toc: false
date: 2025-05-26 08:57:21
categories:
    - go
    - basic
---

**想象一下场景：**

你现在是一位咖啡师，你的面前有多个顾客在不同的窗口等待点单（每个窗口可以看作一个 channel）。你不可能同时处理所有窗口的请求，但你希望尽快处理到来的请求。`select` 语句就像你的耳朵和眼睛，让你能够同时监听所有窗口，一旦有顾客按下呼叫铃（channel 有数据可接收），你就可以优先处理那个窗口的请求。

**`select` 语句的基本结构：**

```go
select {
case <-chan1:
    // 如果 chan1 可读，执行这里的代码
case val := <-chan2:
    // 如果 chan2 可读，执行这里的代码，并将接收到的值赋给 val
case chan3 <- expr:
    // 如果 chan3 可写（非满），执行这里的代码，并将 expr 发送到 chan3
default:
    // 如果没有任何 case 准备好，执行这里的代码（可选）
}
```

**核心要点：**

1.  **多路复用：** `select` 允许你同时等待多个 channel 的操作（发送或接收）。
2.  **非阻塞：** `select` 本身不会阻塞。它会检查所有的 `case`，如果其中一个 `case` 满足条件（可以进行收发操作），就执行该 `case` 对应的代码。
3.  **随机选择：** 如果有多个 `case` 同时满足条件，Go 语言会**随机**选择一个 `case` 执行。
4.  **`default` 子句（可选）：** 如果没有 `case` 准备好，并且存在 `default` 子句，那么会执行 `default` 中的代码。如果不存在 `default`，`select` 语句会阻塞，直到至少有一个 `case` 可以执行。

**生动解释每个 `case`：**

* `case <-chan1:`：这表示尝试从通道 `chan1` 接收数据。如果 `chan1` 中有数据，这个 `case` 就会被选中，并且接收到的数据会被丢弃（因为你没有用变量接收）。你可以把它想象成你听到一个窗口的铃响了，你知道有人要点单了。
* `case val := <-chan2:`：这同样是尝试从通道 `chan2` 接收数据。如果 `chan2` 中有数据，这个 `case` 会被选中，并且接收到的数据会存储在变量 `val` 中。这就像你听到另一个窗口的铃响了，并且你知道了顾客想要点什么。
* `case chan3 <- expr:`：这表示尝试向通道 `chan3` 发送数据 `expr`。如果 `chan3` 有足够的空间（非满），这个 `case` 就会被选中，并且 `expr` 会被发送到 `chan3`。这就像你准备好了一杯咖啡，准备送到一个等待的窗口。
* `default:`：如果没有任何顾客按下铃（没有任何 channel 可读或可写），并且你不想一直等待，你可以使用 `default` 子句来做一些其他的事情，比如擦桌子或者看看还有哪些咖啡豆。

**实际应用示例代码：**

让我们模拟一个场景，有两个不同的服务在向我们的主程序发送消息。我们希望能够同时监听这两个服务，并处理先到达的消息。

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

// 模拟服务 1 发送消息
func service1(ch chan string) {
	for i := 1; ; i++ {
		time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond) // 模拟处理时间
		message := fmt.Sprintf("来自服务 1 的消息 #%d", i)
		fmt.Println("服务 1 发送:", message)
		ch <- message
	}
}

// 模拟服务 2 发送消息
func service2(ch chan string) {
	for i := 1; ; i++ {
		time.Sleep(time.Duration(rand.Intn(1500)) * time.Millisecond) // 模拟处理时间
		message := fmt.Sprintf("来自服务 2 的消息 #%d", i)
		fmt.Println("服务 2 发送:", message)
		ch <- message
	}
}

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)

	go service1(ch1)
	go service2(ch2)

	for i := 0; i < 5; i++ { // 我们只接收前 5 条消息
		select {
		case msg1 := <-ch1:
			fmt.Println("主程序接收到:", msg1)
		case msg2 := <-ch2:
			fmt.Println("主程序接收到:", msg2)
		}
		fmt.Println("--- 处理了一条消息 ---")
	}

	fmt.Println("处理完毕！")
}
```

**代码解释：**

1.  我们创建了两个通道 `ch1` 和 `ch2`，分别用于接收来自 `service1` 和 `service2` 的消息。
2.  `service1` 和 `service2`  Goroutine 会不断地生成消息并通过各自的通道发送。它们发送消息的时间间隔是随机的，模拟了不同的服务可能在不同的时间发送数据。
3.  在 `main` 函数的 `for` 循环中，我们使用 `select` 语句同时监听 `ch1` 和 `ch2`。
4.  哪个通道先有数据到达，对应的 `case` 就会被执行，主程序会接收并打印该消息。
5.  由于 `select` 的随机性，你每次运行程序，接收到的消息顺序可能会不同。

**更进一步的应用场景：**

* **超时控制：** 你可以在 `select` 中加入一个 `time.After` 的 `case`，如果在指定时间内没有从其他通道接收到数据，就可以执行超时处理逻辑。
* **取消操作：** 可以使用一个 done channel，在需要取消某个 Goroutine 的时候关闭这个 channel，然后在 `select` 中监听这个 done channel，一旦关闭就退出 Goroutine。
* **多路网络请求：** 同时等待多个网络请求的响应，哪个先返回就先处理哪个。


### 1. 超时控制

**场景：** 假设我们正在尝试从一个外部服务获取数据，但这个服务有时响应很慢或者没有响应。我们不希望程序一直等待下去，而是希望在一定时间后放弃并进行其他处理。

**实现方式：** 在 `select` 语句中加入一个 `time.After(timeout)` 的 `case`。`time.After(timeout)` 会返回一个在 `timeout` 时间后会收到一个 `time.Time` 值的 channel。

**示例代码：**

```go
package main

import (
	"fmt"
	"time"
)

func fetchData(ch chan string) {
	// 模拟耗时操作
	time.Sleep(2 * time.Second)
	ch <- "来自外部服务的数据"
}

func main() {
	result := make(chan string, 1) // 使用带缓冲的 channel，防止 Goroutine 阻塞
	go fetchData(result)

	timeout := 1 * time.Second

	select {
	case data := <-result:
		fmt.Println("成功获取到数据:", data)
	case <-time.After(timeout):
		fmt.Println("请求超时，放弃等待。")
	}

	fmt.Println("程序继续执行...")
}
```

**代码解释：**

1.  `fetchData` 函数模拟了一个需要 2 秒才能完成的操作，并将结果发送到 `result` channel。
2.  在 `main` 函数中，我们启动了一个 Goroutine 来执行 `fetchData`。
3.  我们设置了一个 `timeout` 为 1 秒。
4.  `select` 语句同时监听 `result` channel 和 `time.After(timeout)` 返回的 channel。
5.  如果 `fetchData` 在 1 秒内完成，我们会从 `result` 接收到数据并打印。
6.  如果超过 1 秒后，`time.After(timeout)` 返回的 channel 会收到一个值，`case <-time.After(timeout):` 就会被选中，我们打印 "请求超时，放弃等待。"。

**运行结果（可能）：**

由于 `fetchData` 需要 2 秒，而超时时间是 1 秒，所以很可能会输出：

```
请求超时，放弃等待。
程序继续执行...
```

如果你将 `time.Sleep(2 * time.Second)` 改为一个小于 1 秒的值，你就会看到成功获取数据的输出。

### 2. 取消操作

**场景：** 有时我们需要在某个操作进行到一半的时候取消它，例如用户关闭了一个页面，我们不再需要等待后台任务完成。

**实现方式：** 使用一个 "done" channel。当需要取消操作时，关闭这个 done channel。在执行任务的 Goroutine 中，使用 `select` 监听这个 done channel。一旦 done channel 被关闭，Goroutine 就可以清理资源并退出。

**示例代码：**

```go
package main

import (
	"fmt"
	"time"
)

func worker(done <-chan struct{}) {
	for i := 0; ; i++ {
		select {
		case <-done:
			fmt.Println("工作者收到取消信号，退出。")
			return
		default:
			fmt.Printf("工作者正在努力工作... (%d)\n", i)
			time.Sleep(500 * time.Millisecond)
		}
	}
}

func main() {
	done := make(chan struct{})

	fmt.Println("启动工作者...")
	go worker(done)

	// 模拟一段时间后需要取消工作者
	time.Sleep(2 * time.Second)
	fmt.Println("发送取消信号...")
	close(done) // 关闭 done channel 通知 worker 停止

	// 等待一段时间，确保 worker 有机会退出
	time.Sleep(1 * time.Second)
	fmt.Println("主程序结束。")
}
```

**代码解释：**

1.  `worker` 函数在一个无限循环中模拟工作。
2.  它使用 `select` 同时监听 `done` channel 和默认情况。
3.  如果 `done` channel 接收到值（当它被关闭时，会一直可以接收到零值），则 `case <-done:` 会被选中，worker 打印退出信息并返回。
4.  `default` case 表示如果没有收到取消信号，worker 就继续工作一段时间。
5.  在 `main` 函数中，我们创建了一个 `done` channel 并启动了 `worker` Goroutine，并将 `done` channel 传递给它。
6.  主程序等待 2 秒后，通过 `close(done)` 关闭了 `done` channel，向 `worker` 发送取消信号。
7.  `worker` 接收到信号后会退出。

**运行结果（大致）：**

```
启动工作者...
工作者正在努力工作... (0)
工作者正在努力工作... (1)
工作者正在努力工作... (2)
工作者正在努力工作... (3)
发送取消信号...
工作者收到取消信号，退出。
主程序结束。
```
