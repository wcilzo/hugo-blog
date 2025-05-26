---
title: Java类加载机制笔记
date: 2025-03-30T14:21:12+08:00
lastmod: 2025-03-30T14:21:12+08:00
author: XieStr
avatar: ../../img/avatar.jpg
# authorlink: https://author.site
# cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - Java
tags:
  - Java
  - JVM
# nolastmod: true
draft: false
---

### Java类加载运行过程

当我们需要启动某个自己编写的类的时候，要编写非常经典的public static void main方法块。整个文件或者项目通过该main方法编译运行。那么是类是如何加载进内存的呢？

流程大致为![img](https://note.youdao.com/yws/public/resource/35faf7c95e69943cdbff4642fcfd5318/xmlnote/959BC07ABE464141BB096AD41BC3708A/102280)

乍一看好像确实有点复杂，不过我们慢慢来，先将整个图抽象成2块。

左边由windows系统启动jvm.dll和创建引导类加载器实例实际上是spot完成的。这部分代码由C++编写。

右边粉色箭头开始直到红色 JVM销毁是由JAVA代码完成的。

loadClass的类加载过程为如下几步：

**加载->验证->准备->解析->初始化**->使用->卸载

>  [!note]
>
>  **加载**:将硬盘中的**字节码**文件(.class)进行读取。
>
>  **验证**:验证字节码文件的正确性。字节码文件格式是固定的。
>
>  **准备**:将类的静态变量分配内存，并给上默认值。这里的默认值不是指代码中给予的默认值。
>
>  **解析**:将**符号引用**替换为直接引用，该阶段会把一些静态方法(符号引用，比如main()方法)替换为指向数据所存内存的指针或句柄等(直接引用)，这是所谓的**静态链接**过程(类加载期间完成)，**动态链接**是在程序运行期间完成的将符号引用替换为直接引用。
>
>  >  [!Tip]
>  >
>  > 静态链接：就是将代码中的方法(main等自定义命名)以及类型(Integer)等内容抽象成符号，这部分符号又直接映射到内存表中，解析过程是将这些符号变成内存表中存储key，方便直接从内存表中获取value。
>  >
>  > 动态链接：方法中由抽象接口，多态等不能确定的类类型。只有在执行阶段才能确定的类型。
>
>  **初始化**：对类的静态变量给予代码中指定的值，并执行静态代码块。
>
>  ![https://note.youdao.com/yws/public/resource/35faf7c95e69943cdbff4642fcfd5318/xmlnote/8ED5F11F99ED4C29BBBFCBB9C874F20A/102279](https://note.youdao.com/yws/public/resource/35faf7c95e69943cdbff4642fcfd5318/xmlnote/8ED5F11F99ED4C29BBBFCBB9C874F20A/102279)

类被加载到方法区中后主要包含 **运行时常量池、类型信息、字段信息、方法信息、类加载器的引用、对应class实例的引用**等信息。

**类加载器的引用**：这个类到类加载器实例的引用

**对应class实例的引用**：类加载器在加载类信息放到方法区中后，会创建一个对应的Class 类型的对象实例放到堆(Heap)中, 作为开发人员访问方法区中类定义的入口和切入点。

> [!Warning] 
>
> 主类在运行过程中如果使用到其它类，会逐步加载这些类。
>
> jar包或war包里的类不是一次性全部加载的，是使用到时才加载。

### 类加载器初始化过程

一共有4种类加载器:

引导类加载器 ：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库。

拓展类加载器 ExtClassLoader()：负责加载支撑JVM运行的位于JRE的lib目录下ext拓展目录的jar包。该加载器父节点为null。因为引导类加载器是C++代码生成的。

应用类加载器 AppClassLoader()：负责加载用户自己编写的java类包。该加载器的父节点是拓展类加载器。依赖于拓展类加载器。

自定义加载器：负责加载用户自定义路径下的类包。该加载器的父节点是应用类加载器。依赖于应用类加载器。

JVM默认使用Launcher的getClassLoader()方法返回的类加载器AppClassLoader的实例加载我们的应用程序。

... 未完待续
