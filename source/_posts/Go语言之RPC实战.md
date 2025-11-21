---
title: Go语言之RPC实践：push VS. pull
description: ''
tags: []
toc: false
date: 2025-11-16 18:14:44
categories:
---

在本文中，我们使用两个示例学习一下go中的rpc的基本使用。这是一个非常经典的 RPC 应用场景：Master/Worker，其中Master负责分发任务，Worker负责接收并执行任务，然后返回结果。

<!--more-->

## Push模式

这个例子可以理解为：

> Master有很多的任务要做，但是不可能自己去做，他要检查自己有哪些Worker，然后并发调用Worker的RPC函数得到结果，仿佛是自己完成了所有工作。

1. 项目结构

```
rpc-master-worker/
├── go.mod
├── master/
│   └── main.go
├── shared/
│   └── common.go
└── worker/
    └── main.go
```

2. 共享数据结构（shared/common.go）

为了让 Master 和 Worker 能够共享数据结构（如任务参数和回复），我们创建一个 shared 包。
这是 Master 和 Worker 之间的“契约”。

```go
package shared

import "fmt"

// 任务参数
type TaskArgs struct {
	TaskID   int
	TaskData string
}

// 任务回复
type TaskReply struct {
	WorkerID   int
	ResultData string
}

// 定义一个通用的服务名称，以便双方都能引用
const WorkerServiceName = "WorkerService"

// 辅助函数，用于格式化任务
func (t *TaskArgs) String() string {
	return fmt.Sprintf("Task(ID: %d, Data: '%s')", t.TaskID, t.TaskData)
}

// 辅助函数，用于格式化回复
func (r *TaskReply) String() string {
	return fmt.Sprintf("Reply(Worker: %d, Result: '%s')", r.WorkerID, r.ResultData)
}
```

3. 服务端(worker/main.go)

Worker 是这个例子中的 RPC 服务端。它注册一个 WorkerService，实现一个 DoTask 方法，然后在指定的端口上监听来自 Master 的连接。

我们将使用 flag 包允许从命令行指定每个 Worker 的 ID 和端口，这样我们就可以轻松地启动多个 Worker。

```go
package main

import (
	"flag"
	"fmt"
	"log"
	"net"
	"net/rpc"
	"rpc-master-worker/shared" // 导入共享包
	"strings"
)

// WorkerService 是 RPC 服务端暴露的对象
type WorkerService struct {
	ID int
}

// DoTask 是 RPC 调用的方法
// 必须遵循 net/rpc 规范: (args *ArgType, reply *ReplyType) error
func (w *WorkerService) DoTask(args *shared.TaskArgs, reply *shared.TaskReply) error {
	log.Printf("Worker %d: 接收到任务 %s\n", w.ID, args)

	// 模拟执行任务（例如：将字符串转换为大写）
	result := strings.ToUpper(args.TaskData)

	// 填充回复
	reply.WorkerID = w.ID
	reply.ResultData = result

	log.Printf("Worker %d: 完成任务 %d, 结果: %s\n", w.ID, args.TaskID, result)
	return nil
}

func main() {
	// 1. 使用 flag 允许从命令行传入 ID 和 端口
	workerID := flag.Int("id", 0, "Worker ID")
	port := flag.String("port", ":9001", "Worker 监听的端口 (例如 :9001)")
	flag.Parse()

	// 2. 创建 WorkerService 实例
	workerService := &WorkerService{
		ID: *workerID,
	}

	// 3. 注册 RPC 服务
	err := rpc.RegisterName(shared.WorkerServiceName, workerService)
	if err != nil {
		log.Fatalf("Worker %d: 注册服务失败: %v", *workerID, err)
	}

	// 4. 在指定端口上启动 TCP 监听(其他的也差不多)
	listener, err := net.Listen("tcp", *port)
	if err != nil {
		log.Fatalf("Worker %d: 监听 %s 失败: %v", *workerID, *port, err)
	}
	defer listener.Close()

	log.Printf("Worker %d: 正在 %s 上监听 RPC 请求...\n", *workerID, *port)

	// 5. 接受并处理连接
	// rpc.Accept 会阻塞并等待新的连接，为每个连接启动一个 goroutine 处理
	rpc.Accept(listener)
}
```

