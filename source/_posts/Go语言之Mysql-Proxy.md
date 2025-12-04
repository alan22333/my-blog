---
title: Go语言之Mysql-Proxy
description: '一个用 Go 实现的 MySQL 代理程序，可以拦截、过滤和记录客户端与 MySQL 服务器之间的所有查询'
tags: ['go']
toc: false
date: 2025-11-29 15:57:16
categories:
    - go
    - project
---

> 前言：最近想学学Agent开发（eino），但是又不太想照着别人的去学，所以想了一个小应用自己做做看。主要的思路就是做一个Mysql的代理程序，然后加上AI功能（安全过滤、自然语言转SQL、AI数据脱敏、SQL学习助手等等一些功能），主要技术就用Go + Eino + go-mysql/server（或者加一个TUI来作为前端），而本文则作为一个基本模板，也就是一个Mysql代理服务器，由go-mysql库来提供Mysql协议解析等等，避免重复造轮子了。
---

## 项目概述

这是一个使用 Go 语言实现的 MySQL 代理服务器，可以拦截、记录和过滤客户端与 MySQL 服务器之间的所有通信。

---

## 核心概念

### 1. 代理模式 (Proxy Pattern)

```
客户端 <---> 代理服务器 <---> 真实 MySQL 服务器
```

- **客户端**: Navicat、MySQL CLI 等任何 MySQL 客户端
- **代理服务器**: 本项目实现的中间层
- **真实 MySQL**: 实际存储数据的 MySQL 服务器

### 2. 透明代理

客户端无需知道代理的存在，就像直接连接到 MySQL 服务器一样。代理完全模拟 MySQL 服务器的行为。

---

## 架构设计

### 目录结构

```
.
├── cmd/proxy/main.go          # 程序入口，处理命令行参数和启动逻辑
├── internal/proxy/
│   ├── server.go             # TCP 服务器，监听客户端连接
│   └── handler.go            # 连接处理器，实现 MySQL 协议
├── pkg/config/
│   └── config.go             # 配置管理
└── go.mod                    # 依赖管理
```

### 模块职责

**main**：
- 程序入口，提供服务器启动和优雅关闭
- 解析命令行参数到配置
- 设置线程间的通信管道

**server**：
- 监听端口上的TCP连接（客户端）
- 创建线程处理客户端连接

**handler**：
- 实现 MySQL 协议的服务端（代理后端）
- 解析客户端发送的 MySQL 命令
- 转发查询到真实 MySQL
- 返回结果给客户端
- 实现查询过滤和记录

**config**：
- 定义相关配置（代理地址端口、真实数据库地址端口、用户密码等等）

### 实现的接口方法

| 方法 | 对应 MySQL 命令 | 说明 |
|------|----------------|------|
| `Handle()` | - | 主循环，持续处理命令 |
| `UseDB(dbName)` | `USE database;` | 切换数据库 |
| `HandleQuery(query)` | `SELECT`, `INSERT`, `UPDATE`, `DELETE` 等 | 处理 SQL 查询 |
| `HandleFieldList(table, wildcard)` | `COM_FIELD_LIST` | 获取表字段列表 |
| `HandleStmtPrepare(query)` | `PREPARE stmt FROM ...` | 预处理语句准备 |
| `HandleStmtExecute(context, query, args)` | `EXECUTE stmt USING ...` | 执行预处理语句 |
| `HandleStmtClose(context)` | `DEALLOCATE PREPARE stmt` | 关闭预处理语句 |
| `HandleOtherCommand(cmd, data)` | 其他命令 | 处理不常用命令 |

### 客户端执行流程

