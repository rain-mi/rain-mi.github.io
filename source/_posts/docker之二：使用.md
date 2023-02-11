---
title: docker之二：使用
date: 2021-05-22 17:09:40
categories: docker
tags:
---

## 一、创建容器
### 1.抓取mysql容器镜像
```
docker pull mysql:5.7

查看镜像
docker images
```

### 2.启动镜像
```
mkdir -pv /data/mysql/{log,conf,data}

docker run -p 31306:3306 --name mysql \
--privileged=true -v /data/mysql/log:/var/log/mysql \
-v /data/mysql/data:/var/lib/mysql \
-v /data/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

数据库用户名root,密码root。 
```

## 二、将运行中的容器进行修改
```
更新安装源
apt-get update
apt-get install -y vim
```

## 三、创建自定义镜像
### 1.将运行的容器打包为镜像
```
docker commit -m "add vim" -a "new mysql" 136a93f66a27 mysql:5.7

其中，-m 来指定提交的说明信息，跟我们使用的版本控制工具一样；
     -a 可以指定更新的用户信息；
     之后是用来创建镜像的容器的 ID；
     最后指定目标镜像的仓库名和 tag 信息。
创建成功后会返回这个镜像的 ID 信息。
```
### 2.在阿里云创建镜像仓库命名为mysql

### 3.在docker主机登录阿里云镜像仓库
```
登录阿里云Docker Registry
sudo docker login --username=[username] registry.cn-hangzhou.aliyuncs.com
用于登录的用户名为阿里云账号全名，密码为开通服务时设置的密码。
```

### 4.将镜像推送到Registry
```
$ sudo docker login --username=[username] registry.cn-hangzhou.aliyuncs.com
$ sudo docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/[username]-repository/mysql:[镜像版本号]
$ sudo docker push registry.cn-hangzhou.aliyuncs.com/[username]-repository/mysql:[镜像版本号]
```