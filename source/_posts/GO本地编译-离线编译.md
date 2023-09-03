---
title: GO本地编译&&离线编译
date: 2023-09-03 20:51:34
tags:
- "Go编译"
categories:
- ["编程语言","GO"]
---
1. 设置环境变量:
```sh
export GOROOT=/home/go
export PATH=$PATH:$GOROOT/bin
```
打开本地代理（在线，可以下载依赖的module）
```sh
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```
将本地下载的go module $GOROOT/bin/pkg 拷贝到离线服务器的 $GOROOT/bin/pkg 下
在离线服务器上设置
```sh
go env -w GOPROXY=file:///$GOROOT/bin/pkg/mod/cache/download
```
2. 编译命令：
```sh
go build -o sql_exporter main.go
# alpine环境下运行编译命令
# alpine环境下运行需要go env -w CGO_ENABLED=0
CGO_ENABLED=0 go build -o sql_exporter main.go
```
关闭安全性校验： go env -w GOSUMDB=off
