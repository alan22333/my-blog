---
title: Go语言之nunu学习记录
description: '学习nunu项目中的一些第三方包'
tags: ['go']
toc: false
date: 2025-08-29 09:35:44
categories:
    - go
    - project
---


关键知识点系统整理成 9 个章节：**cobra、survey、wire、模板 + embed、context、filepath、signal、fsnotify、常见 syscall**

---

## 目录

1. [Cobra：命令行框架与命令树](#cobra)
2. [Survey：交互式问答（选择/输入/校验）](#survey)
3. [Wire：依赖注入与对象装配](#wire)
4. [模板 + embed：把模板打包进二进制并生成文件](#embed)
5. [Context：取消、超时与并发协作](#context)
6. [filepath：跨平台路径与文件遍历](#filepath)
7. [signal：优雅退出与中断处理](#signal)
8. [fsnotify：文件变更监听与热更新](#fsnotify)
9. [常见 syscall：低层能力与高层替代建议](#syscall)

---

<a id="cobra"></a>

## 1. Cobra：命令行框架与命令树

**它是啥**：`spf13/cobra` 是 Go 里最常用的 CLI 框架，支持子命令、旗标（flags）、自动补全、帮助文档等。典型结构是一个 `root` 根命令挂一串子命令（`new`, `init`, `build`, `dev` 等）。

**脚手架中的用途**：

- `root`：显示版本/全局 flags（例如 `--verbose`, `--config`）。
- 子命令：
  - `new`/`init`：交互式生成项目骨架（结合 survey + embed 模板）。
  - `dev`：本地开发（结合 fsnotify 监听变化、signal 优雅退出）。
  - `build`：打包产物（结合 context 控时与并发）。

### 快速上手目录结构

```
mycli/
  cmd/
    root.go
    new.go
  internal/
    generator/
      generator.go
  main.go
  go.mod
```

### 代码示例

**main.go**

```go
package main

import "mycli/cmd"

func main() { cmd.Execute() }
```

**cmd/root.go**

```go
package cmd

import (
    "fmt"
    "github.com/spf13/cobra"
)

var (
    cfgFile string
    verbose bool
)

var rootCmd = &cobra.Command{
    Use:   "mycli",
    Short: "A modern project scaffolding CLI",
    Long:  `mycli 是一个用于快速生成 Go 项目骨架的 CLI 工具。`,
}

func Execute() { _ = rootCmd.Execute() }

func init() {
    rootCmd.PersistentFlags().StringVarP(&cfgFile, "config", "c", "", "配置文件路径")
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "输出更多日志")
}
```

**cmd/new.go**

```go
package cmd

import (
    "context"
    "fmt"
    "time"

    "github.com/spf13/cobra"
)

var projectName string

var newCmd = &cobra.Command{
    Use:   "new",
    Short: "创建新项目",
    RunE: func(cmd *cobra.Command, args []string) error {
        // 给每个子命令一个可取消的 context
        ctx, cancel := context.WithTimeout(cmd.Context(), 30*time.Second)
        defer cancel()
        fmt.Println("✨ 创建项目:", projectName)
        // TODO: 调用 generator 生成模板
        _ = ctx
        return nil
    },
}

func init() {
    rootCmd.AddCommand(newCmd)
    newCmd.Flags().StringVarP(&projectName, "name", "n", "demo", "项目名称")
}
```

### 小技巧与坑

- **PersistentFlags vs Flags**：前者对下游子命令可见；后者只对当前命令生效。
- **命令上下文**：Cobra 1.7+ 支持 `cmd.Context()`，便于和 `context` 打通（超时/取消）。
- **自动补全**：可用 `cobra.GenBashCompletion()` 等生成脚本。
- **PreRun/PostRun**：做参数校验/统一的前后置逻辑。

---

<a id="survey"></a>

## 2. Survey：交互式问答（选择/输入/校验）

**它是啥**：`AlecAivazis/survey/v2` 提供人性化的命令行交互组件：输入、单选、多选、编辑器、密码、确认等。

**脚手架中的用途**：在 `mycli new` 时向用户询问模块名、所选框架、是否启用数据库、License 等，再把答案灌入模板。

### 常用题型与校验

```go
package prompt

import (
    "fmt"
    survey "github.com/AlecAivazis/survey/v2"
)

type Answers struct {
    Name    string
    License string
    Features []string
}

func Ask() (*Answers, error) {
    a := &Answers{}

    qName := &survey.Input{Message: "项目名:"}
    if err := survey.AskOne(qName, &a.Name, survey.WithValidator(survey.Required)); err != nil { return nil, err }

    qLicense := &survey.Select{
        Message: "选择 License:",
        Options: []string{"MIT", "Apache-2.0", "GPL-3.0"},
        Default: "MIT",
    }
    if err := survey.AskOne(qLicense, &a.License); err != nil { return nil, err }

    qFeatures := &survey.MultiSelect{
        Message: "启用特性:",
        Options: []string{"HTTP API", "CLI", "DB(MySQL)", "Config(viper)"},
    }
    if err := survey.AskOne(qFeatures, &a.Features); err != nil { return nil, err }

    fmt.Printf("\n> 你的选择：%+v\n", *a)
    return a, nil
}
```

**最佳实践**

- 使用 `survey.WithValidator` 做**必填**、正则校验、长度限制等。
- `AskOne` 适合单题，`survey.Ask` 适合集合题并支持 struct tag：`survey:"field"`。
- Windows 终端兼容性较好，但遇到中文输入法问题可回退到基础 `fmt` 输入。

---

<a id="wire"></a>

## 3. Wire：依赖注入与对象装配

**它是啥**：`google/wire` 是**编译期**依赖注入工具。通过生成代码把构造函数（providers）组合起来，避免手写装配过程，既类型安全又清晰。

**脚手架中的用途**：在 `new`/`build` 子命令里，用 Wire 统一装配 `Logger`、`Config`、`Generator`、`Renderer` 等依赖；让业务代码只关心接口。

### 基本示例

**internal/generator/generator.go**

```go
package generator

import (
    "fmt"
)

type Config struct { ProjectName string }

type Logger interface { Infof(string, ...any) }

type SimpleLogger struct{}
func (SimpleLogger) Infof(f string, a ...any) { fmt.Printf(f+"\n", a...) }

// 业务对象
 type Generator struct {
    cfg Config
    log Logger
}

func NewConfig(name string) Config { return Config{ProjectName: name} }
func NewLogger() Logger { return SimpleLogger{} }
func NewGenerator(cfg Config, log Logger) *Generator { return &Generator{cfg: cfg, log: log} }

func (g *Generator) Run() { g.log.Infof("generate %s ...", g.cfg.ProjectName) }
```

**internal/generator/wire.go**

```go
//go:build wireinject
// +build wireinject

package generator

import "github.com/google/wire"

func InitGenerator(projectName string) *Generator {
    wire.Build(NewConfig, NewLogger, NewGenerator)
    return nil
}
```

**internal/generator/wire_gen.go**（生成文件，勿手写）

```go
// 运行: go run github.com/google/wire/cmd/wire@latest ./...
// 生成后会出现本文件，里面把 providers 串起来。
```

**使用**

```go
package main

import "mycli/internal/generator"

func main() {
    g := generator.InitGenerator("demo")
    g.Run()
}
```

**最佳实践与坑**

- `wire` 通过 `go:build wireinject` 隔离注入入口，生成物落到同目录 `wire_gen.go`。
- Provider 要么是构造函数（`func NewX(...) (*X, error)`），要么是值/接口绑定（`wire.Bind`）。
- 不要在 provider 中做 I/O 副作用，把副作用放到运行时方法里，便于测试。

---

<a id="embed"></a>

## 4. 模板 + embed：把模板打包进二进制并生成文件

**它是啥**：Go 1.16+ 的 `embed` 允许在编译时把静态资源（模板、脚本、README 等）打到二进制里，无需在用户机器上再下载。结合 `text/template` 或 `tmpl` 系统即可渲染生成项目。

### 目录与代码

```
templates/
  go.mod.tmpl
  main.go.tmpl
  README.md.tmpl
```

**internal/tpl/tpl.go**

```go
package tpl

import (
    "embed"
    "fmt"
    "io/fs"
    "os"
    "path/filepath"
    "strings"
    "text/template"
)

//go:embed templates/*
var templatesFS embed.FS

var funcMap = template.FuncMap{
    "ToLower": strings.ToLower,
}

// RenderDir 把 embed 中的模板渲染到目标目录
func RenderDir(dst string, data any) error {
    return fs.WalkDir(templatesFS, "templates", func(path string, d fs.DirEntry, err error) error {
        if err != nil { return err }
        if d.IsDir() { return nil }
        rel, _ := filepath.Rel("templates", path)
        // 去掉 .tmpl 后缀
        out := filepath.Join(dst, strings.TrimSuffix(rel, ".tmpl"))
        if err := os.MkdirAll(filepath.Dir(out), 0o755); err != nil { return err }
        b, err := fs.ReadFile(templatesFS, path)
        if err != nil { return err }
        t, err := template.New(rel).Funcs(funcMap).Parse(string(b))
        if err != nil { return err }
        f, err := os.OpenFile(out, os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0o644)
        if err != nil { return err }
        defer f.Close()
        if err := t.Execute(f, data); err != nil { return err }
        fmt.Println("+", out)
        return nil
    })
}
```

**在 `new` 命令里调用**

```go
// data 可以来自 survey 的答案
_ = tpl.RenderDir(projectName, struct{ Name string }{Name: projectName})
```

**最佳实践**

- 模板文件建议以 `.tmpl` 结尾，避免和真实文件冲突。
- 注意 Windows 的换行符与可执行权限（用 `0o755`/`0o644` 明确文件权限）。
- 复杂模板建议拆函数：使用 `template.FuncMap` 自定义方法（如驼峰/蛇形转换）。

---

<a id="context"></a>

## 5. Context：取消、超时与并发协作

**它是啥**：`context` 通过树状的截止时间、取消信号、键值对，在 goroutine 之间传递“**应该停止了**”的信息。

比较麻烦，另外再出文章

---

<a id="filepath"></a>

## 6. filepath：跨平台路径与文件遍历

**它是啥**：标准库 `path/filepath` 处理**本地文件系统**路径（区分 `\` 与 `/`、盘符、符号链接）。

**脚手架中的用途**：创建项目目录、相对路径、模板展开、忽略列表等。

### 常用 API 速查

```go
p := filepath.Join("/home", "user", "project") // 拼接
abs, _ := filepath.Abs("./..")                     // 绝对路径
rel, _ := filepath.Rel("/home/user", "/home/user/project/app") // 相对
clean := filepath.Clean("/a/../b//c/")            // 规范化
match, _ := filepath.Match("*.go", "main.go")     // 简单匹配
files, _ := filepath.Glob("**/*.tmpl")            // 通配

// 遍历（优先用 WalkDir，开销更小）
filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error { return nil })
```

**符号链接与真实路径**

```go
real, err := filepath.EvalSymlinks(path)
```

**最佳实践**

- 统一使用 `filepath`（**不是** `path`）处理本地路径。
- 用户输入的路径先 `Clean` 再使用，避免路径穿越。
- 大型目录遍历时考虑忽略 `.git`、`node_modules`、`vendor` 等目录。

---

<a id="signal"></a>

## 7. signal：优雅退出与中断处理

**它是啥**：`os/signal` 捕获来自操作系统的中断信号（如 Ctrl+C -> `SIGINT`、`SIGTERM`），用于**优雅关闭**。

**脚手架中的用途**：`dev` 子命令启动本地进程/服务器时，用户按下 Ctrl+C，停止 watcher、杀子进程并清理临时文件。

### 典型用法（与 Context 打通）

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
if err := run(ctx); err != nil { log.Fatal(err) }
```

**示例：启动子进程并随信号退出**

```go
func run(ctx context.Context) error {
    cmd := exec.CommandContext(ctx, "go", "run", ".")
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    // 若需要把子进程放入新进程组，便于整组杀死（类 Unix）
    if runtime.GOOS != "windows" {
        cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
    }
    if err := cmd.Start(); err != nil { return err }
    // 等待或被 ctx 取消
    return cmd.Wait()
}
```

**最佳实践**

- 用 `signal.NotifyContext` 替代手动 channel。
- 若有**多个**后台任务，用 `errgroup` 收口，任一失败触发取消。
- Windows 上信号语义不同（没有 `SIGTERM`），可退化为 `os.Interrupt`。

---

<a id="fsnotify"></a>

## 8. fsnotify：文件变更监听与热更新

**它是啥**：`fsnotify/fsnotify` 提供跨平台的文件系统事件监听（Create/Write/Rename/Remove/Chmod）。

**脚手架中的用途**：监听模板目录或配置文件，一旦变更则重新生成或重载。

### 最小示例（带防抖）

```go
package watch

import (
    "log"
    "path/filepath"
    "time"

    "github.com/fsnotify/fsnotify"
)

func WatchDir(dir string, onChange func()) error {
    w, err := fsnotify.NewWatcher()
    if err != nil { return err }
    defer w.Close()

    // 递归监听
    add := func(d string) error {
        return filepath.WalkDir(d, func(path string, de fs.DirEntry, err error) error {
            if err != nil { return err }
            if de.IsDir() { return w.Add(path) }
            return nil
        })
    }
    if err := add(dir); err != nil { return err }

    timer := time.NewTimer(time.Hour)
    timer.Stop()

    for {
        select {
        case e := <-w.Events:
            log.Println("fs: ", e)
            // 防抖：聚合 300ms 内的事件
            timer.Reset(300 * time.Millisecond)
        case <-timer.C:
            onChange()
        case err := <-w.Errors:
            return err
        }
    }
}
```

**最佳实践**

- 监听**目录**优于单文件；Rename/Remove 可能导致监听失效，需要**重新 Add**。
- 合理的**防抖**与**节流**，避免重复构建/重载。
- Linux inotify、Mac FSEvents、Windows ReadDirectoryChangesW 行为略有差异，注意测试。

---

<a id="syscall"></a>

## 9. 常见 syscall：低层能力与高层替代建议

**它是啥**：`syscall` 暴露系统调用接口；但 **Go 官方建议**：新代码尽量使用更高层的标准库或 `golang.org/x/sys/*`（如 `x/sys/unix`）。

**脚手架中的用途**：少量场景需要低层控制，例如：

- 设置进程组（便于一键终止子进程组）。
- 文件锁（防并发执行两个生成器实例）。
- 修改文件权限/Umask。

### 示例 1：文件锁（类 Unix）

```go
// go:build !windows
package lock

import (
    "fmt"
    "os"
    "golang.org/x/sys/unix"
)

type FileLock struct { f *os.File }

func TryLock(path string) (*FileLock, error) {
    f, err := os.OpenFile(path, os.O_CREATE|os.O_RDWR, 0o644)
    if err != nil { return nil, err }
    if err := unix.Flock(int(f.Fd()), unix.LOCK_EX|unix.LOCK_NB); err != nil {
        f.Close(); return nil, fmt.Errorf("another process holds the lock: %w", err)
    }
    return &FileLock{f: f}, nil
}

func (l *FileLock) Unlock() error {
    defer l.f.Close()
    return unix.Flock(int(l.f.Fd()), unix.LOCK_UN)
}
```

### 示例 2：设置进程组 + 杀全组（类 Unix）

```go
cmd := exec.Command("your-server")
cmd.SysProcAttr = &syscall.SysProcAttr{Setpgid: true}
_ = cmd.Start()
// 终止整个进程组
pgid, _ := syscall.Getpgid(cmd.Process.Pid)
_ = syscall.Kill(-pgid, syscall.SIGTERM) // 负号表示进程组
```

**高层替代建议表**

| 需求      | 建议优先使用                                   |
| --------- | ---------------------------------------------- |
| 进程信号  | `os/signal`, `exec.CommandContext`             |
| 文件权限  | `os.OpenFile` + 明确 `0o644/0o755`；`os.Chmod` |
| 文件锁    | `x/sys/unix`.Flock 或 跨平台第三方库           |
| 定时/取消 | `context` + `time`                             |

> ⚠️ 注意：Windows 上没有 POSIX 信号/锁的完全等价物，需用平台特定方案。
