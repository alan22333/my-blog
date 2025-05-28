---
title: go语言之恐慌与恢复
description: 'panic 和 recover 机制提供了异常处理能力，但 Go 社区普遍认为应谨慎使用'
tags: ['go']
toc: false
date: 2025-05-28 10:03:51
categories:
    - go
    - basic
---

# Go 语言中 panic 和 recover 的使用准则

在 Go 语言中，panic 和 recover 机制提供了异常处理能力，但 Go 社区普遍认为应谨慎使用。以下是关于何时使用 panic 和 error 的原则：

## 主要原则：优先使用 error

在大多数情况下，应使用返回 error 的方式而非 panic，原因如下：

- **显式错误处理**：调用者可明确知道哪些操作可能失败，并决定如何处理。
- **代码可读性**：错误处理流程清晰可见。
- **控制流明确**：避免意外的程序终止。
- **符合 Go 语言哲学**：Go 鼓励显式错误处理而非异常机制。

## 适合使用 panic 的场景

以下情况可考虑使用 panic：

### 1. 不可恢复的错误

程序遇到无法继续执行的严重问题：

```go
if !criticalCondition {
    panic("系统处于不可恢复状态")
}
```

### 2. 程序初始化阶段的错误

如配置文件缺失、数据库连接失败等：

```go
func loadConfig() {
    if configFile == "" {
        panic("配置文件路径不能为空")
    }
    // ...
}
```

### 3. 编程错误（bug）

如违反接口约定、不可达的代码路径等：

```go
switch value := v.(type) {
case int:
    // 处理int
case string:
    // 处理string
default:
    panic(fmt.Sprintf("未处理的类型: %T", v))
}
```

### 4. 在测试中标记测试失败

```go
func TestSomething(t *testing.T) {
    if result != expected {
        panic("测试失败")
    }
}
```

## 使用 recover 的场景

recover 主要用于：

### 1. 防止 goroutine 崩溃影响整个程序

```go
func safeDo(f func()) {
    defer func() {
        if r := recover(); r != nil {
            log.Println("捕获到panic:", r)
        }
    }()
    f()
}
```

### 2. 顶级 HTTP 处理器中捕获 panic

```go
func handler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            http.Error(w, "内部服务器错误", http.StatusInternalServerError)
            log.Printf("处理器panic: %v", err)
        }
    }()
    // 处理器逻辑
}
```

### 3. 在库的边界处恢复 panic（若库内部使用了 panic）

## 最佳实践建议

- **在应用程序代码中**：几乎总是使用 error 而非 panic。
- **在库代码中**：更加谨慎，通常应返回 error 而非 panic。
- **保持一致性**：若一个包使用了 panic，整个包需采用相同策略。
- **文档记录**：若函数可能 panic，必须在文档中明确说明。

## 示例对比

### 使用 error（推荐）

```go
func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("除数不能为零")
    }
    return a / b, nil
}

// 调用处
result, err := Divide(10, 0)
if err != nil {
    // 处理错误
}
```

### 使用 panic（不推荐，除非特殊情况）

```go
func MustDivide(a, b int) int {
    if b == 0 {
        panic("除数不能为零")
    }
    return a / b
}

// 调用处
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("错误:", r)
        }
    }()
    result := MustDivide(10, 0)
    // ...
}
```

## 总结

在 Go 中，error 是处理错误的默认和推荐方式，panic 应保留给真正异常的情况（如不可恢复的错误或编程错误）。recover 则用于捕获 panic，防止程序或 goroutine 崩溃。
