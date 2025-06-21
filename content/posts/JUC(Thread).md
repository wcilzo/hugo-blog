---
title: JUC(Thread)
date: 2025-06-14T23:50:37+08:00
lastmod: 2025-06-14T23:50:37+08:00
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
  - JUC
# nolastmod: true
draft: false 
---

### 深入理解并发、线程、与登台通知机制学习笔记

进程： 在操作系统角度，进程是程序运行资源分配(以内存为主)的最小单位。

线程：线程是CPU调度的最小单位。计算机中有很多的程序。但是CPU核心数量是有限的。计算机通过进程去运行程序。但是有限的进程数不足以驱动大量的程序运行。于是就有线程。线程是依赖于进程存在。是CPU调度和分配资源的基本单位。

线程独立运行。但是自己没有系统资源。只能与其他线程共享进程中的所有资源。 一个进程可以拥有多个线程， 一个线程必须有一个父进程。线程， 有时也被称为轻量级进程 (Lightweight Process ，LWP），早期 Linux 的线程实现几乎就是复用的进程，后来才独立出自己的 API。

> [!note]
>
> java 对象中 finalize 方法在销毁对象的时候，如果有也好回收的资源写入这个方法，又Finalizer线程执行回收。

#### 进程间的通讯方式：

同一台计算机进程之间的通信被称为IPC(Inter-process communication)， 不同计算机之间的进程通信被称为RPC (Remote process communication)。需要通过网络并遵守相关的协议。比如阿里的Dubbo就是一个RPC框架。HTTP协议也经常用在RPC上。比如SpringCloud 微服务。

1、管道(pipe): 分为匿名管道和命名管道。匿名管道可以用于父由父进程创建的子进程中父子间的通信。命名管道既可以用于父子进程通信，也可以用于其他进程之间的通信。一般来说取决于程序是否开放管道的通信方式。

2、信号(signal)：是软件层面上对中断机制的一种模拟，是一种比较复杂的通信方式。一个进程收到一个信号与处理器收到一个中断请求效果是一样的。

3、消息队列(message queue)：消息队列是消息的连接表，克服了上面两种方式中信号量有限的缺点。写权限的一方可以往消息队列中写消息，读权限的一方可以从消息队列中读取消息。

4、共享内存(shared memory)：可以说是最有用的进程间的通信方式。使得多个进程可以共同访问一块内存空间。不同进程可以看到对方进程中对共享内存中某些数据的更新。但是这种方式依赖于同步操作。比如互斥锁和信号量等。

5、信号量(semaphore)：作为进程之间以及同一进程的不同线程之间的同步和互斥手段。

6、套接字(socket)：常用于网络中不同机器的进程间通讯方式。同一台机器的进程中可以使用Unix domain socket （比如同一机器中 MySQL  中的控制台 mysql shell 和 MySQL 服 务程序的连接），这种方式不需要经过网络协议栈， 不需要打包拆包、计算校验 和、维护序号和应答等，比纯粹基于网络的进程间通信肯定效率更高。

> [!note] 
>
> sychronized和原子操作automic ，什么情况下使用sychronized更快？
>
> 在以下特定情况下，`synchronized`可能比`Atomic`更快：
>
> 1. **高竞争场景**：当多个线程频繁争用同一个原子变量时，CAS操作会频繁失败并重试，导致性能下降。而`synchronized`在这种情况下可能更高效。
> 2. **批量操作**：当需要执行多个原子操作作为一个整体时，使用`synchronized`块比多个单独的原子操作更高效。
> 3. **JVM优化**：现代JVM对`synchronized`进行了大量优化(如偏向锁、轻量级锁、锁消除等)，在低竞争情况下性能接近无锁。
> 4. **复杂操作**：对于需要多个变量同步更新的复杂操作，`synchronized`可能比尝试用多个原子变量更高效。
> 5. **长时间持有锁**：如果线程需要长时间持有锁(不是短暂操作)，`synchronized`可能更合适。

> [!warning]
>
> CPU核心数和线程数的关系
>
> CPU内核和线程是 1 : 1的对应关系。一个内核只能运行一个线程。Intel引入的超线程技术后，产生了逻辑处理器的概念。使得核心数和线程数形成了 1 : 2的关系。
>
> 在Java中，Runtime.getRuntime().availableProcessors(), 可以直接获取当前CPU的核心数，如果是Intel CPU获取的是逻辑处理器数量。并发编程中提升性能 跟 CPU核心数密切相关。

#### 上下文切换

CPU核心数是有限的，一个核心只能运行一个线程。计算机中很多程序在同时运行。每个程序都有自己的一些进程。计算机通过上下文切换使得这些进程正常运行。从而保证用户使用体验。

线程之间是如何进行上下文切换的呢？

上下文是CPU寄存器和程序计数器在任何时间片上运行的内容。主要是通过程序技术器记录当前进程在CPU中指令位置。也就是下一条要执行的指令地址。

从程序的角度看，上下文是将方法调用中各种局部变量与资源。

从线程角度看，上下文是方法调用栈中存储的各类信息。

引发上下文切换的原因一般包括：线程、进程切换、系统调用等。上下文切换通常是计算密集型的。一次上下文切换大概需要5000-20000时钟周期。一个简单的指令需要几个到十几个左右的执行始终周期，所以上下文切换的成本非常大。在程序中要尽量避免。

#### 并行和并发

并发Concurrent是指应用程序能交替执行不能的任务。并发利用线程进行任务切换执行。看起来好像是同时执行的，单实际上是由于计算机速度太快，多个线程切换执行达到类似于同时执行的效果。

并行Parallel 指应用程序在同时执行不同的任务。实际上就是同时执行的。

----

#### Java中的线程

Java程序是多线程程序。一个程序从Main方法开始执行。Main方法通常被成为主线程，同时存在一些守护线程一并执行。**当主线程结束时，守护线程一并结束。守护线程不会单独存在。可以说只要有守护线程就一定有主线程。但是有主线程不一定有守护线程。**

Java程序中，就算没用户自己开启的线程，也有一些**JVM**自启动的线程。一般来说有：

[6] Monitor Ctrl-Break 	// 监控 Ctrl-Break 中断信号的

[5] Attach Listener 	      // 内存 dump，线程 dump，类信息统计， 获取系统属性等 

[4] Signal Dispatcher    	// 分发处理发送给 JVM 信号的线程

[3] Finalizer    			// 调用对象 finalize 方法的线程

[2] Reference Handler	  // 清除 Reference 的线程

[1] main 				  // main 线程， 用户程序入口

尽管这些线程根据不同的 JDK 版本会有差异， 但是依然证明了 Java 程序天生就是多线程的。

##### Java中线程的启动与结束

启动线程的方式官方给出的有两种

1、X extend Thread 继承于线程类Thread，然后 X.start()

2、X implement Runnable 实现Runnable接口，交给Thread运行

```Java
public class NewThread {
	/*扩展自Thread类*/
	private static class UseThread extends Thread{
		@Override
		public void run() {
			super.run();
			SleepTools.second(1);
			// do my work;
			System.out.println("I am extendec Thread");
		}
	}

	
	/*实现Runnable接口*/
	private static class UseRunnable implements Runnable{

		@Override
		public void run() {
			// do my work;
			System.out.println("I am implements Runnable");

		}
	}
	

	public static void main(String[] args) 
			throws InterruptedException, ExecutionException {
		UseThread useThread = new UseThread();
		useThread.start();

		UseRunnable useRunnable = new UseRunnable();
		new Thread(useRunnable).start();
		System.out.println("main end");

		
	}
}
```

Thread 和 Runnable 区别

Thead是Java里面对线程的抽象，Runnable是对任务的抽象，Thread可以接受任何Runnale的实例执行。                                                                                                                                                                                                                              

再来看**另外一种实现方式 Callable、Future、FutureTask**

有些时候需要线程执行后返回一些数据。但是Runnable接口和Thread的Run方法都是void类型。

Callable也是一个接口，位于JUC包下面。其中有一个call方法返回一个泛型。类型是传递进来V类型。

但是Callable不能直接交给Thread去运行。并且，这里只是给出了返回类型，其他线程如何获取这返回值呢？

Future可以做到。Future这个接口中有get方法能够获取执行结果。但是这个方法会阻塞直到获取到这个返回结果。

但是Future依然不能直接交给Thread运行。FutureTask可以。FutureTask继承了一个RunnableFuture接口，而RunnableFuture接口继承了Runnable和Funture接口，这就使得Thread能够直接使用FutureTask类。因为FutureTask中有关于Runnable的实现。

```java
public class UseFuture {
	
	
	/*实现Callable接口，允许有返回值*/
	private static class UseCallable implements Callable<Integer>{
		private int sum;
		@Override
		public Integer call() throws Exception {
			System.out.println("Callable子线程开始计算！");  
//			Thread.sleep(1000);
	        for(int i=0 ;i<5000;i++){
	        	if(Thread.currentThread().isInterrupted()) {
					System.out.println("Callable子线程计算任务中断！");
					return null;
				}
	            sum=sum+i;
				System.out.println("sum="+sum);
	        }  
	        System.out.println("Callable子线程计算结束！结果为: "+sum);  
	        return sum; 
		}
	}
	
	public static void main(String[] args) 
			throws InterruptedException, ExecutionException {

		UseCallable useCallable = new UseCallable();
		//包装
		FutureTask<Integer> futureTask = new FutureTask<>(useCallable);
		Random r = new Random();
		new Thread(futureTask).start();

		Thread.sleep(1);
		if(r.nextInt(100)>50){
			System.out.println("Get UseCallable result = "+futureTask.get());
		}else{
			System.out.println("Cancel................. ");
			futureTask.cancel(true);
		}

	}

}
```

这种callable实现方式实际上还是属于Runnable的实现方式。将callable的对象通过FutureTask包装成Runnable，再交付给Thread去执行。看起来就和实现了Runnable差不多。

还有一种基于线程池的，这实际上是一种池化技术。比较偏向于资源的复用，本质上与这两种方式没什么区别。

- 中止：

线程自然运行结束或者抛出异常错误提前结束。

###### 为什么不建议用resume、suspend、stopAPI？

分别对应恢复、暂停、结束方法。这些API现在已过期。不建议使用。stop只结束线程，但是不是放资源。并不是有始有终的，很多时候我们的程序都希望无论是正常运行终止还是意外终止至少要返回一个结果。而suspend只通知CPU暂停，也不释放资源。占有着这部分资源进入睡眠状态。如果有其他线程需要调用这部分资源，就会进入死锁。

- 中断

安全的中止是通过其他线程调用**某个线程**A的interupt()方法，告诉线程A他应该中断了。但是这不代表A马上进行中断操作。A线程可能完全不理会这个中断请求。

线程中通过方法**isInterrupted()**方法判断是否中断，也可以调用静态方法**Thread.interrupted()**来判断线程是否被中断。不过该静态方法会将中断标识位置改为false。

如果一个线程处于阻塞状态(比如线程调用了thread.sleep、thread.join、thread.wait等方法)，则线程在检查中断标志时如果是true。则会在这些阻塞方法处抛出InterruptedException异常。**并在抛出异常后将中断标志位设置为false。**

##### 为什么不建议自定义属性进行中断？

已经有了中断标志位，就没有必要添加属性进行中断判断了。在执行sleep方法或者进入了阻塞队列中。如果是添加的属性，是无法判断中断的。此时线程暂停运行。必须等阻塞调用返回后才会检查这个标志。这样在某些情况下可能会对程序产生影响。而已经实现的中断标志位中，sleep方法等本身就支持中断检查，一旦发生中断，马上抛出异常。

> [!warning]
>
> **处于死锁状态下的线程无法中断。**能够检测中断标志，但是无法解除锁。



start方法和run方法

Thread是对线程这个概念的对象抽象。是一个对象。new Thread中只是在内存中创建了这么一个对象实例。跟操作系统中的线程没有直接联系，当调用方法start()的时候，才会跟操作系统的线程进行关联。达到真正意义上的启动线程。

​	Thread的源码中，start方法调用了start0方法，start0是一个native的本地方法，这个本地方法跟操作系统密切相关。start方法让线程进入就绪队列等待分配CPU，分到CPU后才调用实现run()方法。**star()方法不能重复调用，重复调用会抛出异常**。run方法是业务逻辑实现的方法。可以重复执行或者调用。



##### 线程的状态/生命周期

初始、运行、等待、超时等待、阻塞、终止。

1、初始(New)： 新建了一个线程，但是没有开始start方法。

2、运行(RUNNABLE)：Java线程中将就绪状态(Ready)和运行中(Running)状态笼统称为运行。

线程对象创建后，其他线程(如main线程)调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU使用权。此时位于就绪状态。就绪状态的线程获得了CPU时间片后变为运行中状态。

3、阻塞(BLOCKED): 表示线程阻塞于锁。

4、等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作(通知或中断)

5、超市等待(TIME_WAITING): 该状态不同于WAITING，它可以在指定时间后自行返回。

6、终止(TERMINATED)：表示该线程已经执行完毕。

![image15](../../static/articleImage/image15.png)

##### 其他线程的相关方法

yield()方法: 使得当前线程让出CPU占有权，让出时间不可设置。不会释放锁资源。同时执行yield的线程有可能让出CPU占有权进入就绪状态后被操作系统选中马上又执行。

> [!note]
>
> ConcurrentHashMap的initTable方法就用到了这个方法。在多线程情况下，如果有多个线程一同初始化这个map，这里能保证只有一个线程成功创建这个map。其他线程只能被阻塞为或者等待。为了避免这种操作引发的上下文切换等开销，干脆就让其他不进行初始化的线程执行yield()方法。让出CPU执行权。使得初始化线程更快的执行完成。

##### 线程的优先级

Java线程中通过成员priority变量控制优先级。优先级的范围为1-10.在线程new过程中可以通过setPriority(int)方法来修改优先级。默认优先级是5.优先级高的线程分配时间片数量多余优先级低的线程。

设置线程优先级：针对频繁阻塞(休眠或者I/O操作)的线程需要设置较高优先级。而偏重计算的（需要更多CPU时间或者偏运算)的线程则设置较低的优先级。确保处理器不会被独占。

