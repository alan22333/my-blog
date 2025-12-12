---
title: GeeWeb学习笔记
description: '记录一下在geektutu系列博客中学到的一些知识以及这个小项目的设计'
tags: ['go']
toc: false
date: 2025-12-06 16:43:49
categories:
    - go
    - project
---

> 未完待续。。。

## day3 前缀树路由

### Trie树

在前一节我们使用map结构存储路由表（key是具体请求的路由，value是对应的HandlerFunc），使用map存储键值对，索引非常高效。但是有一个弊端，键值对的存储的方式，只能用来索引静态路由。

那如果我们想支持类似于/hello/:name这样的动态路由怎么办呢？所谓动态路由，即一条路由规则可以匹配某一类型而非某一条固定的路由。

我们想要实现的是：
    - 请求 `/hello/alan` or `/hello/vhsj`       --路由到--> `/hello/:name`索引到的处理函数下 （获取路径参数）
    - 请求 `/asset/anything/like/CSSfiles/path` --路由到--> `/asset/*filename`索引到的处理函数下 （静态资源请求）

![前缀树](https://geektutu.com/post/gee-day3/trie_eg.jpg)

所以我们引入前缀树来实现动态路由，添加路由表的时候我们会使用带有通配符（:/*）的路由注册到树中并所引导对应的处理函数，然后到查询的时候我们会解析request的具体路径，从根节点开始递归的查询，一层层往下匹配（通配符会有特殊的处理，例如“:”需要解析路径参数保存，而“\*”则需要结束匹配保存路径的后半部分）直到叶子结点（在中间停止也是失败的，例如请求路径“/user/alan” 不可以 匹配路由“/user/:name/hahaha”）

规则不用多说，直接看怎么实现：

```Go
package gee

import "strings"

type node struct {
	pattern  string  // 目标路由，例如 /hello/:name
	part     string  // 路由一部分，或者说树的一层内容，例如 /:name
	children []*node // 子节点
	isWild   bool    // 是否模糊匹配，part 含有 : 或 * 时为true，此时需要按照对应规则进行处理
}

// 第一个匹配成功的节点，用于插入
func (n *node) matchChild(part string) *node {
	for _, c := range n.children {
		if c.part == part || c.isWild {
			return c
		}
	}
	return nil
}

// 所有匹配成功的节点，用于查找
func (n *node) matchChildren(part string) []*node {
	nodes := []*node{}
	for _, c := range n.children {
		if c.part == part || c.isWild {
			nodes = append(nodes, c)
		}
	}
	return nodes
}

// 新建路由：前缀树插入
func (n *node) insert(pattern string, parts []string, height int) {

	// 插入结束，每一个part都加入树中，递归结束
	if len(parts) == height {
		n.pattern = pattern
		return
	}

	// 当前层要处理part
	part := parts[height]

	// 找到第一个匹配的子节点
	child := n.matchChild(part)

	// 如果这个位置是空，则创建一个子节点，把这个part放在这个位置
	if child == nil {
		child = &node{
			part:   part,
			isWild: part[0] == ':' || part[0] == '*',
		}
		n.children = append(n.children, child)
	}

	// 向下递归匹配
	child.insert(pattern, parts, height+1)
}

// 路由检索：前缀树插入
func (n *node) search(parts []string, height int) *node {
	
	// 递归终止条件：到底了或者遇到通配符*（表示后面都可以匹配）
	if len(parts) == height || strings.HasPrefix(n.part, "*") {
		// 说明不是当前节点不是完整路由，而是中间某一块
		if n.pattern == "" {
			return nil
		}
		return n
	}

	// 当前要搜索的块
	part := parts[height]

	// 匹配的所有子节点
	children := n.matchChildren(part)

	// 分别去递归搜索
	for _, child := range children {
		result := child.search(parts, height+1)
		if result != nil {
			return result
		}
	}

	return nil
}
```

### 实现动态路由

现在使用新的数据结构来存储路由表实现我们的目标：通配符路由注册 + 动态请求路径匹配

1. 定义路由：roots中key为Method，value是树根，意为一种req method一棵树；handlers中key为注册的路由（含有通配符的），value为对应的处理函数

```Go
type router struct {
	roots    map[string]*node
	handlers map[string]HandlerFunc
}

func NewRouter() *router {
	return &router{
		roots:    make(map[string]*node),
		handlers: make(map[string]HandlerFunc),
	}
}
```

2. 解析路径：把实际的请求路径切割成一个个part

```Go
// Only one * is allowed
func parsePattern(pattern string) []string {
	vs := strings.Split(pattern, "/")

	parts := make([]string, 0)
	for _, item := range vs {
		if item != "" {
			parts = append(parts, item)
			if item[0] == '*' {
				break
			}
		}
	}
	return parts
}
```

3. 添加和查询路由：对应TrieTree的插入和搜索

```Go
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	parts := parsePattern(pattern)

	key := method + "-" + pattern
	_, ok := r.roots[method]
	if !ok {
		r.roots[method] = &node{}
	}
	r.roots[method].insert(pattern, parts, 0)
	r.handlers[key] = handler
}

// 返回：对应的路由、路径参数列表，例如 /user/:id/hello、{"id":"114514"}
func (r *router) getRoute(method string, path string) (*node, map[string]string) {
	searchParts := parsePattern(path)
	params := make(map[string]string)

	root, ok := r.roots[method]
	if !ok {
		return nil, nil
	}

	n := root.search(searchParts, 0)

	if n != nil {
		parts := parsePattern(n.pattern)
		for index, part := range parts {
			if part[0] == ':' {
				params[part[1:]] = searchParts[index]
			}
			if part[0] == '*' && len(part) > 1 {
				params[part[1:]] = strings.Join(searchParts[index:], "/")
				break
			}
		}
		return n, params
	}

	return nil, nil
}

```

## day4分组控制

### 什么是分组控制

分组控制(Group Control)是 Web 框架应提供的基础功能之一。简单理解就是，很多的路由有共同的需求可以集中处理设置（中间件），例如：

- 以/post开头的路由匿名可访问。
- 以/admin开头的路由需要鉴权。
- 以/api开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

所以路由分组主要是可以方便功能扩展、提升代码注册逻辑性、减少大量重复代码编写。

### 实现逻辑

首先一个分组需要什么呢？一个公共的前缀prefix、一组公共处理逻辑（中间件）middlewares、领导/管理者/上层抽象，即我们的引擎engine，所以RouterGroup可以定义为：

```Go
type RouterGroup struct {
	prefix      string
	middlewares []HandlerFunc 
	engine      *Engine
}
```

然后我们在引擎结构中也嵌入（embeding）Group，那么enigin就可以直接使用Group的所有方法，相当于engine是group的子类。
作为子类，是一定比父类功能更多的，作为路由分组，他需要处理路由相关的功能（注册、查询等等），而引擎则需要更多功能（服务器启动关闭等等）。那么为什么不把分组作为属性放到引擎中呢？

```go
// 引擎就是一个根路由
type Engine struct {
    *RouterGroup
    router *router
    groups []*RouterGroup
}

func New() *Engine {
	engine := &Engine{router: NewRouter()}
	engine.groups = []*RouterGroup{}
	engine.RouterGroup = &RouterGroup{engine: engine}
	return engine
}

```

#### 1.嵌入（Embedding）的作用

`Engine`通过嵌入`*RouterGroup`，可以直接使用`RouterGroup`的所有方法（`GET`、`POST`、`Group`等）。但这**不是继承**，而是**方法提升（method promotion）**。

```go
type Engine struct {
    *RouterGroup  // 嵌入
    router *router
    groups []*RouterGroup
}

// 因为嵌入了 *RouterGroup，所以可以这样调用：
engine.GET("/", handler)  // 实际是 engine.RouterGroup.GET()
engine.Group("/v1")       // 实际是 engine.RouterGroup.Group()
```

#### 2.为什么用嵌入而不是普通属性？

1. **API一致性和便利性**
如果用普通属性：
```go
type Engine struct {
    group RouterGroup  // 普通属性
    router *router
}
// 使用时必须：
engine.group.GET("/", handler)
engine.group.Group("/v1")
```

用嵌入后，`Engine`和`RouterGroup`有相同的API接口，使用者不需要关心是操作根路由组还是子路由组。

2. **根路由组的特殊性**
`Engine`本质上**就是一个根路由组**，它具有：
- 能注册路由（`GET`、`POST`）
- 能创建子路由组（`Group`）
- 另外还能启动服务器（`Run`、`ServeHTTP`）

用嵌入表示这种"is-a"关系（Engine **是一个** RouterGroup）。


然后把所有根路由有关的方法定义到Group上面即可：

```Go
// Group is defined to create a new RouterGroup
// remember all groups share the same Engine instance
func (rg *RouterGroup) Group(prefix string) *RouterGroup {
	engine := rg.engine // share engine
	newRg := &RouterGroup{
		prefix: rg.prefix + prefix,
		engine: engine,
	}
	engine.groups = append(engine.groups, newRg)
	return newRg
}

// 路由相关的方法全部定义到 rg 上
// 由于engine中嵌入了 rg ，所以engine也可以直接调用
func (rg *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	pattern := rg.prefix + comp
	log.Printf("Route %4s - %s", method, pattern)
	rg.engine.router.addRoute(method, pattern, handler)
}

// 定义一些基本的方法的添加
func (rg *RouterGroup) GET(pattern string, handler HandlerFunc) {
	rg.addRoute("GET", pattern, handler)
}

func (rg *RouterGroup) POST(pattern string, handler HandlerFunc) {
	rg.addRoute("POST", pattern, handler)
}
```

eigine作为根可以使用上面的函数，另外也有其他：
```Go
func (e *Engine) Run(addr string) error {
	// e 必须得实现接口方法
	return http.ListenAndServe(addr, e)
}

func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// 一次请求，创建一个context
	context := newContext(w, r)
	e.router.handle(context)
}
```

### 使用示例

```Go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.Html(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=alan
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})

	r.GET("/hello/:name", func(c *gee.Context) {
		// expect /hello/geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
	})

	r.GET("/assets/*filepath", func(c *gee.Context) {
		c.JSON(http.StatusOK, gee.H{"filepath": c.Param("filepath")})
	})

	r.Run(":9999")
}
```

## day5中间件

