---
title: Gee微框架学习记录(第三天)
date: 2025-04-04T14:03:16+08:00
lastmod: 2025-04-04T14:03:16+08:00
author: xieStr
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

对7天系列中Gee框架学习的记录。推荐和Gee第三天一起食用。

第三天前缀树路由是Gee框架的重难点。

主要内容为

- 使用Trie树实现动态路由(dynamic route)解析

- 增加支持使用`:name`和`*filepath`

**本文将把重点放在Trie树在架构种的实现上**

---

### 为什么不用原来的Map作为动态路由？

引入动态路由需要使用新的数据结构。

之前我们在router中使用map作为路由映射。键为路由路径和方法的字符串，值为对应需要执行的`HandlerFunc`方法。有些时候，我们的请求需要带有一个可变的参数。

如，Get方法中需要获取一个跟`id`有关的数据。

`Get: localhost:8080/:id` -> `Get: localhost:8080/1001`

map对于动态路由处理不太好。找到需要键`:id`需要遍历所有的键做正则匹配。

也无法匹配通配符`/*filepath`。

对于类似这种`/user/:id/xxx`，需要拆分多个map层级，复杂度++。

当路由数量足够大，或者包含更多的动态参数，map性能是不够的。我们不可能浪费过多的时间在解析路由这件事情上面。因此需要一种足够快的数据结构支持动态路由。

而Trie树是常用的动态路由实现的数据结构。

Trie树相关的简介7Days中有提到。略过

---

### Gee中Trie的实现

7day中的一些注释解释可能不太容易理解。因此这里对他进行了一些额外的解释。

#### Trie树实现

trie.go

```go
// 节点结构体
type node struct {
	pattern  string // 待匹配路由，例如 /p/:lang
	part     string // 路由中的一部分，例如 :lang
	children []*node // 子节点，例如 [doc, tutorial, intro]
	isWild   bool  // 是否精确匹配，part 含有 : 或 * 时为true 
}
```

pattern -> 需要匹配的路由，只有在最后一段(叶子节点)的时候才赋值，其他时候为 `null` 或者 为 `*`。当只有是通配符的时候才会赋值为`*`。

> 最后一段是指
>
> 如 `/p/:lang/a` 中的 `a`, 或者是`/p/b/c` 中的`c`

isWild  

| 字段     | 值      | 含义                                                         |
| :------- | :------ | :----------------------------------------------------------- |
| `isWild` | `true`  | 当前节点的 `part` 是动态参数（如 `:lang`）或通配符（如 `*filepath`） |
| `isWild` | `false` | 当前节点是静态路径（如 `docs`），需完全匹配                  |

>假设有如下路由树
>
>```text
>/
>├─ docs               (isWild=false)
>├─ :lang              (isWild=true)
>│   ├─ tutorial       (isWild=false)
>│   └─ *filepath      (isWild=true)
>└─ static/*filepath   (isWild=true)
>```
>
>场景1：匹配 `/docs`
>
>- 根节点 `/` 查找子节点 `docs`（`isWild=false`）→ 完全匹配 ✅
>
>- 返回 `pattern=/docs`
>
>场景2：匹配 `/go/tutorial`
>
>- 根节点 `/` 查找子节点：
>  - `docs`（`isWild=false`）→ 不匹配 ❌
>  - `:lang`（`isWild=true`）→ 匹配 `lang="go"` ✅
>
>- 在 `:lang` 下查找子节点 `tutorial`（`isWild=false`）→ 完全匹配 ✅
>
>- 返回 `pattern=/:lang/tutorial`，参数 `{"lang": "go"}`
>
>场景3：匹配 `/static/js/main.js`
>
>- 根节点 `/` 查找子节点 `static`（`isWild=false`）→ 完全匹配 ✅
>
>- 在 `static` 下查找通配子节点 `*filepath`（`isWild=true`）→ 捕获剩余部分 ✅
>
>- 返回 `pattern=/static/*filepath`，参数 `{"filepath": "js/main.js"}`
>
>---
>
>为什么需要 `isWild`？
>
>- **`isWild=true`** 的节点是动态路由的核心，使以下功能成为可能：
>  - 参数捕获（`/user/:id` → `id=123`）
>  - 通配匹配（`/static/*filepath` → `filepath=js/main.js`）
>- **匹配优先级**：静态节点优先于动态节点，避免冲突。
>- **性能关键**：通过 `isWild` 快速过滤不匹配的节点分支。