在现在的JDK版本中，java线程都是1：1对应内核线程。而内核线程中有自己的优先级。所以一般来说调整线程的优先级作用不大。

##### 线程的调度

线程调度是指系统为线程分配CPU使用权的过程。主要调度方式有两种。一种是协同式线程调度(Cooperative Threads-Scheduling)，一种式抢占式线程调度(Preemptive Threads-Scheduling)。

协同式调度，线程执行时间由线程本身控制。线程自己的工作执行完后才会通知其他线程。使用协同式调度的最大好处是实现简单。线程就像是串行的。所以不会有同步问题。坏处也很明显，如果一个线程出现问题阻塞，后面的线程都不能运行。

抢占式调度，每个线程的执行时间由系统决定，线程的执行时间不可控，不会出现一个线程导致整个进程阻塞的问题。

> [!note] 
>
> Java线程调度是抢占式调度。

##### 线程和协程

任何语言实现线程主要有三种方式：内核线程(1:1), 用户线程(1:n), 用户线程和轻量级进程混合实现(n:m)

**内核线程**：由内核来完成线程切换，内核通过操纵调度器(Scheduler)对线程进行调度。并负责将线程的任务映射到各个处理器上。即使一个线程阻塞，也不会对程序造成影响，因为已经交给操作系统处理了。

