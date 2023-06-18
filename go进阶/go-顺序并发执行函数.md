---
title: 顺序并发执行函数
date: 2023-06-18
categories: go
tags: [go, channel]
description: 使用channel、select 实现协程间的通信
---


## 问题描述

> 有三个函数，分别打印"cat", "fish","dog"要求每一个函数都用一个goroutine，按照顺序打印100次


### 思路

- 使用channel、select 实现协程间的通信
- 使用有序的channel来保证函数的执行顺序
- 使用了sync.WaitGroup来等待所有协程执行完成
- 抽离出一个调度器来控制要并发执行的函数和执行次数

## 代码

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	schedule(100, cat, fish, dog)
}

func schedule(cnt int, fnList ...func()) {
	fnCount := len(fnList)
	chList := make([]chan int, fnCount+1)
	for i := 0; i < len(chList); i++ {
		chList[i] = make(chan int)
	}

	wg := sync.WaitGroup{}

	for i := 0; i < fnCount; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			for {
				_, isOpen := <-chList[i]
				if !isOpen {
					close(chList[i+1])
					return
				}
				fnList[i]()
				chList[i+1] <- 0
			}
		}(i)
	}

	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < cnt; i++ {
			fmt.Printf("----第%d次----\n", i+1)
			chList[0] <- 0
			<-chList[fnCount]
			if i == cnt-1 {
				close(chList[0])
			}
		}
	}()
	wg.Wait()
}

func cat() {
	fmt.Println("cat")
}

func fish() {
	fmt.Println("fish")
}

func dog() {
	fmt.Println("dog")
}

```