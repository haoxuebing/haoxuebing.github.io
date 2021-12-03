---
title: gRPC的使用
date: 2021-12-03
categories: go
tags: [go, gRPC]
description: gRPC  是一个高性能、开源和通用的 RPC 框架
---


## gRPC简介

gRPC目前提供 C、Java 和 Go 语言版本，分别是 grpc, grpc-java, grpc-go

其中 C 版本支持 C, C++, Node.js, Python, Ruby, Objective-C, PHP, C#

gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特。这些特性使得其在移动设备上表现更好，更省电和节省空间占用

