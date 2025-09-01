---
title: Go语言之调试利器Air
description: 'Air是一个专为 Go 应用打造的热重载工具。它能够在你保存代码时，自动帮你完成编译、重启应用的流程，让你专注于代码开发，无需手动操作重启程序。'
tags: ['go']
toc: false
date: 2025-07-26 23:46:46
categories:
    - go
    - basic
---

## 一、Air 是什么？

[Air](https://github.com/cosmtrek/air) 是一个专为 Go 应用打造的 **热重载工具**。它能够在你保存代码时，自动帮你完成编译、重启应用的流程，让你专注于代码开发，无需手动操作重启程序。

使用 Air，就像在 Node.js 世界中使用 nodemon、在前端使用 webpack 的 watch 模式，极大地提升了开发体验。

---

## 二、Air 的核心特性

* ✅ 自动监听 `.go` 文件变更并重启程序
* ✅ 支持自定义构建命令和路径
* ✅ 支持忽略特定目录或文件（如 `vendor/`、`test/`）
* ✅ 支持 `.toml` 格式配置文件，易读易写
* ✅ 无需修改代码，开箱即用

---

## 三、如何安装 Air？

### 方式一：使用 Go 安装（推荐）

```bash
go install github.com/cosmtrek/air@latest
```

注意：确保 `$GOPATH/bin` 或 `~/go/bin` 已加入环境变量 `$PATH`。

### 方式二：使用 Homebrew（macOS）

```bash
brew install air
```

---

## 四、快速上手

在你的 Go 项目根目录下，直接运行：

```bash
air
```

如果第一次运行，会自动生成 `.air.toml` 配置文件。

Air 会自动监听你的 Go 源文件（如 `main.go`），并在检测到变动后自动构建和运行程序。

---

## 五、自定义配置：`.air.toml`

Air 支持通过 `.air.toml` 文件来自定义行为，比如构建命令、忽略目录等。以下是一个常见的示例配置：

```toml
# .air.toml

[build]
  cmd = "go build -o tmp/app ."    # 构建命令
  bin = "tmp/app"                  # 编译后的二进制路径
  include_ext = ["go", "tpl", "html"]   # 监听这些扩展名
  exclude_dir = ["vendor", "tmp", "logs"]  # 忽略目录
  exclude_file = ["*_test.go"]     # 忽略文件
  delay = 1000                     # 防抖延迟（ms）

[log]
  time = true                      # 显示时间戳
```

然后运行：

```bash
air -c .air.toml
```

---

## 六、项目目录结构建议

```
your-project/
├── main.go
├── go.mod
├── .air.toml
└── ...
```

推荐在项目根目录放置 `.air.toml` 文件。

---

## 七、配合 Gin 等 Web 框架使用

Air 非常适合用于开发基于 Gin、Fiber、Echo 等 Web 框架的应用。只要你把入口文件设置正确（一般是 `main.go`），Air 就能帮你自动热重载，非常适合调试接口、Web 服务等项目。

---

## 八、Air vs 其他热重载工具

| 工具名           | 语言适配 | 热重载 | 配置支持    | 社区活跃 |
| ------------- | ---- | --- | ------- | ---- |
| **Air**       | Go   | ✅   | `.toml` | 非常活跃 |
| fresh         | Go   | ✅   | 简单      | 一般   |
| reflex        | 通用   | ✅   | 正则匹配    | 一般   |
| CompileDaemon | Go   | ✅   | 简单      | 一般   |

在 Go 项目中，**Air 是目前最推荐的选择**。

---