4. 客户端（master/main.go）

Master 是这个例子中的 RPC 客户端。它持有一组任务和一个 Worker 地址列表（理解为我需要使用的服务们）。
它将并发地（使用 goroutine 和 WaitGroup）将任务分发给 Worker，并收集所有的结果。

```go
package main

import (
	"log"
	"net/rpc"
	"rpc-master-worker/shared"
	"sync"
	"time"
)

func main() {
	log.Println("Master 启动...")

	// 1. 定义 Worker 列表
	// 在生产环境中，这应该通过服务发现（如 Consul, Etcd）动态获取
	workerAddresses := []string{
		"localhost:9001",
		"localhost:9002",
	}

	// 2. 定义任务列表
	tasks := []*shared.TaskArgs{
		{TaskID: 1, TaskData: "hello world"},
		{TaskID: 2, TaskData: "go rpc is simple"},
		{TaskID: 3, TaskData: "learn by example"},
		{TaskID: 4, TaskData: "concurrency is fun"},
		{TaskID: 5, TaskData: "master worker pattern"},
	}

	var wg sync.WaitGroup
	numWorkers := len(workerAddresses)

	for i, task := range tasks {
		wg.Add(1)

		// 3. 选择一个 Worker (这里使用简单的轮询)
		workerAddr := workerAddresses[i%numWorkers]
		
		// 4. 为每个任务启动一个 goroutine
		go func(task *shared.TaskArgs, addr string) {
			defer wg.Done()

			// a. 连接 Worker
			client, err := rpc.Dial("tcp", addr)
			if err != nil {
				log.Printf("Master: 连接 Worker %s 失败: %v", addr, err)
				return
			}
			defer client.Close()

			// b. 准备回复
			var reply shared.TaskReply

			log.Printf("Master: 分发任务 %s 到 Worker %s\n", task, addr)

			// c. 发起同步 RPC 调用
			// 我们也可以使用 client.Go() 进行异步调用，但这里在 goroutine 中，Call() 已经实现了并发
			err = client.Call(shared.WorkerServiceName+".DoTask", task, &reply)
			if err != nil {
				log.Printf("Master: 调用 Worker %s 失败: %v", addr, err)
				return
			}

			// d. 打印结果
			log.Printf("Master: 收到结果: %s\n", &reply)

		}(task, workerAddr)
		
		// 稍微错开任务分发，便于观察
		time.Sleep(100 * time.Millisecond)
	}

	// 5. 等待所有任务完成
	wg.Wait()
	log.Println("Master: 所有任务已完成。")
}
```

6. 运行
打开 3 个终端窗口。

```bash
cd rpc-master-worker
go run . -id=1 -port=:9001
```

```bash
cd rpc-master-worker
go run . -id=2 -port=:9002
```

等待两个 Worker 都启动后，运行 Master：

```bash
cd rpc-master-worker
go run .
```

## Pull模式

这个例子可以理解为：

> Master有很多的任务要做，但是不可能自己去做，他的Worker会不停的询问他有没有待完成的工作，如果有请求+工作，Master就会把工作分发出去，否则阻塞工作进程。另外，我们再加入一点容错机制，例如超时机制（一个worker领取了任务但是没有及时完成，则任务会被重置等待新的Worker来领取）。

1. 项目结构

和之前一样

2. 共享数据结构（shared/common.go）

依旧定义一些约定