trie.go

```go
// 第一个匹配成功的节点，用于插入
func (n *node) matchChild(part string) *node {
	for _, child := range n.children {
		if child.part == part || child.isWild {
			return child
		}
	}
	return nil
}
// 所有匹配成功的节点，用于查找
func (n *node) matchChildren(part string) []*node {
	nodes := make([]*node, 0)
	for _, child := range n.children {
		if child.part == part || child.isWild {
			nodes = append(nodes, child)
		}
	}
	return nodes
}
```

上面是节点的查询方法，一个是查找一个节点，一个是查找所有匹配成功的节点。

> 为什么matchChild是匹配第一个成功的节点呢？
>
> trie可以作为动态路由和静态路由的存储结构。但当我们进行节点的插入的时候，trie树的某些节点可能存储了`:lang` 或者 `:id`这样的节点。也就是动态路由的节点。为了精确匹配待插入的路由，从第一个成功匹配的节点进行递归匹配是至关重要的。

有了查询方法后，下面就是插入和搜索方法

trie.go

```go
// 插入
func (n *node) insert(pattern string, parts []string, height int) {
	// 如果是最后一段，则赋值pattern，并结束递归
    if len(parts) == height {
		n.pattern = pattern
		return
	}
	// 取出当前段
	part := parts[height]
	// 用上面的查找方法匹配第一个精准匹配的节点 静态 or 动态
    child := n.matchChild(part)
    // 如果找不到，那么就到了插入位置
	if child == nil {
         // 新增node，赋值为当前的part，并判断是否是动态路由
		child = &node{part: part, isWild: part[0] == ':' || part[0] == '*'}
		// 将新增的节点加入树中
         n.children = append(n.children, child)
	}
    // 继续递归
	child.insert(pattern, parts, height+1)
}
// 查询
func (n *node) search(parts []string, height int) *node {
	// 如果是最后一段或者是当前 part 匹配 到了通配符*
    if len(parts) == height || strings.HasPrefix(n.part, "*") {
        // 判断pattern是否是空，如果是空则表示没找到。
        // pattern只有是最后一段(叶子节点)才会赋值。
         if n.pattern == "" {
			return nil
		}
        // 否则返回当前node，表示找到了
		return n
	}
	// 取出当前段
	part := parts[height]
	// 用上面的查找方法匹配第一个精准匹配的节点 静态 or 动态
    children := n.matchChildren(part)
	// 循环节点数组，找到对应的node
	for _, child := range children {
        // 递归查找 树的高度要 + 1
		result := child.search(parts, height+1)
		// 如果找到了，返回找到的node节点
         if result != nil {
			return result
		}
	}
	// 没找到返回空
	return nil
}
```

实际上就是树的遍历和查询，还是比较好理解的。 

#### Router

实现了Trie树的查询和插入，接下来就是如何将这个数据结构用于Gee框架中。

Trie树是用于替换router的map路由表，因此Router需要进行修改。

在原来的handlers map的基础上，添加node的map。

为什么是map？RestFul 请求中有不同的请求方式，可能是`GET\POST\...`。Gee框架只实现了GET和POST方式，使用map后续还可以比较方便的拓展。

这样的话，每种不同的方法都有一颗自己的Trie结构树。

我们就能让router 和 Trie树关联起来了。

router.go

