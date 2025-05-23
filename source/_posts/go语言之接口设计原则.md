---
title: go语言之接口设计原则
description: '接口设计要聚焦最小功能集（遵循接口隔离原则）,并使用组合接口，将多个小接口组合成更大接口'
tags: ['go']
toc: false
date: 2025-05-15 10:05:08
categories:
    - go
    - basic
---

假设接口定义如下：

```go
type MyInterface interface {
    Foo()
    Bar()
}
```

然后你有一个结构体只实现了 `Foo()`，没有实现 `Bar()`：

```go
type MyStruct struct{}

func (m MyStruct) Foo() {
    fmt.Println("Foo called")
}
```

### ❌ 此时的行为

如果你尝试这样赋值：

```go
var x MyInterface = MyStruct{}  // ❌ 编译错误
```

**会报编译错误：**

> `MyStruct` does not implement `MyInterface` (missing method `Bar`)

---

### ✅ Go 的接口是结构性的

Go 使用结构性类型系统，**只要类型实现了接口所需的所有方法，就隐式满足该接口。**
所以如果缺失任意一个方法，即便名字完全匹配，**也不能赋值给该接口变量。**

---

### ✅ 正确实现方式

```go
type MyStruct struct{}

func (m MyStruct) Foo() {
    fmt.Println("Foo called")
}

func (m MyStruct) Bar() {
    fmt.Println("Bar called")
}

var x MyInterface = MyStruct{}  // ✅ OK
```

---

### 实际开发建议

* **接口设计要聚焦最小功能集**（遵循接口隔离原则），以便结构体可以按需实现。
* **使用组合接口**：将多个小接口组合成更大接口，例如：

```go
type Fooer interface {
    Foo()
}

type Barer interface {
    Bar()
}

type FooBar interface {
    Fooer
    Barer
}
```

---