```Go
package shared

import (
	"fmt"
	"time"
)

// Master 提供的服务名
const MasterServiceName = "MasterService"

// MasterTaskTimeout 定义任务超时时间
const MasterTaskTimeout = 3 * time.Second

// --- 1. Worker 请求任务 ---

// Worker 请求任务的参数
type RequestTaskArgs struct {
	WorkerID string // Worker 用自己的ID来注册
}

// Master 返回给 Worker 的任务
// (注意：Task 本身也定义在这里)
type Task struct {
	ID   int
	Data string
}

// Master 返回任务的回复
type RequestTaskReply struct {
	Task        *Task // 要执行的任务 (如果为 nil，表示暂时没有)
	NoMoreTasks bool  // 如果为 true，表示所有任务都完成了
}

// --- 2. Worker 提交结果 ---

// Worker 提交结果的参数
type SubmitTaskArgs struct {
	WorkerID   string
	TaskID     int
	ResultData string
}

// Master 对提交的回复（简单确认）
type SubmitTaskReply struct {
	Acknowledged bool
}

// --- 辅助函数 ---
func (t *Task) String() string {
	return fmt.Sprintf("Task(ID: %d, Data: '%s')", t.ID, t.Data)
}

// 模拟任务执行
func (t *Task) Process() string {
	fmt.Printf("... 正在处理任务 %d ...\n", t.ID)
	// 模拟耗时工作
	time.Sleep(1 * time.Second)
	// 简单的字符串处理
	return fmt.Sprintf("Processed data from task %d", t.ID)
}

```

3. 客户端（Worker/main.go）

Worker 现在是客户端。它在一个无限循环中工作(RPC注册/连接后)：请求 -> 处理 -> 提交。

```Go
package main

import (
	"flag"
	"fmt"
	"io"
	"log"
	"net/rpc"
	"rpc/shared"
	"time"
)

// Worker 结构（无状态，只持有配置）
type Worker struct {
	ID         string
	MasterAddr string
	client     *rpc.Client
}

// NewWorker 创建 Worker
func NewWorker(id, masterAddr string) *Worker {
	return &Worker{
		ID:         id,
		MasterAddr: masterAddr,
	}
}

// connect 连接到 Master
func (w *Worker) connect() error {
	client, err := rpc.Dial("tcp", w.MasterAddr)
	if err != nil {
		return fmt.Errorf("连接 Master (%s) 失败: %v", w.MasterAddr, err)
	}
	log.Printf("Worker %s: 已连接到 Master %s\n", w.ID, w.MasterAddr)
	w.client = client
	return nil
}

// Run 开始工作循环
func (w *Worker) Run() {
	if err := w.connect(); err != nil {
		log.Fatalf(err.Error())
	}
	defer w.client.Close()

	log.Printf("Worker %s: 开始请求任务...\n", w.ID)

	for {
		// 1. 请求任务
		reqArgs := &shared.RequestTaskArgs{WorkerID: w.ID}
		reply := &shared.RequestTaskReply{}

		err := w.client.Call(shared.MasterServiceName+".RequestTask", reqArgs, &reply)

		if err != nil {
			// 如果连接断开 (io.EOF 或 rpc.ErrShutdown)，说明 Master 可能已关闭
			if err == io.EOF || err == rpc.ErrShutdown {
				log.Printf("Worker %s: 与 Master 的连接已断开。退出。\n", w.ID)
				return
			}
			log.Printf("Worker %s: 请求任务失败: %v. 正在重试...\n", w.ID, err)
			time.Sleep(1 * time.Second) // 发生错误时稍等
			continue
		}

		// 2. 检查 Master 是否已无任务
		if reply.NoMoreTasks {
			log.Printf("Worker %s: Master 通知所有任务已完成。退出。\n", w.ID)
			return
		}

		// 3. 检查是否真的拿到了任务
		if reply.Task == nil {
			// 这种情况在本例中不应发生（因为 Master 队列是阻塞的），但作为健壮性检查
			log.Printf("Worker %s: 暂时没有任务，1秒后重试...\n", w.ID)
			time.Sleep(1 * time.Second)
			continue
		}

		// 4. 拿到了任务，开始处理
		task := reply.Task
		log.Printf("Worker %s: 领到任务 %s\n", w.ID, task)

		// 模拟w01处理任务3 处理超时
		if task.ID == 3 && w.ID == "w01" {
			log.Printf("Worker %s: !!! 模拟处理任务 3 超时 !!!", w.ID)
			time.Sleep(shared.MasterTaskTimeout + 2*time.Second)
		}

		// 在自己的地址空间里执行任务
		resultData := task.Process() // 模拟执行

		// 5. 提交结果
		submitArgs := &shared.SubmitTaskArgs{
			WorkerID:   w.ID,
			TaskID:     task.ID,
			ResultData: resultData,
		}
		submitReply := &shared.SubmitTaskReply{}

		err = w.client.Call(shared.MasterServiceName+".SubmitTaskResult", submitArgs, &submitReply)
		if err != nil {
			log.Printf("Worker %s: 提交任务 %d 结果失败: %v\n", w.ID, task.ID, err)
			// (在实际应用中，这里需要重试策略)
		} else if submitReply.Acknowledged {
			log.Printf("Worker %s: 成功提交任务 %d 结果\n", w.ID, task.ID)
		}
	}
}

func main() {
	workerID := flag.String("id", "worker-01", "Worker 的唯一ID")
	masterAddr := flag.String("master", "localhost:9000", "Master 的地址")
	flag.Parse()

	worker := NewWorker(*workerID, *masterAddr)
	worker.Run()
}

```

