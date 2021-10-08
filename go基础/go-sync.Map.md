---
title: go语言sync.Map
date: 2021-10-08
categories: go
tags: [go, sync, map]
description: Go语言中的 map 在并发情况下，只读是线程安全的，同时读写是线程不安全的,
---

### sync.Map 简介
> Go语言在 1.9 版本中提供了一种效率较高的并发安全的 sync.Map，sync.Map 和 map 不同，不是以语言原生形态提供，而是在 sync 包下的特殊结构
#### sync.Map特点
- 无须初始化，直接声明使用
- sync.Map 有自己的存储、读取和删除方式， Store 表示存储，Load 表示获取，Delete 表示删除
- 使用 Range 配合一个回调函数进行遍历操作，通过回调函数返回内部遍历出来的值
- sync.Map 为了保证并发安全有一些性能损失，因此在非并发情况下，使用 map 相比使用 sync.Map 会有更好的性能

### API
```golang
// 存储时key和value都是interface类型
func (m *Map) Store(key, value interface{})

// 读取时 ok 为true表示这个key的value存在
func (m *Map) Load(key interface{}) (value interface{}, ok bool)

// 删除一个key
func (m *Map) Delete(key interface{})

// 遍历sync.Map中的键值对
func (m *Map) Range(f func(key, value interface{}) bool)

// 如果当前key的值存在就返回当前key的值，否则设置key的值，如果当前key的值存在loaded为true，否者为false 
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)
```

### 用例
```golang
import (
	"fmt"
	"sync"
)

func main() {
	var scene sync.Map
	// 将键值对保存到sync.Map
	scene.Store("greece", 97)
	scene.Store("london", 100)
	scene.Store("bobo", "hehe")
	scene.Store("egypt", true)
	// 从sync.Map中根据键取值
	fmt.Println(scene.Load("london"))
	// 根据键删除对应的键值对
	scene.Delete("london")
	// 遍历所有sync.Map中的键值对
	scene.Range(func(k, v interface{}) bool {
		fmt.Println("iterate:", k, v)
		return true
	})
}
```
输出：
```
100 true
iterate: egypt true
iterate: greece 97
iterate: bobo hehe
```

### 并发下的map
并发情况下读写 map 时会出现问题
```golang
func main() {
    // 创建一个int到int的映射
    m := make(map[int]int)
    // 开启一段并发代码
    go func() {
        // 不停地对map进行写入
        for {
            m[1] = 1
        }
    }()
    // 开启一段并发代码
    go func() {
        // 不停地对map进行读取
        for {
            _ = m[1]
        }
    }()
    // 无限循环, 让并发程序在后台执行
    for {
    }
}
```
输出:
```
fatal error: concurrent map read and map write
```

### 使用map+锁 

```golang
import (
	"fmt"
	"sync"
	"time"
)

var (
	myMap  = make(map[string]interface{})
	mylock = new(sync.RWMutex)
)

func main() {
	val1 := mySyncMap("key1")
	fmt.Println(val1)
	fmt.Println(myMap)
	time.Sleep(time.Second * 61)
	fmt.Println(myMap)
}

func mySyncMap(keys string) interface{} {

	if myMap[keys] != nil {
		return myMap[keys]
	}
    // 在需要写map的时候加锁
	mylock.Lock()
	defer mylock.Unlock()
	//防止击穿
	if myMap[keys] != nil {
		return myMap[keys]
	}

	resp := "do something get value..."
	myMap[keys] = resp

	go func() {
		time.Sleep(time.Second * 60) // 缓存1min
		// tt := time.After(time.Second * 60) 
		// <-tt
		myMap = make(map[string]interface{})
	}()

	return resp
}
```
输出:
```
do something get value...
map[key1:do something get value...]
map[]
```