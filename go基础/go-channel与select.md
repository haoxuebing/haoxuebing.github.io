---
title: channel与select
date: 2021-09-07
categories: go
tags: [go, channel]
description: 通道（channel）是用来传递数据的一个数据结构
---

## channel 简介
>channel用于goroutines之间的通信，让它们之间可以进行数据交换。像管道一样，一个goroutine_A向channel_A中放数据，另一个goroutine_B从channel_A取数据。

channel是指针类型数据，初始化时使用`make`, `chan TYPE`为channel类型，所以在使用channel作为参数或者返回值时，以int类型channel为例：`paramName chan int`

`<-`代表channel的方向。如果没有指定方向，那么Channel就是双向的，既可以接收数据，也可以发送数据

```golang
chan T          // 可以接收和发送类型为 T 的数据
chan<- float64  // 只可以用来发送 float64 类型的数据
<-chan int      // 只可以用来接收 int 类型的数据

// 声明一个不带缓冲区的channel
ch := make(chan int) 

// 声明一个带缓冲区的channel
ch := make(chan int, 100)

// 把 v 发送到通道 ch
ch <- v    

// 从 ch 接收数据,并把值赋给 v
v := <-ch 

// 关闭通道
close(ch)
```


### 无缓冲区通道例子

使用两个 goroutine 来计算数字之和，在 goroutine 完成计算后，它会计算两个结果的和
```golang
package main

import "fmt"

func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 把 sum 发送到通道 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从通道 c 中接收

	fmt.Println(x, y, x+y)
}

```


### 使用缓冲区通道例子

斐波那契数列
```golang
package main

import (
	"fmt"
)

func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
	close(c)
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	// range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
	// 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
	// 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
	// 会结束，从而在接收第 11 个数据的时候就阻塞了。
	for i := range c {
		fmt.Println(i)
	}
}
```

## select 简介

select语句选择一组可能的send操作和receive操作去处理，类似switch,但是只是用来处理通讯操作

```golang
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}
func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

## 超时
timer是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间，它提供一个Channel，在将来的那个时间那个Channel提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行。
```golang
timer1 := time.NewTimer(time.Second * 2)
<-timer1.C
fmt.Println("Timer 1 expired")
```
当然如果你只是想单纯的等待的话，可以使用time.Sleep来实现。

你还可以使用timer.Stop来停止计时器。
```golang
timer2 := time.NewTimer(time.Second)
go func() {
    <-timer2.C
    fmt.Println("Timer 2 expired")
}()
stop2 := timer2.Stop()
if stop2 {
    fmt.Println("Timer 2 stopped")
}
```
## 同步 
channel可以用在goroutine之间的同步

下面的例子中main goroutine通过done channel等待worker完成任务。 worker做完任务后只需往channel发送一个数据就可以通知main goroutine任务完成。
```golang
import (
    "fmt"
    "time"
)
func worker(done chan bool) {
    time.Sleep(time.Second)
    // 通知任务已完成
    done <- true
}
func main() {
    done := make(chan bool, 1)
    go worker(done)
    // 等待任务完成
    
```

## 参考文献
[Go Channel 详解](https://www.runoob.com/w3cnote/go-channel-intro.html)  