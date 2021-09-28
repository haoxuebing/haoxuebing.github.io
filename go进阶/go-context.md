---
title: go context包
date: 2021-09-18
categories: go
tags: [go, context]
description: Context 几乎成为了并发控制和超时控制的标准做法
---

### Context介绍
>Context 的主要作用还是在多个 Goroutine 或者模块之间同步取消信号或者截止日期，用于减少对资源的消耗和长时间占用，避免资源浪费

Context 包主要就是下面四个方法
```
// 手动调用 cancel 
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 设置一个截止时间
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
// 设置一个运行时间
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
// WithValue 函数能够将请求作用域的数据与 Context 对象建立关系
func WithValue(parent Context, key, val interface{}) Context

```
###  WithCancel 
WithCancel 返回带有新 Done 通道的父节点的副本，当调用返回的 cancel 函数或当关闭父上下文的 Done 通道时，将关闭返回上下文的 Done 通道，无论先发生什么情况。

取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用 cancel
```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx, "【监控1】")
	go watch(ctx, "【监控2】")
	go watch(ctx, "【监控3】")
	time.Sleep(6 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	time.Sleep(1 * time.Second)
	fmt.Println("再等5秒退出进程...")
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}
func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name, "监控退出，停止了...")
			return
		default:
			fmt.Println(name, "goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```
运行结果
```
【监控3】 goroutine监控中...
【监控2】 goroutine监控中...
【监控1】 goroutine监控中...
【监控2】 goroutine监控中...
【监控3】 goroutine监控中...
【监控1】 goroutine监控中...
【监控1】 goroutine监控中...
【监控3】 goroutine监控中...
【监控2】 goroutine监控中...
可以了，通知监控停止
【监控2】 监控退出，停止了...
【监控1】 监控退出，停止了...
【监控3】 监控退出，停止了...
再等5秒退出进程...
```


### WithDeadline
WithDeadline 函数会返回父上下文的副本，并将 deadline 调整为不迟于 d。如果父上下文的 deadline 已经早于 d，则 WithDeadline(parent, d) 在语义上等同于父上下文。当截止日过期时，当调用返回的 cancel 函数时，或者当父上下文的 Done 通道关闭时，返回上下文的 Done 通道将被关闭，以最先发生的情况为准。

取消此上下文将释放与其关联的资源，因此代码应该在此上下文中运行的操作完成后立即调用 cancel
```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	d := time.Now().Add(50 * time.Millisecond)
	ctx, cancel := context.WithDeadline(context.Background(), d)
	// 尽管ctx会过期，但在任何情况下调用它的cancel函数都是很好的实践。
	// 如果不这样做，可能会使上下文及其父类存活的时间超过必要的时间。
	defer cancel()
	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())
	}
}
```
运行结果
```
context deadline exceeded
```

### WithTimeout
WithTimeout 函数返回 WithDeadline(parent, time.Now().Add(timeout))

取消此上下文将释放与其相关的资源，因此代码应该在此上下文中运行的操作完成后立即调用 cancel
```golang
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	// 传递带有超时的上下文
	// 告诉阻塞函数在超时结束后应该放弃其工作。
	ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
	defer cancel()
	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err()) // 终端输出"context deadline exceeded"
	}
}
```
运行结果
```
context deadline exceeded
```

### WithValue
WithValue 函数接收 context 并返回派生的 context，其中值 val 与 key 关联，并通过 context 树与 context 一起传递。这意味着一旦获得带有值的 context，从中派生的任何 context 都会获得此值。不建议使用 context 值传递关键参数，函数应接收签名中的那些值，使其显式化。
```golang
package main

import (
	"context"
	"fmt"
)

func main() {
	type favContextKey string // 定义一个key类型
	// f:一个从上下文中根据key取value的函数
	f := func(ctx context.Context, k favContextKey) {
		if v := ctx.Value(k); v != nil {
			fmt.Println("found value:", v)
			return
		}
		fmt.Println("key not found:", k)
	}
	k := favContextKey("language")
	// 创建一个携带key为k，value为"Go"的上下文
	ctx := context.WithValue(context.Background(), k, "Go")
	f(ctx, k)
	f(ctx, favContextKey("color"))
}
```
运行结果
```
found value: Go
key not found: color
```

### Background()
context的四个方法的第一个参数都是 context.Background()  
因为background 本质上是 emptyCtx 结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的 Context  
Background() 主要用于 main 函数、初始化以及测试代码中，作为 Context 这个树结构的最顶层的 Context，也就是根 Context  

## 参考文献
[Go语言Context（上下文）](http://c.biancheng.net/view/5714.html)  