4. 服务端（Master/main.go）

Master 现在是有状态的服务端。它需要维护任务队列，并处理并发（因为多个 Worker 会同时请求）。我们将使用 Mutex (互斥锁) 来保护任务状态。

```Go
package main

import (
	"fmt"
	"log"
	"net"
	"net/rpc"
	"rpc/shared"
	"sync"
	"time"
)

// 任务状态
type TaskStatus int

const (
	StatusIdle       TaskStatus = 0 // 待分配
	StatusInProgress TaskStatus = 1 // 已分配，处理中
	StatusCompleted  TaskStatus = 2 // 已完成
)

// Master 内部维护的任务结构
type MasterTask struct {
	Task      *shared.Task
	Status    TaskStatus
	WorkerID  string    // 哪个 Worker 正在处理
	StartTime time.Time // 开始时间（用于处理超时）
}

// Master 服务
type Master struct {
	mu             sync.Mutex
	tasks          map[int]*MasterTask // 所有任务的 "数据库"
	taskQueue      chan int            // 待分配的任务ID队列
	totalTasks     int
	completedTasks int
}

// NewMaster 创建并初始化 Master
func NewMaster() *Master {
	m := &Master{
		tasks:     make(map[int]*MasterTask),
		taskQueue: make(chan int, 100), // 带缓冲的通道
	}

	// 1. 初始化任务列表
	m.initTasks()
	m.totalTasks = len(m.tasks)
	log.Printf("Master 初始化完成，总共有 %d 个任务待处理。\n", m.totalTasks)

	// 2. 启动扫描器，定时检查有没有超时任务
	go m.runScanner()

	return m
}

func (m *Master) runScanner() {
	ticker := time.NewTicker(1 * time.Second) // 每秒扫描一次
	defer ticker.Stop()

	for range ticker.C {
		m.mu.Lock()

		// 检查是否所有任务都已完成
		if m.totalTasks == m.completedTasks {
			m.mu.Unlock()
			return // 所有任务完成，扫描器退出
		}

		log.Println("Scanner: 正在扫描超时任务...")

		var tasksToRequeue []int
		for taskID, masterTask := range m.tasks {
			// 检查正在进行中 且 超时 的任务
			if masterTask.Status == StatusInProgress &&
				time.Since(masterTask.StartTime) > shared.MasterTaskTimeout {

				log.Printf("Master: 任务 %d 在 Worker %s 上已超时! 准备重新排队。", taskID, masterTask.WorkerID)

				// 1. 重置任务状态
				masterTask.Status = StatusIdle
				masterTask.WorkerID = ""
				tasksToRequeue = append(tasksToRequeue, taskID)
			}
		}

		// 【重要】在操作 channel 前解锁
		m.mu.Unlock()

		// 2. 将超时的任务ID重新放回队列
		for _, taskID := range tasksToRequeue {
			m.taskQueue <- taskID
		}
	}
}

// initTasks 辅助函数：填充初始任务
func (m *Master) initTasks() {
	tasksData := []string{
		"task data 1",
		"task data 2",
		"task data 3",
		"task data 4",
		"task data 5",
	}
	for i, data := range tasksData {
		taskID := i + 1
		m.tasks[taskID] = &MasterTask{
			Task:   &shared.Task{ID: taskID, Data: data},
			Status: StatusIdle,
		}
		// 推入待办队列
		m.taskQueue <- taskID
	}
}

// --- RPC 方法 ---

// RequestTask (Worker 调用此方法来请求任务)
func (m *Master) RequestTask(args *shared.RequestTaskArgs, reply *shared.RequestTaskReply) error {
	// 1. 从任务队列中获取一个任务ID
	//    这是一个阻塞操作，如果没有任务，goroutine会在此等待
	taskID, ok := <-m.taskQueue
	if !ok {
		// 如果通道被关闭，意味着所有任务都已完成
		reply.NoMoreTasks = true
		return nil
	}

	// 2. 锁定状态，更新任务
	m.mu.Lock()
	defer m.mu.Unlock()

	masterTask, exists := m.tasks[taskID]
	if !exists {
		// 理论上不应该发生
		return fmt.Errorf("内部错误：任务ID %d 不存在", taskID)
	}

	// 3. 分配任务
	masterTask.Status = StatusInProgress
	masterTask.WorkerID = args.WorkerID
	masterTask.StartTime = time.Now()

	// 4. 填充回复
	reply.Task = masterTask.Task
	reply.NoMoreTasks = false

	log.Printf("Master: 分配任务 %d 给 Worker %s\n", taskID, args.WorkerID)
	return nil
}

// SubmitTaskResult (Worker 调用此方法来提交结果)
func (m *Master) SubmitTaskResult(args *shared.SubmitTaskArgs, reply *shared.SubmitTaskReply) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	taskID := args.TaskID
	masterTask, exists := m.tasks[taskID]

	// 1. 验证
	if !exists {
		return fmt.Errorf("任务 %d 不存在", taskID)
	}

	// 情况1: 已经超时了，但尚未被分配出去（来迟半步）
	if masterTask.Status == StatusIdle {
		log.Printf("Master: 收到任务 %d 的【过时】提交 (来自 %s)，任务已超时并正在等待重新分配。忽略。",
			taskID, args.WorkerID)
		reply.Acknowledged = true
		return nil
	}

	// 情况2: 已经超时了，并且分配给了其他worker，且正在处理（来迟一步）
	if masterTask.Status == StatusInProgress && masterTask.WorkerID != args.WorkerID {
		log.Printf("Master: 收到 Worker %s 对任务 %d 的【过时】提交 (任务现分配给 %s)。忽略此结果。",
			args.WorkerID, taskID, masterTask.WorkerID)
		reply.Acknowledged = true // 仍然告诉它 "收到了"，但我们不做处理
		return nil
	}

	// 情况3: 已经超时了，并且分配给了其他worker，且已经完成（来迟两步）
	if masterTask.Status == StatusCompleted {
		log.Printf("Master: 收到任务 %d 的【重复】提交 (来自 %s)，任务已由 %s 完成。忽略。",
			taskID, args.WorkerID, masterTask.WorkerID)
		reply.Acknowledged = true
		return nil
	}

	// 通过验证，说明是一个及时的提交
	// 2. 更新状态
	masterTask.Status = StatusCompleted
	masterTask.WorkerID = args.WorkerID // 记录最后由谁完成
	m.completedTasks++

	log.Printf("Master: 收到 Worker %s 对任务 %d 的结果: '%s'\n", args.WorkerID, taskID, args.ResultData)

	// 3. 检查是否所有任务都已完成
	if m.completedTasks == m.totalTasks {
		log.Println("--- 所有任务已完成！即将关闭任务队列。 ---")
		// 关闭任务队列，这将导致所有阻塞在 <-m.taskQueue 上的 Worker 收到 !ok
		close(m.taskQueue)
	}

	reply.Acknowledged = true
	return nil
}

// --- 主程序 ---

func main() {
	master := NewMaster()

	// 1. 注册 MasterService
	err := rpc.RegisterName(shared.MasterServiceName, master)
	if err != nil {
		log.Fatalf("注册 Master 服务失败: %v", err)
	}

	// 2. 监听端口
	port := ":9000"
	listener, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("监听 %s 失败: %v", port, err)
	}
	defer listener.Close()

	log.Printf("Master 服务端正在 %s 上监听...", port)

	// 3. 接受连接
	rpc.Accept(listener)
}

```

