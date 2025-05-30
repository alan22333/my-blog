---
title: go语言之日志
description: ''
tags: ['go']
toc: false
date: 2025-05-30 18:38:52
categories:
    - go
    - basic
---

Go 语言内置了基本的日志库 `log`，自 Go 1.21 起又引入了更现代、结构化的日志库 `log/slog`
<!--more-->

## 🧱 一、标准库 `log` 简介

Go 标准库中的 `log` 提供了基本的同步日志功能。适合简单项目或单机程序，特点是：

* 支持日志前缀（Prefix）
* 支持设置时间戳、文件路径等输出信息（通过 `SetFlags`）
* 支持自定义输出目的地（Writer）

### 常用方法

```go
log.Print("普通日志")
log.Println("自动换行")
log.Printf("格式化日志: %d", 100)
log.Fatal("打印后调用 os.Exit(1)")
log.Panic("打印后 panic()")
```

### 设置日志属性

```go
log.SetPrefix("DEBUG: ")                     // 设置日志前缀
log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile) // 显示日期、时间、文件名:行号
log.SetOutput(os.Stdout)                     // 改变输出目标（默认是 stderr）
```

---

## 💡 二、自定义日志记录器 (`log.New`)

用于模块化日志输出、写入文件或缓存等：

```go
file, _ := os.Create("app.log")
logger := log.New(file, "APP: ", log.LstdFlags|log.Lshortfile)

logger.Println("hello from logger")
```

---

## 🌟 三、结构化日志库 `log/slog`（Go 1.21+）

### 特点

* 支持 **JSON 格式** 输出
* 支持 **键值对结构化日志**
* 可插拔的日志后端（可用于发送到远程、文件、服务等）

### 示例：基本用法

```go
import (
    "log/slog"
    "os"
)

func main() {
    handler := slog.NewJSONHandler(os.Stdout, nil)
    logger := slog.New(handler)

    logger.Info("user logged in", "userID", 1234, "ip", "192.168.1.1")
}
```

输出如下（JSON 格式）：

```json
{"time":"2025-05-30T12:00:00Z","level":"INFO","msg":"user logged in","userID":1234,"ip":"192.168.1.1"}
```

---

## 📁 四、最佳实践：构建通用日志模块（推荐用于实际项目）

### `logger/logger.go`

```go
package logger

import (
    "log/slog"
    "os"
)

var Log *slog.Logger

func Init(env string) {
    var handler slog.Handler

    if env == "dev" {
        handler = slog.NewTextHandler(os.Stdout, nil)
    } else {
        handler = slog.NewJSONHandler(os.Stdout, nil)
    }

    Log = slog.New(handler)
}
```

### 使用日志模块

```go
// main.go
package main

import (
    "yourapp/logger"
)

func main() {
    logger.Init("dev")
    logger.Log.Info("server started", "port", 8080)
    logger.Log.Error("failed to connect DB", "error", "timeout")
}
```

---

## 🛠️ 五、进阶用法与建议

### 写入文件（使用 `io.MultiWriter`）

```go
file, _ := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
multi := io.MultiWriter(os.Stdout, file)
log.SetOutput(multi)
```

### 加入日志级别（标准库不支持，需要自己实现或用 slog）

```go
log.Println("[INFO] service started")
log.Println("[ERROR] database unavailable")
```

### 使用第三方库（如 zerolog、zap）

```go
// 使用 zap
logger, _ := zap.NewProduction()
defer logger.Sync()
logger.Info("hello zap", zap.String("user", "Alice"), zap.Int("age", 25))
```

---

## 📌 六、推荐日志策略（开发建议）

| 场景         | 建议做法                                        |
| ---------- | ------------------------------------------- |
| 本地开发       | 使用 `slog.TextHandler`，方便阅读                  |
| 生产环境       | 使用 `slog.JSONHandler`，适合日志采集与分析工具（ELK、Loki） |
| 多模块项目      | 自定义日志模块，统一日志格式与等级                           |
| 需要性能或日志量极大 | 使用 `zap`（Uber）或 `zerolog`（高性能）              |
| 日志等级控制     | `slog.Level` 支持 DEBUG、INFO、WARN、ERROR 可设置过滤 |
| 测试中使用      | 写入 `bytes.Buffer`，便于断言日志输出                  |

---

## ✅ 总结

| 日志方式          | 优点              | 缺点              |
| ------------- | --------------- | --------------- |
| `log`         | 简单快速，内置         | 无结构化、不支持等级      |
| `log.New`     | 可自定义输出          | 管理复杂、功能有限       |
| `log/slog`    | 结构化日志、JSON/文本输出 | Go 1.21+ 限制     |
| `zap/zerolog` | 高性能、等级控制、生产级    | 引入外部依赖，API 学习成本 |

---
