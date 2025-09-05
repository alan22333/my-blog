---
title: Go语言之Swaggo
description: 'Swaggo + Gin'
tags: ['go']
toc: false
date: 2025-09-02 18:25:17
categories:
    - go
    - basic
---

---

# 使用 Swaggo 为 Gin 项目自动生成 Swagger 文档

在开发 Go Web 项目的时候，API 文档的维护一直是一个痛点。手写文档容易过时，缺乏统一标准，而 Swaggo 为我们提供了一种 **自动化生成 Swagger 文档** 的方式，尤其适合和 Gin 框架搭配使用。本文将介绍 Swaggo 的基本用法，并结合 Gin 展示最佳实践。

---

## 一、为什么选择 Swaggo + Gin？

* **Swaggo (github.com/swaggo/swag)** 是 Go 语言最常用的 Swagger 文档生成工具。
* 它通过 **注释（Annotation）** 自动解析 API 信息，生成 `swagger.json`/`swagger.yaml` 文件。
* Gin 作为高性能 Web 框架，与 Swaggo 配合，可以让我们在写路由和 Handler 时，顺便写上注释，做到 **代码即文档**。

最终效果是：

* 在浏览器访问 `http://localhost:8080/swagger/index.html`
* 自动看到漂亮的 Swagger UI 界面，方便前后端联调。

---

## 二、安装与初始化

### 1. 安装依赖

```bash
go get -u github.com/gin-gonic/gin
go get -u github.com/swaggo/swag/cmd/swag
go get -u github.com/swaggo/files
go get -u github.com/swaggo/gin-swagger
```

> `swag` 是命令行工具，用于扫描项目注释生成文档。

### 2. 初始化 Swagger

进入项目根目录，执行：

```bash
swag init
```

该命令会扫描 `main.go` 和其他源文件的注释，生成 `docs/` 目录，里面包含 `docs.go` 和 `swagger.json` 等。

---

## 三、在 Gin 中接入 Swaggo

### 1. 项目结构推荐

```
.
├── main.go
├── docs/         # swag 自动生成
├── router/       # 路由管理
├── controller/   # 业务逻辑
└── model/        # 数据模型
```

### 2. 在 `main.go` 添加 Swagger 配置

```go
package main

import (
	"github.com/gin-gonic/gin"

	_ "your_project/docs" // 这里要引入 docs 包，确保 swagger 文档能被注册

	ginSwagger "github.com/swaggo/gin-swagger"
	"github.com/swaggo/files"
)

// @title           Gin + Swaggo 示例 API
// @version         1.0
// @description     一个使用 Swaggo 自动生成文档的 Gin 项目
// @host            localhost:8080
// @BasePath        /api/v1
func main() {
	r := gin.Default()

	// 注册 Swagger 路由
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// 注册业务路由
	api := r.Group("/api/v1")
	{
		api.GET("/ping", PingHandler)
	}

	r.Run(":8080")
}

// PingHandler godoc
// @Summary      健康检查
// @Description  返回 pong
// @Tags         系统
// @Produce      json
// @Success      200  {string}  string "pong"
// @Router       /ping [get]
func PingHandler(c *gin.Context) {
	c.JSON(200, gin.H{"message": "pong"})
}
```

此时运行 `swag init && go run main.go`，访问：

👉 `http://localhost:8080/swagger/index.html`
即可看到 Swagger UI 页面。

---

## 四、常用注释写法

Swaggo 注释写在 **Handler 函数上方**，以下是常见标签：

* `@Summary`：API 简短说明
* `@Description`：详细描述
* `@Tags`：接口分类（分组）
* `@Param`：请求参数
* `@Success`：返回成功响应
* `@Failure`：返回失败响应
* `@Router`：路由及方法（必须，格式：`/path [method]`）

### 示例：带参数的接口

```go
// GetUser godoc
// @Summary      获取用户
// @Description  根据ID获取用户信息
// @Tags         用户
// @Param        id   path      int  true  "用户ID"
// @Produce      json
// @Success      200  {object}  model.User
// @Failure      400  {object}  map[string]string
// @Router       /user/{id} [get]
func GetUser(c *gin.Context) {
	id := c.Param("id")
	c.JSON(200, gin.H{"id": id, "name": "Tom"})
}
```

其中 `model.User` 是我们定义的数据模型，可以在 Swagger 文档中自动展示。

---

## 五、最佳实践

### 1. 项目启动时自动生成文档

在 CI/CD 或本地开发时，可以把 `swag init` 放到 `Makefile` 或脚本里，避免手动执行。

```makefile
swagger:
	swag init --parseDependency --parseInternal
```

### 2. 合理使用 `@Tags`

把接口分组（如 用户、订单、系统），这样在 Swagger UI 页面结构清晰。

### 3. 模型定义与返回值一致

如果接口返回 JSON 结构，最好在 `model/` 中定义结构体，并在注释里引用。

```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}
```

然后在注释里写：

```go
// @Success 200 {object} model.User
```

这样前端在 Swagger UI 上就能看到完整的返回结构。

### 4. 避免文档与代码不一致

* 尽量在写 Handler 的时候就写好注释。
* 定期运行 `swag init` 确保文档更新。
* 可以在 CI 里加个检查，避免遗漏。

### 5. 安全认证（JWT / API Key）

Swaggo 也支持在文档中描述认证方式，比如 JWT：

```go
// @Security BearerAuth
```

需要在 `main.go` 顶部添加：

```go
// @securityDefinitions.apikey BearerAuth
// @in header
// @name Authorization
```

---