---
title: Go语言之接口指针误区
description: '经典的初学者错误 *Interface'
tags: ['go']
toc: false
date: 2025-07-05 09:00:13
categories:
    - go
    - basic
---

# 🧠 Go语言接口误区解析：为什么不能使用 `[]*Interface`？

## 📝 前言

在 Go 语言中，接口是实现“面向抽象编程”的关键工具。很多初学者在使用接口切片时会犯一个**常见错误**：将接口类型声明为指针切片 `[]*Interface`。一旦这样写，Go 编译器就会报出如下错误：

```Shell
cannot use x (variable of type *MyStruct) as *MyInterface value in argument to append:
*MyStruct does not implement *MyInterface (type *MyInterface is pointer to interface, not interface)
```


这到底是为什么？本篇文章将从实战场景出发，深入剖析背后的机制，并给出正确写法。

---

## ❌ 错误示例：*Interface 是什么？

假设我们想维护一个接口类型的切片，并将实现类注册进去。

```Go
type MyInterface interface {
    DoWork()
}

type System struct {
    Workers []*MyInterface  // ❌ 错误：不应该是指针接口
}
```


实现这个接口的结构体如下：

```Go
type Worker struct {
    Name string
}

func (w *Worker) DoWork() {
    fmt.Println(w.Name, "is working")
}
```


然后我们想注册一个 `*Worker` 实例：

```Go
sys := &System{}
worker := &Worker{Name: "Alice"}
sys.Workers = append(sys.Workers, worker)
```


结果编译器报错：

```Go
cannot use worker (type *Worker) as type *MyInterface in append
```


### 😵 原因解析

> `*MyInterface` 是“**指向接口的指针**”，但在 Go 中接口本身已经是一个引用类型！

换句话说：

- `MyInterface` 是接口 ✅

- `*MyInterface` 是“指向接口的指针” ❌ —— Go 根本不允许这种类型的存在！

---

## ✅ 正确做法：接口是引用类型，直接用 `[]MyInterface`

```Go
type System struct {
    Workers []MyInterface  // ✅ 正确：接口本身是引用类型
}
```


这样我们就可以放心地将实现接口的结构体指针加入进去：

```Go
sys := &System{}
worker := &Worker{Name: "Alice"}
sys.Workers = append(sys.Workers, worker)  // ✅ 正常
```


因为 `*Worker` 实现了 `MyInterface`，可以自动被赋值给 `MyInterface` 类型。

---

## 📦 延伸知识：结构体 vs 指针接收者实现接口

在 Go 中，**只有结构体或其指针的“方法集”完整实现接口的方法集，才能认为实现了接口**：

```Go
type MyInterface interface {
    DoWork()
}

type Worker struct{}

func (w Worker) DoWork() { ... }      // 实现了接口
func (w *Worker) DoWork() { ... }     // 也可以实现接口（常见）
```


### 总结：

|方法定义|能否让 `Worker` 实现接口|能否让 `*Worker` 实现接口|
|-|-|-|
|`(w Worker)`|✅ 是|✅ 是（值自动取地址）|
|`(*w Worker)`|❌ 否（值不能用）|✅ 是|

建议：**接口方法接收者多数使用 `*T`，以便避免值复制和保持一致性。**

---

## 💡 实战建议

- 始终使用 `[]Interface`，不要使用 `[]*Interface`

- 实现接口的结构体通常使用指针接收者（`*T`）

- 接口是引用类型，不要再取地址（`&iface` 是没有意义的）

- 若你持有的是多个类型实例，实现接口是最佳解耦方式

---

## 🧾 小结

|错误写法|原因|
|-|-|
|`[]*Interface`|Go 中接口本身就是引用类型，无需加 `*`|
|`*Interface` 参数|Go 不允许指针指向接口类型|

|正确写法|场景|
|-|-|
|`[]Interface`|保持接口多态、模块解耦|
|`*Struct` 实现接口|避免值复制，更灵活|

---
