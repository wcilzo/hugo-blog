---
title: Gee微框架学习记录(第五天)
date: 2025-04-10T21:25:08+08:00
lastmod: 2025-04-10T21:25:08+08:00
author: xiestr
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

对7天系列中Gee框架学习的记录。推荐和Gee第五天一起食用。

> [!note]
>
> 对于初学者来说，可能会曲解这一节的中间件的意思。可能会认为是Redis、MQ等中间件。
>
> 但是这两个实际上不是一个东西。
>
> Go的中间件`【代码逻辑层面】`
>
> - 是Web请求处理管道中的软件层。是位于Http服务器和业务处理程序之间的应用层。通常是指日志、认证、限流等处理程序。
>
> Redis/MQ等`【系统架构层面】`
>
> - 是独立分布式系统组件。作为应用间连接的桥梁。
>
> 我更偏向于这样理解：Redis/MQ等中间件是指`独立的中间件服务`。而Go中的中间件是指一些`约定程序`(或者说`拦截器`，`处理器链`)。

为什么Web框架需要有中间件呢？

对于Web框架来说，不可能实现所有的服务。如果实现所有的服务，整个程序将过于臃肿，并且有些服务并不是使用该框架的开发者所需要的。不必要的服务在启动后浪费内存，增加企业和开发者的硬件成本。甚至来说这已经不能是一个框架，而是一个系统。

为了方便用户在使用基本服务的情况下，进行一些必要的业务拓展，Web框架需要一个“接口”，允许开发者拓展功能。

如何做到用户拓展的同时不影响原来框架自身基础服务呢？这就是中间件设计需要考虑的地方。

### Gee框架实现

前面提到过，框架用Engine进行统一管理，那么中间件应该在Engine的结构体中配置。

中间件的参数又靠什么进行获取和传递呢？

在第二天的学习记录中有提到，为了简化开发者的接口调用处理以及动态路由参数传递，引入了`Context`进行处理。

并且 `Context随着每一个请求的出现而产生，请求的结束而销毁。` , 符合中间件的处理需要。

现在有了中间件的定义和参数传递，但是还不知道中间件如何生效。

再重新思考一下，中间件是为了给某个请求在`处理流程前`或者`处理流程后`进行一些特殊的处理。第一天的笔记中，我们将所有的请求都交给了`ServerHTTP`处理。所谓的处理逻辑(流程)也就是`HandlerFunc`在这里调用，那么中间件应该在这里生效。

现在，中间件的定义、参数传递，调用的大体逻辑都已经了解。接下来应该讨论具体的实现细节了。

---

#### Gee中间件定义

上一篇笔记中配置了分组控制，有了分组控制，就可以给不同的前缀路由进行不同的分组处理。这些处理实际上就是指中间件。也就是每个分组都有自己的中间件进行处理。

在上一篇的分组控制中，在`RouterGroup`中定义了中间件`middlewares`， 并且`Engine`通过继承的方式继承了`RouterGroup`的所有属性(包括middlewares)。因此，已经完成了中间件定义。

server.go

```go
type {
    // Group结构体
    RouterGroup struct {
        // 当前路由前缀，用于区分分组
        prefix      string
        // 组级别中间件
        middlewares []HandlerFunc // support middleware
        // 类似树结构，父节点
        parent      *RouterGroup  // support nesting
        // 设定Engine，所有Group都应该指向同一个Engine
        engine      *Engine       // all groups share a Engine instance
	}
    // Engine结构体
    Engine struct {
        // 继承来自RouterGroup的参数，共享内存，Engine是最顶层Group
		*RouterGroup
        // router 路由树
		router        *router
		// 存储groups数组，group不可能只有一个
         groups        []*RouterGroup // store all groups
	}
}
```

> [!Tip]
>
> 中间件本质是处理方法(HandlerFunc)

#### Gee中间件参数处理和链调用

Context中已经存在一些参数，为了适配中间件，还需要加上`index`和`handler`参数。

index是用来标识当前context执行到中间件处理链的哪个位置。可以理解为数组下标。

