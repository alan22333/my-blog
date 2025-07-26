---
title: Go语言之Gorm入门
description: '快速学习gorm操作mysql'
tags: ['go']
toc: false
date: 2025-07-17 18:02:00
categories:
    - go
    - datebase
---


## 使用 GORM 操作 MySQL 数据库：从入门到实践

在 Go 语言中进行数据库操作，除了 `database/sql` 标准库和像 `sqlx` 这样的扩展库外，还有许多优秀的 ORM (Object-Relational Mapping) 框架。其中，**GORM** 以其简洁的 API、强大的功能和活跃的社区支持，成为了 Go 开发者进行数据库操作的首选之一。

本文将从一个 `sqlx` 的示例出发，逐步讲解如何使用 GORM 实现数据库的增删改查以及事务操作，并分享一些 GORM 的高级用法和开发技巧。

### 为什么选择 GORM？

在深入 GORM 之前，我们先来探讨一下为什么选择 ORM，以及 GORM 相较于 `sqlx` 有何优势：

  * **减少 SQL 编写**: ORM 的核心优势在于将数据库操作抽象为 Go 结构体和方法调用，大大减少了手动编写 SQL 语句的工作量，降低了出错的概率。
  * **提高开发效率**: 结构化的 API 和内置的各种便利函数，能够让开发者更专注于业务逻辑，而非底层的数据库细节。
  * **类型安全**: GORM 通过 Go 结构体来映射数据库表，利用 Go 的类型系统，在编译期就能发现一些潜在的类型错误。
  * **跨数据库兼容性**: GORM 支持多种数据库（MySQL, PostgreSQL, SQLite, SQL Server 等），在切换数据库时，大部分代码无需修改。

相较于 `sqlx`，GORM 的优势主要体现在：

  * **更彻底的抽象**: `sqlx` 依然需要手动编写 SQL 语句，而 GORM 更多地通过方法链来构建查询。
  * **更丰富的功能**: GORM 内置了预加载、关联查询、自动迁移等高级功能，这些在 `sqlx` 中需要手动实现或依赖其他库。
  * **更友好的链式 API**: GORM 的方法链式调用让代码更具可读性和流畅性。

### GORM 基础入门：连接、模型与CRUD

#### 1\. 安装 GORM

首先，你需要安装 GORM 及其 MySQL 驱动：

```bash
go get -u gorm.io/gorm
go get -u gorm.io/driver/mysql
```

#### 2\. 连接数据库

在 GORM 中，连接数据库非常简单。我们通常在 `init` 函数中完成数据库的初始化，并将其存储在一个全局变量中。

```go
package main

import (
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var DB *gorm.DB

func init() {
	dsn := "username:password@(localhost:3306)/study?charset=utf8mb4&parseTime=True&loc=Local"
	var err error
	DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalf("failed to connect database: %v", err)
	}

	// 自动迁移
	// GORM 会根据结构体定义自动创建或更新表结构
	err = DB.AutoMigrate(&Person{})
	if err != nil {
		log.Fatalf("failed to auto migrate: %v", err)
	}
	log.Println("Database connection and migration successful!")
}

// ... 结构体定义和CRUD操作
```

**代码解析**:

  * `dsn`: DSN (Data Source Name) 是连接数据库的字符串，格式与 `sqlx` 类似。
  * `mysql.Open(dsn)`: 使用 GORM 提供的 MySQL 驱动打开数据库连接。
  * `gorm.Open(...)`: 初始化 GORM 数据库连接，并可以传入 `&gorm.Config{}` 进行一些配置，例如日志模式、命名策略等。
  * `DB.AutoMigrate(&Person{})`: 这是 GORM 的一个非常方便的功能。它会根据 `Person` 结构体定义，自动创建 `user` 表（如果不存在），或者根据结构体的字段变化更新表结构。这在开发阶段非常有用。

#### 3\. 定义数据结构 (模型)

GORM 使用 Go 结构体来定义数据库表。默认情况下，GORM 会将结构体名称的复数形式作为表名（例如，`Person` 结构体对应 `people` 表），但我们可以通过 `TableName()` 方法或者在 `gorm:"table:user"` 标签中指定表名。字段名默认转换为蛇形命名（例如 `UserId` 对应 `user_id`）。

