---
title: go语言之测试与基准测试入门
description: 'Go 鼓励使用简洁的语法编写测试用例，帮助开发者保证代码质量和性能。本文科普向，鼠鼠认为这一块知识想要深入学习还需要在具体项目中实践'
tags: ['go']
toc: false
date: 2025-05-30 18:21:45
categories:
    - go
    - basic
---

## 📌 一、Go 的测试框架介绍

Go 内置了强大的测试框架，不需要第三方库，只需引入：

```go
import "testing"
```

所有测试文件必须以 `_test.go` 结尾，所有测试函数以 `Test` 开头，并接受 `*testing.T` 参数。

---

## 📘 二、单元测试（Unit Test）

### ✅ 基本格式

```go
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Expected 5, got %d", result)
    }
}
```

### 📁 文件结构示例

```
calc/
 ├── calc.go
 └── calc_test.go
```

### 📌 示例代码

```go
// calc.go
package calc

func Add(a, b int) int {
    return a + b
}
```

```go
// calc_test.go
package calc

import "testing"

func TestAdd(t *testing.T) {
    got := Add(1, 2)
    want := 3

    if got != want {
        t.Errorf("Add(1, 2) = %d; want %d", got, want)
    }
}
```

### 🚀 运行测试

```bash
go test
```

---

## 🧪 三、表驱动测试（推荐）

适合多个输入输出的函数测试，易扩展、代码更清晰。

```go
func TestAddTableDriven(t *testing.T) {
    cases := []struct{
        a, b, expected int
    }{
        {1, 2, 3},
        {2, 2, 4},
        {10, 5, 15},
    }

    for _, c := range cases {
        got := Add(c.a, c.b)
        if got != c.expected {
            t.Errorf("Add(%d, %d) = %d; want %d", c.a, c.b, got, c.expected)
        }
    }
}
```

---

## 🕓 四、基准测试（Benchmark）

用于性能评估。函数名以 `Benchmark` 开头，接收 `*testing.B` 参数。

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(10, 20)
    }
}
```

执行命令：

```bash
go test -bench=.
```

---

## 🧪 五、测试覆盖率

查看测试覆盖率：

```bash
go test -cover
```

生成详细报告：

```bash
go test -coverprofile=cover.out
go tool cover -html=cover.out
```

---

## 🧰 六、实际开发中的测试最佳实践

| 做法                   | 描述                                                       |
| -------------------- | -------------------------------------------------------- |
| ✅ 使用表驱动测试            | 可读性强、维护方便                                                |
| ✅ 测试覆盖边界条件           | 不仅测常规输入，还要测试错误输入、零值等                                     |
| ✅ 使用子测试              | `t.Run("case name", func(t *testing.T) {...})` 有助于分组管理测试 |
| ✅ 保持测试快速             | 测试应能快速运行，不阻塞开发                                           |
| ✅ 对接口进行测试            | 方便 mock 和替换                                              |
| ✅ 使用 `go test -race` | 检查并发代码中的数据竞争问题                                           |

---

## 🧪 七、测试中的 Mock 与 Stub（进阶）

Go 没有内置 mock 框架，但可以手动定义接口并注入假实现。

```go
type DB interface {
    GetUser(id int) string
}

type MockDB struct{}

func (m *MockDB) GetUser(id int) string {
    return "mock-user"
}
```

测试中使用：

```go
func TestService(t *testing.T) {
    service := NewService(&MockDB{})
    user := service.GetUserName(1)
    if user != "mock-user" {
        t.Fail()
    }
}
```

---

## ✅ 总结

| 类型      | 用法           |
| ------- | ------------ |
| 单元测试    | 测试函数是否输出正确结果 |
| 表驱动测试   | 测试多个输入组合     |
| 基准测试    | 测试性能         |
| 子测试     | 多维度组织测试      |
| 覆盖率     | 检查测试是否全面     |
| Mock 接口 | 模拟依赖         |

---