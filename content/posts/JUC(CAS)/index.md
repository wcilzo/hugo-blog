---
title: CAS和Atomic以及简论线程安全
date: 2025-06-24T22:22:53+08:00
lastmod: 2025-06-24T22:22:53+08:00
author: xiestr
avatar: ../../img/avatar.jpg
# authorlink: https://author.site
# cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - Java
tags:
  - Java
  - JUC
# nolastmod: true
draft: false
---

### CAS

CAS 全程 compare and swap，对比和交换。在了解CAS之前，得明白什么是原子操作。

> [!tip] 
>
> 什么是原子操作？
>
> 应该有相当一部分人说不清楚这个概念。原子性是数据库的四大特性，相信大家都不陌生。一个事务包含多个操作，要么全都做，要么全都不做。这就是原子性。原子操作也是同理的，如果说一个方法里面又有查询，插入又有修改，并且他们都被事务包裹，那么就满足了全部做或者不做。那么就称这个方法具有原子性。
>
> 简单的说：如果一个方法的所有操作要么全都做，要么全都不做，那么就实现了原子性。
>
> 原子操作是不可分割的单个操作，在CPU层面保证执行过程不被中断。是一种底层实现机制。可以结合原子性理解。
>
> 如何实现原子操作呢？
>
> 答案是加锁。锁确保了一个线程/进程只能在同一时间访问这部分资源，其他线程无法操作。要么全部完成，要么完全不执行。

synchronized关键字是基于阻塞的锁机制，但是这把锁中涉及到的CPU指令集有点多，如果只是修改一个值而不是一个对象，并且这个值可能被许多线程修改。那么这里的资源消耗是非常大的。

为了解决这种问题，Java引入了Atomic系列的原子操作类。这些操作都是基于CPU底层提供的CAS操作指令实现的。

CAS操作包含三个运算符: 一个内存地址V，一个期望值A，一个新值B。操作的时候如果这个地址V上的值等于旧值A，那么将他直接修改为新值B。否则不做任何操作。

CAS的基本思路是：如果这个值和期望值A相同，那么赋值为新值A，返回旧值B。当有大量的线程对一个内存地址进行操作时，那么自然就只能有一个线程可以完成。其他的不能完成怎么办呢？这里就有一个自旋操作。不能成功修改的进行自旋，获取旧值计算后再尝试修改。直到所有线程完成为止。

![image-20250624231149161](./images/image-20250624231149161.png)

Java中的Automic就是用了循环CAS完成。

#### CAS的三大问题

- ABA问题 

CAS中是，比较地址V中的值，如果等于旧值B，那么可以替换为A。如果说出现一个线程1想把B改为C的线程。但是有另外的线程把B先改为D，又把D改为B了，这个动作很快。在B被改为D之前呢，线程 1 先比较了B，准备改为C，但是被先一步改为D，又改回B了。这时候如果线程1去替换成C，那么这个值就有可能不对了。这种使用CAS进行检查发现没有变化，但是实际变化了的情况。叫做ABA问题。

可通过版本字段控制

在改变量的同时一并改同一条记录的版本号。比如一开始是B，版本号是1，在改为C的时候版本号 + 1，就是C 和 版本号2，再改为B，版本号 + 1，就是B 版本号 3. 过程是 (B 版本号1 -> D 版本号2 -> B版本号3)。这样就可以知道，原来实际上这个值B已经变动过一次了。

- 自旋

多线程情况下，其他线程太多导致等待次数剧增。Java中Automic的CAS操作是循环CAS。如果说有大量的线程比如 10000个，要同时改一个地址V的旧值B。第一个线程修改成功了，改为了A。那么第二个改变的线程得自旋1次才能改成C，第三个改变的线程得自旋2次才能改为D。以此类推，当第10000个线程改动的时候，就得自旋9999次。这对CPU资源消耗是巨大的。

> [!note]
>
> 这里第几个线程改动不是指按顺序的改动，因为每个线程的执行时间和调度顺序可能不一样。这里只是特指，在第n个线程改动时，会经历 n - 1次自旋。

- 一个变量的原子操作

CAS一般只能修改一个变量，在上面的介绍中。了解到CAS只能对一个地址V进行比较更改。如果说多个CAS操作呢？也不能保证操作的原子性。这时候只能用锁。或者是将多个共享变量合并为一个共享变量进行操作。JDK提供了AtomicReference类实现。可以将多个变量放在一个对象中进行CAS操作。

### Automic









### 线程安全相关
