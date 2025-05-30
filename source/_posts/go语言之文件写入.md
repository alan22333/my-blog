---
title: go语言之文件写入
description: 'Go 的 os 包和 bufio 包提供了多种方式来向文件写入数据'
tags: ['go']
toc: false
date: 2025-05-29 16:45:21
categories:
    - go
    - basic
---

## Go 语言文件写入详解与最佳实践

在 Go 语言中，文件写入是另一个重要的文件操作任务。Go 的 `os` 包和 `bufio` 包提供了多种方式来向文件写入数据。本文将通过你提供的代码示例，深入探讨 Go 语言中文件写入的常用方法，并总结一些工程中的最佳实践。

### 代码示例

首先，我们来看一下你提供的示例代码：

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {
	// 使用 os.WriteFile 一次性写入数据到文件
	d1 := []byte("hello\ngo\n")
	err := os.WriteFile("./tmp/dat1", d1, 0644)
	check(err)

	// 使用 os.Create 创建文件并获取 *os.File 对象，然后使用 Write 方法写入
	f, err := os.Create("./tmp/dat2")
	check(err)
	defer f.Close() // 确保文件在使用完毕后关闭

	// 使用 Write 方法写入 byte slice
	d2 := []byte{115, 111, 109, 101, 10}
	n2, err := f.Write(d2)
	check(err)
	fmt.Printf("wrote %d bytes\n", n2)

	// 使用 WriteString 方法写入字符串
	n3, err := f.WriteString("writes\n")
	check(err)
	fmt.Printf("wrote %d bytes\n", n3)

	// 调用 Sync 将缓冲区的数据写入磁盘
	f.Sync()

	// 使用 bufio.NewWriter 创建带缓冲的写入器，提高写入效率
	w := bufio.NewWriter(f)
	n4, err := w.WriteString("buffered\n")
	check(err)
	fmt.Printf("wrote %d bytes\n", n4)

	// 调用 Flush 将缓冲区中的数据写入底层写入器（在这里是文件）
	w.Flush()
}

```


### 代码详解

1.  **导入包**:
    ```go
    import (
        "bufio"
        "fmt"
        "os"
    )
    ```
    * `os`: 提供了操作系统相关的功能，包括文件操作。
    * `fmt`: 用于格式化输入输出。
    * `bufio`: 提供了带缓冲的 I/O 操作，可以提高读写效率。

2.  **错误检查函数 `check`**:
    ```go
    func check(e error) {
        if e != nil {
            panic(e)
        }
    }
    ```
    这个错误处理函数在遇到错误时会触发 `panic`。在实际项目中，你可能需要更优雅的错误处理方式。

3.  **`os.WriteFile`**:
    ```go
    d1 := []byte("hello\ngo\n")
    err := os.WriteFile("./tmp/dat1", d1, 0644)
    check(err)
    ```
    `os.WriteFile` 函数会将给定的 `[]byte` 数据一次性写入到指定的文件中。第三个参数 `0644` 是文件的权限模式。这适用于简单地写入少量数据到文件的场景。

4.  **`os.Create` 和 `f.Write` / `f.WriteString`**:
    ```go
    f, err := os.Create("./tmp/dat2")
    check(err)
    defer f.Close()

    d2 := []byte{115, 111, 109, 101, 10}
    n2, err := f.Write(d2)
    check(err)
    fmt.Printf("wrote %d bytes\n", n2)

    n3, err := f.WriteString("writes\n")
    check(err)
    fmt.Printf("wrote %d bytes\n", n3)
    ```
    * `os.Create` 函数会创建一个新的文件用于写入。如果文件已存在，它会截断（清空）该文件。它返回一个 `*os.File` 类型的文件对象 `f`。
    * `defer f.Close()`: 同样，这是一个重要的最佳实践，确保文件在使用完毕后被关闭。
    * `f.Write(d2)`: 将一个 `[]byte` 写入到文件中，返回实际写入的字节数 `n2` 和一个 `error`。
    * `f.WriteString("writes\n")`: 将一个字符串写入到文件中，返回实际写入的字节数 `n3` 和一个 `error`。

5.  **`f.Sync()`**:
    ```go
    f.Sync()
    ```
    `Sync` 方法会将文件底层驱动的任何缓存中的数据写入到磁盘。这可以确保数据的持久性，但在频繁写入的场景下可能会影响性能。

6.  **`bufio.NewWriter` 和 `w.WriteString` / `w.Flush`**:
    ```go
    w := bufio.NewWriter(f)
    n4, err := w.WriteString("buffered\n")
    check(err)
    fmt.Printf("wrote %d bytes\n", n4)

    w.Flush()
    ```
    * `bufio.NewWriter(f)` 创建一个新的带缓冲的写入器。写入到这个 writer 的数据会先被缓存在内存中，当缓冲区满或者显式调用 `Flush` 方法时，才会批量写入到底层的文件中，这样可以减少系统调用次数，提高写入性能。
    * `w.WriteString("buffered\n")`: 将字符串写入到 writer 的缓冲区中。
    * `w.Flush()`: 将 writer 缓冲区中的所有数据写入到底层的 `io.Writer`（在这里是文件）。**务必在完成写入后调用 `Flush`，以确保所有数据都被写入磁盘。**

### 工程中的最佳实践

1.  **及时关闭文件**: 始终使用 `defer f.Close()` 来确保文件在使用完毕后关闭，释放系统资源。

2.  **错误处理**: 示例代码中使用了 `panic` 进行错误处理，这在生产环境中通常是不合适的。应该使用更健壮的错误处理机制，例如返回 `error` 值并进行处理。

3.  **选择合适的写入方法**:
    * 对于简单的、一次性的少量数据写入，可以使用 `os.WriteFile`。
    * 对于更复杂的写入操作或需要追加写入等，可以使用 `os.Create` (或 `os.OpenFile` with appropriate flags) 获取 `*os.File` 并使用 `Write` 或 `WriteString` 方法。
    * 对于需要提高写入性能的场景，尤其是当进行多次小块写入时，应该使用 `bufio.Writer`，并在完成写入后调用 `Flush`。

4.  **考虑数据持久性**: 如果对数据的持久性有较高的要求，可以在适当的时候调用 `f.Sync()`，但这可能会牺牲一定的性能。

5.  **文件权限**: 使用 `os.WriteFile` 或 `os.Create` 时需要注意设置正确的文件权限模式。
