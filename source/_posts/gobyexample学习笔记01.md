---
title: gobyexample学习笔记01
description: ''
tags: ['go']
toc: false
date: 2025-05-12 19:41:03
categories:
    - go
    - basic
---

# Go 语言学习笔记 - Lesson 01

## 1. 基础语法特点

### 1.1 包管理
- Go 语言使用 `package` 关键字声明包
- 主程序必须使用 `package main`
- 导入包使用 `import` 关键字，支持多种导入方式：
  ```go
  import "fmt"
  import (
      "fmt"
      "math"
  )
  ```

### 1.2 变量声明
- 使用 `var` 关键字声明变量
- 支持类型推导：`:=` 操作符
- 变量声明后必须使用，否则会编译错误
- 支持多变量声明：
  ```go
  var a, b int = 1, 2
  c, d := 3, 4
  ```

### 1.3 常量
- 使用 `const` 关键字声明常量
- 常量可以是字符、字符串、布尔值或数值
- 常量不能使用 `:=` 语法声明

## 2. 数据结构

### 2.1 数组和切片
- 数组：固定长度，声明时需要指定长度
  ```go
  var a [5]int
  ```
- 切片：动态数组，更灵活
  ```go
  s := make([]int, 5)
  ```
- 切片操作：
  - 追加：`append()`
  - 截取：`s[low:high]`
  - 长度：`len()`
  - 容量：`cap()`

### 2.2 Map
- 键值对集合
- 声明和初始化：
  ```go
  m := make(map[string]int)
  m["key"] = 42
  ```
- 检查键是否存在：
  ```go
  value, exists := m["key"]
  ```

## 3. 控制结构

### 3.1 循环
- Go 只有 `for` 循环，没有 `while`
- 支持多种 for 循环形式：
  ```go
  for i := 0; i < 10; i++ {}
  for i < 10 {}
  for {}
  ```

### 3.2 条件语句
- `if` 语句可以包含初始化语句
  ```go
  if v := math.Pow(x, n); v < lim {
      return v
  }
  ```
- `switch` 语句：
  - 不需要 `break`
  - 可以使用 `fallthrough`
  - 支持多条件

## 4. 函数特性

### 4.1 函数声明
- 支持多返回值
- 支持命名返回值
- 支持可变参数
  ```go
  func sum(nums ...int) int {}
  ```

### 4.2 闭包
- 函数可以返回函数
- 闭包可以访问外部变量
- 常用于实现函数工厂

## 5. 面向对象特性

### 5.1 结构体
- 使用 `struct` 关键字
- 支持匿名字段
- 可以定义方法

### 5.2 方法
- 可以为任何类型定义方法
- 方法接收者可以是值或指针
  ```go
  func (v Vertex) Abs() float64 {}
  func (v *Vertex) Scale(f float64) {}
  ```

### 5.3 接口
- 隐式实现接口
- 接口可以包含多个方法
- 空接口 `interface{}` 可以存储任何类型

## 6. 指针
- 使用 `&` 获取地址
- 使用 `*` 获取值
- 支持指针接收者方法

## 7. 字符串和符文
- 字符串是不可变的字节序列
- 使用 `rune` 处理 Unicode 字符
- 支持字符串切片和连接

## 8. 递归
- 支持函数递归调用
- 常用于树结构和分治算法

## 9. 范围遍历
- 使用 `range` 遍历数组、切片、map 等
- 可以获取索引和值
  ```go
  for i, v := range slice {}
  for k, v := range m {}
  ```

## 注意事项
1. Go 语言强调简洁性和实用性
2. 错误处理通常使用返回值而不是异常
4. 代码格式化使用 `go fmt`