```
1. 客户端发起连接
   ↓
2. Server.Accept() 接受连接
   ↓
3. 创建 ConnectionHandler
   ↓
4. server.NewConn() 建立 MySQL 协议连接
   ↓
5. 进入循环：conn.HandleCommand()
   ↓
6. 客户端发送查询: SELECT * FROM users
   ↓
7. 调用 HandleQuery("SELECT * FROM users")
   ↓
8. filterQuery() 过滤查询
   │   ├─ 检测危险操作 (UPDATE/DELETE/DROP)
   │   ├─ 记录日志
   │   └─ 可以修改或阻止查询
   ↓
9. 连接到后端 MySQL (如果还未连接)
   ↓
10. backendDB.Execute(filteredQuery)
   ↓
11. 后端 MySQL 执行查询并返回结果
   ↓
12. 将结果返回给客户端
   ↓
13. 继续等待下一个命令
```
---

## go-mysql

使用 `github.com/go-mysql-org/go-mysql` 开源库处理 MySQL 协议：

#### 客户端模式（连接到真实 MySQL）
```go
conn, err := client.Connect(
    "127.0.0.1:3306",  // MySQL 地址
    "test",             // 用户名
    "test",             // 密码
    "",                 // 数据库
)
result, err := conn.Execute("SELECT * FROM users")
```

#### 服务端模式（接受客户端连接）
```go
// 创建 MySQL 协议服务端连接
conn, err := server.NewConn(
    clientConn,  // TCP 连接
    "root",      // 用户名（用于握手）
    "",          // 密码
    handler,     // 实现 Handler 接口的处理器
)

// 循环处理客户端命令
for {
    err := conn.HandleCommand()  // 自动调用相应的 Handle* 方法
}
```

---

## 代码实现

也可以不看上面，下面代码里有海量AI注释，一看就懂喵

**main**：

```Go
package main

import (
	"flag"
	"os"
	"os/signal"
	"syscall"

	"demo/internal/proxy"
	"demo/pkg/config"

	"github.com/siddontang/go-log/log"
)

// main 程序入口函数
// 负责：
//  1. 解析命令行参数
//  2. 配置日志级别
//  3. 创建并启动代理服务器
//  4. 处理优雅关闭
func main() {
	// 解析命令行参数
	// flag 包提供了命令行参数解析功能
	// 格式：flag.类型("参数名", 默认值, "说明")
	proxyAddr := flag.String("proxy-addr", "0.0.0.0", "Proxy server address")
	proxyPort := flag.Int("proxy-port", 3307, "Proxy server port")
	backendAddr := flag.String("backend-addr", "127.0.0.1", "Backend MySQL server address")
	backendPort := flag.Int("backend-port", 3306, "Backend MySQL server port")
	backendUser := flag.String("backend-user", "test", "Backend MySQL username")
	backendPassword := flag.String("backend-password", "test", "Backend MySQL password")
	backendDatabase := flag.String("backend-db", "", "Backend MySQL database")
	logLevel := flag.String("log-level", "info", "Log level (debug, info, warn, error)")

	// 解析命令行参数
	// 调用后，所有 flag 变量会被赋值
	flag.Parse()

	// 设置日志级别
	// 根据用户指定的日志级别配置日志输出
	switch *logLevel {
	case "debug":
		log.SetLevel(log.LevelDebug) // 最详细，包含调试信息
	case "info":
		log.SetLevel(log.LevelInfo) // 一般信息，推荐使用
	case "warn":
		log.SetLevel(log.LevelWarn) // 警告信息
	case "error":
		log.SetLevel(log.LevelError) // 只显示错误
	default:
		log.SetLevel(log.LevelInfo) // 默认使用 info 级别
	}

	// 创建配置对象
	// 将命令行参数组装成配置结构
	cfg := &config.Config{
		ProxyAddr:       *proxyAddr, // 注意：*proxyAddr 是解引用，获取指针指向的值
		ProxyPort:       *proxyPort,
		BackendAddr:     *backendAddr,
		BackendPort:     *backendPort,
		BackendUser:     *backendUser,
		BackendPassword: *backendPassword,
		BackendDatabase: *backendDatabase,
	}

	// 打印启动配置
	// 让用户知道代理使用的配置
	log.Infof("Starting MySQL Proxy with configuration:")
	log.Infof("  Proxy Address: %s:%d", cfg.ProxyAddr, cfg.ProxyPort)
	log.Infof("  Backend MySQL: %s:%d", cfg.BackendAddr, cfg.BackendPort)
	log.Infof("  Backend User: %s", cfg.BackendUser)
	log.Infof("  Backend Database: %s", cfg.BackendDatabase)

	// 创建代理服务器实例
	server := proxy.NewServer(cfg)

	// 设置信号处理
	// 用于优雅关闭服务器（响应 Ctrl+C 等信号）
	sigChan := make(chan os.Signal, 1) // 创建信号通道，缓冲大小为 1
	// 注册要监听的信号
	// SIGINT: Ctrl+C 触发
	// SIGTERM: kill 命令默认信号
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	// 创建错误通道
	// 用于从 goroutine 中接收服务器启动错误
	errChan := make(chan error, 1)

	// 在新的 goroutine 中启动服务器
	// 这样主 goroutine 可以继续处理信号
	go func() {
		if err := server.Start(); err != nil {
			errChan <- err // 如果启动失败，将错误发送到错误通道
		}
	}()

	// 等待服务器关闭信号或错误
	// select 会阻塞，直到某个 case 可以执行
	select {
	case <-sigChan:
		// 收到关闭信号（如 Ctrl+C）
		log.Infof("Received shutdown signal, stopping server...")
		if err := server.Stop(); err != nil {
			log.Errorf("Error stopping server: %v", err)
			os.Exit(1) // 以错误码 1 退出
		}
		log.Infof("Server stopped gracefully")
		// 正常退出，退出码 0
	case err := <-errChan:
		// 服务器启动或运行时出错
		log.Errorf("Server error: %v", err)
		os.Exit(1) // 以错误码 1 退出
	}
}
```