```go
// Person 结构体定义
type Person struct {
	// GORM 约定 `ID` 或 `Id` 字段为主键，会自动识别为自增主键。
	// 这里我们使用 `UserId` 来映射 `id` 列，并且指定其为主键。
	// GORM 约定: 如果是 `ID` 字段，默认就是主键且自增。
	// 如果不是 `ID` 字段，需要显式加上 `gorm:"primaryKey"`
	UserId   string `gorm:"column:id;primaryKey"` // 对应数据库的 id 列，并且是主键
	Username string `gorm:"column:name"`          // 对应数据库的 name 列
	Age      int    `gorm:"column:age"`           // 对应数据库的 age 列
	Address  string `gorm:"column:address"`       // 对应数据库的 address 列
}

// TableName 方法用于指定 GORM 对应的数据库表名
func (Person) TableName() string {
	return "user" // 将 Person 结构体映射到 user 表
}
```

**代码解析**:

  * `gorm:"column:id;primaryKey"`: 通过 `gorm` 标签来指定字段与数据库列的映射关系以及其他属性。
      * `column:id`: 指定结构体字段 `UserId` 映射到数据库表的 `id` 列。
      * `primaryKey`: 将 `UserId` 标记为主键。
  * `TableName()` 方法: 这是一个 GORM 的约定，通过实现这个方法，我们可以自定义结构体对应的表名，这里我们将其设置为 `user`，与你的 `sqlx` 示例保持一致。

#### 4\. CRUD 操作 (增删改查)

##### 查询 (Retrieve)

**查询单条记录**:

```go
// query 查询单条记录
func query() {
	var p Person
	// First 根据主键查找第一条记录
	// DB.First(&p, "12132")
	// Where 子句用于构建查询条件
	result := DB.Where("id = ?", "12132").First(&p)
	if result.Error != nil {
		if result.Error == gorm.ErrRecordNotFound {
			log.Printf("query fail: record not found for id %s\n", "12132")
		} else {
			log.Fatalf("query fail: %v\n", result.Error)
		}
		return
	}
	log.Printf("query succ: %v\n", p)
}
```

**代码解析**:

  * `DB.Where("id = ?", "12132")`: 使用 `Where` 方法构建查询条件。GORM 会自动将 `Person` 结构体中的 `UserId` 字段映射到 `id` 列。
  * `First(&p)`: 查找满足条件的第一条记录，并将其扫描到 `p` 结构体中。如果没有找到记录，会返回 `gorm.ErrRecordNotFound` 错误。

**查询多条记录 (列表)**:

```go
// list 查询多条记录
func list() {
	var ps []Person
	// Find 查询所有记录
	result := DB.Find(&ps)
	if result.Error != nil {
		log.Fatalf("list fail: %v\n", result.Error)
		return
	}
	for _, p := range ps {
		log.Printf("list succ: %v\n", p)
	}
}
```

**代码解析**:

  * `DB.Find(&ps)`: 查找所有 `Person` 记录，并将其扫描到 `ps` 切片中。

##### 新增 (Create)

```go
// insert 插入单条记录
func insert() {
	newPerson := Person{UserId: "1145", Username: "alan", Age: 24, Address: "China"}
	result := DB.Create(&newPerson)
	if result.Error != nil {
		log.Fatalf("insert fail: %v\n", result.Error)
		return
	}
	// 如果主键是自增的，GORM 会自动填充到结构体中
	log.Printf("insert succ, ID: %s, RowsAffected: %d\n", newPerson.UserId, result.RowsAffected)
}
```

**代码解析**:

  * `DB.Create(&newPerson)`: 将 `newPerson` 结构体插入到数据库中。GORM 会自动根据结构体字段生成 `INSERT` 语句。
  * `result.RowsAffected`: 返回受影响的行数。

##### 更新 (Update)

