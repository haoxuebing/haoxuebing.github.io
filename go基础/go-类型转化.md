---
title: go类型转化
date: 2021-08-31
categories: go
tags: [go, 类型转化]
description: 强类型语言编程必然面临各种类型之间相互转化 
---


## []byte和string相互转换

```golang
func main() {
	str := "hello"
	data := []byte(str)
	fmt.Println(data) // [104 101 108 108 111]
	str = string(data)
	fmt.Println(str)  //  hello
}
```

## string和int互转
```golang
func main() {
	str := "123"
	inum, _ := strconv.Atoi(str)
	snum := strconv.Itoa(inum)
	fmt.Printf("inum: %T, snum: %T", inum, snum) //inum: int, snum: string
}
```

## string和int64互转
```golang
func main() {
	str := "123"
	inum, _ := strconv.ParseInt(str, 10, 64)
	snum := strconv.FormatInt(inum, 10)
	fmt.Printf("inum: %T, snum: %T", inum, snum) //inum: int64, snum: string
}
```

## int和int64互转
```golang
func main() {
	num := 123
	i64num := int64(num)
	inum := int(i64num)
	fmt.Printf("i64num: %T, inum: %T", i64num, inum) //i64num: int64, inum: int
}
```

## map和json互转
```golang
func main() {
	map1 := map[string]string{
		"key1": "Abc",
		"key2": "Bbc",
		"key3": "Cbc",
	}
	// map转json
	mjson, _ := json.Marshal(map1)
	mString := string(mjson)
	fmt.Printf("mjson:%T,mString:%T", mjson, mString) //mjson:[]uint8,mString:string

	// json 转map
	map2 := make(map[string]string)
	err := json.Unmarshal(mjson, &map2)
	// err := json.Unmarshal([]byte(mString), &map2)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Printf("map2:%T", map2) // map2:map[string]string
}
```
