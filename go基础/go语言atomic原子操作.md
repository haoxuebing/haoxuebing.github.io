---
title: go语言atomic原子操作
date: 2021-10-11
categories: go
tags: [go, atomic]
description: go中CAS操作可以有效的减少使用锁所带来的开销，但是需要注意在高并发下这是使用cpu资源做交换的
---

## 原子操作有啥用

先来看段代码

```golang
import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var count int32
	fmt.Println("main start...")
	var wg sync.WaitGroup
	for i := 0; i < 100000; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			fmt.Println("goroutine:", i, "start...")
			atomic.AddInt32(&count, 1)
			// count = count + 1

			fmt.Println("goroutine:", i, "count:", count, "end...")
		}(i)
	}
	wg.Wait()
	fmt.Println("count:", count)
	fmt.Println("main end...")
}
```
#### 使用 `count++`的运行结果:
```golang
goroutine: 58034 count: 93274 end...
goroutine: 62190 count: 82272 end...
goroutine: 63531 start...
goroutine: 63531 count: 99975 end...
goroutine: 19539 start...
goroutine: 19539 count: 99976 end...
goroutine: 55809 count: 99807 end...
goroutine: 68441 count: 98248 end...
goroutine: 43291 count: 99963 end...
count: 99976
main end...
```
最终发现`count`始终增加不到100000，协程数量越多count与协程数量的差值越大
#### 使用原子操作的运行结果：
```golang
goroutine: 56406 count: 97893 end...
goroutine: 49946 count: 97674 end...
goroutine: 76579 start...
goroutine: 76579 count: 99997 end...
goroutine: 83385 start...
goroutine: 83385 count: 99998 end...
goroutine: 89151 start...
goroutine: 89151 count: 99999 end...
goroutine: 56843 start...
goroutine: 56843 count: 100000 end...
count: 100000
main end...
```
最终我们能看到`count`增加到了100000

原子操作是在执行中不能被中断的操作，通常由CPU芯片级能力来保证，并由操作系统提供调用，golang基于操作系统的能力，提供了基于原子操作的支持

## atomic介绍
sync/atomic包提供了原子操作的能力，直接有底层CPU硬件支持，因而一般要比基于操作系统API的锁方式效率高些；这些功能需要非常小心才能正确使用。 除特殊的底层应用程序外，同步更适合使用channel或sync包的功能。 通过消息共享内存; 不要通过共享内存进行通信

### sync/atomic 的数据类型和原子操作 
- 数据类型共6个，包括int32、int64、uint32、uint64、uintptr和unsafe.Pointer    
- 提供的原子操作共有5种，即：增或减(Add)、比较并交换(CompareAndSwap)、载入(Load)、存储(Store)和交换(Swap)
- 原子操作函数的第一个参数都是传变量地址的，因为原子函数需要改变变量的值

### atomic 与 sync.Mutex
- sync.Mutex的底层也是用sync/atomic实现的
- atomic 无死锁问题，但限制多

### CAS 与 乐观锁

乐观锁，数据库里的乐观锁并不是真的使用了锁的机制，而是一种程序的实现思路。查数据的时候记录下该数据的版本号`version`，如果成功修改的话，会修改该数据的版本号，如果修改的时候版本号和查询的时候版本号不一致，则认为数据已经被修改过，会重新尝试查询再次操作。

CAS（compare and swap），先比对，然后再进行交换，和数据库里的乐观锁的做法很相似

代码如下：

```golang
for {
    user := getUserByName(A)
    version := user.version
    paied()
    if atomic.CompareAndSwapInt32(&user.version, version, version + 1) {
        user.money -= 10
    } else {
        rollback()
    }
}
```

### API
底层是汇编实现的，目录在 `src/sync/atomic`
```golang
// SwapInt32 atomically stores new into *addr and returns the previous *addr value.
func SwapInt32(addr *int32, new int32) (old int32)

// SwapInt64 atomically stores new into *addr and returns the previous *addr value.
func SwapInt64(addr *int64, new int64) (old int64)

// SwapUint32 atomically stores new into *addr and returns the previous *addr value.
func SwapUint32(addr *uint32, new uint32) (old uint32)

// SwapUint64 atomically stores new into *addr and returns the previous *addr value.
func SwapUint64(addr *uint64, new uint64) (old uint64)

// SwapUintptr atomically stores new into *addr and returns the previous *addr value.
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)

// SwapPointer atomically stores new into *addr and returns the previous *addr value.
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)

// CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

// CompareAndSwapInt64 executes the compare-and-swap operation for an int64 value.
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

// CompareAndSwapUint32 executes the compare-and-swap operation for a uint32 value.
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

// CompareAndSwapUint64 executes the compare-and-swap operation for a uint64 value.
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)

// CompareAndSwapUintptr executes the compare-and-swap operation for a uintptr value.
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)

// CompareAndSwapPointer executes the compare-and-swap operation for a unsafe.Pointer value.
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)

// AddInt32 atomically adds delta to *addr and returns the new value.
func AddInt32(addr *int32, delta int32) (new int32)

// AddUint32 atomically adds delta to *addr and returns the new value.
// To subtract a signed positive constant value c from x, do AddUint32(&x, ^uint32(c-1)).
// In particular, to decrement x, do AddUint32(&x, ^uint32(0)).
func AddUint32(addr *uint32, delta uint32) (new uint32)

// AddInt64 atomically adds delta to *addr and returns the new value.
func AddInt64(addr *int64, delta int64) (new int64)

// AddUint64 atomically adds delta to *addr and returns the new value.
// To subtract a signed positive constant value c from x, do AddUint64(&x, ^uint64(c-1)).
// In particular, to decrement x, do AddUint64(&x, ^uint64(0)).
func AddUint64(addr *uint64, delta uint64) (new uint64)

// AddUintptr atomically adds delta to *addr and returns the new value.
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)

// LoadInt32 atomically loads *addr.
func LoadInt32(addr *int32) (val int32)

// LoadInt64 atomically loads *addr.
func LoadInt64(addr *int64) (val int64)

// LoadUint32 atomically loads *addr.
func LoadUint32(addr *uint32) (val uint32)

// LoadUint64 atomically loads *addr.
func LoadUint64(addr *uint64) (val uint64)

// LoadUintptr atomically loads *addr.
func LoadUintptr(addr *uintptr) (val uintptr)

// LoadPointer atomically loads *addr.
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

// StoreInt32 atomically stores val into *addr.
func StoreInt32(addr *int32, val int32)

// StoreInt64 atomically stores val into *addr.
func StoreInt64(addr *int64, val int64)

// StoreUint32 atomically stores val into *addr.
func StoreUint32(addr *uint32, val uint32)

// StoreUint64 atomically stores val into *addr.
func StoreUint64(addr *uint64, val uint64)

// StoreUintptr atomically stores val into *addr.
func StoreUintptr(addr *uintptr, val uintptr)

// StorePointer atomically stores val into *addr.
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
```

## 参考文献
[Go并发编程之美-CAS操作](https://zhuanlan.zhihu.com/p/56733484)  
[Golang atomic原子操作](https://turbock79.cn/?p=3708)
[如何做到资源并发安全](https://blog.sbb.fun/blog/Go/SafeComplicate.html)