```go
// update 更新记录
func update() {
	// 更新单个字段
	// result := DB.Model(&Person{}).Where("id = ?", "1145").Update("name", "alan223")

	// 更新多个字段
	result := DB.Model(&Person{}).Where("id = ?", "1145").Updates(map[string]interface{}{"name": "alan223", "age": 25})

	// 或者直接传入结构体，GORM 会更新非零值字段
	// p := Person{UserId: "1145", Username: "alan223", Age: 25}
	// result := DB.Model(&p).Where("id = ?", "1145").Updates(p) // 注意这里Updates(p)会更新所有非零值字段

	if result.Error != nil {
		log.Fatalf("update fail: %v\n", result.Error)
		return
	}
	if result.RowsAffected == 0 {
		log.Printf("update warning: no records updated for id %s\n", "1145")
	} else {
		log.Println("update succ")
	}
}
```

**代码解析**:

  * `DB.Model(&Person{})`: 指定要操作的模型。
  * `Where("id = ?", "1145")`: 指定更新条件。
  * `Update("name", "alan223")`: 更新单个字段。
  * `Updates(map[string]interface{}{"name": "alan223", "age": 25})`: 更新多个字段。传入 `map` 可以精确控制更新的字段。
  * `Updates(p)`: 传入结构体进行更新，GORM 默认会更新所有**非零值**字段。如果你想更新所有字段，包括零值，可以使用 `Select("*").Updates(p)`。

##### 删除 (Delete)

```go
// delete 删除记录
func delete() {
	// 硬删除 (物理删除)
	result := DB.Where("id = ?", "1145").Delete(&Person{})
	// 或者
	// result := DB.Delete(&Person{}, "1145") // 根据主键删除

	if result.Error != nil {
		log.Fatalf("delete fail: %v\n", result.Error)
		return
	}
	if result.RowsAffected == 0 {
		log.Printf("delete warning: no records deleted for id %s\n", "1145")
	} else {
		log.Println("delete succ")
	}
}
```

**代码解析**:

  * `DB.Where("id = ?", "1145").Delete(&Person{})`: 根据条件删除记录。
  * `DB.Delete(&Person{}, "1145")`: 根据主键删除记录。
  * **软删除**: GORM 支持软删除。如果你在模型中包含 `gorm.DeletedAt` 字段，GORM 在执行 `Delete` 操作时，不会真正删除记录，而是将 `DeletedAt` 字段设置为当前时间。查询时会自动过滤掉被软删除的记录。这在很多业务场景下非常有用。

#### 5\. 事务 (Transactions)

GORM 提供了两种方式进行事务操作：**块事务**和**手动事务**。块事务更推荐，因为它会自动处理提交和回滚，并且在函数退出时自动回滚未提交的事务。

##### 块事务 (推荐)

```go
// tx GORM 的事务操作 (块事务)
func tx() {
	err := DB.Transaction(func(tx *gorm.DB) error {
		// 事务中的所有操作都使用 tx 对象
		// 例如：
		// 插入一条记录
		newPerson := Person{UserId: "9999", Username: "tx_test", Age: 30, Address: "Transaction Land"}
		if err := tx.Create(&newPerson).Error; err != nil {
			log.Printf("transaction insert failed: %v\n", err)
			return err // 返回错误，事务将回滚
		}

		// 更新一条记录
		if err := tx.Model(&Person{}).Where("id = ?", "12132").Update("name", "UpdatedByTx").Error; err != nil {
			log.Printf("transaction update failed: %v\n", err)
			return err // 返回错误，事务将回滚
		}

		// 如果所有操作都成功，GORM 会自动提交事务
		return nil
	})

	if err != nil {
		log.Printf("transaction failed: %v\n", err)
	} else {
		log.Println("transaction succ")
	}
}
```

**代码解析**:

  * `DB.Transaction(func(tx *gorm.DB) error {...})`: GORM 的块事务方法。你传入一个函数，所有在这个函数内部使用 `tx` 对象进行的操作都会在同一个事务中。
  * 如果函数返回 `nil`，事务会被提交。如果函数返回任何错误，事务会自动回滚。

##### 手动事务 (不推荐，除非有特殊需求)

