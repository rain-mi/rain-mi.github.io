---
title: SRS流媒体之一：单节点部署
date: 2021-06-09 21:55:31
categories: srs
tags:
---

## 1.安装ffmpeg源
```
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro
rpm -Uvh http://li.nux.ro/download/nux/dextop/el7/x86_64/nux-dextop-release-0-1.el7.nux.noarch.rpm
```

## 2.安装常用工具,srs依赖
```
yum install -y wget lrzsz zip unzip vim
yum install -y epel-release redhat-lsb
yum install -y ffmpeg ffmpeg-devel   
```

## 3.安装SRS
```
unzip SRS-CentOS7-x86_64-3.0.160.zip 
cd SRS-CentOS7-x86_64-3.0.160
bash INSTALL 
```

## 4.编辑配置文件
```
vim /usr/local/srs/conf/srs.conf
在 vhost defaultVhost内部加如下配置

vhost __defaultVhost__ {
    ingest live-02 {
        enabled      on;
        input {
            type    stream;
            url     rtmp://58.200.131.2:1935/livetv/hunantv;   #拉取湖南卫视流到服务器
        }
        ffmpeg      /usr/bin/ffmpeg;
        engine {
            enabled          on;
            output          rtmp://localhost:1935/live/bbb;#输出流
        }
    }
}
```

## 5.启动及命令
```
启动
sudo /etc/init.d/srs start

其它命令
service srs start
service srs stop
service srs restart
```

## 6.测试
```
使用VCL测试
选择 “媒体”--“打开网络串流”
输入 rtmp://172.18.0.147:1935/live/bbb
```