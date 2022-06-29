
---
title: go反射
date: 2022-06-29
categories: go
tags: [go, reflect]
description: 反射

---

> concreteCat,_ := reflect.ValueOf(cat).Interface().(Cat)
see http://golang.org/doc/articles/laws_of_reflection.html fox example

```golang
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```