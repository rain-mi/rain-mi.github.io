---
title: SRS流媒体之二：SRS Cluster
date: 2021-06-09 22:03:34
categories: srs
tags:
---
## 一、简介
### 开源流媒体服务器SRS Cluster集群
单台服务器做直播，总归有单点风险，利用SRS的Forward机制 + Edge Server设计，搭建一个大规模的高可用集群。

源站服务器集群：origin server cluster，可以借助forward机制，仅用少量的服务器，专用于处理推流请求。 
边缘服务器集群：edge server cluster，可以用N台机器，从源站拉流，用于较大规模的实时播放。
源站前置负载均衡（硬件或软件负载均衡都行），上图中用haproxy来实现tcp的软负载均衡。
边缘服务器前置反向代理（比如：nginx），用于提供统一的播放地址，同时解决跨域问题，给客户端拉流播放。

### 这样架构的好处有以下：
1）不管是源站集群，还是连缘服务器集群，均可水平扩展，理论上没有上限。
2）源站可以仅用较少的机器，比如2主2从，就能实现一个高可用且性能尚可的集群（如果业务量不大，连slave server都可以省掉）
3）边缘服务器集群，可以根据实际用户量随时调整规模，另外hls切片，可以放在edge server上切片，减轻源站服务器压力。

## 二、部署
### 1.服务器架构
```
ip	            rtmp port    http api port   http server port   role
172.18.0.147   1945		     1995            8180              master
               1946		     1996            8181              slave
 		         1947		     1997            8182              edge
172.18.0.131	 1945		     1995            8180              master
 		          1946		     1996            8181              slave　　
                1947		  1997            8182              edge　
```

### 2.1 配置master
```
vim /usr/local/srs/conf/master.conf
----------------------------------------
listen              1945;
max_connections     1000;
pid                 ./objs/srs.master.pid
srs_log_tank        file;
srs_log_file        ./objs/srs.master.log;
 
http_api {
    enabled         on;
    listen          1995;
}
 
http_server {
    enabled         on;
    listen          8180;
    dir             ./objs/nginx/html;
}
 
stats {
    network         0;
    disk            sda sdb xvda xvdb;
}
 
vhost __defaultVhost__ {
        forward        172.18.0.147:1946 172.18.0.131:1946;
}
注：最后一段的forward，表示将视频流转发到2台slave服务器
```

### 2.2 slave配置
```
vim /usr/local/srs/conf/slave.conf
----------------------------------------
listen              1946;
max_connections     1000;
pid                 ./objs/srs.slave.pid
srs_log_tank        file;
srs_log_file        ./objs/srs.slave.log;
 
http_api {
    enabled         on;
    listen          1996;
}
 
http_server {
    enabled         on;
    listen          8181;
    dir             ./objs/nginx/html;
}
 
stats {
    network         0;
    disk            sda sdb xvda xvdb;
}
 
vhost __defaultVhost__ {
}
```

### 2.3 edge配置
```
vim /usr/local/srs/conf/edge.conf
----------------------------------------
listen              1947;
max_connections     1000;
pid                 ./objs/srs.edge.pid
srs_log_tank        file;
srs_log_file        ./objs/srs.edge.log;
 
http_api {
    enabled         on;
    listen          1997;
}
 
http_server {
    enabled         on;
    listen          8182;
    dir             ./objs/nginx/html;
}
 
stats {
    network         0;
    disk            sda sdb xvda xvdb;
}
 
vhost __defaultVhost__ {
 
    http_remux{
        enabled     on;
        mount       [vhost]/[app]/[stream].flv;
        hstrs       on;
    }
 
    hls{
        enabled         on;
        hls_path        ./objs/nginx/html;
        hls_fragment    10;
        hls_window      60;
    }
 
    mode            remote;
    origin          172.18.0.147:1945 172.18.0.131:1945 172.18.0.147:1946 172.18.0.131:1946;
}
----------------------------------------
注：最后一段的origin 将所有master、slave均做为视频源(origin server)，如果播放时，edge发现自己机器上没有数据，会从origin配置的这些源站上去拉视频流。
```

## 三、启动与测试
### 1 启动
```
每台虚拟机上，依次启动：slave、master、edge
cd /usr/local/srs
sudo ./objs/srs -c ./conf/slave.conf
sudo ./objs/srs -c ./conf/master.conf
sudo ./objs/srs -c ./conf/edge.conf
```

### 2 测试
1）可以用obs向每个master或slave推流试试，比如 rtmp://172.18.0.147:1945/live/livestream 或 rtmp://172.18.0.147:1946/live/livestream，如果推流不报错，说明master/slave工作正常
2）然后用vlc播放器，验证从slave/edge这些服务器上拉流(比如 rtmp://172.18.0.131:1946/live/livestream 或 rtmp://172.18.0.131:1947/live/livestream，是否播放正常

推流命令
```
ffmpeg -re -i source.200kbps.768x320.flv -vcodec copy -acodec copy -f flv -y rtmp://srs_server_ip:1935/live/livestream
ffmpeg -re -i /package/qryt.flv -vcodec copy -acodec copy -f flv -flvflags no_duration_filesiz -y rtmp://172.18.0.147:1945/live/livestream
```

## 四、配置高可用集群
### 1 配置haproxy
```
yum install -y haproxy 
vim /etc/haproxy/haproxy.cfg 
----------------------------------------
global
    log         127.0.0.1 local2
 
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
 
    stats socket /var/lib/haproxy/stats
 
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    option                  dontlognull
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
 
listen srs-cluster
    bind *:1935
    mode tcp
    balance roundrobin
    server master1 172.18.0.147:1945
    server master2 172.18.0.131:1945
-----------------------------------------
注：关键是最后一段，把本机1935端口，转发到后端2台master服务器的1945端口。


systemctl restart haproxy
重启haproxy成功后，可以用obs推流到 rtmp://haproxy_server_ip:1935/live/livestream试下推流是否正常。
```

### 2 为edge配置nginx负载均衡
```
vim /usr/local/nginx/conf/nginx.conf
-----------------------------------------
    upstream srs{
        server 172.18.0.147:8182;
        server 172.18.0.131:8182;
    }
 
    server {

        location ~ /* {
            proxy_pass http://srs;
            add_header Cache-Control no-cache;
            add_header Access-Control-Allow-Origin *;
        }
    }
-----------------------------------------

#推送flv视频命令
ffmpeg -re -i /package/qryt.flv -vcodec copy -acodec copy -f flv -y rtmp://172.18.0.147:1935/live/qryt.flv
```
注：新增一个upstream用于指定要转发的edge服务器节点，然后在location ~ /* 这里proxy_pass 指定upstream的名字（location ~ /* 切记要写在 location / 前面）
这样配置后，访问 http://nginx_server_ip/live/qryt.flv 理论上就能转到后端的edge服务器。实测：浏览器会提示下载视频，可用VCL观看视频