**server**:
```Go
package proxy

import (
	"fmt"
	"net"

	"demo/pkg/config"

	"github.com/siddontang/go-log/log"
)

// Server 代表代理服务器的主结构体
// 负责监听客户端连接并分发给处理器
type Server struct {
	cfg      *config.Config // 服务器配置
	listener net.Listener   // TCP 监听器，用于接收客户端连接
	running  bool           // 服务器运行状态标志
}

// NewServer 创建一个新的代理服务器实例
// 参数：
//
//	cfg - 服务器配置，包含代理和后端 MySQL 的连接信息
//
// 返回：
//
//	*Server - 新创建的服务器实例
func NewServer(cfg *config.Config) *Server {
	return &Server{
		cfg:     cfg,
		running: false, // 初始状态为未运行
	}
}

// Start 启动代理服务器
// 工作流程：
//  1. 在指定地址和端口上创建 TCP 监听器
//  2. 进入循环，等待客户端连接
//  3. 为每个新连接创建独立的 goroutine 处理
//
// 返回：
//
//	error - 启动失败时返回错误信息
func (s *Server) Start() error {
	// 构建监听地址，格式：IP:端口
	addr := fmt.Sprintf("%s:%d", s.cfg.ProxyAddr, s.cfg.ProxyPort)

	// 创建 TCP 监听器
	// net.Listen 会绑定指定的地址和端口，准备接收连接
	listener, err := net.Listen("tcp", addr)
	if err != nil {
		return fmt.Errorf("failed to listen on %s: %w", addr, err)
	}

	s.listener = listener
	s.running = true // 标记服务器为运行状态

	// 记录服务器启动信息
	log.Infof("MySQL Proxy server started on %s", addr)
	log.Infof("Backend MySQL server: %s:%d", s.cfg.BackendAddr, s.cfg.BackendPort)

	// 主循环：持续接收客户端连接
	for s.running {
		// Accept() 会阻塞，直到有新的客户端连接
		conn, err := s.listener.Accept()
		if err != nil {
			// 如果服务器正在运行但接收连接失败，记录错误
			if s.running {
				log.Errorf("Failed to accept connection: %v", err)
			}
			continue // 继续等待下一个连接
		}

		// 记录新连接的客户端地址
		log.Infof("New connection from %s", conn.RemoteAddr())

		// 为每个连接启动独立的 goroutine 处理
		// 这样可以同时处理多个客户端连接，不会阻塞主循环
		go s.handleConnection(conn)
	}

	return nil
}

// Stop 停止代理服务器
// 关闭监听器，不再接收新连接
// 返回：
//
//	error - 关闭失败时返回错误信息
func (s *Server) Stop() error {
	s.running = false // 标记服务器为停止状态
	if s.listener != nil {
		return s.listener.Close() // 关闭监听器
	}
	return nil
}

// handleConnection 处理单个客户端连接
// 这个方法在独立的 goroutine 中运行
// 参数：
//
//	clientConn - 客户端的 TCP 连接
func (s *Server) handleConnection(clientConn net.Conn) {
	// 确保连接最终会被关闭，避免资源泄漏
	defer clientConn.Close()

	// 创建连接处理器，负责处理 MySQL 协议和查询转发
	handler := NewConnectionHandler(clientConn, s.cfg)

	// 处理连接，如果出错则记录错误信息
	if err := handler.Handle(); err != nil {
		log.Errorf("Error handling connection from %s: %v", clientConn.RemoteAddr(), err)
	}

	// 连接处理完成，记录日志
	log.Infof("Connection from %s closed", clientConn.RemoteAddr())
}

```

