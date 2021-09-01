---
title: go读写文件
date: 2021-09-01
categories: go
tags: [go, file]
description: go 读写文件汇总
---

### 判断文件是否存在

```golang
func checkFileIsExist(filename string) bool {
	var exist = true
	if _, err := os.Stat(filename); os.IsNotExist(err) {
		exist = false
	}
	return exist
}
```

### 读文件
方式一：
```golang
func readFile(filename string) {
	data, _ := ioutil.ReadFile(filename)
	fmt.Printf("Data as string: %s\n", data)
}
```
方式二：
```golang
func readFile2(filename string) {
	file, _ := os.OpenFile(filename, os.O_CREATE, 0666)
	defer file.Close()
	data, _ := ioutil.ReadAll(file)
	fmt.Printf("Data as string: %s\n", data)
}

```

### 写文件
方式一：使用 file.WriteString 或 file.Write
```golang
func writeFile(filename, data string) {
    // 打开文件不存在则创建
	file, _ := os.OpenFile(filename, os.O_CREATE, 0666)
	defer file.Close()
	file.WriteString(data) // 写入文件 字符串
	// or
	file.Write([]byte(data)) // 写文件   字节数组
}

```

方式二：使用 io.WriteString
```golang
func writeFile2(filename, data string) {
	file, _ := os.OpenFile(filename, os.O_CREATE, 0666)
	defer file.Close()
	io.WriteString(file, data)
}
```

方式三：使用 ioutil.WriteFile
```golang
func writeFile3(filename, data string) {
	ioutil.WriteFile(filename, []byte(data), 0666)
}
```

方式四： 使用 bufio.NewWriter
```golang
func writeFile4(filename, data string) {
	file, _ := os.OpenFile(filename, os.O_CREATE, 0666)
	w := bufio.NewWriter(file) //创建新的 Writer 对象
	w.WriteString(data)
	w.Flush()
}
```