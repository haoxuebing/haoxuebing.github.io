---
title: go判断两个变量是否相等
date: 2021-09-07
categories: go
tags: [go, reflect]
description: 判断变量是否相等是编程过程很常见的操作
---

## 前言
判断结构体是否为空不能用nil
reflect.DeepEqual本质上是递归,使用反射类型测试深度相等性，可以比较任意两个go语言变量是否相等


```golang
type Person struct {
	Id   int
	Name string
}

func main() {
	p := Person{}

	fmt.Println(p == (Person{}))                //true
	fmt.Println(reflect.DeepEqual(p, Person{})) //true

	ptr := &Person{}
	fmt.Println(ptr == nil) //false

	m1 := map[string]interface{}{"a": "1", "b": 2, "c": 3}
	m2 := map[string]interface{}{"a": "1", "c": 3, "b": 2}
	fmt.Println(reflect.DeepEqual(m1, m2)) //true

	arr1 := []int{1, 2, 3, 4, 5}
	arr2 := []int{5, 4, 3, 2, 1}
	fmt.Println(reflect.DeepEqual(arr1, arr2)) //false
	arr3 := []int{1, 2, 3, 4, 5}
	fmt.Println(reflect.DeepEqual(arr1, arr3)) //true
}
```

## 疑问
> reflect.DeepEqual源码中有个packEface方法很是神奇，变量e是emptyInterface指针类型，对e的属性赋值后，变量i居然获取到了外部变量的属性值(eg:p.Id,p.Name的值)，功力不够目前没明白

```golang

// emptyInterface is the header for an interface{} value.
type emptyInterface struct {
	typ  *rtype
	word unsafe.Pointer
}
// Value is the reflection interface to a Go value.
type Value struct {
	typ *rtype
	ptr unsafe.Pointer
	flag
}


// packEface converts v to the empty interface.
func packEface(v Value) interface{} {
	t := v.typ
	var i interface{}
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// First, fill in the data portion of the interface.
	switch {
	case ifaceIndir(t):
		if v.flag&flagIndir == 0 {
			panic("bad indir")
		}
		// Value is indirect, and so is the interface we're making.
		ptr := v.ptr
		if v.flag&flagAddr != 0 {
			// TODO: pass safe boolean from valueInterface so
			// we don't need to copy if safe==true?
			c := unsafe_New(t)
			typedmemmove(t, c, ptr)
			ptr = c
		}
		e.word = ptr
	case v.flag&flagIndir != 0:
		// Value is indirect, but interface is direct. We need
		// to load the data at v.ptr into the interface data word.
		e.word = *(*unsafe.Pointer)(v.ptr)
	default:
		// Value is direct, and so is the interface.
		e.word = v.ptr
	}
	// Now, fill in the type portion. We're very careful here not
	// to have any operation between the e.word and e.typ assignments
	// that would let the garbage collector observe the partially-built
	// interface value.
	e.typ = t
	return i
}
```