**Handler**:
```Go
package proxy

import (
	"fmt"
	"net"
	"strings"

	"demo/pkg/config"

	"github.com/go-mysql-org/go-mysql/client"
	"github.com/go-mysql-org/go-mysql/mysql"
	"github.com/go-mysql-org/go-mysql/server"
	"github.com/siddontang/go-log/log"
)

// ConnectionHandler 连接处理器
// 负责处理单个客户端连接的所有 MySQL 协议交互
// 实现了 server.Handler 接口，用于处理各种 MySQL 命令
type ConnectionHandler struct {
	clientConn net.Conn       // 客户端的 TCP 连接
	cfg        *config.Config // 代理配置信息
	backendDB  *client.Conn   // 与后端 MySQL 服务器的连接（懒加载）
}

// NewConnectionHandler 创建一个新的连接处理器
// 参数：
//
//	clientConn - 客户端的 TCP 连接
//	cfg - 代理配置，包含后端 MySQL 的连接信息
//
// 返回：
//
//	*ConnectionHandler - 新创建的连接处理器实例
func NewConnectionHandler(clientConn net.Conn, cfg *config.Config) *ConnectionHandler {
	return &ConnectionHandler{
		clientConn: clientConn,
		cfg:        cfg,
		// backendDB 初始为 nil，在第一次执行查询时才会连接
	}
}

// Handle 处理客户端连接的主函数
// 工作流程：
//  1. 使用 go-mysql 库创建 MySQL 协议连接
//  2. 进入循环，持续处理客户端发来的命令
//  3. 每个命令会调用相应的 Handle* 方法（如 HandleQuery、UseDB 等）
//
// 返回：
//
//	error - 连接处理过程中的错误
func (h *ConnectionHandler) Handle() error {
	// 创建 MySQL 协议连接
	// 参数说明：
	//   - h.clientConn: 客户端的 TCP 连接
	//   - "root": 客户端看到的用户名（用于协议握手）
	//   - "": 密码（空密码，实际认证在后端进行）
	//   - h: Handler 接口实现，用于处理各种 MySQL 命令
	conn, err := server.NewConn(h.clientConn, "root", "", h)
	if err != nil {
		return fmt.Errorf("failed to create MySQL connection: %w", err)
	}

	// 主循环：持续处理客户端命令
	// HandleCommand() 会阻塞等待客户端发送命令，然后调用相应的处理方法
	for {
		if err := conn.HandleCommand(); err != nil {
			// mysql.ErrBadConn 表示连接已关闭，这是正常情况
			if err != mysql.ErrBadConn {
				log.Errorf("Handle command error: %v", err)
			}
			return err
		}
	}
}

// UseDB 处理客户端切换数据库的请求
// 对应 SQL: USE database_name;
// 参数：
//
//	dbName - 要切换到的数据库名
//
// 返回：
//
//	error - 切换失败时返回错误
func (h *ConnectionHandler) UseDB(dbName string) error {
	log.Infof("Client switching to database: %s", dbName)

	// 如果还没有连接到后端 MySQL，先建立连接
	// 这是懒加载模式，只在需要时才连接
	if h.backendDB == nil {
		if err := h.connectBackend(""); err != nil {
			return err
		}
	}

	// 在后端 MySQL 上执行数据库切换
	if err := h.backendDB.UseDB(dbName); err != nil {
		return fmt.Errorf("failed to switch database on backend: %w", err)
	}

	return nil
}

// HandleQuery 处理客户端发送的 SQL 查询
// 这是最核心的方法，所有 SELECT、INSERT、UPDATE、DELETE 等查询都经过这里
// 参数：
//
//	query - 客户端发送的 SQL 查询字符串
//
// 返回：
//
//	*mysql.Result - 查询结果（包含结果集或影响的行数）
//	error - 查询执行失败时返回错误
func (h *ConnectionHandler) HandleQuery(query string) (*mysql.Result, error) {
	// 记录收到的原始查询
	log.Infof("Query received: %s", query)

	// 过滤和处理查询
	// 这是实现查询拦截、修改、阻止的关键位置
	filteredQuery := h.filterQuery(query)
	log.Infof("Filtered query: %s", filteredQuery)

	// 如果还没有连接到后端 MySQL，先建立连接
	if h.backendDB == nil {
		if err := h.connectBackend(""); err != nil {
			return nil, err
		}
	}

	// 在后端 MySQL 上执行过滤后的查询
	result, err := h.backendDB.Execute(filteredQuery)
	if err != nil {
		log.Errorf("Failed to execute query on backend: %v", err)
		return nil, err
	}

	// 记录执行结果
	log.Infof("Query executed successfully, affected rows: %d", result.AffectedRows)
	return result, nil
}

// HandleFieldList 处理字段列表请求
// 对应 MySQL 协议的 COM_FIELD_LIST 命令
// 一些客户端（如旧版 MySQL 命令行）使用这个命令获取表结构
// 参数：
//
//	table - 表名
//	fieldWildcard - 字段通配符（通常不使用）
//
// 返回：
//
//	[]*mysql.Field - 字段列表
//	error - 获取失败时返回错误
func (h *ConnectionHandler) HandleFieldList(table string, fieldWildcard string) ([]*mysql.Field, error) {
	log.Infof("Field list request for table: %s, wildcard: %s", table, fieldWildcard)

	// 如果还没有连接到后端 MySQL，先建立连接
	if h.backendDB == nil {
		if err := h.connectBackend(""); err != nil {
			return nil, err
		}
	}

	// 执行 SHOW COLUMNS 查询来获取表结构
	// 使用反引号包裹表名，避免表名是保留字时出错
	query := fmt.Sprintf("SHOW COLUMNS FROM `%s`", table)
	result, err := h.backendDB.Execute(query)
	if err != nil {
		return nil, err
	}

	// 解析查询结果，构建字段列表
	var fields []*mysql.Field
	for i := 0; i < result.RowNumber(); i++ {
		// GetString(行号, 列号) 获取结果集中的数据
		// SHOW COLUMNS 返回格式：Field, Type, Null, Key, Default, Extra
		fieldName, _ := result.GetString(i, 0) // 第 0 列是字段名
		fieldType, _ := result.GetString(i, 1) // 第 1 列是字段类型

		// 创建字段描述
		field := &mysql.Field{
			Name: []byte(fieldName),           // 字段名
			Type: h.parseFieldType(fieldType), // 解析字段类型
		}
		fields = append(fields, field)
	}

	return fields, nil
}

// HandleStmtPrepare 处理预处理语句的准备请求
// 预处理语句（Prepared Statement）可以提高性能并防止 SQL 注入
// 对应 SQL: PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?';
// 参数：
//
//	query - 要预处理的 SQL 语句（可能包含 ? 占位符）
//
// 返回：
//
//	params - 参数数量（SQL 中 ? 的个数）
//	columns - 结果列数量
//	context - 预处理语句的上下文（后续执行时使用）
//	error - 准备失败时返回错误
func (h *ConnectionHandler) HandleStmtPrepare(query string) (int, int, interface{}, error) {
	log.Infof("Prepare statement: %s", query)

	// 确保已连接到后端 MySQL
	if h.backendDB == nil {
		if err := h.connectBackend(""); err != nil {
			return 0, 0, nil, err
		}
	}

	// 对查询进行过滤（与普通查询一样的过滤逻辑）
	filteredQuery := h.filterQuery(query)

	// 在后端 MySQL 上准备预处理语句
	stmt, err := h.backendDB.Prepare(filteredQuery)
	if err != nil {
		return 0, 0, nil, err
	}

	// 返回参数数量、列数量和语句上下文
	return stmt.ParamNum(), stmt.ColumnNum(), stmt, nil
}

// HandleStmtExecute 处理预处理语句的执行请求
// 对应 SQL: EXECUTE stmt USING @param1, @param2;
// 参数：
//
//	context - 之前 HandleStmtPrepare 返回的上下文
//	query - 原始查询字符串（用于日志）
//	args - 执行参数（替换 SQL 中的 ? 占位符）
//
// 返回：
//
//	*mysql.Result - 执行结果
//	error - 执行失败时返回错误
func (h *ConnectionHandler) HandleStmtExecute(context interface{}, query string, args []interface{}) (*mysql.Result, error) {
	log.Infof("Execute prepared statement with %d args", len(args))

	// 类型断言：确保 context 是预处理语句对象
	stmt, ok := context.(*client.Stmt)
	if !ok {
		return nil, fmt.Errorf("invalid statement context")
	}

	// 在后端 MySQL 上执行预处理语句，传入参数
	result, err := stmt.Execute(args...)
	if err != nil {
		return nil, err
	}

	return result, nil
}

// HandleStmtClose 处理关闭预处理语句的请求
// 对应 SQL: DEALLOCATE PREPARE stmt;
// 参数：
//
//	context - 之前 HandleStmtPrepare 返回的上下文
//
// 返回：
//
//	error - 关闭失败时返回错误
func (h *ConnectionHandler) HandleStmtClose(context interface{}) error {
	log.Infof("Close prepared statement")

	// 类型断言：确保 context 是预处理语句对象
	stmt, ok := context.(*client.Stmt)
	if !ok {
		return fmt.Errorf("invalid statement context")
	}

	// 关闭预处理语句，释放资源
	return stmt.Close()
}

// HandleOtherCommand 处理其他不常用的 MySQL 命令
// 这是一个兜底方法，处理库未明确支持的命令
// 参数：
//
//	cmd - MySQL 命令类型（字节码）
//	data - 命令数据
//
// 返回：
//
//	error - 默认返回"不支持"错误
func (h *ConnectionHandler) HandleOtherCommand(cmd byte, data []byte) error {
	log.Infof("Other command received: %d, data length: %d", cmd, len(data))
	// 返回 MySQL 错误：未知错误
	return mysql.NewDefaultError(mysql.ER_UNKNOWN_ERROR, "command not supported")
}

// connectBackend 连接到后端 MySQL 服务器
// 这个方法实现了懒加载连接，只在第一次需要时才建立连接
// 参数：
//
//	dbName - 要连接的数据库名（可以为空）
//
// 返回：
//
//	error - 连接失败时返回错误
func (h *ConnectionHandler) connectBackend(dbName string) error {
	// 构建后端 MySQL 的地址
	addr := fmt.Sprintf("%s:%d", h.cfg.BackendAddr, h.cfg.BackendPort)

	log.Infof("Connecting to backend MySQL server at %s", addr)

	// 使用 go-mysql 的 client 包连接到后端 MySQL
	// 参数：
	//   - addr: MySQL 服务器地址
	//   - user: 用户名
	//   - password: 密码
	//   - dbName: 默认数据库（可选）
	conn, err := client.Connect(
		addr,
		h.cfg.BackendUser,
		h.cfg.BackendPassword,
		dbName,
	)
	if err != nil {
		return fmt.Errorf("failed to connect to backend MySQL: %w", err)
	}

	// 保存连接到实例变量
	h.backendDB = conn
	log.Infof("Connected to backend MySQL server successfully")
	return nil
}

// filterQuery 过滤和修改查询
// 这是实现查询拦截、审计、安全控制的核心方法
// 你可以在这里实现各种过滤逻辑：
//   - 阻止危险操作（如 DROP DATABASE）
//   - 记录敏感操作（如 UPDATE、DELETE）
//   - 修改查询（如添加 WHERE 条件）
//   - SQL 注入检测
//   - 查询性能分析
//
// 参数：
//
//	query - 原始 SQL 查询
//
// 返回：
//
//	string - 过滤/修改后的查询（或原查询）
func (h *ConnectionHandler) filterQuery(query string) string {
	// 将查询转换为大写，便于匹配（不影响原查询）
	trimmedQuery := strings.TrimSpace(strings.ToUpper(query))

	// 示例1: 检测 DROP DATABASE 命令
	// 在生产环境中，你可能想要阻止这类危险操作
	if strings.HasPrefix(trimmedQuery, "DROP DATABASE") {
		log.Warnf("Blocked DROP DATABASE command: %s", query)
		// 你可以在这里：
		// 1. 返回一个错误查询，让后端返回错误
		// 2. 返回一个安全的替代查询
		// 3. 抛出错误（需要修改返回类型）
	}

	// 示例2: 记录危险操作（DELETE、UPDATE、DROP）
	// 这些操作可能会修改或删除数据，需要特别关注
	if strings.HasPrefix(trimmedQuery, "DELETE") ||
		strings.HasPrefix(trimmedQuery, "UPDATE") ||
		strings.HasPrefix(trimmedQuery, "DROP") {
		log.Warnf("Dangerous operation detected: %s", query)
		// 在实际应用中，你可以：
		// 1. 将操作记录到审计日志
		// 2. 发送告警通知
		// 3. 根据用户权限决定是否允许执行
	}

	// 示例3: 你可以添加更多过滤规则
	// if strings.Contains(trimmedQuery, "PASSWORD") {
	//     log.Warnf("Query contains sensitive keyword: PASSWORD")
	// }

	// 示例4: 查询重写
	// if strings.HasPrefix(trimmedQuery, "SELECT * FROM USERS") {
	//     return "SELECT id, name, email FROM users" // 隐藏敏感字段
	// }

	// 返回原始查询（未修改）
	// 如果要修改查询，返回修改后的字符串
	return query
}

// parseFieldType 解析 MySQL 字段类型字符串到类型常量
// MySQL 的 SHOW COLUMNS 返回的是字符串形式的类型（如 "varchar(255)"）
// 需要转换为 MySQL 协议使用的数字类型常量
// 参数：
//
//	fieldType - 字段类型字符串（如 "INT", "VARCHAR(50)", "TEXT"）
//
// 返回：
//
//	uint8 - MySQL 协议类型常量
func (h *ConnectionHandler) parseFieldType(fieldType string) uint8 {
	// 转换为大写，便于匹配
	fieldType = strings.ToUpper(fieldType)

	// 使用 switch 匹配字段类型
	// 注意：使用 Contains 而不是精确匹配，因为类型可能包含长度信息
	// 例如：VARCHAR(255)、INT(11) 等
	switch {
	case strings.Contains(fieldType, "INT"):
		// INT, TINYINT, SMALLINT, MEDIUMINT, BIGINT
		return mysql.MYSQL_TYPE_LONG
	case strings.Contains(fieldType, "VARCHAR"), strings.Contains(fieldType, "CHAR"):
		// VARCHAR, CHAR
		return mysql.MYSQL_TYPE_VAR_STRING
	case strings.Contains(fieldType, "TEXT"):
		// TEXT, TINYTEXT, MEDIUMTEXT, LONGTEXT
		return mysql.MYSQL_TYPE_BLOB
	case strings.Contains(fieldType, "DECIMAL"):
		// DECIMAL, NUMERIC
		return mysql.MYSQL_TYPE_DECIMAL
	case strings.Contains(fieldType, "DATETIME"):
		// DATETIME
		return mysql.MYSQL_TYPE_DATETIME
	case strings.Contains(fieldType, "TIMESTAMP"):
		// TIMESTAMP
		return mysql.MYSQL_TYPE_TIMESTAMP
	case strings.Contains(fieldType, "DATE"):
		// DATE（注意：要在 DATETIME 之后检查，避免误匹配）
		return mysql.MYSQL_TYPE_DATE
	case strings.Contains(fieldType, "TIME"):
		// TIME（注意：要在 DATETIME 和 TIMESTAMP 之后检查）
		return mysql.MYSQL_TYPE_TIME
	default:
		// 未知类型，默认返回字符串类型
		return mysql.MYSQL_TYPE_VAR_STRING
	}
}

// Close 关闭处理器和后端连接
// 释放资源，避免连接泄漏
// 返回：
//
//	error - 关闭失败时返回错误
func (h *ConnectionHandler) Close() error {
	if h.backendDB != nil {
		return h.backendDB.Close()
	}
	return nil
}

// 编译时检查：确保 ConnectionHandler 实现了 server.Handler 接口
// 如果没有实现接口的所有方法，编译会失败
// 这是 Go 语言的一个常用技巧，用于在编译期发现接口实现错误
var _ server.Handler = (*ConnectionHandler)(nil)

```

