---
title: go语言之错误处理
description: 'Go 不使用异常机制（try-catch），而是通过返回错误值来显式地处理错误'
tags: ['go']
toc: false
date: 2025-05-15 09:47:12
categories:
    - go
    - basic
---

# Go语言中的错误处理详解

Go 语言的错误处理机制与许多其他语言有所不同。Go 不使用异常机制（`try-catch`），而是通过返回错误值来显式地处理错误。虽然这种方式比异常机制更加简洁和明确，但它也带来了更多的冗长和细节。

## 📌 一、Go 错误处理的基本方式

Go 语言的错误通常是通过 `error` 接口来表示的。`error` 接口本身非常简单：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error()` 方法的类型，都可以作为 `error` 类型使用。

### 返回错误

Go 中大多数函数会返回一个 `error` 类型的值来表示执行过程中可能出现的错误。典型的错误处理模式如下：

```go
func someFunction() (string, error) {
    // 返回数据和错误
    return "", fmt.Errorf("something went wrong")
}
```

调用时，可以通过检查返回的 `error` 值来判断是否发生错误：

```go
result, err := someFunction()
if err != nil {
    fmt.Println("Error:", err)
    return
}
fmt.Println("Success:", result)
```

## 📌 二、创建和包装错误

Go 提供了标准库中的 `errors` 包来创建简单的错误，也提供了 `fmt.Errorf` 来包装错误并增加上下文信息。

### 1. 使用 `errors.New` 创建简单错误

```go
import "errors"

var ErrNotFound = errors.New("resource not found")
```

### 2. 使用 `fmt.Errorf` 包装错误

```go
return fmt.Errorf("failed to load config file: %w", err)
```

### 3. 错误包装与错误链

```go
if errors.Is(err, ErrNotFound) {
    fmt.Println("Resource not found")
}
```

## 📌 三、自定义错误类型

### 定义自定义错误类型

```go
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string {
    return fmt.Sprintf("[code %d] %s: %v", e.Code, e.Message, e.Err)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

func NewAppError(code int, msg string, err error) *AppError {
    return &AppError{
        Code:    code,
        Message: msg,
        Err:     err,
    }
}
```

### 使用自定义错误类型

```go
func loadConfig(file string) error {
    data, err := os.ReadFile(file)
    if err != nil {
        return NewAppError(1001, "failed to load config", err)
    }
    fmt.Println("Config data:", string(data))
    return nil
}

func main() {
    err := loadConfig("config.json")
    if err != nil {
        var appErr *AppError
        if errors.As(err, &appErr) {
            fmt.Printf("Custom error caught: %s\n", appErr.Error())
        } else {
            fmt.Println("Unknown error:", err)
        }
    }
}
```

## 📌 四、错误判断与处理

```go
if errors.Is(err, ErrNotFound) {
    fmt.Println("Resource not found")
}

var appErr *AppError
if errors.As(err, &appErr) {
    fmt.Printf("AppError with code: %d\n", appErr.Code)
}
```

## 📌 五、最佳实践

1. 错误返回而非 panic。
2. 错误信息应简洁并富有上下文。
3. 使用自定义错误类型增强错误可处理性。
4. 使用统一的错误处理策略与工具函数。

## 📌 六、使用 panic 和 recover（仅限不可恢复场景）

```go
func SafeRun(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    fn()
    return nil
}
```

## 📌 七、Starter 封装实战：构建统一错误处理框架

在项目中，可以封装一个 starter 模板，集中管理错误定义与处理。以下是一个 starter 错误模块的例子：

### 定义错误类型

```go
package errs

import (
    "fmt"
)

type ErrorCode int

const (
    ErrInternal ErrorCode = 1000
    ErrDatabase ErrorCode = 1001
    ErrValidation ErrorCode = 1002
)

type AppError struct {
    Code ErrorCode
    Msg  string
    Err  error
}

func (e *AppError) Error() string {
    return fmt.Sprintf("[%d] %s: %v", e.Code, e.Msg, e.Err)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

func New(code ErrorCode, msg string, err error) *AppError {
    return &AppError{Code: code, Msg: msg, Err: err}
}
```

### 使用封装的错误模块

```go
package service

import (
    "errors"
    "fmt"
    "project/errs"
)

func LoadUser(id int) (string, error) {
    if id == 0 {
        return "", errs.New(errs.ErrValidation, "user id is invalid", nil)
    }
    // 模拟数据库错误
    dbErr := errors.New("db connection failed")
    return "", errs.New(errs.ErrDatabase, "failed to load user", dbErr)
}

func main() {
    name, err := LoadUser(0)
    if err != nil {
        var appErr *errs.AppError
        if errors.As(err, &appErr) {
            fmt.Println("Handled error:", appErr)
        }
    } else {
        fmt.Println("Loaded user:", name)
    }
}
```

通过构建统一错误处理模块，我们可以提升项目结构清晰度，使错误处理更加系统化和可维护。

---

## 📌 八、总结

Go 的错误处理机制强调**显式、清晰和简单**。通过定义错误类型、错误链和自定义错误，我们能够创建更加清晰和易于管理的错误处理流程。

✅ 错误应返回而不是 panic。

✅ 通过 `fmt.Errorf` 增加上下文信息。

✅ 自定义错误类型 + 错误链判断提升可读性与可维护性。

✅ 封装 starter 错误模块，提高开发效率。