```go
// manualTx 手动事务操作
func manualTx() {
	tx := DB.Begin() // 开始事务
	if tx.Error != nil {
		log.Fatalf("failed to begin transaction: %v", tx.Error)
	}

	defer func() {
		if r := recover(); r != nil {
			tx.Rollback() // 发生 panic 时回滚
			panic(r)
		}
	}()

	// 事务中的操作
	newPerson := Person{UserId: "8888", Username: "manual_tx_test", Age: 28, Address: "Manual Land"}
	if err := tx.Create(&newPerson).Error; err != nil {
		tx.Rollback() // 出现错误时回滚
		log.Fatalf("manual transaction insert failed: %v", err)
		return
	}

	if err := tx.Model(&Person{}).Where("id = ?", "12132").Update("name", "ManualUpdated").Error; err != nil {
		tx.Rollback() // 出现错误时回滚
		log.Fatalf("manual transaction update failed: %v", err)
		return
	}

	if err := tx.Commit().Error; err != nil { // 提交事务
		log.Fatalf("failed to commit transaction: %v", err)
	}
	log.Println("manual transaction succ")
}
```

**代码解析**:

  * `DB.Begin()`: 开始一个事务。
  * `tx.Commit()`: 提交事务。
  * `tx.Rollback()`: 回滚事务。
  * `defer` 结合 `recover()`: 确保即使在事务中发生 `panic`，事务也能被正确回滚。手动事务需要开发者自己处理提交和回滚逻辑，容易出错，所以通常推荐使用块事务。

### 示例代码 (GORM 版本)

```go
package main

import (
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var DB *gorm.DB

func init() {
	// 连接数据库
	// DSN (Data Source Name) 格式：username:password@(host:port)/database?charset=utf8mb4&parseTime=True&loc=Local
	// parseTime=True 是为了让 GORM 能正确解析 MySQL 中的 DATETIME 和 TIMESTAMP 类型
	// loc=Local 是为了让时间以本地时区解析
	dsn := "root:vader20011014@(localhost:3306)/study?charset=utf8mb4&parseTime=True&loc=Local"
	var err error
	DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalf("failed to connect database: %v", err)
	}

	// 自动迁移
	// GORM 会根据结构体定义自动创建或更新表结构
	err = DB.AutoMigrate(&Person{})
	if err != nil {
		log.Fatalf("failed to auto migrate: %v", err)
	}
	log.Println("Database connection and migration successful!")
}

// 定义数据结构
type Person struct {
	// GORM 约定 `ID` 或 `Id` 字段为主键，会自动识别为自增主键。
	// 这里我们使用 `UserId` 来映射 `id` 列，并且指定其为主键。
	UserId   string `gorm:"column:id;primaryKey"` // 对应数据库的 id 列，并且是主键
	Username string `gorm:"column:name"`          // 对应数据库的 name 列
	Age      int    `gorm:"column:age"`           // 对应数据库的 age 列
	Address  string `gorm:"column:address"`       // 对应数据库的 address 列
}

// TableName 方法用于指定 GORM 对应的数据库表名
func (Person) TableName() string {
	return "user" // 将 Person 结构体映射到 user 表
}

// -----查询-----
func query() {
	var p Person
	// First 根据主键查找第一条记录
	// DB.First(&p, "12132")

	// Where 子句用于构建查询条件
	result := DB.Where("id = ?", "12132").First(&p)
	if result.Error != nil {
		if result.Error == gorm.ErrRecordNotFound {
			log.Printf("query fail: record not found for id %s\n", "12132")
		} else {
			log.Fatalf("query fail: %v\n", result.Error)
		}
		return
	}
	log.Printf("query succ: %v\n", p)
}

func list() {
	var ps []Person
	// Find 查询所有记录
	result := DB.Find(&ps)
	if result.Error != nil {
		log.Fatalf("list fail: %v\n", result.Error)
		return
	}
	for _, p := range ps {
		log.Printf("list succ: %v\n", p)
	}
}

// -----新增-----
func insert() {
	newPerson := Person{UserId: "1145", Username: "alan", Age: 24, Address: "China"}
	result := DB.Create(&newPerson)
	if result.Error != nil {
		log.Fatalf("insert fail: %v\n", result.Error)
		return
	}
	// 如果主键是自增的，GORM 会自动填充到结构体中
	log.Printf("insert succ, ID: %s, RowsAffected: %d\n", newPerson.UserId, result.RowsAffected)
}

// -----更新-----
func update() {
	// 更新多个字段
	result := DB.Model(&Person{}).Where("id = ?", "1145").Updates(map[string]interface{}{"name": "alan223", "age": 25})

	if result.Error != nil {
		log.Fatalf("update fail: %v\n", result.Error)
		return
	}
	if result.RowsAffected == 0 {
		log.Printf("update warning: no records updated for id %s\n", "1145")
	} else {
		log.Println("update succ")
	}
}

// -----删除-----
func delete() {
	// 硬删除 (物理删除)
	result := DB.Where("id = ?", "1145").Delete(&Person{})

	if result.Error != nil {
		log.Fatalf("delete fail: %v\n", result.Error)
		return
	}
	if result.RowsAffected == 0 {
		log.Printf("delete warning: no records deleted for id %s\n", "1145")
	} else {
		log.Println("delete succ")
	}
}

// -----事务-----
// GORM 推荐使用块事务
func tx() {
	err := DB.Transaction(func(tx *gorm.DB) error {
		// 事务中的所有操作都使用 tx 对象

		// 插入一条记录
		newPerson := Person{UserId: "9999", Username: "tx_test", Age: 30, Address: "Transaction Land"}
		if err := tx.Create(&newPerson).Error; err != nil {
			log.Printf("transaction insert failed: %v\n", err)
			return err // 返回错误，事务将回滚
		}

		// 更新一条记录
		// 注意：更新不存在的ID并不会报错，只会影响0行。如果业务需要严格控制，需要单独检查RowsAffected
		if err := tx.Model(&Person{}).Where("id = ?", "12132").Update("name", "UpdatedByTx").Error; err != nil {
			log.Printf("transaction update failed: %v\n", err)
			return err // 返回错误，事务将回滚
		}

		// 如果所有操作都成功，GORM 会自动提交事务
		return nil
	})

	if err != nil {
		log.Printf("transaction failed: %v\n", err)
	} else {
		log.Println("transaction succ")
	}
}

func main() {
	query()
	insert() // 先插入一条记录，方便后续操作
	update()
	delete()
	list()
	tx() // 执行事务
}
```