**config**
```Go
package config

// Config 代理服务器的配置结构体
// 包含代理服务器和后端 MySQL 服务器的所有配置信息
type Config struct {
	// Proxy server settings - 代理服务器配置
	ProxyAddr string // 代理服务器监听的 IP 地址，0.0.0.0 表示监听所有网络接口
	ProxyPort int    // 代理服务器监听的端口号，客户端连接到此端口

	// Backend MySQL server settings - 后端真实 MySQL 服务器配置
	BackendAddr     string // 后端 MySQL 服务器的 IP 地址
	BackendPort     int    // 后端 MySQL 服务器的端口号（通常是 3306）
	BackendUser     string // 连接后端 MySQL 的用户名
	BackendPassword string // 连接后端 MySQL 的密码
	BackendDatabase string // 默认连接的数据库名称（可选，为空表示不指定默认数据库）
}

// DefaultConfig 返回一个默认配置
// 默认配置：
// - 代理监听所有接口的 3307 端口
// - 后端 MySQL 在本地 3306 端口
// - 使用 test/test 作为后端认证凭据
func DefaultConfig() *Config {
	return &Config{
		ProxyAddr:       "0.0.0.0",   // 监听所有网络接口
		ProxyPort:       3307,        // 代理端口（避免与 MySQL 默认 3306 冲突）
		BackendAddr:     "127.0.0.1", // 本地 MySQL
		BackendPort:     3306,        // MySQL 标准端口
		BackendUser:     "test",      // 后端用户名
		BackendPassword: "test",      // 后端密码
		BackendDatabase: "",          // 不指定默认数据库
	}
}
```