---
title: gowebexamples学习笔记01
description: 'go web开发基础知识'
tags: ['go']
toc: false
date: 2025-06-04 10:54:46
categories:
    - go
    - basic
---

# Go Web 开发笔记总结

本笔记根据 `gowebexamples` 项目的代码示例，总结 Go Web 开发中的常见操作。

## ch01 - Hello World

- **核心概念**: 构建最简单的 HTTP 服务器，响应客户端请求。
- **实现**: 使用 `net/http` 包的 `http.HandleFunc` 注册路由，`http.ListenAndServe` 启动服务器。
- **示例**: 在根路径 `/` 上响应 "Hello, you've requested: [path]"。

```go
package main

import (
	"fmt"
	"net/http"
)

func main(){
	http.HandleFunc("/",func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w,"Hello, you've requested: %s\n",r.URL.Path)
	})
	http.ListenAndServe(":8080",nil)
}
```

**代码讲解**:
- `http.HandleFunc("/", ...)`: 注册一个 HTTP 处理函数，当用户访问根路径 `/` 时，会执行匿名函数。
- `func(w http.ResponseWriter, r *http.Request)`: 这是 HTTP 处理函数的签名，`w` 用于写入响应，`r` 包含请求信息。
- `fmt.Fprintf(w, ...)`: 将格式化的字符串写入 HTTP 响应。
- `http.ListenAndServe(":8080", nil)`: 启动一个 HTTP 服务器，监听在 `8080` 端口。`nil` 表示使用默认的 ServeMux。

## ch02 - HTTP Server

- **核心概念**: 在 HTTP 服务器中提供静态文件服务。
- **实现**: 使用 `http.FileServer` 创建文件服务器，`http.StripPrefix` 移除 URL 前缀，然后通过 `http.Handle` 注册到特定路径。
- **示例**: 将 `static/` 目录下的文件通过 `/static/` 路径暴露。

```go
package main

import (
	"fmt"
	"net/http"
)

func main(){
	http.HandleFunc("/",func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w,"welcome to my website")
	})

	// set file server
	fs := http.FileServer(http.Dir("static/"))
	http.Handle("/static/",http.StripPrefix("/static/",fs))

	http.ListenAndServe(":8080",nil)
}
```

**代码讲解**:
- `http.FileServer(http.Dir("static/"))`: 创建一个文件服务器，它会从 `static/` 目录下提供文件。
- `http.StripPrefix("/static/", fs)`: 移除请求 URL 中的 `/static/` 前缀，这样文件服务器就能正确地找到 `static/` 目录下的文件。
- `http.Handle("/static/", ...)`: 将文件服务器注册到 `/static/` 路径，所有以 `/static/` 开头的请求都会由这个文件服务器处理。

## ch03 - Routing

- **核心概念**: 路由是根据 URL 路径将请求分发到不同的处理函数。虽然示例代码为空，但在实际应用中，路由是构建复杂 Web 应用的基础。
- **常用库**: `gorilla/mux` 是一个流行的第三方路由库，提供了更强大的路由匹配和中间件支持。

```go
package main

import (
	"fmt"
	"net/http"
	"github.com/gorilla/mux"
)

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/products/{key}", ProductHandler)
	http.ListenAndServe(":8080", r)
}

func ProductHandler(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Category: %v\n", vars["key"])
}
```

**代码讲解**:
- `mux.NewRouter()`: 创建一个新的 `mux` 路由器。
- `r.HandleFunc("/products/{key}", ProductHandler)`: 注册一个带有路径参数的路由。`{key}` 是一个占位符，可以匹配任何值。
- `ProductHandler`: 处理 `/products/{key}` 路径的请求。
- `mux.Vars(r)`: 从请求中提取路径参数。
- `http.ListenAndServe(":8080", r)`: 启动 HTTP 服务器并使用 `mux` 路由器处理请求。

## ch04 - Database