-----

### GORM 开发技巧与高级用法

GORM 不仅仅提供了基本的 CRUD 和事务操作，还有许多强大的功能和技巧可以帮助你更高效地进行 Go 数据库开发。

#### 1\. 结构体标签 (Tags) 详解

GORM 的结构体标签是其强大功能的核心。除了上面用到的 `column` 和 `primaryKey`，还有许多其他有用的标签：

  * `gorm:"-"`: 忽略此字段，不映射到数据库。
  * `gorm:"autoIncrement"`: 标记字段为自增主键（通常不需要，GORM 默认 `ID` 为自增）。
  * `gorm:"unique"`: 字段值唯一。
  * `gorm:"default:value"`: 设置字段的默认值。
  * `gorm:"not null"`: 字段不允许为空。
  * `gorm:"size:255"`: 设置字符串类型字段的长度。
  * `gorm:"type:longtext"`: 指定字段的数据库类型。
  * `gorm:"index"`: 为字段创建普通索引。
  * `gorm:"uniqueIndex"`: 为字段创建唯一索引。

**示例**:

```go
type Product struct {
    ID          uint   `gorm:"primaryKey"`
    Code        string `gorm:"unique;size:100"`
    Price       uint   `gorm:"default:0"`
    Description string `gorm:"type:longtext"`
}
```

#### 2\. 日志与调试

GORM 提供了非常灵活的日志功能，方便你在开发和调试过程中查看生成的 SQL 语句和执行结果。

