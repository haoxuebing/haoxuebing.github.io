---
title: go test
date: 2021-12-01
categories: go
tags: [go, test]
description: go 单元测试与基准测试
---

## 单元测试
默认的情况下，`go test`命令不需要任何的参数，它会自动把你源码包下面所有 test 文件测试完毕

单元测试文件必须以`_test`结尾，例：hello_test.go

每个测试用例函数需要以`Test`为前缀

```
func TestXXX( t *testing.T )
```

### `go test` 命令
- 会自动读取源码目录下面名为 *_test.go 的文件
- 测试用例文件不会参与正常源码编译，不会被包含到可执行文件中
- 不需要 main() 作为函数入口
- 在以_test结尾的源码内以Test开头的函数会自动被执行

### `go test` 参数
- -bench regexp 执行相应的 benchmarks，例如 -bench=.
- -cover 开启测试覆盖率
- -run regexp 只运行 regexp 匹配的函数，例如 -run=Array 那么就执行包含有 Array 开头的函数
- -v 显示测试的详细命令


#指定某测试文件的某方法
```
go test -v -run TestA hello_test.go
```

## 基准测试
基准测试可以测试一段程序的运行性能及耗费 CPU 的程度

基准测试函数需以 `Benchmark` 开头

benchmark_test.go

``` golang
func Benchmark_Add(b *testing.B) {
    var n int
    for i := 0; i < b.N; i++ {
        n++
    }
}
```

```
go test -v -bench=. benchmark_test.go
```
基准测试框架对一个测试用例的默认测试时间是 1 秒

### 自定义测试时间
```
go test -v -bench=. -benchtime=5s benchmark_test.go
```

### 测试内存
基准测试可以对一段代码可能存在的内存分配进行统计
```golang
func Benchmark_Alloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("%d", i)
    }
}
```
`-benchmem`参数以显示内存分配情况
```
go test -v -bench=Alloc -benchmem benchmark_test.go
```