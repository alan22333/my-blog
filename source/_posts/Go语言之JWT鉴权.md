---
title: Go语言之JWT鉴权
description: 'JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在网络应用环境间以一种简洁、安全的方式传递信息，常用于身份认证与授权'
tags: ['go']
toc: false
date: 2025-07-28 16:23:45
categories:
    - go
    - basic
---


## 🧠 一、JWT 是什么？

JWT（JSON Web Token）是一种开放标准（RFC 7519），用于在网络应用环境间以一种简洁、安全的方式传递信息，常用于身份认证与授权。它是一段由三部分组成的字符串，如下格式：

```
<Header>.<Payload>.<Signature>
```

举个例子：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1c2VyX2lkIjoxLCJleHAiOjE3MDA4NzA1OTEsImlzcyI6ImdvLWpzIn0.
hQ4jkHVz4x5cpQZKlyFP2rRokM8HxM1HxRXa4GZkGaQ
```

### JWT 三个组成部分解释：

| 部分        | 内容        | 说明                                   |
| --------- | --------- | ------------------------------------ |
| Header    | 元数据，如算法类型 | 通常为 `{"alg": "HS256", "typ": "JWT"}` |
| Payload   | 有效负载      | 包括自定义字段如 `user_id`、`role`、`exp`      |
| Signature | 签名        | 用密钥对前两部分加密生成，防篡改                     |

---

## 🔐 二、JWT 鉴权原理流程图

```
[客户端提交登录信息]
          ↓
[服务端验证成功，生成JWT]
          ↓
[客户端拿到JWT并保存在Header中]
          ↓
[每次请求携带Authorization: Bearer <token>]
          ↓
[服务端中间件校验Token有效性]
          ↓
[Token有效：提取user_id放入上下文，继续处理]
       无效：拒绝请求
```

---

## 🚀 三、项目实现：Gin + JWT 鉴权系统

我们将实现一个包括：

* 登录接口（生成Token）
* JWT中间件（验证Token）
* 受保护接口（需要Token访问）
* 统一响应结构
* 用户模型（模拟数据库）

---

## 📁 目录结构

```
go-jwt-auth/
├── main.go
├── controller/
│   └── user.go
├── middleware/
│   └── jwt.go
├── model/
│   └── user.go
├── response/
│   └── response.go
```

---

## 🔧 四、代码详解与讲解

---

### （1）中间件部分（middleware/jwt.go）

```go
package middleware

import (
    "strings"
    "time"
    "alan-snippet/response"
    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v4"
)

var jwtSecret = []byte("a_very_secret_key") // 推荐从配置文件或环境变量读取

// MyClaims 自定义 JWT 的 Payload 部分
type MyClaims struct {
    UserID uint `json:"user_id"`
    jwt.RegisteredClaims
}

// JWTAuth 中间件：拦截请求并验证 JWT 的合法性
func JWTAuth() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            response.Unauthorized(c, "缺少Authorization头")
            c.Abort()
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            response.Unauthorized(c, "Token格式应为Bearer {token}")
            c.Abort()
            return
        }

        tokenStr := parts[1]

        // 解析Token，并将结果写入自定义claims结构体
        token, err := jwt.ParseWithClaims(tokenStr, &MyClaims{}, func(token *jwt.Token) (interface{}, error) {
            return jwtSecret, nil
        })

        if err != nil || !token.Valid {
            response.Unauthorized(c, "Token无效或过期")
            c.Abort()
            return
        }

        // 解析成功，提取UserID注入上下文供后续handler使用
        claims, ok := token.Claims.(*MyClaims)
        if !ok {
            response.Unauthorized(c, "Token解析失败")
            c.Abort()
            return
        }

        c.Set("userID", claims.UserID)
        c.Next()
    }
}

// GenerateToken 登录成功后调用，用于生成JWT
func GenerateToken(userID uint) (string, error) {
    claims := MyClaims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)), // 7天有效期
            Issuer:    "go-jwt-auth", // 签发人
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}
```

---

### （2）模拟用户数据库（model/user.go）

```go
package model