因为是由系统进行线程之间的操作，如创建，同步等操作，都需要系统调用。而系统调用会进行线程间的上下文切换。【用户态和内核态直接来回切换。】代价非常高。而计算机中还有别的应用程序也用到了系统内核。内核并不是100%针对某一个应用程序的，因此系统支持的线程数是有限的。

**用户线程**: 严格意义上的用户线程是指完全建立在用户空间的线程库上的。系统内核不能感知到用户线程的存在和情况。用户线程的创建、同步、销毁和调度完全由用户编码实现，在用户态中完成，不需要内核帮助。理想状态下如果实现得当，用户线程消耗非常低。也能够支持更大规模的线程数量。比如说Go的协程。就是用户线程实现的。

用户线程的优势是非常快且低的资源消耗。劣势是需要用户自己实现线程的创建、销毁、切换、调度等操作。并且因为操作系统只会将线程进行分配，不会处理阻塞和线程映射到多个处理器等问题。

**混合实现**：用户线程完全建立在用户空间中，但是也有内核线程的调度和处理器映射。这种混合模式中，用户线程和轻量级进程的数量比是不定的。

> [!note] 
>
> Java线程实现
>
> 早期的Java线程 Jdk(1.2)之前是由用户线程实现。但是从1.3开始，主流商用的Java虚拟机线程模型改为了基于操作系统原生的内核线程实现。
>
> 但是由于近些年分布式、微服务、高并发的兴起，Java又开始支持用户线程的实现。但是目前底层依然是内核线程实现，在`Jdk21`提出了虚拟线程实现(混合实现[n:m])。

