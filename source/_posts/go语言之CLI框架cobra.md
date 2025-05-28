---
title: go语言之CLI框架：cobra
description: 'Cobra 是一个用于创建强大现代 CLI 应用程序的 Go 库，许多知名项目如 Kubernetes、Docker 和 Hugo 都使用它。'
tags: ['go']
toc: false
date: 2025-05-25 08:36:31
categories:
    - go
    - basic
---


## 📦 第一步：安装 Cobra CLI 工具

```bash
go install github.com/spf13/cobra-cli@latest
```

确保 `$GOPATH/bin` 在你的 `PATH` 中，然后你可以运行：

```bash
cobra-cli --help
```

---

## 🧱 第二步：初始化项目

创建你的 Go 项目目录：

```bash
mkdir tasker && cd tasker
go mod init github.com/yourname/tasker
```

使用 `cobra-cli` 初始化：

```bash
cobra-cli init --pkg-name github.com/yourname/tasker
```

这会生成如下结构：

```
tasker/
├── cmd/
│   └── root.go
├── main.go
```

---

## 🚀 第三步：运行项目

在当前目录运行：

```bash
go run main.go
```

输出将会是：

```
tasker is a CLI application
```

你可以查看帮助：

```bash
go run main.go --help
```

---

## ➕ 第四步：添加子命令

### 示例：添加 `add` 命令

```bash
cobra-cli add add
```

编辑 `cmd/add.go`：

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var addCmd = &cobra.Command{
	Use:   "add [task]",
	Short: "添加一个新任务",
	Args:  cobra.ExactArgs(1),
	Run: func(cmd *cobra.Command, args []string) {
		task := args[0]
		fmt.Printf("添加任务: %s\n", task)
	},
}

func init() {
	rootCmd.AddCommand(addCmd)
}
```

现在你可以运行：

```bash
go run main.go add "写 Cobra 教程"
```

---

## 📋 第五步：添加 `list` 命令

```bash
cobra-cli add list
```

编辑 `cmd/list.go`：

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var tasks = []string{}

var listCmd = &cobra.Command{
	Use:   "list",
	Short: "列出所有任务",
	Run: func(cmd *cobra.Command, args []string) {
		if len(tasks) == 0 {
			fmt.Println("没有任务")
			return
		}
		for i, task := range tasks {
			fmt.Printf("%d. %s\n", i+1, task)
		}
	},
}

func init() {
	rootCmd.AddCommand(listCmd)
}
```

### 📝 注意：数据结构在内存中，不持久化。可用 `map[string]string` 或写入文件模拟更复杂逻辑。

---

## 🎛 第六步：添加标志（flags）

比如我们想让 `add` 命令带一个 `--priority` 选项：

在 `add.go` 的 `init()` 函数中添加：

```go
addCmd.Flags().StringP("priority", "p", "normal", "任务优先级")
```

并在 `Run` 中读取：

```go
priority, _ := cmd.Flags().GetString("priority")
fmt.Printf("添加任务: %s（优先级: %s）\n", task, priority)
```

示例运行：

```bash
go run main.go add "学习 Go" --priority high
```

---

## 🧼 第七步：美化和构建项目

构建可执行文件：

```bash
go build -o tasker
./tasker --help
```

可以将 `tasker` 添加到系统 PATH 中，变成全局命令行工具。

---

## 📦 最终项目结构

```
tasker/
├── cmd/
│   ├── add.go
│   ├── list.go
│   └── root.go
├── go.mod
└── main.go
```

---
