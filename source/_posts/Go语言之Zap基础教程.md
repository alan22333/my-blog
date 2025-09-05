---
title: Go语言之Zap基础教程
description: 'Zap 是 Uber 开源的一个 Go 语言日志库，是目前性能最高的开源日志框架'
tags: ['go']
toc: false
date: 2025-09-05 16:03:37
categories:
    - go
    - basic
---

### Zap 日志框架基本教程

Zap 是 Uber 开源的一个 Go 语言日志库，以其超高的性能和可扩展性著称。它在设计时就考虑到了性能，比 Go 标准库的 `log` 或其他一些流行的日志库（如 `logrus`）快得多。如果你正在构建一个对性能有严格要求的应用，或者处理高并发请求，Zap 绝对是一个值得考虑的选择。

#### 1\. 为什么选择 Zap？

  * **极高的性能：** Zap 的设计核心就是性能。它在分配内存、格式化日志方面都做了优化，特别是在生产环境中，能够显著减少 CPU 和内存的开销。
  * **结构化日志：** Zap 鼓励使用结构化的键值对（key-value pairs）记录日志，而不是简单的字符串。这使得日志更易于机器解析、过滤和分析。
  * **可定制化：** 你可以轻松地配置 Zap 的输出格式（JSON 或文本）、日志级别、输出目标（文件、控制台等）。

#### 2\. 安装 Zap

使用 Go 的包管理工具 `go get` 来安装 Zap：

```bash
go get -u go.uber.org/zap
```

#### 3\. 基本用法

Zap 提供了两种主要的日志记录器：`SugaredLogger` 和 `Logger`。

  * `Logger`：这是高性能、非分配的日志记录器。它要求你以严格的键值对形式传递字段，性能最高。
  * `SugaredLogger`：这是 `Logger` 的一个包装，提供了更方便的 API，允许你使用 `printf` 风格的格式化字符串。它比 `Logger` 慢一点，但仍然比其他许多日志库快得多。

在大多数情况下，**`SugaredLogger`** 已经足够快且更易于使用，尤其是在非性能敏感的地方。

##### 3.1 简单的 SugaredLogger 示例

```go
package main

import (
	"go.uber.org/zap"
)

func main() {
	// 创建一个开发环境的日志记录器
	logger, _ := zap.NewDevelopment()
	defer logger.Sync() // 确保所有缓冲的日志都写入了

	// 转换为 SugaredLogger
	sugar := logger.Sugar()

	// 记录不同级别的日志
	sugar.Debug("这是一个 debug 消息")
	sugar.Info("这是一个 info 消息", "用户名", "Alice")
	sugar.Warn("这是一个 warning 消息", "内存占用", "90%")
	sugar.Error("这是一个 error 消息", "错误代码", 500)
}
```

运行这段代码，你会看到类似下面的 JSON 格式输出：

```json
{"level":"info","ts":1672531200.000000,"caller":"main/main.go:12","msg":"这是一个 info 消息","用户名":"Alice"}
{"level":"warn","ts":1672531200.000000,"caller":"main/main.go:13","msg":"这是一个 warning 消息","内存占用":"90%"}
{"level":"error","ts":1672531200.000000,"caller":"main/main.go:14","msg":"这是一个 error 消息","错误代码":500}
```

注意，`zap.NewDevelopment()` 会默认以 **JSON 格式** 输出，并且包含 `caller`（调用者）等信息，非常适合开发环境调试。

##### 3.2 高性能的 Logger 示例

如果你需要极致的性能，可以直接使用 `Logger`。

```go
package main

import (
	"go.uber.org/zap"
)

func main() {
	// 创建一个生产环境的日志记录器
	logger, _ := zap.NewProduction()
	defer logger.Sync()

	// 使用 Logger 记录
	logger.Info("用户登录",
		zap.String("用户名", "Bob"),
		zap.Int("用户ID", 123),
		zap.Bool("成功", true),
	)

	// 使用 Logger 记录错误
	err := "这是一个模拟错误"
	logger.Error("数据库连接失败",
		zap.Error(err),
		zap.Int("重试次数", 3),
	)
}
```

