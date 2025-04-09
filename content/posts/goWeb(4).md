---
title: Gee微框架学习记录(第四天)
date: 2025-04-07T10:50:30+08:00
lastmod: 2025-04-07T10:50:30+08:00
author: xiestr
avatar: /img/author.jpg
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

对7天系列中Gee框架学习的记录。推荐和Gee第四天一起食用。

> [!NOTE]
>
> 前面完成了动态路由的相关配置，Gee第四天主要讨论分组路由实现，第五天的中间件处理需要用到，所以这一节要理解。

**1. 什么是分组路由？**

想象你管理一家超市：

- **没分组**：就像把零食、日用品、生鲜全混在一起，想给所有零食打折时，得一个个商品去贴标签，累死人还容易漏。
- **分组后**：把商品分到不同货架（零食区、日用品区）。想给零食打折？直接在零食区挂个"全场8折"的牌子，简单又不会搞错。

在动态路由系统中，当需要**批量管理路由行为**（如中间件应用、路径前缀统一调整）时，通过个体路由白名单维护会导致：

- **修改成本高**：变更中间件需逐个路由调整
- **一致性风险**：易遗漏部分路由配置

- **可维护性差**：路由与业务逻辑关联不直观



**分组嵌套路由**

指的是路由分组可以包含子分组，也就是分组再分组。形成**多级路由前缀**和**中间件继承链**。每个子分组自动继承父分组的所有属性（路径前缀、中间件），并允许扩展自己的特定配置。

那么在这里应该如何实现呢？

为了能够匹配路由前缀和继承父分组的路由前缀、中间件。通过父分组很容易想到需要一个类似树结构的数据结构。在前面的动态路由中，额外定义了一个Trie树并交给Router进行管理。这里我们也可以用类似的操作，定义一个新的结构体Group，给Engine进行管理。

Group对象应该包含前缀prefix，用于查找父节点的parent，以及中间件middlewares(下一节提到)。为了能够访问路由节点，需要加上Engine。通过Engine，Group就能够访问router。

**为什么rotuer不写在Group中？**

1、如果写在Group中

存在一些重复的前缀 -> 创建不同的路由，而且路径可能会产生冲突。在进行一些特殊路由的匹配，比如说两个

```text
Tree1 (group1):
/api/v1/users

Tree2 (group2):
/admin/users
```

当请求一个与这两个树无关的路由的时候，比如说`/lang`，需要遍历所有树。这显然是不合理的。

又或者是  `/admin/users` 和 `/api/admin/users`， 容易引发冲突。

总结为：**避免重复前缀和冲突**

2、如果写在Engine中

通过继承Group的属性，就可以避免相同前缀重复创建和冲突的问题。相同的前缀只会创建一棵树。这样整个查询效率就大大提升了。

总结为：**提升性能**

3、为了保持**设计一致性**

Engine在设计之初就是为了管理所有跟网络相关的功能，包括各种拓展的中间件和路由等内容。

> [!Warning]
>
> 到这里可能还有疑问，为啥只会创建一颗树呢？
>
> 我们前面不是给每个方法`(GET、POST)`都创建了一颗树吗？
>
> 实际上是**没有问题**的，确实实际上`GET` 和 `POST`方法都创建了一棵树。所有跟方法相关的路由都会合并到同一个方法数。
>
> 比如说 `/api/v1/users` 和 `/v2/users2`，假设这两个都是`GET`方法。
> 那么他们都会合并到同一颗`GET树`中。

---

### Gee实现

Engine 和 RouterGroup(Group)

server.go 

```Go
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

Gee实现Group和Engine的方式实际上跟上面讨论的差不多。

> [!note]
>
> 为什么不是反过来RouterGroup继承Engine呢？
>
> RouterGroup 是牢大， engine继承他，从而获得他的方法，而不是反过来。反过来的话，
> RouterGroup 会增加一些方法，比如 run，但是这个方法，在Engine中并不需要。

gee.go

```go
// New is the constructor of gee.Engine
// 变更new Engine 方法
func New() *Engine {
	engine := &Engine{router: newRouter()}
    // 初始化参数，将所有的Group指向同一个Engine
	engine.RouterGroup = &RouterGroup{engine: engine}
    // 初始化参数，Groups为RouterGroup类型的数组
	engine.groups = []*RouterGroup{engine.RouterGroup}
	return engine
}

