---
title: go语言之嵌入实战案例
description: '句几个例子说明//go:embed的具体用法'
tags: ['go']
toc: false
date: 2025-05-30 17:46:14
categories:
    - go
    - basic
---


## ✅ 案例 1：嵌入静态网页文件到 Web 服务器

**应用场景**：构建前端静态页面并打包进 Go 的后端程序中。

```go
package main

import (
	"embed"
	"fmt"
	"io/fs"
	"net/http"
)

//go:embed static/*
var content embed.FS

func main() {
	// 子文件系统，从 static 开始
	staticFiles, err := fs.Sub(content, "static")
	if err != nil {
		panic(err)
	}

	fmt.Println("Serving static files on http://localhost:8080")
	http.Handle("/", http.FileServer(http.FS(staticFiles)))
	http.ListenAndServe(":8080", nil)
}
```

📂 `项目结构` 示例：

```
main.go
static/
  index.html
  style.css
  app.js
```

启动后访问 [http://localhost:8080](http://localhost:8080) 可以直接看到网页内容。

---

## ✅ 案例 2：嵌入默认配置文件

**应用场景**：命令行工具或服务程序中嵌入默认配置（如 YAML、JSON、TOML）。

```go
package main

import (
	_ "embed"
	"encoding/json"
	"fmt"
)

//go:embed config.default.json
var defaultConfig []byte

type Config struct {
	Port int    `json:"port"`
	Name string `json:"name"`
}

func main() {
	var cfg Config
	err := json.Unmarshal(defaultConfig, &cfg)
	if err != nil {
		panic(err)
	}
	fmt.Printf("Loaded config: %+v\n", cfg)
}
```

📄 `config.default.json`：

```json
{
  "port": 8080,
  "name": "my-app"
}
```

开发者可以选择使用默认配置或在运行时覆盖配置。

---

## ✅ 案例 3：嵌入模板文件

**应用场景**：Go Web 应用中需要渲染 HTML 模板（如邮件模板、用户界面等）。

```go
package main

import (
	"embed"
	"html/template"
	"os"
)

//go:embed templates/email.html
var tmplFS embed.FS

func main() {
	tmpl, err := template.ParseFS(tmplFS, "templates/email.html")
	if err != nil {
		panic(err)
	}

	data := map[string]string{
		"Username": "Alice",
		"Content":  "Welcome to our service!",
	}

	err = tmpl.Execute(os.Stdout, data)
	if err != nil {
		panic(err)
	}
}
```

📄 `templates/email.html`：

```html
<h1>Hello, {{.Username}}</h1>
<p>{{.Content}}</p>
```

---

## ✅ 案例 4：嵌入版本信息和帮助文档

**应用场景**：CLI 工具显示版本信息或帮助文件。

```go
package main

import (
	_ "embed"
	"fmt"
)

//go:embed VERSION
var version string

//go:embed HELP.md
var helpDoc string

func main() {
	fmt.Println("Version:", version)
	fmt.Println("Help:")
	fmt.Println(helpDoc)
}
```

📄 `VERSION`:

```
v1.0.3
```

📄 `HELP.md`:

```
Usage:
  myapp [command]

Available Commands:
  help        Show help info
  version     Show version
```

---

## 🧠 总结：实际场景汇总

| 场景     | 使用资源        | 类型    | 推荐变量类型     |
| ------ | ----------- | ----- | ---------- |
| 嵌入静态文件 | HTML/CSS/JS | 多个文件  | `embed.FS` |
| 默认配置   | JSON/YAML   | 单个文件  | `[]byte`   |
| 模板引擎   | HTML 模板     | 单/多文件 | `embed.FS` |
| 帮助文档   | 文本/Markdown | 单个文件  | `string`   |
| 字体/图片  | 二进制文件       | 单个文件  | `[]byte`   |

---