```go
import (
    "gorm.io/gorm/logger"
    "time"
)

func init() {
    dsn := "root:vader20011014@(localhost:3306)/study?charset=utf8mb4&parseTime=True&loc=Local"
    var err error
    DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // 设置日志模式为 Info
        // Other options
        // NamingStrategy: schema.NamingStrategy{
        //     SingularTable: true, // 使用单数表名，即 Person 对应 person 表
        // },
    })
    if err != nil {
        log.Fatalf("failed to connect database: %v", err)
    }
    DB.AutoMigrate(&Person{})
    log.Println("Database connection and migration successful!")
}
```

**日志模式**:

  * `logger.Silent`: 不打印任何日志。
  * `logger.Error`: 只打印错误日志。
  * `logger.Warn`: 打印错误和警告日志。
  * `logger.Info`: 打印所有日志，包括 SQL 语句。

你也可以自定义 Logger 来满足更复杂的日志需求。

#### 3\. 链式方法与作用域

GORM 最大的特点之一就是其优雅的链式方法调用。你可以将多个方法链接起来构建复杂的查询。

```go
// 查找年龄大于20，且地址在中国，并按年龄降序排列，取前10条记录
var adults []Person
DB.Where("age > ?", 20).Where("address = ?", "China").Order("age desc").Limit(10).Find(&adults)
```

GORM 还支持**作用域 (Scopes)**，它允许你将常用的查询条件封装成可复用的函数：

```go
func OlderThan(age int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("age > ?", age)
    }
}

func InAddress(address string) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("address = ?", address)
    }
}

// 使用作用域
var result []Person
DB.Scopes(OlderThan(25), InAddress("China")).Find(&result)
```

作用域可以大大提高代码的复用性和可读性。

#### 4\. 预加载 (Preload) 与关联查询

在处理关联数据时，GORM 的预加载功能非常强大，可以避免 N+1 查询问题。假设你有 `User` 和 `CreditCard` 两个模型，一个用户可以有多张信用卡：

```go
type CreditCard struct {
    gorm.Model
    Number string
    UserID uint
}

type User struct {
    gorm.Model
    Name       string
    CreditCards []CreditCard
}

// 查询用户及其所有信用卡
var user User
DB.Preload("CreditCards").First(&user, 1)
// 此时 user.CreditCards 字段会被自动填充
```

`Preload` 会执行额外的查询来加载关联数据，但比手动多次查询要高效得多。

#### 5\. 原生 SQL

尽管 GORM 旨在减少原生 SQL 的使用，但在某些复杂场景下，你可能仍然需要执行原生 SQL。GORM 提供了 `Raw` 和 `Exec` 方法来满足这些需求。

```go
// Raw 用于查询
var p Person
DB.Raw("SELECT id, name, age FROM user WHERE id = ?", "12132").Scan(&p)

// Exec 用于非查询操作 (INSERT, UPDATE, DELETE)
DB.Exec("UPDATE user SET name = ? WHERE id = ?", "RawUpdate", "1145")
```

#### 6\. Hooks (钩子)

GORM 提供了各种钩子方法，允许你在模型生命周期的不同阶段执行自定义逻辑，例如在创建前、更新后等。

```go
// BeforeCreate hook
func (p *Person) BeforeCreate(tx *gorm.DB) (err error) {
    if p.Address == "" {
        p.Address = "Unknown" // 为新记录设置默认地址
    }
    return nil
}

// AfterUpdate hook
func (p *Person) AfterUpdate(tx *gorm.DB) (err error) {
    log.Printf("Person with ID %s was updated.\n", p.UserId)
    return nil
}
```

这些钩子方法非常适合用于数据验证、日志记录、缓存更新等场景。

#### 7\. 错误处理

GORM 的操作结果会返回一个 `*gorm.DB` 对象，你可以通过检查其 `Error` 字段来判断操作是否成功。

  * `result.Error`: 如果操作失败，这里会包含具体的错误信息。
  * `gorm.ErrRecordNotFound`: 特定于未找到记录的错误。
  * `result.RowsAffected`: 对于 DML 操作 (Create, Update, Delete)，表示受影响的行数。

始终检查 `result.Error` 是一个好习惯。