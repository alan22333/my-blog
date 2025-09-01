---
title: Go语言之context应用场景
description: 'go context的常见应用场景和最佳实践，其中很多在复杂业务中的才涉及到，按需阅读即可，例如学习最常用的数据库事务、优雅关闭等等'
tags: ['go']
toc: false
date: 2025-08-26 16:23:09
categories:
    - go
    - basic
---

# 0. 速览：牢记的五条铁律

1. `context.Context` 始终放在函数**第一个参数**。
2. **不要存进结构体字段**；逐层传递。
3. 派生了 `WithTimeout/WithCancel` 就**必须调用 `cancel()`**（循环里千万别 `defer`）。
4. **只**用 `WithValue` 传**小而只读**的请求元数据（traceID、locale、userID…）。
5. 取消是**协作式**的：在你的阻塞点 `select <-ctx.Done()`，不检查就不会停。

---

# 1) HTTP 请求全链路：入口统一时间预算 + 级联取消

## 场景

* 统一限制每个请求的耗时（比如 2s）。
* 下游（DB/外部 HTTP、缓存）全部遵循同一预算。
* 携带 traceID / userID 等请求范围元数据。

## 代码（handler → service → dao）

```go
// --- keys.go ---
package keys
type ctxKey string
const (
    KeyTraceID ctxKey = "traceID"
    KeyUserID  ctxKey = "userID"
)
func WithTraceID(ctx context.Context, id string) context.Context { return context.WithValue(ctx, KeyTraceID, id) }
func TraceID(ctx context.Context) (string, bool) { v, ok := ctx.Value(KeyTraceID).(string); return v, ok }

// --- handler.go ---
func GetProfileHandler(w http.ResponseWriter, r *http.Request) {
    // 1) 入口统一预算：2s
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    // 2) 附加请求元数据（traceID / userID）
    if reqID := r.Header.Get("X-Request-ID"); reqID != "" {
        ctx = keys.WithTraceID(ctx, reqID)
    }
    // 假设从鉴权中取到 userID
    ctx = context.WithValue(ctx, keys.KeyUserID, "u123")

    prof, err := svc.GetProfile(ctx, "u123")
    if err != nil {
        // 3) 细节：区分超时/用户取消/内部错误
        switch {
        case errors.Is(err, context.DeadlineExceeded):
            http.Error(w, "timeout", http.StatusGatewayTimeout)
            return
        case errors.Is(err, context.Canceled):
            // 客户端断开常见会导致这里出现；通常直接返回或写个 408/499（无标准常量）
            return
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
            return
        }
    }
    w.Header().Set("Content-Type", "application/json")
    _, _ = w.Write([]byte(prof))
}

// --- service.go ---
type Service struct {
    DAO *DAO
    Http *http.Client
}
func (s *Service) GetProfile(ctx context.Context, uid string) (string, error) {
    // 4) 可在 service 层“缩短”预算（不能延长）
    //    推荐先看现有剩余时间，避免不必要的二次定时器
    if dl, ok := ctx.Deadline(); ok && time.Until(dl) < 300*time.Millisecond {
        // 剩余时间太短，直接失败或降级
        return "", fmt.Errorf("no budget: %w", context.DeadlineExceeded)
    }

    // 5) 组合调用演示：并行查 DB + 外部画像服务（errgroup）
    g, ctx := errgroup.WithContext(ctx)
    var (
        base string
        enrich string
    )
    g.Go(func() error {
        v, err := s.DAO.QueryBase(ctx, uid)
        if err != nil { return err }
        base = v; return nil
    })
    g.Go(func() error {
        // 外部服务遵循 ctx：NewRequestWithContext
        req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/enrich?uid="+uid, nil)
        resp, err := s.Http.Do(req)
        if err != nil { return err }
        defer resp.Body.Close()
        b, _ := io.ReadAll(resp.Body)
        enrich = string(b)
        return nil
    })
    if err := g.Wait(); err != nil { return "", err }

    return fmt.Sprintf(`{"base":%q,"enrich":%q}`, base, enrich), nil
}

// --- dao.go ---
type DAO struct { DB *sql.DB }
func (d *DAO) QueryBase(ctx context.Context, uid string) (string, error) {
    // 6) 一定使用 *Context API
    row := d.DB.QueryRowContext(ctx, "SELECT name FROM users WHERE id=?", uid)
    var name string
    if err := row.Scan(&name); err != nil {
        return "", err
    }
    return name, nil
}
```

### 关键细节（最佳实践）

* **入口即预算**：在 `handler` 一次性设置；下游只“遵循”不再另设更长超时。
* **errgroup.WithContext**：任一子任务失败/取消会**自动取消其它**。
* **外部请求**用 `NewRequestWithContext`，DB 用 `QueryRowContext/ExecContext`。
* **错误判别**：使用 `errors.Is(err, context.DeadlineExceeded)` / `context.Canceled`。
* **不要假设 499 常量存在**（没有标准常量），想返回可用数字或直接中断。