```go
// route 结构体
type router struct {
    // 添加roots 关联node
	roots    map[string]*node
	handlers map[string]HandlerFunc
}

// roots key eg, roots['GET'] roots['POST']
// handlers key eg, handlers['GET-/p/:lang/doc'], handlers['POST-/p/book']

// new Router需要新的roots参数初始化
func newRouter() *router {
	return &router{
		roots:    make(map[string]*node),
		handlers: make(map[string]HandlerFunc),
	}
}

// Only one * is allowed
// router Path 路径解析，只支持一个通配符 *
func parsePattern(pattern string) []string {
	// 将通过'/'分割的字符串转化成一个数组
    vs := strings.Split(pattern, "/")
	
    // 定义一个parts 数组用于存储解析后的路径分段
	parts := make([]string, 0)
	// 循环判断，如果有通配符的情况，后续不需要添加到parts中，提升性能。其他的添加到parts中
    for _, item := range vs {
        // 当前元素不为空
		if item != "" {
		   // 加入到parts分段数组中
             parts = append(parts, item)
			// 如果当前元素首字母是通配符 '*'，那么后续全都跳过
             if item[0] == '*' {
				break
			}
		}
	}
    // 返回
	return parts
}

// 路径添加，需要调用Trie树的insert插入方法
func (r *router) addRoute(method string, pattern string, handler HandlerFunc) {
	// 解析路径
    parts := parsePattern(pattern)
	
    // 原来的string映射不变，用于handler处理
	key := method + "-" + pattern
	// 取对应方法的Trie树
    _, ok := r.roots[method]
    // 如果Trie树为空，则初始化
	if !ok {
		r.roots[method] = &node{}
	}
    // 对应方法的Trie树插入新节点
	r.roots[method].insert(pattern, parts, 0)
	// handlers 处理不变，key 映射 handler方法
    r.handlers[key] = handler
}

// 通过path查询树节点
func (r *router) getRoute(method string, path string) (*node, map[string]string) {
	// 解析路径
    searchParts := parsePattern(path)
	// 初始化一个方法用于存储匹配的路由参数
    params := make(map[string]string)
	// 通过method获取对应的方法Trie树
    root, ok := r.roots[method]
	// 如果Trie树不存在，返回空
	if !ok {
		return nil, nil
	}
	// 调用Trie树的查找方法，搜对应节点
	n := root.search(searchParts, 0)
	// 如果找到
	if n != nil {
		// 解析节点的pattern，转化成分段数组parts
         parts := parsePattern(n.pattern)
         // 循环解析的parts分段
		for index, part := range parts {
            // 如果是动态路由且为 ':'动态参数匹配，将params参数赋值，不需要':'号
			if part[0] == ':' {
				params[part[1:]] = searchParts[index]
			}
            // 如果是动态路由且为 '*'通配符匹配，判断part长度是否大于1，
            // 如果符合，同上处理params参数，并跳过后续处理，提升性能。
			if part[0] == '*' && len(part) > 1 {
				params[part[1:]] = strings.Join(searchParts[index:], "/")
				break
			}
		}
        // 返回 查找到的节点和params参数
		return n, params
	}
	// 如果没找到，返回空
	return nil, nil
}
```

这部分的handlers似乎可以进行一些修改，如果足够细心，会发现handlers和node的insert分开处理，实际上应该是可以一起处理的，在node中加入对应的HandlerFunc。在获取node的时候实际上就获取了对应的HandlerFunc。当然这样做的话树的处理难度上升和内存的消耗也响应的增加。

#### Context 和 Handle 修改

开发中我们可能需要拿到解析的参数，因此对Context对象增加一个属性和方法。将参数存储到`Params`中，通过`c.Param("动态参数")`[动态参数指的是`:lang`这类参数]获取对应的值。

context.go

```go
type Context struct {
	// origin objects
	Writer http.ResponseWriter
	Req    *http.Request
	// request info
	Path   string
	Method string
    // 添加的参数，用于获取解析参数
	Params map[string]string
	// response info
	StatusCode int
}

// 新增的参数方法
func (c *Context) Param(key string) string {
	value, _ := c.Params[key]
	return value
}
```

router.go

修改handle

```go
func (r *router) handle(c *Context) {
	// 通过path查询树节点
    n, params := r.getRoute(c.Method, c.Path)
	// 如果节点存在
    if n != nil {
        // 给context的params赋值参数
		c.Params = params
		key := c.Method + "-" + n.pattern
        // 执行router 中 的handlers map映射的 handle
		r.handlers[key](c)
	} else {
		c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
	}
}
```

main.go

```go
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})

	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
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

---

在router的getRoute中，Gin框架有一个root.travel, Gee没有体现。这是可以拓展的地方。

建议这部分多看几遍，再多梳理一下。

有时间要多看看评论，评论提出的问题可能是没有注意到的问题。或者是一些优化思路。