## 谁来管理状态？

这是一个经典问题。答案是：“尽可能地将状态推给服务端（接收方），让客户端保持无状态。”

我们来分析不同的模式：

### 模式一：服务端管理状态（Stateful Server）- 我们的 Pull 模式
谁管理？ 接收方（Master）。

状态：Master 维护着所有任务的完整状态（空闲、处理中、已完成）、哪个 Worker 在处理、什么时候开始的。

客户端（Worker）：完全无状态。它不记得自己做过什么，它只知道“请求 -> 执行 -> 提交”。如果一个 Worker 崩溃重启了，它会像一个新 Worker 一样简单地重新加入工作池，开始请求任务。

实例：

我们的 Master/Worker：这是最佳实践。

数据库：数据库（服务端）管理所有数据状态，客户端只是发出 CRUD 请求。

Web Sessions (服务端存储)：用户登录后，服务器创建一个 Session ID，并将用户信息（如购物车）存储在服务器的内存或 Redis 中。

#### 优点：

- 客户端简单、健壮：客户端（Worker）可以随意启停、崩溃、扩展，对系统整体状态没有影响（除了短暂的劳动力损失）。

- 逻辑集中：所有复杂的业务逻辑（如超时、重试、任务去重）都集中在 Master 处，易于管理和维护。