---

# 2) 优雅关停（graceful shutdown）：服务整体的“父”上下文

## 场景

* 收到 `SIGINT/SIGTERM` 时停止接收新请求，让在途请求在 5s 内收尾。

## 代码

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() { _ = srv.ListenAndServe() }()

// Go 1.16+：直接用信号派生 Context
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()

<-ctx.Done() // 收到信号

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

_ = srv.Shutdown(shutdownCtx) // 等待在途完成；超时则强制关闭连接
```

**细节**

* 关闭顺序：**先**停止新连接（`Shutdown`），**再**等待在途完成。
* `Shutdown` 里会把 `r.Context()` 取消，下游协程应响应 `Done()`。
* 记录**最长耗时的在途请求**，以便调整 `Shutdown` 的预算。

---

# 3) 并发流水线与 worker：每一环都要“感知取消”

## 场景

* jobs → stage1 → stage2 → sink 的流水线；或一组 worker 从队列消费。

## 代码

```go
// 上游生产
func producer(ctx context.Context, out chan<- int) error {
    defer close(out)
    for i := 0; i < 100; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case out <- i:
        }
    }
    return nil
}

// 一个 stage（可并行开 N 份）
func stageDouble(ctx context.Context, in <-chan int, out chan<- int) error {
    defer close(out)
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case v, ok := <-in:
            if !ok { return nil }
            // 计算本身如耗时，也要支持 ctx（见 ctx-aware sleep）
            if err := Sleep(ctx, 10*time.Millisecond); err != nil {
                return err
            }
            out <- 2 * v
        }
    }
}

