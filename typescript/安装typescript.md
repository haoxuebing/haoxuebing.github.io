---
title: 安装typescript
date: 2024-09-21
categories: typescript
tags: [typescript]
description: typescript 安装与使用
---

## 安装

```shell
# 安装
npm install -g typescript

# 查看版本
tsc -v  
```



## 编译运行

```shell
#编译 生成 hello.js
tsc hello.ts

# 运行hello.js
node hello.js
```





## 使用ts-node

把编译与运行结合起来

```shell
# 安装 ts-node
npm install -t ts-node

# 直接运行ts
ts-node hello.ts
```



## 生成tsconfig.json

```shell
tsc --init
```



## 生成package.json

```shell
npm init -y
```