// Group is defined to create a new RouterGroup
// remember all groups share the same Engine instance
// 创建分组
func (group *RouterGroup) Group(prefix string) *RouterGroup {
	// 取出Group的engine
    engine := group.engine
    // 创建新的Group
	newGroup := &RouterGroup{
        // 前缀连接 当前Group中的前缀加上需要加入Group的前缀
		prefix: group.prefix + prefix,
        // parent 为当前Group
		parent: group,
        // 所有Group指向同一个Engine
		engine: engine,
	}
    // 将创建的Group加入到Engine的Groups中，这里就彻底创建结束了
	engine.groups = append(engine.groups, newGroup)
    // 返回创建的Group
	return newGroup
}

// addRoute方法由Engine转化为RouterGroup
// 分组处理添加Route
func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
	// 分组前缀 + 当前请求pattern
    pattern := group.prefix + comp
    log.Printf("Route %4s - %s", method, pattern)
	// 还是用原来的router中的addRoute加入到路由树
    group.engine.router.addRoute(method, pattern, handler)
}

// GET defines the method to add GET request
// GET 方法改用Group
func (group *RouterGroup) GET(pattern string, handler HandlerFunc) {
	group.addRoute("GET", pattern, handler)
}

// POST defines the method to add POST request
// POST 方法改用Group
func (group *RouterGroup) POST(pattern string, handler HandlerFunc) {
	group.addRoute("POST", pattern, handler)
}
```

> [!note]
>
> 为什么这里addRoute 用 RouterGroup 而不是 Engine 
>
> - 职责不同：
>   首先要明确各自的职责，RouterGroup 设计初就是为了分组
>   Engine是全局引擎， 负责协同路由、中间件 和 前缀树
>
> - 避免方法冗余：如果不这样设计，每次在调用如 GET 方法的时候，要显示传递分组前缀，这种做法是不合理的。
>
>   ```go
>   engine.addRoute("GET", "/api/users", handler)  // 需要手动拼接前缀
>   ```
>
>   而
>
>   ```go
>   apiGroup := engine.Group("/api")
>   apiGroup.GET("/users", handler)  // 自动处理前缀拼接
>   ```
>
> - 支持中间件绑定
>   中间件是路由级别的功能，应该与路由绑定
>
> **实际上，我觉得最重要的是职责不同出发更容易理解。**
>
> 如果将将prefix 写到engine 里面 会怎么样？
> - 无法实现最初的目的，分组隔离。
>   所有的路由都将公用一个分组。
>   而在RouterGroup中，每个分组各自维护自己的prefix 和 middlewares
>
> - 由 1) 导致中间件管理混乱，所有路由全局中间件。
>   所有中间件不能针对特定路由组开启或者关闭。
>
> - 代码维护性下降
>   需要同时维护路由匹配、前缀拼接、中间件执行等逻辑，违反单一职责原则
>
>   | 架构对比   | RouterGroup vs 直接写 Engine     |                                 |
>   | ---------- | -------------------------------- | ------------------------------- |
>   | 特性       | RouterGroup 设计                 | prefix/middlewares 写 Engine 中 |
>   | 分组隔离   | ✅ 每个分组独立配置               | ❌ 全局共享状态                  |
>   | 中间件控制 | ✅ 支持按分组绑定中间件           | ❌ 所有路由强制全局中间件        |
>   | 嵌套分组   | ✅ 支持无限层级嵌套               | ❌ 难以实现                      |
>   | 代码职责   | ✅ Engine 和 RouterGroup 各司其职 | ❌ Engine 臃肿，承担过多职责     |
>   | 扩展性     | ✅ 轻松添加新分组功能             | ❌ 修改会影响所有路由            |
>
> 总结
>         将 prefix 和 middlewares 放到 Engine 中的危害本质是：
>         破坏了分组架构的核心优势（隔离性、嵌套性、中间件灵活性）。
>         RouterGroup 的设计是 Web 框架经过多年验证的最佳实践，看似多了一层抽象，实则为后续扩展留足了空间。

建议这部分多看几遍，再多梳理一下。

有时间要多看看评论，评论提出的问题可能是没有注意到的问题。或者是一些优化思路。

> [!Tip]
>
> 这一篇记录可能比较粗糙，分成了好几天来写，有一些点可能不是特别清楚。后续抽时间再维护看看。
