---
title: go语言之http
description: 'Go 语言内置了功能强大的 `net/http` 包，用于构建 高性能 HTTP 服务 和 客户端请求'
tags: ['go']
toc: false
date: 2025-05-30 21:07:30
categories:
    - go
    - basic
---

Go 语言内置了功能强大的 `net/http` 包，用于构建 **高性能 HTTP 服务** 和 **客户端请求**。这是 Go 成为服务端开发热门语言的核心原因之一。

本文从背景、服务端开发、客户端请求、处理常见需求等多个方面 **系统讲解 Go HTTP 的相关知识与实际用法**

---

## 📌 一、背景介绍：Go HTTP 是什么？

Go 的 `net/http` 是标准库的一部分，包含：

| 模块                    | 说明            |
| --------------------- | ------------- |
| `http.Server`         | 用于创建 Web 服务器  |
| `http.Handler`        | 接口，处理 HTTP 请求 |
| `http.Client`         | 发起 HTTP 请求    |
| `http.Request`        | 客户端请求的信息封装    |
| `http.ResponseWriter` | 服务端写入响应的接口    |

**特性：**

* 开箱即用，不依赖第三方框架
* 支持中间件、自定义路由
* 支持并发请求（默认支持）
* 可配合 `context` 实现超时控制、取消等高级功能

---

## 🚀 二、服务端开发：快速搭建一个 HTTP 服务器

### ✅ 示例：创建一个基本 HTTP 服务

```go
package main

import (
    "fmt"
    "net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, Go HTTP!")
}

func main() {
    http.HandleFunc("/", helloHandler) // 注册路由
    fmt.Println("Listening on :8080")
    http.ListenAndServe(":8080", nil) // 启动服务器
}
```

### 📘 说明：

* `HandleFunc` 将 URL 路径 `/` 绑定到处理函数
* `http.ListenAndServe` 启动服务器并监听端口
* 每一个请求都会在 goroutine 中处理（自动并发）

---

## 🔄 三、自定义路由与多路径处理

```go
func aboutHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "This is the about page.")
}

func main() {
    http.HandleFunc("/", helloHandler)
    http.HandleFunc("/about", aboutHandler)
    http.ListenAndServe(":8080", nil)
}
```

---

## 🧱 四、自定义 `http.Handler` 实现更复杂逻辑

```go
type myHandler struct{}

func (h myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 任何 URL 都由这个 Handler 处理
    fmt.Fprintf(w, "Handled by custom handler: %s\n", r.URL.Path)
}

func main() {
    handler := myHandler{}                     // 实例化自定义 handler
    http.ListenAndServe(":8080", handler)      // 使用自定义 handler 启动服务
}

```

---

## 🧰 五、中间件机制（Logging、Auth 等）

```go
// 定义一个中间件函数，打印每次访问的路径
func logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("Received request: %s\n", r.URL.Path)
        next.ServeHTTP(w, r) // 调用下一个 handler
    })
}

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello with middleware!")
}

func main() {
    mux := http.NewServeMux()          // 创建多路复用器
    mux.HandleFunc("/", hello)         // 注册路由处理函数
    http.ListenAndServe(":8080", logger(mux)) // 启用中间件
}

```

---

## 🌐 六、HTTP 客户端请求（GET/POST）

### ✅ GET 请求

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func main() {
    resp, err := http.Get("https://httpbin.org/get") // 发起 GET 请求
    if err != nil {
        log.Fatal(err) // 请求失败则打印错误
    }
    defer resp.Body.Close() // 确保释放资源

    body, _ := io.ReadAll(resp.Body) // 读取响应体内容
    fmt.Println(string(body))        // 打印响应
}

```

### ✅ POST 请求

```go
resp, err := http.Post("https://httpbin.org/post", "application/json", bytes.NewBuffer([]byte(`{"name":"go"}`)))
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
```

---

## ⏱ 七、结合 context 处理超时与取消

```go
func slowHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // 获取请求上下文
    fmt.Println("Handler started")
    defer fmt.Println("Handler ended")

    select {
    case <-time.After(5 * time.Second):
        fmt.Fprintln(w, "Finished work")
    case <-ctx.Done(): // 客户端取消请求
        http.Error(w, "Request canceled", http.StatusRequestTimeout)
    }
}

```

---

## 🔒 八、生产场景推荐配置（含超时）

```go
func main() {
    srv := &http.Server{
        Addr:         ":8080",      // 服务监听端口
        Handler:      http.DefaultServeMux,
        ReadTimeout:  5 * time.Second, // 读请求最大时间
        WriteTimeout: 10 * time.Second, // 写响应最大时间
        IdleTimeout:  15 * time.Second, // Keep-alive 空闲连接最大时间
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello with timeout!")
    })

    log.Fatal(srv.ListenAndServe()) // 启动服务器
}
```

---

## ✅ 九、实际开发中 HTTP 最佳实践

| 实践           | 说明                                        |
| ------------ | ----------------------------------------- |
| ✅ 路由封装       | 使用 `ServeMux` 或第三方框架如 `chi`、`gorilla/mux` |
| ✅ 中间件设计      | 封装日志、认证、限流等                               |
| ✅ context 控制 | 控制取消、超时（防止内存泄露）                           |
| ✅ 合理使用缓存     | 设置 `Cache-Control` 头、ETag                 |
| ✅ 日志记录       | 打印响应耗时、状态码等                               |
| ✅ 优雅关闭       | 使用 `context` 和 `os/signal` 优雅退出           |

---