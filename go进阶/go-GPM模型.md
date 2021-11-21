---
title: go-GPM模型
date: 2021-11-19
categories: go
tags: [go, gpm]
description: 从进程/线程/协程到golang的GPM模型，以及其调度策略与生命周期
---

## 进程/线程/协程

说到GPM，进程、线程、协程是绕不开的话题。web服务器经历了从进程处理请求模型（代表作php），再到用线程处理请求模型（代表作Java、.Net），后来出现风靡一时的Node.js用“事件”的方式处理请求模型，现如今golang带起来的协程，请求处理模型一代比一代先进，把服务器资源充分的利用更在有意义的事情上，使服务性能得到一次又一次提升。  

**进程**：进程是系统进行资源分配的基本单位，有独立的内存空间，进程的管理模块为PCB。

**线程**：线程是 CPU 调度和分派的基本单位，线程依附于进程存在，每个线程会共享父进程的资源，线程的管理模块为TCB。

CPU在上下文切换时，PCB消耗的资源远大于TCB，所以诞生了用线程处理请求的模型，绕开了PCB。 一个线程的最小占用内存是2M，在使用线程模型并发处理请求时，CPU大量的时间片会消耗在TCB上。想要提高服务器并发处理的性能，就要绕开TCB。

Node.js的思路是在单进程上利用`事件驱动`+`非阻塞IO`来处理请求，这种模型中上下文的切换是在单线程下完成的，也就没有了CPU时间片在PCB和TCB上的消耗。缺陷是难以利用CPU多核，加上JavaScript的历史包袱，最终也只是风靡一时。

**协程**：是一种用户态的轻量级线程，最小占用内存只有2kb，协程的调度完全由用户控制，没有内核的开销，减少了TCB对CPU时间片的开销。

虽然其他语言（python和php-swoole的co-routine）也逐渐引入了协程库的概念，但在协程的调度上，golang是在语言级别实现，用更少的CPU指令完成了协程的调度，这个调度也就是我们现在常叫的G-P-M模型。

## go调度器历史
go的调度器模型也是经历了多次迭代才有现在优异性能

- 单线程调度器 0.x 
  - 线程和协程的关系 1:N
  - 跟node.js相似，只有一个活跃线程，一旦某协程阻塞，造成线程阻塞
- 多线程调度器 1.0
  - 线程和协程的关系 M:N
  - 多个活跃线程M，但还是一个全局的goroutine队列，M每次获取G都需要加锁，竞争严重，性能差
- 任务窃取调度器 1.1
  - 引入了处理器 P，构成了目前的 G-M-P 模型
  - 在处理器 P 的基础上实现了基于工作窃取的调度器  
  - 不支持抢占式调度，程序只能依靠 Goroutine 主动让出 CPU 资源才能触发调度 
  - 时间过长的垃圾回收（Stop-the-world，STW）会导致程序长时间无法工作
- 抢占式调度器 1.2 ~ 至今
  - 基于协作的抢占式调度器 - 1.2 ~ 1.13
    - 通过编译器在函数调用时插入抢占检查指令，在函数调用时检查当前 Goroutine 是否发起了抢占请求，实现基于协作的抢占式调度；
    - Goroutine 可能会因为垃圾回收和循环长时间占用资源导致程序暂停
  - 基于信号的抢占式调度器 - 1.14 ~ 至今
    - 实现基于信号的真抢占式调度
    - 垃圾回收在扫描栈时会触发抢占调度
    - 抢占的时间点不够多，还不能覆盖全部的边缘情况

## GPM模型
- GPM含义：
    - G: goroutine协程，它是一个待执行的任务
    - P: processor处理器，它可以被看做运行在线程上的本地调度器
    - M: thread操作系统的线程，它由操作系统的调度器调度和管理

![GPM调度器](../images/gpm调度器.png)

1. **全局队列（Global Queue）**：存放等待运行的G
2. **P的本地队列**：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列
3. **P列表**：所有的P都在程序启动时创建，并保存在数组中，最多有GOMAXPROCS(可配置)个
4. **M**：Go语言本身是限定M的最大量是10000，runtime/debug包中的SetMaxThreads函数来设置。如果有一个M阻塞，会创建一个新的M。如果有M空闲，那么就会回收和睡眠

