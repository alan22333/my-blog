---
title: go语言之枚举
description: '为什么 Go 没有 enum？'
tags: ['go']
toc: false
date: 2025-05-13 16:25:43
categories:
    - go
    - basic
---

---

# 📘 Go 语言中的枚举

---

## 🔸 1. 为什么 Go 没有 enum？

Go 的设计哲学之一是 **“少即是多（Less is more）”**。它刻意避免加入 C/C++ 或 Java 风格的 `enum` 语法，而鼓励开发者通过现有机制构建自己的“枚举”，使语言更简洁。

---

## 🔸 2. 使用 `const + iota` 自定义类型模拟枚举

### ✅ 示例：订单状态（OrderStatus）

```go
// 自定义一个新类型
type OrderStatus int

// 声明常量并用 iota 自动编号
const (
	Pending OrderStatus = iota
	Processing
	Shipped
	Delivered
	Cancelled
)
```

此时你定义了一个类型 `OrderStatus`，并赋值了几个状态枚举。使用时：

```go
var s OrderStatus = Shipped
fmt.Println("Order status:", s) // 输出：Order status: 2（不直观）
```

---

## 🔸 3. 让枚举可读：添加 `String()` 方法

要让输出更清晰（而不是数字），实现 `fmt.Stringer` 接口：

```go
func (s OrderStatus) String() string {
	switch s {
	case Pending:
		return "Pending"
	case Processing:
		return "Processing"
	case Shipped:
		return "Shipped"
	case Delivered:
		return "Delivered"
	case Cancelled:
		return "Cancelled"
	default:
		return "Unknown"
	}
}
```

现在输出变成：

```go
fmt.Println("Order status:", s) // 输出：Order status: Shipped
```

---

## 🔸 4. 提高开发效率：使用 `stringer` 工具自动生成

Go 提供了官方工具 [`stringer`](https://pkg.go.dev/golang.org/x/tools/cmd/stringer) 来自动生成 `String()` 方法。

### ✅ 步骤：

1. 安装：

   ```sh
   go install golang.org/x/tools/cmd/stringer@latest
   ```

2. 在代码顶部加注释：

   ```go
   //go:generate stringer -type=OrderStatus
   ```

3. 执行：

   ```sh
   go generate
   ```

4. 它会生成一个 `orderstatus_string.go` 文件，自动添加 `String()` 方法！

---

## 🔸 5. 在实际开发中的典型应用场景

| 枚举类型   | 场景举例                               |
| ------ | ---------------------------------- |
| 用户角色   | Admin, Moderator, User             |
| 支付方式   | WeChat, Alipay, CreditCard         |
| 网络协议类型 | HTTP, HTTPS, FTP                   |
| 日志级别   | Debug, Info, Warn, Error           |
| 服务状态   | Starting, Running, Stopped, Failed |

示例代码（支付方式）：

```go
type PaymentMethod int

const (
	Alipay PaymentMethod = iota
	WeChat
	CreditCard
)

func (p PaymentMethod) String() string {
	switch p {
	case Alipay:
		return "Alipay"
	case WeChat:
		return "WeChat"
	case CreditCard:
		return "CreditCard"
	default:
		return "Unknown"
	}
}
```

用法：

```go
pay := WeChat
fmt.Println("Chosen method:", pay) // 输出：Chosen method: WeChat
```

---

## 🔸 6. 枚举类型优势总结

| 特性          | 使用枚举自定义类型的好处               |
| ----------- | -------------------------- |
| 类型安全        | 不会误传其他类型，编译时检查             |
| 可读性高        | 使用名称而非数字，便于维护和调试           |
| 易于扩展        | 可以集中管理、统一处理                |
| 支持 `switch` | 可用于 `switch case` 表达逻辑分支处理 |
| 与接口搭配灵活使用   | 可以与接口模式组合形成更复杂的业务逻辑        |

---

## 🔸 7. 一个完整小案例：订单状态切换器

```go
package main

import "fmt"

type OrderStatus int

const (
	Pending OrderStatus = iota
	Confirmed
	Shipped
	Delivered
)

func (s OrderStatus) String() string {
	switch s {
	case Pending:
		return "Pending"
	case Confirmed:
		return "Confirmed"
	case Shipped:
		return "Shipped"
	case Delivered:
		return "Delivered"
	default:
		return "Unknown"
	}
}

func nextStatus(s OrderStatus) OrderStatus {
	switch s {
	case Pending:
		return Confirmed
	case Confirmed:
		return Shipped
	case Shipped:
		return Delivered
	default:
		return s
	}
}

func main() {
	status := Pending
	for i := 0; i < 4; i++ {
		fmt.Println("Current status:", status)
		status = nextStatus(status)
	}
}
```

**输出：**

```
Current status: Pending
Current status: Confirmed
Current status: Shipped
Current status: Delivered
```

---

## ✅ 总结

* Go 没有内置枚举语法，但我们可以用 `const + type + iota` 灵活模拟。
* 枚举用得非常广泛，尤其是在：

  * 状态管理
  * 类型控制
  * 协议识别
  * 日志处理 等场景
* 可以手写 `String()` 方法提高可读性，或用 `stringer` 工具自动生成。

---

