---
title: go context包
date: 2023-06-18
categories: go
tags: [go, context]
description: Context 几乎成为了并发控制和超时控制的标准做法
---

## 场景描述

> A方法中调用B方法和C方法，当A再次被调用时，如果上次的A方法还没执行完，则取消还在执行的A方法，并重新开始执行


### 思路

- 使用context.Context的context.WithCancel来实现取消操作
- 使用了互斥锁sync.Mutex来确保在方法A执行期间不会同时执行多个
- 使用time.Sleep用于模拟耗时任务
- 使用了sync.WaitGroup来等待所有协程执行完成
- 在函数内部，使用select语句监听ctx.Done()通道，当接收到取消信号时，立即返回函数


## 代码
```
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func main() {
	c := NewControl()
	go c.A()
	go c.A()
	time.Sleep(2500 * time.Millisecond)
	go c.A()
	time.Sleep(5 * time.Second)
}

type Control struct {
	Cancel context.CancelFunc
	mu     sync.Locker
}

func NewControl() *Control {
	return &Control{
		Cancel: nil,
		mu:     &sync.Mutex{},
	}
}

func (c *Control) A() {
	c.mu.Lock()
	if c.Cancel != nil {
		c.Cancel()
	}

	ctx, cancel := context.WithCancel(context.Background())
	c.Cancel = cancel
	c.mu.Unlock()

	wg := sync.WaitGroup{}

	wg.Add(3)
	go func() {
		defer wg.Done()
		c.B(ctx)
	}()
	go func() {
		defer wg.Done()
		c.C(ctx)
	}()

	go func() {
		defer wg.Done()
		select {
		case <-time.After(3 * time.Second):
			fmt.Println("Func A Done.")
		case <-ctx.Done():
			fmt.Println("Func A Cancel.")
		}
	}()

	wg.Wait()
}

func (c *Control) B(ctx context.Context) {
	select {
	case <-ctx.Done():
		fmt.Println("Func B Cancel.")
	case <-time.After(2 * time.Second):
		fmt.Println("Func B Done.")
	}
}

func (c *Control) C(ctx context.Context) {
	select {
	case <-ctx.Done():
		fmt.Println("Func C Cancel.")
	case <-time.After(3 * time.Second):
		fmt.Println("Func C Done.")
	}
}

```

## 输出