##### 协程

互联网架构服务在处理一次对外业务请求的响应，往往需要分不知不同机器上的大量服务共同协作来实现，也就是微服务。细分的微服务在减少单个服务复杂度，增加复用性的同时，也增加服务数量，压缩了请求响应时间。这就要求每一个服务必须在极短的时间内完成计算。这样每个请求的耗时就不会太长。也要求每一个服务要能处理更多的请求，保证不会出现某个服务被堵塞出现等待。

面对这样的要求，Java的基于内核线程的的并发机制就显得不太够用了。基于内核的每次业务线程切换开销，等同于计算本身的开销。同时由于Go语言等支持用户线程的语言能够很轻松的承载微服务，给Java带来了挑战。

协程又叫用户线程。最初用户线程被设计成协同式调度(Cooperative Scheduling)，所以叫协程(Gorouting)。完整的做调用栈保护，恢复工作，今天也被成为“有栈协程”。

协程的主要优势是轻量。无论有没有栈，都比线程轻量的多。并且需要在应用层面(调用栈、调度器等)实现多。同时拥有协同式调度的缺点。

因此，协程适用于被阻塞的，高并发(网络io)场景，不适合计算密集型场景。	

##### 纤程-java的协程

Loom项目在2018年被OPENJDK提出。并使用 纤程(Fiber)这个名字。Loom实际上是使得两个线程模型存在与java虚拟机中。可以在程序中同时使用。使用纤程跟使用线程的方法类似。没有多大改动。

