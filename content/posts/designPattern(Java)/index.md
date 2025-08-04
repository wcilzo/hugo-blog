---
title: 常见设计模式(JAVA)
date: 2025-08-04T22:59:17+08:00
lastmod: 2025-08-04T22:59:17+08:00
author: xiestr
avatar: ../../img/avatar.jpg
# authorlink: https://author.site
# cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - 设计模式
tags:
  - Java
  - 设计模式
# nolastmod: true
draft: false
---

持续更新

### 策略模式

个人理解：定义一组可完成相同功能的不同算法（策略），通过接口和依赖注入的方式进行解耦。这样策略能够动态的切换，相对来说是比较简单常见的设计模式。

**核心思想**：定义一系列算法（策略），将每个算法封装起来，并使它们可以互相替换。策略模式让算法的变化独立于使用算法的客户端。

#### 结构

一个接口 interface xx

多个可实现接口的类 class implements xx

需要该行为的类(业务类)将接口作为成员。

```java
public interface Strategy {
    void doStrategy();
}

public class StrategyA implements Strategy {
    @Override
    public void doStrategy() {
        //xxxx
    }
}

public class StrategyB implements Strategy {
    @Override
    public void doStrategy() {
        //xxxx
    }
}

public class StrategyC implements Strategy {
    @Override
    public void doStrategy() {
        //xxxx
    }
}

public class EntityX {
    private Strategy strategy;
    
	EntityX (Strategy strategy) {
        this.strategy = strategy;
    }
    public void doSomething() {
        strategy.doStrategy();
    }
}
```

通过这种方式，不必关注Strategy的具体实现类是哪一个，直接调用接口的方法，在运行时程序会帮我们执行对应实现类的方法。