- **核心概念**: Go 语言中数据库操作，以 MySQL 为例，涵盖连接、创建表、插入、查询和删除等基本 CRUD 操作。
- **实现**: 使用 `database/sql` 包进行数据库交互，通过 `_ "github.com/go-sql-driver/mysql"` 导入 MySQL 驱动。
- **操作**: 
    - `sql.Open`: 连接数据库。
    - `db.Ping`: 检查数据库连接。
    - `db.Exec`: 执行 DDL (如 `CREATE TABLE`) 和 DML (如 `INSERT`, `DELETE`) 语句。
    - `db.QueryRow`: 查询单行数据。
    - `db.Query`: 查询多行数据。

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// connect to mysql
	db, err := sql.Open("mysql", "root:password@(localhost:3306)/database?parseTime=true")

	if err != nil {
		log.Fatal(err)
	}

	if err := db.Ping(); err != nil {
		log.Fatal(err)
	}
	fmt.Println("connected!")

	// create a new table
	query := `
            CREATE TABLE IF NOT EXISTS users(
                id INT AUTO_INCREMENT,
                username TEXT NOT NULL,
                password TEXT NOT NULL,
                created_at DATETIME,
                PRIMARY KEY (id)
            );`
	
	if _, err := db.Exec(query); err != nil {
		log.Fatal(err)
	}

	// insert a user
	username := "testuser"
	password := "testpass"
	create_at := time.Now()

	insertQuery := `INSERT INTO users (username,password,created_at) VALUES (?,?,?)`

	res, err := db.Exec(insertQuery, username, password, create_at)
	if err != nil {
		log.Fatal(err)
	}

	id, err := res.LastInsertId()
	fmt.Println("Inserted user with ID:", id)
}
```

**代码讲解**:
- `sql.Open("mysql", ...)`: 打开一个到 MySQL 数据库的连接。请替换连接字符串中的 `root:password@(localhost:3306)/database` 为您的实际数据库凭据。
- `db.Ping()`: 验证数据库连接是否仍然活跃。
- `db.Exec(query)`: 执行 SQL 语句，例如创建表或插入数据。对于 `INSERT` 操作，`res.LastInsertId()` 可以获取新插入行的 ID。

## ch05 - Template

- **核心概念**: 使用 Go 的 `html/template` 包进行 HTML 模板渲染，将动态数据填充到 HTML 页面中。
- **实现**: 
    - `template.ParseFiles`: 解析模板文件。
    - `template.Must`: 辅助函数，用于处理模板解析错误。
    - `tmp.Execute`: 将数据与模板结合并写入 `http.ResponseWriter`。
- **数据结构**: 定义结构体来封装需要传递给模板的数据。
- **常用库**: `gorilla/mux` 用于路由。

```go
package main

import (
	"html/template"
	"net/http"

	"github.com/gorilla/mux"
)

type Todo struct{
	Title string
	Done bool
}

type TodoPageData struct {
	PageTitle string
	Todos []Todo
}

func main(){
	tmp := template.Must(template.ParseFiles("layout.html"))
	
	r := mux.NewRouter()
	r.HandleFunc("/",func(w http.ResponseWriter, r *http.Request) {
		// encapsulate data
		data := TodoPageData{
			PageTitle: "Todo-List",
			Todos: []Todo{
				{"eat",true},
				{"drink",true},
				{"sleep",false},
			},
		}
		// return template
		tmp.Execute(w,data)
	})

	// start server
	http.ListenAndServe(":8080",r)
}
```

**代码讲解**:
- `type Todo struct{...}` 和 `type TodoPageData struct{...}`: 定义了用于模板渲染的数据结构。
- `template.Must(template.ParseFiles("layout.html"))`: 解析名为 `layout.html` 的模板文件。`template.Must` 会在解析失败时 panic。
- `tmp.Execute(w, data)`: 将 `data` 结构体中的数据填充到模板中，并将结果写入 `http.ResponseWriter`。

## ch06 - Asset and File

- **核心概念**: 提供静态资源（如 CSS、JavaScript、图片）服务，与 ch02 类似，但更侧重于“资产”的概念。
- **实现**: 同样使用 `http.FileServer` 和 `http.StripPrefix`。
- **示例**: 将 `asset/` 目录下的文件通过 `/static/` 路径暴露。

```go
package main