目前Java比较出名的协程库是Quasar。Quasar的实现原理是字节码注入，在字节码的层面对当前的调用函数局部变量进行保存和恢复。这种方式虽然不依赖Java虚拟机，但是影响性能。

> [!warning]
>
> jdk19有了虚拟线程概念。后面再补充。

##### 守护线程

Daemon守护线程是一种支持性线程，他主要给程序中后台调度和支持性工作。当不存在任何一个非Daemon线程的时候，Java虚拟机会推出。可以用`Thread.setDaemon(true)`将线程设置为Daemon线程。**Daemon线程中的finally快在JVM退出时不一定会执行。**



##### 管道输入输出流

进程间有好几种通信方式，其中就有管道。Java中管道用于线程之间数据传输，传输媒介是内存。

Java 中的管道输入/输出流主要包括了如下 4 种具体实现：

PipedOutputStream 、PipedInputStream 、PipedReader 和 PipedWriter，前两种面 向字节，而后两种面向字符。

设想这么一个应用场景：通过 Java  应用生成文件， 然后需要将文件上传到 云端，比如：

页面点击导出后， 后台触发导出任务， 然后将 mysql中的数据根据导出条件查询出来， 生成 Excel 文件， 然后将文件上传到 oss，最后发步一个下载文件的链接。

