---
title: gRPC的使用
date: 2021-12-03
categories: go
tags: [go, gRPC]
description: gRPC  是一个高性能、开源和通用的 RPC 框架
---


### gRPC简介

gRPC目前提供 C、Java 和 Go 语言版本，分别是 grpc, grpc-java, grpc-go

其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP, C#

#### gRPC服务端
定义一个服务，指定其能够被远程调用的方法（包含参数和返回类型）。在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用
#### gRPC客户端
在 gRPC客户端拥有一个存根能够像服务端一样的方法，客户端应用可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法，能够更容易地创建分布式应用和服务

#### Protocol buffers简介
protocol buffer是google推出的一种数据描述语言，类似于XML能够将结构化数据序列化，可用于数据存储、通信协议等方面，在 gRPC 里使用 proto3，这样可以使用 gRPC 支持全部范围的的语言

![grpc](../images/grpc.png)



#### 安装protobuf

github 获取 protobuf 源码 [https://github.com/google/protobuf/releases](https://github.com/google/protobuf/releases)

- windows 系统可以直接下载 exe 文件
    
-  linux 环境使用源码进行安装的步骤  

```
# 获取源码包
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.1/protobuf-all-3.19.1.tar.gz
cd protobuf-3.19.1/
./configure
make
make install
刷新共享库 （很重要的一步啊）
sudo ldconfig 
# 成功后需要使用命令测试
protoc -h 

# 安装protoc-gen-go，linux下这个插件不装没法生成go代码
go get github.com/golang/protobuf/{protoc-gen-go,proto}

cd $GOPATH

# 解压后获得protoc-gen-go，直接复制到bin目录
cp ./protoc-gen-go /usr/local/bin

```

#### 编写 .proto 文件

vim arith.proto
```
syntax = "proto3";   // 使用proto3版本
option go_package = "./pb"; // 生成的.go 文件放在 pb文件夹下


// 算术运算请求结构
message ArithRequest {
    int32 a = 1;   // 1代表顺序
    int32 b = 2;   // 2代表顺序
}

// 算术运算响应结构
message ArithResponse {
    int32 pro = 1;  // 乘积
    int32 quo = 2;  // 商
    int32 rem = 3;  // 余数
}

// rpc方法
service ArithService {
    rpc multiply (ArithRequest) returns (ArithResponse);    // 乘法运算方法
    rpc divide (ArithRequest) returns (ArithResponse);      // 除法运算方法
}
```

使用protoc工具，把 .proto 文件转化成 .go 文件，使用protoc工具生成代码
```
protoc --go_out=plugins=grpc:. arith.proto  # 用这条命令去生成go文件

```

~~protoc --go_out=. --plugin=grpc arith.proto~~ # 这条命令生成的go文件少内容，坑死我了


sever、client都需要引入grpc
```
go get google.golang.org/grpc
```
#### server 端
基于生成的arith.pb.go代码来实现一个rpc服务端

vim server.go
``` golang
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"go_rpc/pb"
	"log"
	"net"

	"google.golang.org/grpc"
)

var (
	port = flag.Int("port", 8097, "The server port")
)

// 算术运算结构体
type Arith struct {
	pb.UnimplementedArithServiceServer
}

// 乘法运算方法
func (*Arith) Multiply(ctx context.Context, req *pb.ArithRequest) (*pb.ArithResponse, error) {
	res := new(pb.ArithResponse)
	res.Pro = req.GetA() * req.GetB()
	return res, nil
}

// 除法运算方法
func (*Arith) Divide(ctx context.Context, req *pb.ArithRequest) (*pb.ArithResponse, error) {
	res := new(pb.ArithResponse)
	if req.GetB() == 0 {
		return res, errors.New("divide by zero")
	}
	res.Quo = req.GetA() / req.GetB()
	res.Rem = req.GetA() % req.GetB()
	return res, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf("127.0.0.1:%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterArithServiceServer(s, new(Arith))
	log.Printf("server listening at %v", lis.Addr())
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

运行上述程序，将会监听本地的8097端口并接收客户端的tcp连接。

#### client 端
基于生成的arith.pb.go再来实现一个客户端程序
vim client.go 
``` golang
package main

import (
	"context"
	"flag"
	"go_rpc/pb"
	"log"
	"time"

	"google.golang.org/grpc"
)

const (
	defaultName = "ArithService"
)

var (
	addr = flag.String("addr", "127.0.0.1:8097", "the address to connect to")
	name = flag.String("name", defaultName, "Name to ArithService")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.Dial(*addr, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewArithServiceClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	r, err := c.Multiply(ctx, &pb.ArithRequest{
		A: 11,
		B: 12,
	})
	if err != nil {
		log.Fatalf("could not ArithService: %v", err)
	}
	log.Printf("%s Multiply : %d", *name, r.GetPro())
}

```

运行结果：
```
ArithService Multiply : 132
```


#### 总结
如果使用protoc生成的 .go 文件有问题，缺少一些结构体或者方法 
- 首先看下使用的命令 `protoc --go_out=plugins=grpc:. *.proto` 这个笔者用的没问题 
- 再就是检查下 `protoc-gen-go`插件，github源码库下载的可能也是有问题的，最好用go get 安装 