---
title: Go语言之Redis入门
description: 'Redis 是一个高性能的K-V内存数据库，本文将详细讲解如何在Go语言中使用Redis和一些常见的最佳实践'
tags: ['go','redis']
toc: false
date: 2025-07-26 18:13:45
categories:
    - go
    - database
---


## 一、Redis 简介与 Go 适用场景

Redis 是一个高性能的**Key-Value**内存数据库，常用于：

* 数据缓存（热点数据、数据库查询结果）
* 计数器（网站访问量、点赞数）
* 排行榜系统（使用 `ZSet`）
* 消息队列（Pub/Sub 或 Redis Stream）
* 分布式锁
* 限流器（如滑动窗口、令牌桶）

在 Go 微服务架构中，Redis 通常扮演**缓存层 + 通讯中间件**的角色，性能至关重要。

---

## 二、Go 中操作 Redis 基础：使用 go-redis/v9

```bash
go get github.com/redis/go-redis/v9
```

### 初始化连接客户端（全局复用）

```go
package redisx

import (
	"context"
	"github.com/redis/go-redis/v9"
	"time"
)

var (
	ctx = context.Background()
	Rdb *redis.Client
)

func InitRedis() {
	Rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",  // no password
		DB:       0,   // default DB
		PoolSize: 10,  // 连接池大小
	})
	if err := Rdb.Ping(ctx).Err(); err != nil {
		panic("redis 连接失败: " + err.Error())
	}
}
```

---

## 三、常见 Redis 数据结构 + Go 操作实战

### 1. String（缓存、计数）

```go
rdb.Set(ctx, "views", 100, time.Hour)
val, _ := rdb.Get(ctx, "views").Int()
rdb.Incr(ctx, "views")
```

### 2. Hash（结构化数据）

```go
rdb.HSet(ctx, "user:1", "name", "Tom", "age", 30)
rdb.HGetAll(ctx, "user:1")
```

### 3. List（队列）

```go
rdb.LPush(ctx, "task:queue", "job1", "job2")
rdb.RPop(ctx, "task:queue") // 消费任务
```

### 4. Set（去重）

```go
rdb.SAdd(ctx, "ip:banlist", "192.168.1.1")
rdb.SIsMember(ctx, "ip:banlist", "192.168.1.1")
```

### 5. Sorted Set（排行榜）

```go
rdb.ZAdd(ctx, "rank", redis.Z{Score: 100, Member: "u1"})
rdb.ZRevRangeWithScores(ctx, "rank", 0, 2)
```

---

## 四、典型应用场景实战（代码完整示例）

### 1. 缓存数据库查询结果

```go
func GetUser(uid string) (string, error) {
	key := "user:" + uid
	// 查 Redis
	val, err := rdb.Get(ctx, key).Result()
	if err == redis.Nil {
		// 未命中，查询数据库
		val = "UserFromDB" // 模拟
		rdb.Set(ctx, key, val, time.Minute*10)
	} else if err != nil {
		return "", err
	}
	return val, nil
}
```

### 2. 使用 ZSet 实现游戏排行榜

```go
func AddScore(user string, score float64) {
	rdb.ZIncrBy(ctx, "game:rank", score, user)
}

func TopRank(n int64) {
	top, _ := rdb.ZRevRangeWithScores(ctx, "game:rank", 0, n-1).Result()
	for i, z := range top {
		fmt.Printf("%d: %s -> %.0f\n", i+1, z.Member, z.Score)
	}
}
```

### 3. 分布式锁实现（推荐 Lua 方式）

```go
func AcquireLock(key, value string, ttl time.Duration) bool {
	ok, _ := rdb.SetNX(ctx, key, value, ttl).Result()
	return ok
}

func ReleaseLock(key, value string) {
	lua := `if redis.call("get", KEYS[1]) == ARGV[1] then
	         return redis.call("del", KEYS[1]) else return 0 end`
	rdb.Eval(ctx, lua, []string{key}, value)
}
```

### 4. 消息发布/订阅

```go
// 发布
rdb.Publish(ctx, "channel:news", "Hello Subscriber!")

// 订阅
sub := rdb.Subscribe(ctx, "channel:news")
ch := sub.Channel()
for msg := range ch {
	fmt.Println("收到消息：", msg.Payload)
}
```

---

## 五、封装 Redis 为模块化中间件（强烈推荐）

### 封装 Redis 接口层（泛型支持）

```go
// redisx/client.go
package redisx

type Client struct {
	rdb *redis.Client
}

func NewClient(opt *redis.Options) *Client {
	rdb := redis.NewClient(opt)
	return &Client{rdb: rdb}
}

func (c *Client) SetString(key string, val string, ttl time.Duration) error {
	return c.rdb.Set(ctx, key, val, ttl).Err()
}

func (c *Client) GetString(key string) (string, error) {
	return c.rdb.Get(ctx, key).Result()
}

func (c *Client) Incr(key string) (int64, error) {
	return c.rdb.Incr(ctx, key).Result()
}
```

### 使用方式：

```go
redisClient := redisx.NewClient(&redis.Options{Addr: "localhost:6379"})
redisClient.SetString("user:1:name", "Tom", time.Hour)
```

---

## 六、性能优化与常见问题

| 问题       | 建议                |
| -------- | ----------------- |
| Redis 雪崩 | 设置不同 TTL、加入本地缓存降级 |
| 缓存击穿     | 使用互斥锁或缓存空值        |
| 缓存穿透     | 对非法请求返回默认值并缓存     |
| 热 Key 过载 | 考虑引入本地缓存、Sharding |

### 实用建议：

* 使用连接池（默认开启）
* 多操作使用 Pipeline 批处理
* 配置监控：`INFO memory`、`slowlog get`

---
