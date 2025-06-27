---
title: Gee微框架学习记录(第二天)
date: 2025-03-30T16:31:18+08:00
lastmod: 2025-03-30T16:31:18+08:00
author: XieStr
avatar: ../../img/avatar.jpg
# authorlink: https://author.site
# cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - GoWeb
tags:
  - GoWeb
# nolastmod: true
draft: false
---

对7天系列中Gee框架学习的记录。推荐和Gee第二天一起食用。

Context部分主要实现两个目标

- 将`路由(router)`独立出来，方便之后增强。
- 设计`上下文(Context)`，封装 Request 和 Response ，提供对 JSON、HTML 等返回类型的支持。

为什么要使用(设计)Context ?

>- 简化接口调用
>
>  在前面设计的框架中，请求是根据`*http.Request`和`http.ResponseWriter`处理的。但是这两个对象提供的接口粒度太细，在开发的过程中，有时候要够减一个完整的响应，一些参数设置就必须自己一个个写上去。[Header中的一些参数，如StatusCode、ContentType等]如果是需要重复利用的情况下，就会出现大量的冗余代码。一般来说，这种时候都会手动封装一个工具进行使用，简化开发。
>
>  而Context在这里的作用不止于此。
>
>- 动态路由参数传递
>
>- 中间件信息承载，天然隔离不同请求的数据(协程安全)

特点：Context随着每一个请求的出现而产生，请求的结束而销毁。

和当前请求强相关的信息由Context承载是非常合适的。

---

### Context 实现

context.go

```go
// map[string]interface{} 设置别名，够减JSON更方便
type H map[string]interface{}

// 定义Context 结构体，方便后续拓展
// 加入相关参数，冗余Path 和 Method、StatusCode方便处理
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	// response info
	StatusCode int
}

// new Context 方法，给相关参数赋初值
func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Writer: w,
		Req:    req,
		Path:   req.URL.Path,
		Method: req.Method,
	}
}

// 以下都是实现Context的方法，方便请求处理数据

// 表单处理 如：账号密码提交
func (c *Context) PostForm(key string) string {
	return c.Req.FormValue(key)
}

// 从 URL ? 后的查询参数中获取 key 对应的值 如 /home?q=golang&page=1
func (c *Context) Query(key string) string {
	return c.Req.URL.Query().Get(key)
}

// Status赋值
func (c *Context) Status(code int) {
	c.StatusCode = code
	c.Writer.WriteHeader(code)
}

// 设置Header
func (c *Context) SetHeader(key string, value string) {
	c.Writer.Header().Set(key, value)
}

// String方法处理
func (c *Context) String(code int, format string, values ...interface{}) {
	c.SetHeader("Content-Type", "text/plain")
	c.Status(code)
	c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

// JSON 方法处理
func (c *Context) JSON(code int, obj interface{}) {
	c.SetHeader("Content-Type", "application/json")
	c.Status(code)
	encoder := json.NewEncoder(c.Writer)
	if err := encoder.Encode(obj); err != nil {
		http.Error(c.Writer, err.Error(), 500)
	}
}

// 二进制数据处理
func (c *Context) Data(code int, data []byte) {
	c.Status(code)
	c.Writer.Write(data)
}

// HTML内容处理
func (c *Context) HTML(code int, html string) {
	c.SetHeader("Content-Type", "text/html")
	c.Status(code)
	c.Writer.Write([]byte(html))
}
```

### 路由Router 实现

将原来在Engine中的相关路由结构和方法提出出来。方便后续增强。如动态路由等。由于我们已经实现了Context代替原来的`http.ResponseWriter` 和`*http.Request`，这里要将handle方法的参数转变成Context。

router.go

```go
// 定义路由结构体，内部为map数据结构对应的路由映射表
type router struct {
	handlers map[string]HandlerFunc
}

// new 方法，初始化router内部参数
func newRouter() *router {
	return &router{handlers: make(map[string]HandlerFunc)}
}

// addRoute方法变更
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	log.Printf("Route %4s - %s", method, pattern)
	key := method + "-" + pattern
	r.handlers[key] = handler
}

// handle 参数变更 重点，这里原来是ServeHTTP的处理逻辑。被提取出来到这里作为单独方法使用。
func (r *router) handle(c *Context) {
	key := c.Method + "-" + c.Path
	if handler, ok := r.handlers[key]; ok {
		handler(c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

### Engine 部分相关更改

gee.go

```go
// HandlerFunc defines the request handler used by gee
// HandlerFunc参数更改为 Context
type HandlerFunc func(*Context)

// Engine implement the interface of ServeHTTP
// 原来的router 映射表改为 定义的rouer结构体
type Engine struct {
	router *router
}

// New is the constructor of gee.Engine
// new 方法也要做更新
func New() *Engine {
	return &Engine{router: newRouter()}
}

// engine 的 addRoute更新成调用router的addRoute方法，简化了
func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	engine.router.addRoute(method, pattern, handler)
}

// 省略 GET 和 POST 、Run 无变更

// 由于抽离了handle到router，这里直接new Context 后使用router的handle就可以了
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := newContext(w, req)
	engine.router.handle(c)
}
```

### 调用更改后的框架

main.go

```go
func main() {
	r := gee.New()
    // Handlerfunc 参数变更成Context
	r.GET("/", func(c *gee.Context) {
        // 调用Context的HTML响应方法
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})
    // Handlerfunc 参数变更Context
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
        // 调用Context的Query响应方法
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path)
	})
	// Handlerfunc 参数变更Context
	r.POST("/login", func(c *gee.Context) {
        // 调用Context的JSON响应方法，使用设置的别名新建一个JSON
		c.JSON(http.StatusOK, gee.H{
			"username": c.PostForm("username"),
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
```

这部分Context的响应方法有些可能存在一定的问题，但是作为一个以构建框架为主的系列，不必苛求太严格。后续如果要使用的话记得优化即可。