##### Join方法

如何保证线程之间顺序执行呢？

现在有 T1、T2、T3 三个线程， 你怎样保证 T2 在 T1 执行完后执行， T3 在 T2 执行完后执行？

答：用 Thread#join 方法即可， 在 T3  中调用 T2.join，在 T2 中调用 T1.join。

把指定的线程加入到当前线程， 可以将两个交替执行的线程合并为顺序执行。 比如在线程 B 中调用了线程 A 的 Join()方法， 直到线程 A 执行完毕后， 才会继续执行线程 B 剩下的代码。

##### synchronized 内置锁

Java中synchronized 关键字支持多个线程访问同一个对象或者对象的成员变量。synchronized可以修饰方法或者以同步块的方式，允许多个线程访问资源。保证变量的可见性和排他性质，又叫做内置锁机制。

##### 对象锁和类锁

对象锁用于实例对象，或者实例方法。类锁是用于静态方法或者类的class对象上。

类锁：

```java
public static void main(String[] args) {
    test();
    System.out.println(count);
}
```

类锁锁每个类对应的class对象。每个类只有一个class对象。所以只有一个类锁。

对象锁:

```java
private synchronized void testNoStatic() {
    for (int i = 0; i < 10000; i++) {
        count ++;
    }
}

private void testSynTest() {
    synchronized (o) {
        for (int i = 0; i < 10000; i++) {
            count ++;
        }
    }
}
```

测试代码

```java
public class Test1 {

    private static int count;

    private Object o = new Object();

    private static synchronized void test() {
        for (int i = 0; i < 10000; i++) {
            count ++;
        }
    }

    private synchronized void testNoStatic() {
        for (int i = 0; i < 10000; i++) {
            count ++;
        }
    }

    private void testSynTest() {
        synchronized (o) {
            for (int i = 0; i < 10000; i++) {
                count ++;
            }
        }
    }

    private Runnable runTestForThread() {
        return new Runnable() {
            @Override
            public void run() {
//                test();
//                testNoStatic();
                testSynTest();
            }
        };
    }

    private void runThreads() {
        for (int i = 0; i < 5; i++) {
            new Thread(runTestForThread()).start();
        }
    }

    // 由静态方法调用必须是static
    private static class SynTestForThread extends Thread {
        private Test1 test1;

        public SynTestForThread(Test1 test1) {
            this.test1 = test1;
        }
        @Override
        public void run() {
//            test1.test();
//            test1.testNoStatic();
            test1.testSynTest();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Test1 test1 = new Test1();
//        test1.runThreads();
        // 简单的保证线程运行结束，当然用join应该会更好
//        Thread.sleep(50);

        // join的方式 为了方便，需要使用extend Thread 创建线程
        SynTestForThread thread1 = new SynTestForThread(test1);
        SynTestForThread thread2 = new SynTestForThread(test1);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();

        System.out.println(test1.count);
    }
}
```

> [!note]
>
> 锁只能用于同一个对象，哪怕一个是当前对象类锁一个是当前对象对象锁也不行。这种一样锁不住资源。

比如，程序可以这样改

