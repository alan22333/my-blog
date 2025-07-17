---
title: Go语言之再谈异常处理
description: 'panic、recove、defer'
tags: ['go']
toc: false
date: 2025-07-07 18:34:43
categories:
    - go
    - basic
---


# Go语言中的异常处理机制：panic、recover 和 defer 的正确使用方式

Go 语言没有传统意义上的 `try-catch-finally` 异常处理机制，而是以一种更轻量的方式来应对运行时错误。本文将系统地讲解 Go 的异常处理机制，包括：

* 什么是 `panic`、`recover` 和 `defer`
* 它们的使用规则和底层原理
* 开发中的最佳实践
* 常见陷阱与解决方案

---

## 一、Go 的异常处理设计理念

Go 语言的核心哲学是**明确错误而非隐藏错误**，所以它鼓励使用显式的 `error` 类型进行错误处理。只有在真正无法处理的问题（如程序逻辑崩溃）时，才建议使用 `panic`。

> ✅ **常规错误用 `error` 返回，极端错误才用 `panic` 抛出。**

---

## 二、三个关键词详解：`panic`、`recover`、`defer`

### 1. panic

`panic` 会让程序立即停止当前函数的执行，沿调用栈向上传播，直到遇到 `recover()` 或程序彻底崩溃。

```go
func main() {
	panic("程序发生严重错误")
	fmt.Println("这行不会被执行")
}
```

### 2. defer

`defer` 会在当前函数返回前执行，无论是否发生 `panic`。它非常适合用来做清理工作（关闭文件、回收资源等）或配合 `recover` 捕捉异常。

```go
func main() {
	defer fmt.Println("这个一定会执行")
	fmt.Println("开始")
}
```

输出：

```
开始
这个一定会执行
```

### 3. recover

`recover` 是一个内置函数，**只能在 defer 中调用**。它可以捕捉到 panic 并让程序继续运行。

```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("Recovered from panic:", err)
		}
	}()
	panic("崩溃啦！")
}
```

输出：

```
Recovered from panic: 崩溃啦！
```

---

## 三、recover 的使用场景与最佳实践

### ✅ 示例 1：安全的 Web 请求处理

在 Web 应用中，为了防止某个请求 panic 导致整个服务崩溃，推荐写一个中间件：

```go
func recoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				http.Error(w, "Internal Server Error", http.StatusInternalServerError)
				log.Printf("panic: %v\n", err)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

这样即使某个请求触发了 panic，也不会影响服务器整体运行。

---

### ✅ 示例 2：异步任务中的防护（goroutine）

如果你在 goroutine 中没有捕捉 panic，整个程序仍可能崩溃：

```go
func safeGo(f func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				log.Println("goroutine panic:", r)
			}
		}()
		f()
	}()
}

func main() {
	safeGo(func() {
		panic("异步任务崩溃！")
	})
	time.Sleep(time.Second)
}
```

这个模式是高并发场景下的重要保障手段。

---

### ✅ 示例 3：清理资源

```go
func openFile(filename string) {
	f, err := os.Open(filename)
	if err != nil {
		panic(err)
	}
	defer func() {
		fmt.Println("关闭文件")
		f.Close()
	}()
}
```

即使中途 `panic`，文件也能被安全关闭。

---

## 四、何时该 panic，何时该 return error？

| 场景                             | 使用方式                   |
| ------------------------------ | ---------------------- |
| 用户输入错误 / 网络超时等可预期错误            | 返回 `error`             |
| 数组越界 / nil dereference 等程序 bug | 使用 `panic`             |
| 初始化失败（例如配置文件丢失）                | 允许 panic 或 log.Fatal   |
| 多线程崩溃防护                        | 使用 goroutine + recover |

---

## 五、常见陷阱与误区

### ❌ recover 不在 defer 中使用

```go
func bad() {
	if err := recover(); err != nil { // ❌ 失效
		fmt.Println("won't work")
	}
}
```

`recover` 只能在 `defer` 函数内才能捕获 `panic`，否则永远返回 `nil`。

---

### ❌ 忽略 panic 的后果

如果你写库函数时随意 `panic`，调用者没有准备好 `recover`，就可能导致整个程序崩溃。**库函数应该优先返回 error。**

---

## 六、一个完整的最佳实践模板

```go
func SafeRun(handler func()) {
	defer func() {
		if err := recover(); err != nil {
			log.Printf("panic recovered: %v\n", err)
			debug.PrintStack() // 打印堆栈
		}
	}()
	handler()
}

func main() {
	SafeRun(func() {
		// 模拟严重错误
		var list []int
		fmt.Println(list[100]) // panic: index out of range
	})
	fmt.Println("程序没有崩溃")
}
```

输出示例：

```
panic recovered: runtime error: index out of range [100] with length 0
...堆栈信息...
程序没有崩溃
```

---

## 七、总结

| 关键词         | 说明                                             |
| ----------- | ---------------------------------------------- |
| `panic()`   | 抛出异常，程序中断执行                                    |
| `recover()` | 捕获异常并恢复执行，仅在 defer 中有效                         |
| `defer`     | 注册延迟执行函数，可配合 recover 做错误处理                     |
| 最佳实践        | panic 用于不可恢复错误，其他返回 error；goroutine 要保护；中间件要兜底 |

---