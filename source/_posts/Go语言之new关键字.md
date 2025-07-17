---
title: Go语言之new关键字
description: ''
tags: ['go']
toc: false
date: 2025-07-06 11:07:56
categories:
    - go
    - basic
---

在 Go 语言中，内存分配主要有两种方式：一种是使用内建函数 `new`，另一种是使用复合字面量（如结构体字面量）或者 `make`。相比之下，`new` 看起来不常用，甚至被一些初学者误解为“类似 C++ 的 new”。

<!--more-->

# Go 语言中的 `new` 关键字详解与最佳实践

## 一、`new` 的基本语法

```go
ptr := new(Type)
```

### 说明：

* `Type` 是任意类型（如结构体、数组、基本类型等）。
* `new(Type)` 会分配一块内存来存储一个 `Type` 类型的零值，并返回一个指向该内存的指针。
* 返回值类型为 `*Type`。

### 示例：

```go
package main

import "fmt"

func main() {
    x := new(int)
    fmt.Println(*x) // 输出 0，因为是 int 类型的零值
    *x = 42
    fmt.Println(*x) // 输出 42
}
```

---

## 二、`new` 与复合字面量的区别

Go 中更常见的写法其实是结构体的复合字面量，例如：

```go
type User struct {
    Name string
    Age  int
}

u1 := new(User)           // 返回 *User，字段是零值
u2 := &User{}             // 返回 *User，字段是零值（更常用）
u3 := &User{Name: "Tom"}  // 返回 *User，指定字段初始化
```

`new(User)` 和 `&User{}` 在结果上是一样的，都是分配内存并返回指针，但后者更直观、灵活，并且在实际代码中更具可读性。

---

## 三、使用 `new` 的典型场景

虽然 Go 的编码风格更倾向于使用复合字面量，但 `new` 并不是没有价值的，以下是一些适合使用 `new` 的场景：

### 1. 简洁地分配基本类型指针

```go
p := new(bool)  // 分配一个布尔变量并返回指针
*q := new(float64)
```

适合用于需要修改值并传递指针的场景，如状态标志、配置项等。

### 2. 用于通用函数返回指针

```go
func NewZero[T any]() *T {
    return new(T)
}
```

在泛型中，`new` 是一个非常通用的方式，用于为任意类型分配零值内存。

---

## 四、不推荐使用 `new` 的场景

### ❌ 不建议用于结构体初始化：

```go
type Person struct {
    Name string
    Age  int
}

p := new(Person)
// vs
p := &Person{Name: "Alice", Age: 30} // 推荐
```

使用复合字面量初始化更清晰直观，尤其是当需要初始化字段时。

---

## 五、最佳实践建议

| 场景             | 推荐方式        | 说明                         |
| -------------- | ----------- | -------------------------- |
| 基本类型需要指针       | `new(int)`  | 简洁、语义明确                    |
| 结构体初始化         | `&Struct{}` | 更清晰，方便指定字段值                |
| 泛型函数中初始化任意类型   | `new(T)`    | 泛型中几乎是唯一方式                 |
| 与 `make` 混淆的情况 | 避免使用 `new`  | `make` 主要用于 slice、map、chan |

---

## 六、`new` vs `make`

| 特点    | `new`        | `make`               |
| ----- | ------------ | -------------------- |
| 返回值   | 指向类型的指针 `*T` | 初始化好的类型值（非指针）        |
| 适用类型  | 所有类型         | 仅限 slice、map、chan    |
| 初始化行为 | 零值初始化        | 返回可用实例（如具有长度的 slice） |

### 示例：

```go
a := new([]int)   // *[]int，指向一个 nil 的 slice
b := make([]int, 5) // []int，已经有长度为 5 的可用 slice
```

---

## 七、总结

* `new` 是 Go 的内建函数，用于分配一块零值内存并返回指针。
* 对于基本类型或泛型，`new` 是一个便捷且清晰的选择。
* 在结构体初始化时，更推荐使用 `&T{}` 风格。
* 不要将 `new` 与 `make` 混淆，它们服务于完全不同的目的。

---

## 八、参考示例项目

一个真实的小项目中，如果你需要传入一个状态布尔指针，可以这样使用 `new`：

```go
func InitFeature(enabled *bool) {
    if enabled != nil && *enabled {
        fmt.Println("Feature enabled")
    }
}

func main() {
    flag := new(bool)
    *flag = true
    InitFeature(flag)
}
```

这种写法避免了引入多余的布尔变量，同时又保留了指针语义。


