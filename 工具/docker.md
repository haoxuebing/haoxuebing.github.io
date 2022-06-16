---
title: Dockerdate: 2022-01-27
categories: docker
tags: [docker]
description: Docker可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上

---

## docker 简介
> Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上

## docker 命令
```
docker pull                    #获取镜像
docker build                   #编译镜像   

docker images                  #查看所有镜像
docker ps                      #查看docker进程 
docker  ps -a                  #查看所有运行过的镜像

docker run                     #使用镜像启动一个容器
docker rmi 镜像id               #删除某个镜像
docker rm -f  容器id            #多个容器id用空格隔开

docker start 容器id             #启动某个容器
docker stop 容器id              #停止某个容器　
docker restart 容器id           #重启某个容器　
docker logs 容器id              #查看容器运行日志
```

### 用例
```
docker pull ubuntu:13.10  
```
- 使用 docker pull 命令来载入 ubuntu 镜像
- :后为版本号

```
docker run -itd --name ubuntu-test ubuntu /bin/bash
```
- -i 交互式操作。
- -t 终端。
- -d 参数默认不会进入容器
- --name 容器的名字
- ubuntu ubuntu 镜像。
- /bin/bash：交互式 Shell

```
docker run -d -p 5000:5000 -v /data:/data webapp python app.py
```
- -p 设置端口 主机(宿主)端口:容器端口
- -v 目录映射 主机的目录 /data 映射到容器的 /data

```
docker exec -it 容器ID /bin/bash
```
- 使用交互式shell进入容器

```
docker build -t test/myapp:v1 .
```
- -f /path/to/a/Dockerfile 默认是当前目录下的Dockerfile
- -t 镜像的名字及标签，通常 name:tag 或者 name 格式
- . 表示当前目录，也可是其他目录或URL

## Dockerfile
```
FROM golang:alpine
# 为我们的镜像设置必要的环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
	GOPROXY="https://goproxy.cn,direct"

WORKDIR /home/www/code_20220515

# 将代码复制到容器中
COPY . .
# 将我们的代码编译成二进制可执行文件  可执行文件名为 app
RUN go build -o app .

# 移动到用于存放生成的二进制文件的 /dist 目录
WORKDIR /dist

RUN cp /home/www/code_20220515/app .

# 在容器目录 /dist 创建一个目录 为static
RUN mkdir static .

# 在容器目录 把宿主机的静态资源文件 拷贝到 容器/dist/src目录下
# 这个步骤可以略  因为项目是引用到了 外部静态资源
RUN cp -r /home/www/code_20220515/static .

# 声明服务端口
EXPOSE 8080

# 启动容器时运行的命令
CMD ["/dist/app"]
```
\#编译dockerfile
```
docker build -t my_web_0515:v1 .
```

\# 运行编译好的镜像
```
docker run -itd -p 8080:8080 my_web_0515:v1
```   


## docker-compose.yml
```
version: '3.4'

services:
  code1:
    image: code1
    build:
      context: .
      dockerfile: ./dockerfile
    ports:
      - 8080:8080

```
\# 启动并运行整个应用程序
```
docker-compose -f docker-compose.yml up -d
```

## 其他

[docker命令大全](https://www.runoob.com/docker/docker-command-manual.html)