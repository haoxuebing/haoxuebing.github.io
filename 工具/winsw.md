---
title: WinSW
date: 2022-01-27
categories: windows
tags: [服务管理]
description: WinSW 将任何应用程序作为 Windows 服务进行包装和管理
---

## WinSW 
Windows Service Wrapper，简称WinSW，可用于管理windows上的服务

github地址  [https://github.com/winsw/winsw](https://github.com/winsw/winsw)

安装服务 `winsw install myapp.xml`  
启动服务 `winsw start myapp.xml`  
查看服务状态 `winsw status myapp.xml`


## 命令

```
# 安装
winsw install [<path-to-config>] [--no-elevate] [--user|--username <username>] [--pass|--password <password>]
# 卸载
winsw uninstall [<path-to-config>] [--no-elevate]
# 启动
winsw start [<path-to-config>] [--no-elevate]
# 停止
winsw stop [<path-to-config>] [--no-elevate] [--no-wait]
# 重启
winsw restart [<path-to-config>] [--no-elevate]
# 状态
winsw status [<path-to-config>]
# 刷新
winsw refresh [<path-to-config>] [--no-elevate]
# 定制
winsw customize -o|--output <output> --manufacturer <manufacturer>
# 绘制与服务关联的进程树
winsw dev ps [<path-to-config>] [-a|--all]
# 如果服务停止响应，则终止服务
winsw dev kill [<path-to-config>] [--no-elevate]
# 列出由当前可执行文件管理的服务
winsw dev list

```

## 用例
以frp为例，设置为开机启动

- 1.下载 `WinSW-x64.exe` 然后重名为 `winsw`
  下载地址 https://github.com/winsw/winsw/releases/tag/v3.0.0-alpha.10
- 2.在winsw的目录下创建 `winsw.xml` 内容如下

```
<service>
    <!-- 该服务的唯一标识 -->
    <id>frp</id>
    <!-- 该服务的名称 -->
    <name>frpc.exe</name>
    <!-- 该服务的描述 -->
    <description>frpc客户端 这个服务用 frpc 实现内网穿透</description>
    <!-- 要运行的程序路径 -->
    <executable>D:\frp\frpc.exe</executable>
    <!-- 携带的参数 -->
    <arguments>-c frpc.ini</arguments>
    <!-- 第一次启动失败 60秒重启 -->
    <onfailure action="restart" delay="60 sec"/>
    <!-- 第二次启动失败 120秒后重启 -->
    <onfailure action="restart" delay="120 sec"/>
    <!-- 日志模式 -->
    <logmode>append</logmode>
    <!-- 指定日志文件目录(相对于executable配置的路径) -->
    <logpath>logs</logpath>
</service>
```  
使用 `winsw.xml` 这个名字，可在命令中省去指定config

- 3.启动服务
```
$ ./winsw.exe install   # 注册服务 
$ ./winsw.exe start   # 启动服务
``` 
- 4.检测服务

可以通过任务管理器查看

也可使用 `./winsw.exe status` 或者 `./winsw.exe dev list` 查看