handler是用于存储当前请求需要的中间件处理方法。

> [!Note]
>
> 这里可能又有疑问: 
>
> 为什么RouterGroup定义了中间件middlewares，这里Context还要再定义handler呢？是不是冗余了？
>
> 并不是。RouterGroup上定义的中间件方法是针对于不同分组的，是`组级别`的`共用方法`。而Context中定义的handler是针对于当前请求的中间件方法和请求自身需要处理的HandlerFunc方法。
>
> **生命周期**上：在RouterGroup中定义的中间件方法是启动时注册，长期存在。Context中定义的方法在`请求开始时创建，结束时候销毁`。

context.go

```go
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
	Params map[string]string
	// response info
	StatusCode int
	// middleware
    // 中间件需要参数 
	handlers []HandlerFunc
	index    int
}

// 初始化修改
func newContext(w http.ResponseWriter, req *http.Request) *Context {
	return &Context{
		Path:   req.URL.Path,
		Method: req.Method,
		Req:    req,
		Writer: w,
		// 中间件index 为 -1
         index:  -1,
	}
}

// 处理中间件方法链
func (c *Context) Next() {
	c.index++
	s := len(c.handlers)
	for ; c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}
```

Next方法执行后，效果类似于数组下标 + 1，进入到下一个中间件。一直到中间件都处理完，再从后往前回调。处理完方法中执行完`Next()`的剩余逻辑。

**Next中循环的作用? 同时这里for 中的 c.index ++ 会不会影响到的 c.index原来的值** ?
index 的初始值 为 -1，index ++ 使得首次调用 index 为正常值 0
注意： handlers是一个handler数组，每个handler就是我们所谓的 HandlerFunc(method)
**每次循环的时候，并不会调用到 for 循环外面的 c.index ++。** 还在for方法块里面。

注意这里**并不是： 每次执行玩 handler 之后循环内进行 c.index ++**，而是通过下一个Next的第一行的c.index ++
观察整个for， 我们并没有给index 一个初始值 如 (for i := 1; xx < xx; i ++) 这种形式
在中间件中，当我们手动调用 Next 的时候，就是进入一个递归调用的过程，
执行c.index ++（不是for的，而是第一行c.index ++。）帮我们指定到下一个 handler 的 index 值。
而如果我们没有在 中间件 使用 Next，会直接执行当前中间件中的所有逻辑。
然后由Next 中 for 循环的c.index++ 指定下一个中间件的handel 的index值。确保中间件顺利执行。
如果还是不太明白：可以看下面代码解释

中间件定义

```GO
func Middleware1(c *Context) {
   fmt.Println("M1-Start")
   c.Next() // 有的调用，有的不调用
   fmt.Println("M1-End")
}

func Middleware2(c *Context) {
   fmt.Println("M2-Start") // 这里故意不调用 Next()
   fmt.Println("M2-End")
}

func Middleware3(c *Context) {
   fmt.Println("M3-Start")
   c.Next()
   fmt.Println("M3-End")
}
```

- 场景1：所有中间件都调用 Next()
  执行顺序：

       M1-Start →
         M2-Start →
          M3-Start →
          M3-End →
         M2-End →
       M1-End
      
      - 场景2：Middleware2 不调用 Next()
        M1-Start →
        M2-Start →
        M2-End   // 因为没有 Next()，M2 的后置逻辑立即执行！
        M3-Start → // 框架的 for 循环强制推进
        M3-End →
        M1-End

关键：
- M3 没有被跳过（因为 for 循环的 c.index++）

- M2 的 M2-End 提前执行（丢失了后置逻辑的预期效果）

1）使用这种方式能够避免中间件 忘记 调用 Next 方法阻塞后续中间件执行，
即使忘记调用， 循环能够使得他正常调用后续中间件。
2）性能优化：  for 循环 对比于 Next 递归调用， 性能会提高。

> [!Note]
>
> 循环是线性的，递归调用对于中间件过多时，栈深度显著增加。

