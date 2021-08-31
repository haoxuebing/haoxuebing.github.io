---
title: sync包之WaitGroup
date: 2021-08-30
categories: go
tags: [go, sync, WaitGroup]
description: GO语言并发编程-sync包之WaitGroup的使用
---

## 概述
> 正常情况下 goroutine的结束过程是不可控制的，唯一可以保证终止goroutine的行为是main goroutine的终止。也就是说，我们并不知道哪个goroutine什么时候结束。
>通过WaitGroup提供的三个函数：Add,Done,Wait，可以轻松实现等待某个协程或协程组完成的同步操作

## WatiGroup
WatiGroup是sync包中的一个struct类型，用来收集需要等待执行完成的goroutine
``` golang
// A WaitGroup waits for a collection of goroutines to finish.
// The main goroutine calls Add to set the number of
// goroutines to wait for. Then each of the goroutines
// runs and calls Done when finished. At the same time,
// Wait can be used to block until all goroutines have finished.
//
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}

```
WaitGroup结构一旦定义就不能复制

## 主要函数
### func (wg *WaitGroup) Add(delta int) 
Add方法增加WaitGroup计数器的值，注意Add加上正数的调用应在Wait之前，delta可以是负数；如果内部计数器变为0，Wait方法阻塞等待的所有协程都会释放，如果计数器小于0，则调用panic

### func (wg *WaitGroup) Done()
每次需要等待的goroutine在真正完成之前，应该调用该方法来人为表示goroutine完成了，该方法会对等待计数器减1

### func (wg *WaitGroup) Wait() 
Wait方法阻塞直到WaitGroup计数器减为0

## 用例 
### 利用 defer 在协程完成前减一
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


### waitgroup 指针类型做函数参数为
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

### 官方
``` golang
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
执行结果：
```
http://www.baiyuxiong.com/
http://www.google.com/
http://www.golang.org/
over
```

## 总结
- Add()用来增加要等待的goroutine的数量，Done()用来表示goroutine已经完成了，减少一次计数器，Wait()用来等待所有需要等待的goroutine完成
- Add的数量和Done的调用数量必须相等，否则会报出死锁的错误信息
- WaitGroup结构一旦定义就不能复制，所以函数中要传递WaitGroup的指针
- WaitGroup在需要等待多个任务结束再返回的业务来说还是很有用的，但现实中用的更多的可能是，先等待一个协程组，若所有协程组都正确完成，则一直等到所有协程组结束；若其中有一个协程发生错误，则告诉协程组的其他协程，全部停止运行（本次任务失败）以免浪费系统资源。该场景WaitGroup是无法实现的，需要借助通知机制或 channel 来实现