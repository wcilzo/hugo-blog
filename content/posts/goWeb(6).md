---
title: Gee微框架学习记录(第六、七天)
date: 2025-04-12T23:22:56+08:00
lastmod: 2025-04-12T23:22:56+08:00
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

对7天系列中Gee框架学习的记录。推荐和Gee第六、第七天一起食用。

接下来的内容不多，所以将Gee系列第六、第七天的内容放在一起。

第六天是模板HTML Template，这一节我认为相对来说不那么重要。因为现在已经有Next, Nuxt这些比较成熟的SSR前端渲染框架了。但是对于一个基本的网络框架来说，这一点还是必不可少的。

在完成HTML Template模板之前，先完成静态文件读取渲染到网页端。

gee.go

```go
// create static handler
func (group *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {
    // 绝对路径获取
	absolutePath := path.Join(group.prefix, relativePath)
	// 解析请求地址映射到服务器文件上的真实地址
    fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))
	// 返回一个HandlerFunc
    return func(c *Context) {
		file := c.Param("filepath")
		// Check if file exists and/or if we have permission to access it
		// 检查文件是否存在
         if _, err := fs.Open(file); err != nil {
			c.Status(http.StatusNotFound)
			return
		}
		// 交给标准库文件服务器处理
		fileServer.ServeHTTP(c.Writer, c.Req)
	}
}

// serve static files
func (group *RouterGroup) Static(relativePath string, root string) {
    // 1. 创建静态文件处理handler
    handler := group.createStaticHandler(relativePath, http.Dir(root))
    // 2. 构造URL匹配模式
    urlPattern := path.Join(relativePath, "/*filepath")
    // 3. 注册GET路由
    group.GET(urlPattern, handler)
}
```

开发者由Static方法查看静态文件。

### HTML 模板渲染

一样的，由Engine管理HTML模板，因此定义到Engine的结构体中。

gee.go

```go
Engine struct {
	*RouterGroup
	router        *router
	groups        []*RouterGroup     // store all groups
	htmlTemplates *template.Template // for html render
	funcMap       template.FuncMap   // for html render
}

func (engine *Engine) SetFuncMap(funcMap template.FuncMap) {
	engine.funcMap = funcMap
}

func (engine *Engine) LoadHTMLGlob(pattern string) {
	engine.htmlTemplates = template.Must(template.New("").Funcs(engine.funcMap).ParseGlob(pattern))
}
```

`*template.Template` 将所有的模板加载进内存，`template.FuncMap`对象是所有的自定义模板渲染函数。

context.go

```go
type Context struct {
    // ...
	// engine pointer
	engine *Engine
}

// 重写 HTML函数
func (c *Context) HTML(code int, name string, data interface{}) {
	c.SetHeader("Content-Type", "text/html")
	c.Status(code)
	if err := c.engine.htmlTemplates.ExecuteTemplate(c.Writer, name, data); err != nil {
		c.Fail(500, err.Error())
	}
}
```

在 `Context` 中添加了成员变量 `engine *Engine`，这样就能够通过 Context 访问 Engine 中的 HTML 模板(是共享资源)。

gee.go

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	// ...
	c := newContext(w, req)
	c.handlers = middlewares
    // 实例化Context时要赋值
	c.engine = engine
	engine.router.handle(c)
}
```

---

第七天的Panic Recover 错误恢复

- panic

Go中常见的错误处理方法是返回error，交给调用者决定后续如何处理。有时候开发者写的程序逻辑错误也有可能会触发panic。我们可以利用这一点，将无法恢复的错误手动触发panic。再结合defer 和 revover，就完成了错误处理机制。

- defer

panic 导致的程序退出前，会先处理当前协程上defer的任务。执行结束后退出，类似java语言的`try...catch`代码块。可以定义多个defer，在同一个函数中定义的defer会逆序执行，从最后一个定义的defer函数一直逆序执行到第一个定义的defer函数。当全部defer执行完后，抛出panic。

- recover

recover在defer中生效，避免发生panic导致的程序终止。

### Gee的错误处理

recovery.go

```go
package gee

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"strings"
)

// print stack trace for debug
func trace(message string) string {
	var pcs [32]uintptr
    // Callers 用来返回调用栈的程序计数器
    // 0 是他自己
    // 1是上一层trace
    // 2是再上一层的defer func
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller

	var str strings.Builder
	str.WriteString(message + "\nTraceback:")
	for _, pc := range pcs[:n] {
        // 获取对应的函数
		fn := runtime.FuncForPC(pc)
		// 获取到调用该函数的文件名称和行号，打印到日志 
         file, line := fn.FileLine(pc)
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))
	}
	return str.String()
}

func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```

这里将错误处理变成了中间件的形式。

Recover函数:

用defer挂载错误恢复函数，在函数中调用recover捕获panic，最后打印堆栈数据到日志当中。返回错误信息给用户。

trace用于捕获panic。

**使用方式：** 通过前面笔记中定义的Use函数使用(作为一种中间件使用).

---

Gee框架的学习笔记到此完结，最后这一篇并没有涉及到比较深入的讨论，特别是HTML模板这一块。对于错误恢复机制来说，灵活的运用了前面定义的中间件部分，是一个很好的学习例子。