```java
    public static void main(String[] args) throws InterruptedException {
        Test1 test1 = new Test1();
//        test1.runThreads();
        // 简单的保证线程运行结束，当然用join应该会更好
//        Thread.sleep(50);

        // join的方式 为了方便，需要使用extend Thread 创建线程
        SynTestForThread thread1 = new SynTestForThread(test1);
//        SynTestForThread thread2 = new SynTestForThread(test1);
//        thread1.start();
//        thread2.start();
//        thread1.join();
//        thread2.join();

        // 测试锁只能锁相同的类或者对象
        Thread thread3 = new Thread(() -> {
            test1.test();
        });
        thread3.start();
        thread1.start();
        Thread.sleep(50);

        System.out.println(test1.count);
    }
```

只需要改thread3中的实现方法，或者实现线程`SynTestForThread`的run方法。就能测试锁对象或者锁类等能不能锁住的问题。

这里就不进行测试了。可以试着跑跑看。

##### volatile 最轻量级的通信/同步机制

volatile要搞清楚是解决了可见性的问题，而不是锁。他不能锁住资源，只能保证资源能够立即被多个线程观测到被其中一个线程修改的值。

volatile不能保证多个线程的写安全，只能保证读安全。他比较适用于多读一写的程序。

```java
public class VltTest {

    private static volatile int count = 0;

    private static Boolean ready = false;

    private void testVlt() {
        System.out.println("test volatile is start");
        while(!ready) {
//            System.out.println("test count");
        }
        System.out.println("count = " + count);
    }

    private Runnable testVltRunnable = new Runnable() {
        @Override
        public void run() {
            testVlt();
        }
    };
    public static final void second(int seconds) {
        try {
            TimeUnit.SECONDS.sleep(seconds);
        } catch (InterruptedException e) {
        }
    }

    public static void main(String[] args) throws InterruptedException {
        VltTest vltTest = new VltTest();
        Thread thread = new Thread(vltTest.testVltRunnable);
        thread.start();
        second(1);
        count = 5;
        ready = true;
        second(5);
        System.out.println("count = " + count);
        System.out.println("main thread is over");
    }
}

```

ready被赋值为true后，线程一直在循环当中没有结束。如果想要看count是否被更新，取消while循环中的注释，再运行发现count = 5. 这可能不是一个很好的例子，可以仿照上面测试synchronized关键字写一个测试再看看。

##### 等待/通知机制

我们很多工作是需要相互配合完成的。但由于线程的无序性，有时候难以完成预期的目标。在前面介绍过join，join是一种方法，但有时候我们希望尽可能将一个任务分配到一个线程中工作。join只有在线程结束中才能达到效果。并且往往只是单向性的。所有如何让线程之间更高效的相互配合，完成某项工作呢？等待/通知机制是一个好方法。

wait、wait(long) 和 notify 、notifyAll方法在Java的Object对象里面。这几个方法最终都会调用本地native方法，也就是说实际上是由操作系统层面进行的等待通知。

wait 调用方法使线程进入waiting状态，，只有等待其他线程进行notify或者notifyAll唤醒或者中断才会放回。

> [!note] 
>
> 特别 注意的是，调用wait方法会释放对象锁。

wait(long) 带有时间的等待方法，调用该方法后进行timed-waiting状态，如果超过了设置的时间，重新获取锁，获取成功则回到原状态。这个时间参数是毫秒，等待长达n毫秒。没有通知就超时返回。

wait(long, int) 对于超时时间具有更细粒度的控制，达到纳秒级别

notify 通知一个都对象等待的线程，使其从wait或者wait(long)方法返回，前提是拿到了对象锁。没有获得锁的线程会重新进入waiting状态。

notifyAll 通知在一个对象上等待的所有线程，使他们从wait状态返回。

###### 等待通知的标准范式

等待方

1、获取对象锁

2、判断条件是否成立

3、如果条件不满足，调用wait方法，被通知后依然要检查条件

4、条件满足执行剩余逻辑。