`Logger` 的方法（如 `Info`, `Error`）需要你使用 `zap.Field` 类型来传递键值对。Zap 提供了多种 `zap.Xxx` 字段类型来处理不同类型的数据，例如 `zap.String`、`zap.Int`、`zap.Error` 等，这在性能上更优，因为它避免了反射和类型断言的开销。

-----

#### 4\. 配置日志记录器

Zap 提供了 `zap.Config` 结构体来让你更细粒度地控制日志记录器的行为。这是在生产环境中配置日志的最佳方式。

```go
package main

import (
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

func main() {
	// 1. 定义一个配置
	cfg := zap.Config{
		Level:             zap.NewAtomicLevelAt(zap.InfoLevel), // 设置日志级别为 INFO
		Development:       false,                                // 生产环境模式
		DisableCaller:     true,                                 // 禁用 caller
		DisableStacktrace: true,                                 // 禁用堆栈跟踪
		Encoding:          "json",                               // 输出格式为 JSON
		EncoderConfig: zapcore.EncoderConfig{
			TimeKey:        "ts",
			LevelKey:       "level",
			NameKey:        "logger",
			CallerKey:      "caller",
			MessageKey:     "msg",
			StacktraceKey:  "stacktrace",
			LineEnding:     zapcore.DefaultLineEnding,
			EncodeLevel:    zapcore.LowercaseLevelEncoder,
			EncodeTime:     zapcore.ISO8601TimeEncoder,
			EncodeDuration: zapcore.SecondsDurationEncoder,
			EncodeCaller:   zapcore.ShortCallerEncoder,
		},
		OutputPaths:      []string{"stdout"}, // 日志输出到标准输出
		ErrorOutputPaths: []string{"stderr"}, // 错误日志输出到标准错误
	}

	// 2. 根据配置创建日志记录器
	logger, err := cfg.Build()
	if err != nil {
		panic(err)
	}
	defer logger.Sync()

	logger.Info("服务启动成功", zap.String("版本", "1.0"))
	logger.Error("数据库连接失败", zap.Int("重试次数", 3))
}
```

这个例子展示了如何通过 `zap.Config` 来：

  * **设置日志级别：** 使用 `zap.NewAtomicLevelAt(zap.InfoLevel)` 来设置最低日志级别。所有低于此级别的日志（如 `Debug`）都将被忽略。
  * **选择编码器：** `Encoding` 可以设置为 `"json"` 或 `"console"`。
  * **自定义输出格式：** `EncoderConfig` 允许你精确控制 JSON 或控制台输出的键名和格式。
  * **定义输出目标：** `OutputPaths` 可以是文件路径、`stdout` 或 `stderr`。你可以将日志同时输出到多个目标，例如 `[]string{"stdout", "/var/log/my-app.log"}`。

-----

### 总结与建议

  * **开发环境：** 使用 `zap.NewDevelopment()`，它提供了友好的控制台输出和调试信息，非常方便。
  * **生产环境：** 使用 `zap.NewProduction()` 或通过 `zap.Config` 来进行精细配置。推荐使用 JSON 格式输出，这对于日志聚合服务（如 ELK Stack 或 Loki）来说更易于处理和分析。
  * **性能要求：**
      * **常规使用：** `SugaredLogger` 已经提供了足够的性能，且 API 更简单，适合大多数项目。
      * **高性能/高并发场景：** 优先使用 `Logger`，并尽可能使用 `zap.Xxx` 类型来记录结构化数据，以获得最佳性能。

Zap 是一个功能强大且性能卓越的日志库。通过本教程，你应该已经掌握了它的基本用法和配置方法，可以开始在自己的项目中使用了。