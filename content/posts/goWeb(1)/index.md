---
title: Gee微框架学习记录(第一天)
date: 2025-03-30T14:21:12+08:00
lastmod: 2025-03-30T14:21:12+08:00
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

对7天系列中Gee框架学习的记录。推荐和Gee第一天一起食用。

Gee框架参考了Go的Web框架`Gin`实现。

主要为：

>1、基本请求接口 Handler
>
>2、上下文 Context (重要)
>
>3、Trie树路由 Router(动态路由实现，重要。着重理解)
>
>4、分组控制(Group) 
>
>5、中间件 Middleware (重要)
>
>6、HTML 模板 和 错误恢复 (需要理解中间件后再来学习这一部分)

----

### 前置知识 Handler 和 HandlerFunc

> `HandlerFunc` 是 `Handler` 接口的一个函数适配器实现
>
> 通过类型转换，普通函数可以"升级"为 `Handler`

Handler是一个接口类型，HandlerFunc是一个函数类型。

- Handler接口定义了一个ServerHTTP方法，任何实现了这个方法的函数都可以作为HTTP处理器。

```Go
type Handler interface {
    ServerHTTP(ResponseWriter, *Request)
}
```

- HandlerFunc 是一个函数类型，他的签名和 ServerHTTP 方法一致。

```go
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)  // 直接调用函数本身
}
```

但是标准库中HandlerFunc实现了ServerHTTP方法，于是HandlerFunc类型就满足了Handler接口。

在调用的时候直接执行HandlerFunc自身不会有额外开销。

所以当存在一个普通函数参数类型为 `w ResponseWriter, r *Request` 时，可以通过类型转化变成Handler。

```go
func myHandler(w ResponseWriter, r *Request) {
    xxxxx;
    ·····
}

var h http.Handler = http.HandlerFunc(myHandler)
```

如果是按照标准库启动Web服务

那么

```go
func main() {
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":9999", nil))
}

// handler echoes r.URL.Path
func indexHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
}

// handler echoes r.URL.Header
func helloHandler(w http.ResponseWriter, req *http.Request) {
	for k, v := range req.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
}
```

这里实际上是将普通函数通过HandlerFunc转化为Handler。将相关请求交给了ServerHTTP统一处理。

这里的http.ListenAndServe中第二个参数为nil表示使用默认的多路复用器。

> http.HandlerFunc()或者http.Handle()注册的路由会自动绑定到这个默认的多路复用器。
>
> 不需要人为显示创建ServeMux实例
>
> 等价于
>
> ```go
> mux := http.DefaultServeMux
> mux.HandleFunc("/", indexHandler)
> mux.HandleFunc("/hello", helloHandler)
> http.ListenAndServe(":9999", mux)  // 效果完全一致
> ```
>
> 如果不为nil
>
> 那么需要传入一个实现了ServeHTTP方法的函数(隐式转化为Handler)，或者是定义的Handler方法，
>
> 并且这个Handler需要自己实现路由映射。后面将提到Gee的路由映射。

---

了解了以上内容后，应该对Handler 和 HandlerFunc 有了一个初步的认识。

如果我们要实现自己的Web框架， 只需要实现Handler 方法中的 ServerHTTP方法， 将所有请求交给ServerHTTP处理。Gee框架也是这么做的。

> Handler 和 HandlerFunc 标准库已经实现。不用额外实现

在实现ServerHTTP方法之前，需要定义一个`Engine`。 

`Engine`用于后续拓展框架使用。可以理解为，框架通过Engine管理所有内容。包括后续的动态路由、分组以及中间件。

因此，通过Engine实现ServerHTTP从而实现拓展管理能力。

``` go
type Engine struct {}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	case "/hello":
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	default:
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}

func main() {
	engine := new(Engine)
	log.Fatal(http.ListenAndServe(":9999", engine))
}
```

这里传入engine，实现了ServeHTTP方法，可以转化成Handler。不再使用默认的多路复用器。所有的请求都通过ServeHTTP 方法的硬编码解决。

### Gee框架雏形实现

go.mod解释

```go
module example				// 模块名称

go 1.13						// go版本依赖

require gee v0.0.0			 // 依赖模块和版本号

replace gee => ./gee		 // 将依赖gee 替换为本地./gee目录
```

gee.go

```go
// HandlerFunc defines the request handler used by gee
// 定义框架的HandlerFunc方法。
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Engine implement the interface of ServeHTTP
// Engine实现http.Handler接口，替代标准库的默认多路复用器
// 核心原理：
//  1. 使用map[string]HandlerFunc维护路由表
//  2. key的格式为"METHOD-PATH"(如"GET-/hello")
//  3. 接收到请求时，根据方法和路径查找对应的HandlerFunc执行
type Engine struct {
	router map[string]HandlerFunc
}

// New is the constructor of gee.Engine
// 新建结构体方法
func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

// 将路由加入路由表中，key的格式为"METHOD-PATH"(如"GET-/hello")
func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handler
}

// GET defines the method to add GET request
// Engine实现GET方法，将路由和处理方法注册到路由表。
func (engine *Engine) GET(pattern string, handler HandlerFunc) {
	engine.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
// Engine实现POST方法，将路由和处理方法注册到路由表
func (engine *Engine) POST(pattern string, handler HandlerFunc) {
	engine.addRoute("POST", pattern, handler)
}

// Run defines the method to start a http server
// 将默认的监听端口方法进行封装
func (engine *Engine) Run(addr string) (err error) {
	return http.ListenAndServe(addr, engine)
}

// 实现ServerHTTP方法，满足将Engine实例 转化为Handler的能力。
// 逻辑：解析请求路径，查找路由表，找到则执行对应的处理方法，否则返回404 NOT FOUND
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	key := req.Method + "-" + req.URL.Path
	if handler, ok := engine.router[key]; ok {
		handler(w, req)
	} else {
		fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
	}
}
```

细心观察，发现由`Engine`实现 `GET`和`POST`方法，体现了Engine的管理特点。

main.go

```go
func main() {
	// new 了 一个 engine 对象
    r := gee.New()
	// 调用GET方法，实现一个匿名函数Handler处理该路由
    r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})
	// 与上同理
	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		for k, v := range req.Header {
			fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
		}
	})
	// 调用封装的Run方法监听端口
	r.Run(":9999")
}
```



Engine 通过实现 GET/POST 等 HTTP 方法，提供了路由注册接口，这种设计：

- 封装了底层路由表操作

- 提供了清晰的API边界

### 特别注意事项

- 当前实现没有处理：

  - 路径参数（如 `/users/:id`）

  - 通配符匹配

  - 中间件支持

  - 路由分组

- 404处理应该先设置Header：

​	`w.WriteHeader(http.StatusNotFound)  // 应该在输出内容前调用`

- 考虑添加OPTIONS方法的自动支持  (Gee框架未实现)

这些改进方向可以在后续版本中逐步添加。