import "net/http"

func main() {
	fs := http.FileServer(http.Dir("asset/"))
	http.Handle("/static/", http.StripPrefix("/static/", fs))
	http.ListenAndServe(":8080", nil)
}
```

**代码讲解**:
- `http.FileServer(http.Dir("asset/"))`: 创建一个文件服务器，用于提供 `asset/` 目录下的静态文件。
- `http.StripPrefix("/static/", fs)`: 移除请求 URL 中的 `/static/` 前缀，确保文件服务器能正确映射到 `asset/` 目录。
- `http.Handle("/static/", ...)`: 将文件服务器注册到 `/static/` 路径，使得所有 `/static/` 开头的请求都由该文件服务器处理。

## ch07 - Form

- **核心概念**: 处理 HTML 表单提交，获取表单数据。
- **实现**: 
    - 根据 HTTP 请求方法 (`r.Method`) 判断是 GET 请求（显示表单）还是 POST 请求（处理表单数据）。
    - 使用 `r.FormValue("field_name")` 获取表单字段的值。
- **模板**: 结合 `html/template` 来渲染表单页面和显示提交结果。

```go
package main

import (
	"html/template"
	"net/http"
)

type ContactDetail struct {
	Email   string
	Subject string
	Message string
}

func main() {
	tmpl := template.Must(template.ParseFiles("forms.html"))

	http.HandleFunc("/",func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			tmpl.Execute(w,nil)
			return
		}
		contactDetail := ContactDetail{
			Email: r.FormValue("email"),
			Subject: r.FormValue("subject"),
			Message: r.FormValue("message"),
		}
		// do something with it...
		tmpl.Execute(w,contactDetail)
	})

	http.ListenAndServe(":8080", nil)
}
```

**代码讲解**:
- `type ContactDetail struct{...}`: 定义一个结构体来存储表单提交的数据。
- `tmpl := template.Must(template.ParseFiles("forms.html"))`: 解析用于显示表单的 HTML 模板。
- `if r.Method != http.MethodPost`: 判断请求方法，如果是 GET 请求则显示表单，如果是 POST 请求则处理表单数据。
- `r.FormValue("email")`: 获取表单中 `name="email"` 的字段值。`FormValue` 会自动解析 `application/x-www-form-urlencoded` 和 `multipart/form-data` 类型的表单数据。

## ch08 - Middleware

- **核心概念**: 中间件是在请求到达最终处理函数之前或之后执行的代码，用于实现日志记录、认证、压缩等横切关注点。
- **基本实现**: 通过包装 `http.HandlerFunc` 来实现简单的中间件，例如日志记录。
- **高级实现**: 定义 `Middleware` 类型 (`func(http.HandlerFunc) http.HandlerFunc`)，并实现 `Chain` 函数来链式调用多个中间件，提供更灵活的中间件管理。

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

type Middleware func(http.HandlerFunc) http.HandlerFunc

func Logging() Middleware{
	return func(hf http.HandlerFunc) http.HandlerFunc {
		return func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			defer func ()  {
				log.Println(r.URL.Path,time.Since(start))
			}()
			hf(w,r)
		}
	}
}

func Chain(f http.HandlerFunc,ms... Middleware) http.HandlerFunc{
	for _,middleware := range ms{
		f = middleware(f)
	}
	return f
}

func hello(w http.ResponseWriter,r *http.Request){
	fmt.Fprintln(w,"Hello there")
}

func main(){
	http.HandleFunc("/",Chain(hello,Logging()))
	http.ListenAndServe(":8080",nil)
}
```

