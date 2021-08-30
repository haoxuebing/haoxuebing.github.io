---
title: sync包之WaitGroup
date: 2021-08-30
categories: go
tags: [go, sync, WaitGroup]
description: GO语言并发编程-sync包之WaitGroup的使用
---

# 概述
> WaitGroup 用于等待一组协程执行完毕
## 主要函数
### func (wg *WaitGroup) Add(delta int) 
Add方法增加WaitGroup计数器的值，注意Add加上正数的调用应在Wait之前，delta可以是负数；如果内部计数器变为0，Wait方法阻塞等待的所有协程都会释放，如果计数器小于0，则调用panic

### func (wg *WaitGroup) Done()
每次需要等待的goroutine在真正完成之前，应该调用该方法来人为表示goroutine完成了，该方法会对等待计数器减1

### func (wg *WaitGroup) Wait() 
Wait方法阻塞直到WaitGroup计数器减为0

# 实战
## 利用 defer wg.Done()
``` golang
func main() {
	var wg sync.WaitGroup
	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("1 goroutine sleep ...")
		time.Sleep(2e9)
		fmt.Println("1 goroutine exit ...")
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()
		fmt.Println("2 goroutine sleep ...")
		time.Sleep(4e9)
		fmt.Println("2 goroutine exit ...")
	}()

	fmt.Println("waiting for all goroutine ")
	wg.Wait()
	fmt.Println("All goroutines finished!")
}

```
执行结果：

```
waiting for all goroutine 
2 goroutine sleep ...
1 goroutine sleep ...
1 goroutine exit ...
2 goroutine exit ...
All goroutines finished!
```


## waitgroup 指针类型做函数参数为
``` golang
func main() {
	sayHello := func(wg *sync.WaitGroup, id int) {
		defer wg.Done()
		fmt.Printf("%v goroutine start ...\n", id)
		time.Sleep(2 * time.Second)
		fmt.Printf("%v goroutine exit ...\n", id)
	}

	var wg sync.WaitGroup
	const N = 5
	wg.Add(N)
	for i := 0; i < N; i++ {
		go sayHello(&wg, i)
	}

	fmt.Println("waiting for all goroutine ")
	wg.Wait()
	fmt.Println("All goroutines finished!")
}
```
执行结果：
```
waiting for all goroutine 
4 goroutine start ...
3 goroutine start ...
1 goroutine start ...
2 goroutine start ...
0 goroutine start ...
1 goroutine exit ...
4 goroutine exit ...
0 goroutine exit ...
2 goroutine exit ...
3 goroutine exit ...
All goroutines finished!
```
##  google 官方代码
``` golang
package main

import (
    "fmt"
    "sync"
    "net/http"
)

func main() {
    var wg sync.WaitGroup
    var urls = []string{
            "http://www.golang.org/",
            "http://www.google.com/",
            "http://www.baiyuxiong.com/",
    }
    for _, url := range urls {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine to fetch the URL.
        go func(url string) {
            // Decrement the counter when the goroutine completes.
            defer wg.Done()
            // Fetch the URL.
            http.Get(url)
        fmt.Println(url);
        }(url)
    }
    // Wait for all HTTP fetches to complete.
    wg.Wait()
    fmt.Println("over");
}
```
代码执行结果：
```
http://www.baiyuxiong.com/
http://www.google.com/
http://www.golang.org/
over
```

# 总结
- Add()用来增加要等待的goroutine的数量，Done()用来表示goroutine已经完成了，减少一次计数器，Wait()用来等待所有需要等待的goroutine完成
- Add的数量和Done的调用数量必须相等，否则会报出死锁的错误信息
- WaitGroup结构一旦定义就不能复制，所以函数中要传递WaitGroup的指针
- 若其中有一个协程发生错误，则告诉协程组的其他协程，全部停止运行（本次任务失败）以免浪费系统资源。该场景WaitGroup是无法实现的，需要借助通知机制或 channel 来实现