#### 缺点：

服务端成为瓶颈：Master 成了“单点”，所有人都来问它要任务，它的并发和容错能力（SPOF - Single Point of Failure）要求极高。

### 模式二：客户端管理状态（Stateless Server）- 我们的 Push 模式（第一个例子）
谁管理？ 请求方（Master）。

状态：Master（请求方）维护着任务列表，以及 Worker 的地址列表。它需要自己决定“任务1给Worker A，任务2给Worker B”。

服务端（Worker）：无状态。它不记得 Master，它只是被动地接收一个请求，处理，然后返回。

实例：

我们的第一个 Push 示例。

RESTful API (如 JWT 认证)：

Server (API) 是无状态的。它不记得你是否登录。

Client (App) 必须管理自己的状态（即保存 JWT Token），并在每一次请求时，都主动在 Header 中带上 Authorization: Bearer <token> 来证明自己的身份。

函数计算 (Serverless)：云函数本身是无状态的，它被调用时，需要调用方（或事件源）提供所有必需的上下文数据。

#### 优点：

服务端（Worker）极易扩展：因为 Worker 是无状态的，你可以轻松地从1个扩展到1000个，放在负载均衡器后面即可。

#### 缺点：

客户端（Master）逻辑复杂：Master 不仅要管理任务，还要管理 Worker 的“健康状态”和“地址列表”（即服务发现）。如果 Worker A 挂了，Master 怎么知道？它如何重试分配给 A 的任务？这非常复杂。

### 模式三：双方管理状态（Hybrid）
谁管理？ 双方都管理一部分。

实例：TCP 连接。

Server 和 Client 都必须维护一个关于连接状态（ESTABLISHED, FIN_WAIT, CLOSE_WAIT...）的“状态机”。如果一方的状态机与另一方不同步（例如一方认为连接已关闭，另一方还认为已连接），通信就会失败。

#### 优点：
适用于需要持久、双向通信的场景。

#### 缺点：
复杂度极高，状态同步是分布式系统中最难的问题之一。应尽量避免。

### 结论与最佳实践
对于“任务分发”这个场景：

最佳实践永远是“模式一”（服务端管理状态，即我们的 Pull 模式）。

让 Master 成为“状态的唯一事实来源 (Single Source of Truth)”，Worker 则是“无状态的劳动力 (Stateless Workforce)”。

这种架构的弹性是最好的：

Worker 挂了？ 没关系，Master 会通过超时机制把任务交给别人。

要加100个 Worker？ 没关系，启动它们，它们会自动去 Master 领活。

Master 挂了？ ……这才是你唯一需要担心的问题。你需要为 Master 做高可用（HA）集群（例如使用 Raft/Etcd 来保证 Master 状态的一致性），但这是另一个更高级的话题了。