type User struct {
    ID       uint
    Username string
    Password string
}

var users = map[string]User{
    "alan": {ID: 1, Username: "alan", Password: "123456"},
}

func GetUserByUsername(username string) (User, bool) {
    user, ok := users[username]
    return user, ok
}
```

---

### （3）统一响应格式（response/response.go）

```go
package response

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

const (
    CodeSuccess      = 0
    CodeUnauthorized = 401
    CodeBadRequest   = 400
    CodeServerError  = 500
)

func Success(c *gin.Context, data interface{}, msg string) {
    c.JSON(http.StatusOK, gin.H{
        "code": CodeSuccess,
        "msg":  msg,
        "data": data,
    })
}

func Fail(c *gin.Context, code int, msg string) {
    c.JSON(http.StatusOK, gin.H{
        "code": code,
        "msg":  msg,
    })
}

func Unauthorized(c *gin.Context, msg string) {
    Fail(c, CodeUnauthorized, msg)
}
```

---

### （4）用户控制器（controller/user.go）

```go
package controller

import (
    "alan-snippet/middleware"
    "alan-snippet/model"
    "alan-snippet/response"
    "github.com/gin-gonic/gin"
)

// Login 用户登录接口
func Login(c *gin.Context) {
    var req struct {
        Username string `json:"username"`
        Password string `json:"password"`
    }

    if err := c.ShouldBindJSON(&req); err != nil {
        response.Fail(c, response.CodeBadRequest, "请求参数错误")
        return
    }

    user, exists := model.GetUserByUsername(req.Username)
    if !exists || user.Password != req.Password {
        response.Unauthorized(c, "用户名或密码错误")
        return
    }

    token, err := middleware.GenerateToken(user.ID)
    if err != nil {
        response.Fail(c, response.CodeServerError, "Token生成失败")
        return
    }

    response.Success(c, gin.H{"token": token}, "登录成功")
}

// GetProfile 获取当前用户信息（需JWT认证）
func GetProfile(c *gin.Context) {
    userID := c.GetUint("userID")
    response.Success(c, gin.H{
        "user_id": userID,
        "nickname": "Alan",
    }, "用户信息获取成功")
}
```

---

### （5）项目入口（main.go）

```go
package main

import (
    "alan-snippet/controller"
    "alan-snippet/middleware"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()

    // 登录路由，无需鉴权
    r.POST("/login", controller.Login)

    // 受保护路由，使用JWTAuth中间件
    api := r.Group("/api", middleware.JWTAuth())
    {
        api.GET("/profile", controller.GetProfile)
    }

    r.Run(":8080")
}
```

---

## 🔬 五、Postman 测试流程

### 第一步：登录获取Token

```http
POST /login
Content-Type: application/json

{
  "username": "alan",
  "password": "123456"
}
```

响应：

```json
{
  "code": 0,
  "msg": "登录成功",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR..."
  }
}
```

---

### 第二步：携带Token请求受保护接口

```http
GET /api/profile
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
```

响应：

```json
{
  "code": 0,
  "msg": "用户信息获取成功",
  "data": {
    "user_id": 1,
    "nickname": "Alan"
  }
}
```

---

## ✅ 六、最佳实践汇总

| 方面         | 建议                       |
| ---------- | ------------------------ |
| 密钥管理       | 使用 `.env` / 配置文件管理，不要硬编码 |
| Token 有效期  | 7天为推荐值，支持RefreshToken更安全 |
| Token 签名算法 | 推荐 HS256 或 RSA           |
| 用户存储       | 使用数据库如 MySQL/PostgreSQL  |
| 密码存储       | 使用 `bcrypt` 加密存储密码       |
| 权限控制       | 可结合 `Casbin` 或 RBAC 模型   |
| 日志记录       | 每次认证/失败建议记录日志，便于追踪       |

---