---
title: Dockerfile补充
date: 2023-02-28 11:01:26
categories: kubernates
tags:
---



## Dockerfile指令

```
FROM：继承基础镜像
MAINTAINER：镜像制作作者信息
RUN：用来执行shell命令
EXPOSE：暴露端口号
CMD：启动容器默认执行的命令
ENTRYPOINT：启动容器真正执行的命令
VOLUME：创建挂载点
ENV：配置环境变量
ADD：复制文件到容器，压缩文件会自动解压
COPY：复制文件到容器
WORKDIR：设置容器的工作目录
USER：容器使用的用户
```

Dockerfile中CMD和ENTRYPIONT 必须有其一

CMD可以被覆盖，如果有ENTRYPIONT，CMD就是ENTRYPIONT的参数    



## 创建镜像
```
docker build -t centos:user .
```



## 运行容器

```
docker run -it --rm centos:user bash
```





## 制作小镜像

- 不使用centos镜像

- alpine, busybox, scratch, Debian

- Glibc, node:slim, python:slim, net



scratch 为空容器，仅可运行不依赖其系统环境的程序。



## 使用多阶段构建

​		编译操作和生成最终镜像的操作



main.go

```
package main
import "fmt"

func main() {
    fmt.Println("hello.golang")
}
```





Dockerfile

```
## bulid step
FROM golang:1.14.4-alpine as builder

WORKDIR /opt

COPY main.go /opt

RUN go build /opt/main.go   
            
CMD "./main"
            
## create real app image

FROM alpine:3.8 
      
COPY --from=builder /opt/main /

CMD "/opt/main"
```