**代码讲解**:
- `type Middleware func(http.HandlerFunc) http.HandlerFunc`: 定义了一个 `Middleware` 类型，它是一个函数，接受一个 `http.HandlerFunc` 并返回一个新的 `http.HandlerFunc`。
- `Logging() Middleware`: 这是一个中间件工厂函数，它返回一个 `Middleware` 类型的函数。这个中间件会在处理请求前后记录日志。
- `Chain(f http.HandlerFunc,ms... Middleware) http.HandlerFunc`: 这个函数将多个中间件按顺序应用到一个 `http.HandlerFunc` 上，形成一个处理链。请求会依次经过每个中间件，最后到达最终的处理函数 `f`。

## ch09 - Sessions

- **核心概念**: 会话管理，用于在多个 HTTP 请求之间维护用户状态。
- **实现**: 使用 `github.com/gorilla/sessions` 库。
- **操作**: 
    - `sessions.NewCookieStore`: 创建基于 Cookie 的 session 存储。
    - `store.Get`: 获取或创建一个 session。
    - `session.Values`: 存储 session 数据。
    - `session.Save`: 保存 session 到客户端 Cookie。
- **示例**: 模拟用户登录、登出和访问受保护的资源。

```go
package main

import (
	"fmt"
	"net/http"
	"github.com/gorilla/sessions"
)

var (
	key   = []byte("super-secret-key")
	store = sessions.NewCookieStore(key)
)

func secret(w http.ResponseWriter, r *http.Request) {
	session, _ := store.Get(r, "cookie-name")

	if auth, ok := session.Values["authenticated"].(bool); !ok || !auth {
		http.Error(w, "Forbidden", http.StatusForbidden)
		return
	}
	fmt.Fprintln(w, "The cake is a lie!")
}

func login(w http.ResponseWriter, r *http.Request) {
	session, _ := store.Get(r, "cookie-name")
	session.Values["authenticated"] = true
	session.Save(r, w)
	fmt.Fprintln(w, "Successfully logged in!")
}

func main() {
	http.HandleFunc("/secret", secret)
	http.HandleFunc("/login", login)
	http.ListenAndServe(":8080", nil)
}
```

**代码讲解**:
- `sessions.NewCookieStore(key)`: 创建一个新的 Cookie 存储，`key` 用于加密 session 数据。
- `store.Get(r, "cookie-name")`: 从请求中获取名为 "cookie-name" 的 session。如果不存在，则创建一个新的。
- `session.Values["authenticated"] = true`: 在 session 中设置一个键值对，表示用户已认证。
- `session.Save(r, w)`: 将 session 的更改保存到客户端的 Cookie 中。
- `secret` 函数通过检查 `session.Values["authenticated"]` 来判断用户是否登录，从而保护敏感资源。

## ch10 - JSON

- **核心概念**: 处理 JSON 数据的编码（Go 对象转 JSON 字符串）和解码（JSON 字符串转 Go 对象）。
- **实现**: 使用 `encoding/json` 包。
- **操作**: 
    - `json.NewDecoder(r.Body).Decode(&user)`: 从请求体中解码 JSON 到 Go 结构体。
    - `json.NewEncoder(w).Encode(data)`: 将 Go 结构体编码为 JSON 并写入响应体。
- **结构体标签**: 使用 `json:"field_name"` 标签来控制 JSON 字段名。

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
)

type User struct{
	FirstName string `json:"firstname"`
	LastName string `json:"lastname"`
	Age int `json:"age"`
}