```java
synchronized (对象) {
    while(条件不满足) {
        对象.wait();
    }
    // 执行剩余逻辑
}
```

通知方

1、获取对象锁

2、执行逻辑 改变条件

4、notify通知对应对象上的等待线程

```java
synchronized (对象) {
    // 执行处理逻辑改变条件
    对象.notify() or 对象.notifyAll();
}
```

> [!note]
>
> 在调用wait或者notify的相关方法之前，线程必须获取到对象锁。只能在同步方法或者同步方法块中调用wait方法或者notify的相关方法。

尽可能的用notifyAll方法，一般来说需要等待/通知场景的问题，都不是一个线程能解决的。通常代表着存在大量的等待通知线程。notify可能会导致唤醒的一个线程不是所需要的线程，并且可能导致大量线程处于等待状态影响体验。

```java
public class Express {
    public final static String DIST_CITY = "Shenzhen";

    public final static int TOTAL = 500;

    private int km;

    private String site;

    public Express () {
    }

    public Express (int km, String site) {
        this.km = km;
        this.site = site;
    }

    public void change () {
        if (km < TOTAL) {
            km += 100;
            System.out.println("the km is " + this.km);
        }
        if (km >= TOTAL) {
            site = DIST_CITY;
            System.out.println("the Express is arrived");
        }
    }

    public synchronized void waitKm() throws InterruptedException {
        while (this.km < TOTAL) {
            wait();
            System.out.println("Map thread[" +Thread.currentThread().getId()
                    +"] wake,I will change db");
        }
    }

    public synchronized void waitSite() throws InterruptedException {
        while (!this.site.equals(DIST_CITY)) {
            wait();
            System.out.println("Map thread[" +Thread.currentThread().getId()
                    +"] wake,I will change db");
        }
        System.out.println("the site is "+this.site+",I will call user");
    }
}

// ---------------------------------------------------------------------------

public class WnTest {
    private static Express express = new Express(0, "MAOMING");

    private static class CheckKm extends Thread {
        @Override
        public void run() {
            try {
                express.waitKm();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    private static class CheckSite extends Thread {
        @Override
        public void run() {
            try {
                express.waitSite();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 2; i ++) {
            new CheckSite().start();
        }

        for (int i = 0; i < 2; i++) {
            new CheckKm().start();
        }

        Thread.sleep(500);

        for(int i=0; i<5; i++){
            synchronized (express){
                express.change();
                express.notify();
            }
            Thread.sleep(500);
        }
    }
}
```

###### yield, sleep, wait, notify方法对锁的影响

yield、sleep调用后都不会释放当前线程持有的锁。

调用wait后，会释放锁。并且被唤醒后，会重新竞争锁，锁拿到后才会执行wait后面的操作。

调用notify方法后，不会马上释放锁，只有执行完synchronized方法块后才会释放锁。所以一般都是让notify方法放在synchronized的最后一行。

###### 为什么wait和notify方法要在同步块中调用[lost wake up]?

如果没有同步方法块，可能会丢失掉notify唤醒。

比如说

>生产者 [通知方]
>
>count ++;
>
>notify();
>
>消费者 [等待方]
>
>while (check) {
>
>wait();
>
>count --;
>
>}

生产者有可能在消费者还没有wait的时候就notify了，导致丢失了一次notify操作。

这就是所谓的lost wake up问题

所以我们通常通过锁来解决这个问题。让他们竞争同一把锁，只有拿到锁的才能改变count值，这样就不会丢失操作了。

###### 为什么要在循环中检查等待条件？

处于等待状态的线程可能会收到错误的警报和伪唤醒，如果不在循环中检查等待条件，那么程序就会在没有满足条件的情况下直接退出。因此，当一个线程被唤醒的时候，不认为他原来的等待条件依然是有效的，在notify调用后和等待线程的这段时间他有可能改变，所以依然要检查等待状态。
