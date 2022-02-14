---
title: unsafe.Pointer的使用
date: 2022-02-14
categories: go
tags: [go, 指针]
description: 类似C语言中的void*类型的指针，它可以包含任意类型变量的地址
---

## unsafe.Pointer 是什么

一个普通的`*T`类型指针可以被转化为unsafe.Pointer类型指针，并且一个unsafe.Pointer类型指针也可以被转回普通的指针，被转回普通的指针类型并不需要和原始的`*T`类型相同

### 1.用于类型转化

例如,把float64类型转化为unint64类型
```
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}
```

### 2.指针运算
再指针的基础上,通过操作指针地址实现成员变量的访问
```
func main() {
    data := []byte("abcd")
    for i := 0; i < len(data); i++ {
        ptr := unsafe.Pointer(uintptr(unsafe.Pointer(&data[0])) + uintptr(i)*unsafe.Sizeof(data[0])) 
        fmt.Printf("%c,", *(*byte)(unsafe.Pointer(ptr)))
    }
    fmt.Printf("\n")
}
```
结果：
```
a,b,c,d,
```

### 3.读写结构内部成员
string类型的定义
```
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

reflect.StringHeader 的定义
```
type StringHeader struct {
    Data uintptr
    Len  int
}
```
unsafe.Pointer与uintptr在内存结构上是相同的，可以相互转换


```
func main() {
    str1 := "hello world"
    hdr1 := (*reflect.StringHeader)(unsafe.Pointer(&str1)) // 注1
    fmt.Printf("str:%s, data addr:%d, len:%d\n", str1, hdr1.Data, hdr1.Len)

    str2 := "abc"
    hdr2 := (*reflect.StringHeader)(unsafe.Pointer(&str2))

    hdr1.Data = hdr2.Data // 注2
    hdr1.Len = hdr2.Len   // 注3
    fmt.Printf("str:%s, data addr:%d, len:%d\n", str1, hdr1.Data, hdr1.Len)
}
```

运行结果
```
str:hello world, data addr:15595488, len:11
str:abc, data addr:15591112, len:3
```

- 注1：该行代码是把str1转化为unsafe.Pointer后，再把unsafe.Pointer转换来StringHeader的指针，然后通过读写hdr1的成员即可读写str1成员的值
- 注2：通过修改hdr1的Data的值，修改str1的字节数组的指向
- 注3：为了保证字符串的结果是完整的，通过修改hdr1的Len的值，修改str1的长度字段

最后，str1的值，已经被修改成了str2的值，即"abc"。


### 总结

- 其他类型的指针只能转化为unsafe.Pointer，unsafe.Pointer也才能转化成任意类型的指针
- 可以通过uintptr可以进行加减操作，从而实现指针的运算
- unsafe.Pointer与uintptr可以实现相互转换