最后，到这里，应该已经了解了for循环中的c.index ++ 不会覆盖原来的c.index,
因为中间件需要显示调用 Next方法，调用后直接进入了 Next 的嵌套循环，c.index值不会被循环中的c.index++改变
而当所有中间件都调用了 Next 方法，此时for循环的c.index ++ 就变得没有意义了。
这里需要for循环的原因就是为了给 中间件 没有显示调用 Next 方法兜底。

### Gee中间件调用

gee.go

```go
// Use is defined to add middleware to the group
// 对组添加定义的中间件
func (group *RouterGroup) Use(middlewares ...HandlerFunc) {
	group.middlewares = append(group.middlewares, middlewares...)
}

// 增加中间件调用处理
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 暂存中间件数组
	var middlewares []HandlerFunc
	// 请求判断属于哪个组，根据前缀进行判断，添加对应的中间件到暂存数组
    for _, group := range engine.groups {
		if strings.HasPrefix(req.URL.Path, group.prefix) {
			middlewares = append(middlewares, group.middlewares...)
		}
	}
	c := newContext(w, req)
	// 给context中定义的handler中间件数组添加 组需要处理的中间件数组
    c.handlers = middlewares
	engine.router.handle(c)
}
```

router.go

```go
// handle， 让当前handler加入中间件的处理链中执行。一种中间件
func (r *router) handle(c *Context) {
	n, params := r.getRoute(c.Method, c.Path)
	if n != nil {
		c.Params = params
		key := c.Method + "-" + n.pattern
		// 方法的处理逻辑追加到处理链中
		c.handlers = append(c.handlers, r.handlers[key])
	} else {
		c.handlers = append(c.handlers, func(c *Context) {
			c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
		})
	}
	// 手动指定 Next()
	c.Next()
}

```

#### Gee中间件使用

定义一个简单的中间件logger，用于输出请求处理日志。

logger.go

```go
func Logger() HandlerFunc {
	return func(c *Context) {
		// Start timer
		t := time.Now()
		// Process request
		c.Next()
		// Calculate resolution time
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```

main.go

```go
// 测试功能
func onlyForV2() gee.HandlerFunc {
	return func(c *gee.Context) {
		// Start timer
		t := time.Now()
		// if a server error occurred
		c.Fail(500, "Internal Server Error")
		// Calculate resolution time
		log.Printf("[%d] %s in %v for group v2", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}

func main() {
	r := gee.New()
    // 中间件使用
	r.Use(gee.Logger()) // global midlleware
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	v2 := r.Group("/v2")
    // 中间件使用
	v2.Use(onlyForV2()) // v2 group middleware
	{
		v2.GET("/hello/:name", func(c *gee.Context) {
			// expect /hello/geektutu
			c.String(http.StatusOK, "hello %s, you're at %s\n", c.Param("name"), c.Path)
		})
	}

	r.Run(":9999")
}
```

---

到这里可能还有些疑惑。

> [!Note]
>
> 上面Next的处理以及ServerHTTP、handle，似乎并规定说中间件的执行顺序，那么为什么说中间件能在请求处理流程后执行？似乎只体现了逻辑前的情况。
>
> 所谓的中间件`处理逻辑前`和`处理逻辑后`执行是每个中间件都有的功能。根据`Next()`前后进行`隔离`。每个中间件中，执行Next()前的逻辑就是处理逻辑前，Next()后的逻辑就是处理逻辑后。中间件之间互相独立。顺序并不影响。
>
> 那么可能又有一个问题？是否存在一种，中间件之间需要一定交互的形式。必须规定执行顺序，这种就只能交给开发者自身定义了吗？或者说，这种根本就不能是独立的几个中间件，而是只能是一个中间件？
>
> 这种中间件之间需要依赖的情况，分成2种
>
> - 强依赖关系，那么就需要把这些中间件合并成一个
> - 弱依赖关系，交给开发者自身处理

建议这部分还是不太懂的多看几遍，再多梳理一下。

有时间要多看看评论，评论提出的问题可能是没有注意到的问题。或者是一些优化思路。