// ctx-aware Sleep（不要直接 time.Sleep）
func Sleep(ctx context.Context, d time.Duration) error {
    t := time.NewTimer(d)
    defer t.Stop()
    select {
    case <-t.C:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

**细节**

* **所有阻塞点**（读写通道、休眠、IO）都要能被 `ctx.Done()` 打断。
* `ticker`/`timer` 注意 `Stop()`，避免泄漏。
* 关闭通道的一端**由生产者**负责，避免 data race。

---

# 4) 带重试的外部调用（遵守剩余预算）

## 场景

* 对不稳定的外部依赖做重试 + 退避，但**不突破整体时间预算**。

## 代码

```go
func CallWithRetry(ctx context.Context, do func(context.Context) error) error {
    backoff := 100 * time.Millisecond
    for attempt := 0; ; attempt++ {
        // 每次尝试都用“子 ctx”，确保单次尝试不会拖垮总预算
        tryCtx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
        err := do(tryCtx)
        cancel()

        if err == nil { return nil }
        if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
            // 上层取消/无预算，或本次尝试已超时，检查总 ctx
            if ctx.Err() != nil {
                return ctx.Err()
            }
        }

        // 退避前先判断是否还有预算
        if deadline, ok := ctx.Deadline(); ok && time.Now().Add(backoff).After(deadline) {
            return fmt.Errorf("no time to retry: %w", err)
        }
        if err := Sleep(ctx, backoff); err != nil { return err }
        if backoff < 2*time.Second { backoff *= 2 }
    }
}
```

**细节**

* **单次尝试**设置小超时；**总预算**由外层 `ctx` 保证。
* 退避用 `Sleep(ctx, …)`，可随时被取消。
* 返回错误时保留 **原始原因**（可配合 `WithCancelCause`，见 §9）。

---

# 5) 数据库事务（`BeginTx`）与上下文传播

## 场景

* 事务内多次读写，任何一步失败或取消都要回滚。

## 代码

```go
func (d *DAO) Transfer(ctx context.Context, from, to int64, amt int64) error {
    tx, err := d.DB.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
    if err != nil { return err }
    defer func() { _ = tx.Rollback() }() // 安全回滚，成功时由 Commit 覆盖

    if _, err := tx.ExecContext(ctx, "UPDATE acct SET bal=bal-? WHERE id=?", amt, from); err != nil {
        return err
    }
    if _, err := tx.ExecContext(ctx, "UPDATE acct SET bal=bal+? WHERE id=?", amt, to); err != nil {
        return err
    }
    return tx.Commit()
}
```

**细节**

* `BeginTx` 用传入的 `ctx`，**整个事务**共享预算。
* `defer Rollback()` + 最后 `Commit()` 是惯用对。
* 发生取消/超时时，驱动应中断阻塞的等待（依赖具体 driver 实现）。

---

# 6) gRPC（服务端/客户端）与元数据

## 场景

* gRPC 天生携带 `context`；中间件读取/注入 trace、auth。

## 代码（客户端）

```go
ctx, cancel := context.WithTimeout(context.Background(), 800*time.Millisecond)
defer cancel()

md := metadata.Pairs("x-trace-id", "abc-123")
ctx = metadata.NewOutgoingContext(ctx, md)

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "u123"})
```

**细节**

* **把 HTTP 入站的 ctx** 原封不动传给 gRPC 出站，保证链路一致。
* 服务器端可在 interceptor 中读取 metadata 并写入 logger/trace。

---

# 7) CLI / 批处理：Ctrl+C 取消 + 子任务清理

## 场景

* 命令行工具长时间运行，用户 Ctrl+C 或系统发信号时**全局取消**。

## 代码

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()
    if err := run(ctx); err != nil {
        log.Fatal(err)
    }
}

func run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error { return taskA(ctx) })
    g.Go(func() error { return taskB(ctx) })
    return g.Wait()
}
```

**细节**

* `signal.NotifyContext` 是最简范式。
* 子任务**必须**遵循 `ctx`，否则无法“干净退出”。

---

# 8) 中间件：统一注入/读取 `Value`，但克制使用

## 场景

* 在 middleware 中解析鉴权、构造 logger/traceID，并把**小型只读**信息放到 `ctx`。

## 代码（示意）

```go
func InjectMetadata(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        if rid := r.Header.Get("X-Request-ID"); rid != "" {
            ctx = keys.WithTraceID(ctx, rid)
        }
        // 放轻量字段即可；logger 客体建议用依赖注入传参，不要塞进 ctx
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**细节**

* **不要**把可变大对象/配置/连接句柄塞 `ctx`。
* `Value` 仅用于**跨接口传递小元数据**；更复杂的依赖用构造函数/DI 注入。

---

# 9) 进阶：携带“取消原因”（Go 1.20+）

## 场景

* 希望上游知道“是谁/为什么取消”的更具体原因（比如熔断、限流）。

## 代码

```go
ctx, cancel := context.WithCancelCause(parent)
defer cancel(nil) // 正常结束时传 nil

// 某处触发
cancel(fmt.Errorf("rate limited by token bucket"))

// 上游读取
if cause := context.Cause(ctx); cause != nil {
    // 更细粒度记录：cause.Error()
}
```

**细节**

* `WithCancelCause` + `context.Cause(ctx)` 可精准定位来源。
* 仅在**链路诊断**确实需要时使用，避免过度复杂化。

---

# 10) 常见坑位对照表

| 场景                       | 错误做法                     | 正确做法                                               |
| ------------------------ | ------------------------ | -------------------------------------------------- |
| 循环里派生 `WithTimeout`      | `defer cancel()`（在循环里堆积） | `cancel()` 放到每轮末尾立即调用                              |
| 只 `time.After()` 等待      | 上层取消时还在“傻等”              | 用 `select { case <-t.C: …; case <-ctx.Done(): … }` |
| 在库里随意设置超时                | 与上层预算冲突                  | 入口统一预算；库层最多**缩短**预算                                |
| 传 `nil` context          | 下游 panic                 | 没有就 `context.Background()`/`TODO()`                |
| `Value` 传大对象/配置          | 变成“万能垃圾袋”                | 仅传小型只读元信息，依赖显式注入                                   |
| 想“立刻终止”协程                | 误解取消为抢占式                 | 在阻塞点检查 `ctx.Done()` 自己退出                           |
| 忘记 `timer/ticker.Stop()` | goroutine/资源泄漏           | `defer t.Stop()`/`defer tk.Stop()`                 |
| 事务里没用 `*Context` 方法      | 取消不生效                    | `BeginTx/ExecContext/QueryContext` 全用带 Context 版本  |

---

# 11) 可直接复用的小工具

```go
// 剩余预算检测：不足 min 时直接失败
func EnsureBudget(ctx context.Context, min time.Duration) error {
    if dl, ok := ctx.Deadline(); ok && time.Until(dl) < min {
        return context.DeadlineExceeded
    }
    return nil
}

// 在父 ctx 基础上“缩短”预算（如果父没有截止，则设置）：
func Tighten(ctx context.Context, d time.Duration) (context.Context, context.CancelFunc) {
    if dl, ok := ctx.Deadline(); ok {
        // 已有截止：取更早者
        if now := time.Now(); dl.Sub(now) < d { // 父更短
            // 不再额外创建，返回 no-op cancel
            return ctx, func() {}
        }
    }
    return context.WithTimeout(ctx, d)
}
```

---

# 12) 在你的三层项目里落地的“配方清单”

* **handler**：统一 `WithTimeout`（例如 2s）；在入站中间件注入 traceID/userID；**不再额外加更长超时**。
* **service**：并发：用 `errgroup.WithContext`；重试：`CallWithRetry`；任何阻塞点 `select ctx.Done()`；必要时 `Tighten` 缩短。
* **dao**：全部 `*Context` API；事务 `BeginTx` + `ExecContext/QueryContext`；错误上抛，不混淆 `context.Canceled/DeadlineExceeded`。
* **全局**：`signal.NotifyContext` 做优雅停机；日志里输出 `traceID`（从 `ctx` 读）。

---