func main(){
	http.HandleFunc("/decode",func(w http.ResponseWriter, r *http.Request) {
		var user User
		json.NewDecoder(r.Body).Decode(&user)
		fmt.Fprintf(w, "%s %s is %d years old!", user.FirstName, user.LastName, user.Age)
	})

	http.HandleFunc("/encode",func(w http.ResponseWriter, r *http.Request) {
		peter := User{
			FirstName: "Jo",
			LastName: "Dio",
			Age: 100,
		}
		json.NewEncoder(w).Encode(peter)
	})

	http.ListenAndServe(":8080", nil)
}
```

**代码讲解**:
- `type User struct{...}`: 定义一个结构体，用于映射 JSON 数据。`json:"..."` 标签指定了结构体字段对应的 JSON 键名。
- `json.NewDecoder(r.Body).Decode(&user)`: 创建一个 JSON 解码器，从请求体 `r.Body` 中读取 JSON 数据并解码到 `user` 结构体中。
- `json.NewEncoder(w).Encode(peter)`: 创建一个 JSON 编码器，将 `peter` 结构体编码为 JSON 格式，并写入响应体 `w`。

## ch11 - Websocket

- **核心概念**: 实现全双工通信协议 WebSocket，用于实时应用。
- **实现**: 使用 `github.com/gorilla/websocket` 库。
- **操作**: 
    - `websocket.Upgrader`: 配置 WebSocket 升级器，例如 `CheckOrigin` 用于跨域检查。
    - `upgrader.Upgrade`: 将 HTTP 连接升级为 WebSocket 连接。
    - `conn.ReadMessage`: 读取 WebSocket 消息。
    - `conn.WriteMessage`: 发送 WebSocket 消息。
- **示例**: 实现一个简单的 WebSocket 回显服务器，并将 `websockets.html` 作为客户端页面。

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize: 1024,
	WriteBufferSize: 1024,
	CheckOrigin: func(r *http.Request) bool {
		return true
	},
}

func main(){
	http.HandleFunc("/echo",func(w http.ResponseWriter, r *http.Request) {
		conn,err := upgrader.Upgrade(w,r,nil)
		if err != nil {
			http.Error(w, "Could not open websocket connection", http.StatusBadRequest)
			return
		}

		for{
			msgType,msg,err := conn.ReadMessage()
			if err != nil {
				fmt.Println("Read error:", err)
				return
			}
			fmt.Printf("%s sent: %s\n", conn.RemoteAddr(), string(msg))

			if err := conn.WriteMessage(msgType, msg); err != nil {
				fmt.Println("Write error:", err)
				return
			}
		}

	})
	http.ListenAndServe(":8080", nil)
}
```

**代码讲解**:
- `websocket.Upgrader`: 用于将普通的 HTTP 连接升级为 WebSocket 连接。`CheckOrigin` 设置为 `true` 允许跨域连接（生产环境中应谨慎设置）。
- `upgrader.Upgrade(w, r, nil)`: 执行连接升级操作，成功后返回一个 `*websocket.Conn` 对象，代表 WebSocket 连接。
- `conn.ReadMessage()`: 从 WebSocket 连接中读取消息。它返回消息类型、消息数据和可能的错误。
- `conn.WriteMessage(msgType, msg)`: 向 WebSocket 连接写入消息，将接收到的消息原样发送回客户端，实现了“回显”功能。

## ch12 - Password Hashing

- **核心概念**: 使用安全的哈希算法存储用户密码，而不是明文存储，以提高安全性。
- **实现**: 使用 `golang.org/x/crypto/bcrypt` 包。
- **操作**: 
    - `bcrypt.GenerateFromPassword`: 生成密码哈希。
    - `bcrypt.CompareHashAndPassword`: 比较明文密码和哈希值是否匹配。
- **安全性**: `bcrypt` 是一种自适应的哈希函数，可以抵御彩虹表攻击和暴力破解。

```go
package main

import (
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

func HashPassword(pwd string)(string,error){
	bytes,err := bcrypt.GenerateFromPassword([]byte(pwd),14)
	return string(bytes),err
}
func CheckPasswordHash(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash),[]byte(password))
	return err == nil
}

func main(){
    password := "secret"
    hash, _ := HashPassword(password)

    fmt.Println("Password:", password)
    fmt.Println("Hash:    ", hash)

    match := CheckPasswordHash(password, hash)
    fmt.Println("Match:   ", match)
}
```

**代码讲解**:
- `bcrypt.GenerateFromPassword([]byte(pwd), 14)`: 使用 bcrypt 算法对密码进行哈希。第二个参数 `14` 是成本因子（cost factor），值越大，哈希计算越慢，安全性越高，但也会消耗更多计算资源。
- `bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))`: 比较明文密码和哈希值是否匹配。这个函数会处理哈希和盐值的提取，并进行安全的比较，防止时序攻击。