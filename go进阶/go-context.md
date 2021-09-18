---
title: go context包
date: 2021-09-18
categories: go
tags: [go, context]
description: context"上下文"，准确说它是 Goroutine 的上下文，包含 Goroutine 的运行状态、环境、现场等信息
---

## 简介
Context 在 Go1.7 之后就加入到了Go语言标准库中

Context 也叫作“上下文”，是一个比较抽象的概念，一般理解为程序单元的一个运行状态、现场、快照。其中上下是指存在上下层的传递，上会把内容传递给下，程序单元则指的是 Goroutine。

每个 Goroutine 在执行之前，都要先知道程序当前的执行状态，通常将这些执行状态封装在一个 Context 变量中，传递给要执行的 Goroutine 中。

在网络编程下，当接收到一个网络请求 Request，在处理 Request 时，我们可能需要开启不同的 Goroutine 来获取数据与逻辑处理，即一个请求 Request，会在多个 Goroutine 中处理。而这些 Goroutine 可能需要共享 Request 的一些信息，同时当 Request 被取消或者超时的时候，所有从这个 Request 创建的所有 Goroutine 也应该被结束。

随着 Context 包的引入，标准库中很多接口因此加上了 Context 参数，例如 database/sql 包，Context 几乎成为了并发控制和超时控制的标准做法。

### 一个简单的 context 用例

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
	time.Sleep(10 * time.Second)
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

## 参考文献
[Go语言Context（上下文）](http://c.biancheng.net/view/5714.html)  