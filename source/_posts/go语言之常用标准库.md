---
title: go语言之最常用标准库
description: 'Go 的标准库提供了一系列强大且实用的内置包，涵盖了从基本 I/O 到网络编程等各种常见任务，无需外部依赖即可构建可靠的应用程序。'
tags: ['go']
toc: false
date: 2025-05-23 14:23:12
categories:
    - go
    - basic
---

**1. `fmt` 包:**

* **功能:** 提供格式化输入输出的功能，类似于 C 语言的 `printf` 和 `scanf`。
* **为什么必备:** 几乎所有的程序都需要进行输入输出，无论是打印日志、用户交互还是格式化数据。
* **学习建议:** 重点学习 `Printf`、`Sprintf`、`Fprintf` 等格式化输出函数，以及 `Scanf`、`Sscanf`、`Fscanf` 等格式化输入函数。了解各种格式化动词（如 `%d`, `%s`, `%v` 等）的用法。

**2. `os` 包:**

* **功能:** 提供与操作系统交互的功能，如文件操作、进程管理、环境变量等。
* **为什么必备:** 任何需要与底层操作系统交互的程序都会用到这个包。
* **学习建议:** 学习文件和目录的操作（`os.Create`, `os.Open`, `os.Mkdir`, `os.Remove` 等），环境变量的获取和设置（`os.Getenv`, `os.Setenv` 等），以及进程相关的操作（`os.Exit` 等）。

**3. `net/http` 包:**

* **功能:** 提供 HTTP 客户端和服务器的实现。
* **为什么必备:** 在当今互联网时代，Web 开发非常普遍，无论是构建 API 还是简单的 HTTP 客户端，这个包都是必不可少的。
* **学习建议:** 学习如何创建一个简单的 HTTP 服务器（`http.HandleFunc`, `http.ListenAndServe`），以及如何发起 HTTP 请求（`http.Get`, `http.Post` 等）。了解 `http.Request` 和 `http.ResponseWriter` 的结构和用法。

**4. `io` 包:**

* **功能:** 提供基本的 I/O 接口。很多其他的 I/O 相关的包都基于 `io` 包的接口。
* **为什么必备:** 处理输入和输出流是编程中常见的任务。
* **学习建议:** 理解 `io.Reader` 和 `io.Writer` 接口，以及一些常用的实现，如 `bytes.Buffer` 和 `os.File`。

**5. `bufio` 包:**

* **功能:** 提供带缓冲的 I/O 操作，可以提高 I/O 的效率。
* **为什么必备:** 在处理大量数据或需要更精细控制 I/O 的场景下很有用。
* **学习建议:** 学习 `bufio.Reader` 和 `bufio.Writer` 的用法，以及它们提供的缓冲读取和写入方法。

**6. `strings` 包:**

* **功能:** 提供字符串操作的常用函数，如查找、替换、分割等。
* **为什么必备:** 字符串处理在各种应用中都很常见。
* **学习建议:** 学习 `strings.Contains`, `strings.Index`, `strings.ReplaceAll`, `strings.Split`, `strings.Join` 等常用函数。

**7. `strconv` 包:**

* **功能:** 提供字符串和基本数据类型之间的转换功能。
* **为什么必备:** 在处理用户输入、配置文件等场景下，经常需要在字符串和数值之间进行转换。
* **学习建议:** 学习 `strconv.Atoi` (字符串转整数), `strconv.Itoa` (整数转字符串), 以及其他类型转换函数，如 `ParseBool`, `ParseFloat` 等。