### schedule 循环调度
线程M想运行任务就得获取G，调度器启动之后，Go 语言运行时会调用 runtime.mstart 以及 runtime.mstart1，前者会初始化 g0 的 stackguard0 和 stackguard1 字段，后者会初始化线程并调用 [runtime.schedule](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/proc.go#L2977)入调度循环  

1. 首先从P的本地队列获取G
2. 每调度`61`次就会主动去全局队列拿g，避免全局队列饥饿
3. 本地P队列为空时，M也会尝试从全局队列拿一批G放到P的本地队列
4. 如果全局队列为空，就会从其他P的本地队列偷一半放到自己P的本地队列
5. M运行G，G执行之后，M会从P获取下一个G，不断重复下去

```golang
func schedule() {
	_g_ := getg()

top:
	var gp *g
	var inheritTime bool

	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
		gp, inheritTime = findrunnable()
	}

	execute(gp, inheritTime)
}
```
由 runtime.execute 执行获取的 Goroutine，做好准备工作后，它会通过 runtime.gogo 将 Goroutine 调度到当前线程上，当 Goroutine 中运行的函数返回时，程序会跳转到 runtime.goexit 所在位置执行该函数
```golang
TEXT runtime·goexit(SB),NOSPLIT,$0-0
	CALL	runtime·goexit1(SB)

func goexit1() {
	mcall(goexit0)
}

func goexit0(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gdead)
	gp.m = nil
	...
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	dropg()
	gfput(_g_.m.p.ptr(), gp)
	schedule()
}
```
在最后 runtime.goexit0 会重新调用 runtime.schedule 触发新一轮的 Goroutine 调度，Go 语言中的运行时调度循环会从 runtime.schedule 开始，最终又回到 runtime.schedule，我们可以认为调度循环永远都不会返回

### 调度器的设计策略  
- **复用线程** 避免频繁的创建、销毁线程，对线程复用
  - work stealing机制
    - 当本线程无可执行的G时，尝试从全局队列获取，全局队列获取不到就去其他线程绑定的P偷取G，而不是销毁线程
  - hand off 机制
    - 当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行
- **利用并行**
  - GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在多个CPU上同时运行
- **抢占**
  - 在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死
- **全局G队列**
  - 从全局G队列获取G，此时会涉及到加锁解锁，全局队列没有就会执行work stealing 从其他P队列偷G

### "go func()"经历了什么过程
1. 我们通过 go func()来创建一个goroutine；
2. 有两个存储G的队列，一个是局部调度器P的本地队列、一个是全局G队列。新创建的G会先
保存在P的本地队列中，如果P的本地队列已经满了就会保存在全局的队列中
3. G只能运行在M中，一个M必须持有一个P，M与P是1：1的关系。M会从P的本地列列弹出一个可执行状态的G来执行，如果P的本地队列为空，就会想其他的MP组合偷取一个可执行的G来执行 
4. 一个M调度G执行的过程是一个循环机制 
5. 当M执行某一个G时候如果发生了syscall或则其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)来服务于这个P
6. 当M系统调用结束时候，这个G会尝试获取一个空闲的P执行，并放入到这个P的本地队列。如果获取不到P，那么这个线程M变成休眠状态， 加入到空闲线程中，然后这个G会被放入全局队列中。

### 调度器的生命周期

![调度器的生命周期](../images/gpm调度器的生命周期.png)

- M0
  - 启动程序后的编号为0的主线程
  - 在全局变量 runtime.m0中 ，不需要在heap上分配
  - 负责执行初始化操作和启动第一个G
  - 启动第一个G之后，m0就和其他的M一样了
- G0
  - 每次启动一个M,都会第一个创建的goroutine，就是G0
  - G0近用于负责调度其他的G
  - G0不指向任何可执行的函数
  - 每个M都会有一个自己的G0
  - 在调度或系统调用时会使用M切换到G0，来调度其他的G
  - M0的G0会放在全局空间


## 参考文献
[Golang的协程调度器原理及GMP设计思想](https://www.kancloud.cn/aceld/golang/1958305)  
[并发编程-调度器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)