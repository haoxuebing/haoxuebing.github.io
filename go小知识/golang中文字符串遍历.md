---
title: golang中文字符串遍历
date: 2023-05-06
categories: golang
tags: [小知识]
description: golang中文字符串遍历乱码的情况
---

## 问题
golang中文字符串在使用下标遍历时会出现乱码的情况

## 解决
- 方法1：使用 for...range
- 方法2：将字符串转为[]rune类型

```golang
package main

import (
	"fmt"
)

func main() {
	str := "hi,你好"
	fmt.Println(len(str))         // 9
	fmt.Println(len([]rune(str))) // 5

	// 包含中文使用下标遍历会乱码
	for i := 0; i < len(str); i++ {
		fmt.Printf("str[%d]=%v\n", i, string(str[i]))
	}

	// 方法1： for...range遍历
	for i, v := range str {
		fmt.Printf("str[%d]=%v\n", i, string(v))
	}

	// 方法2: 转为[]rune类型，再下标遍历
	strRune := []rune(str)
	for i := 0; i < len(strRune); i++ {
		fmt.Printf("strRune[%d]=%v\n", i, string(strRune[i]))
	}

}
```
