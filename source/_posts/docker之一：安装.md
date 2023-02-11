---
title: docker之一：安装
date: 2021-05-21 09:53:08
categories: docker
tags:
---


## 源码部署docker服务

### 1.安装源码包
```
下载地址：
https://download.docker.com/linux/static/stable/x86_64/docker-19.03.15.tgz

解压安装包
tar -zxvf docker-19.03.15.tgz && cd docker/
mv docker dockerd docker-init docker-proxy /usr/bin/
```

### 2.配置systemd管理
```
添加到systemd管理
--------------------------------------------------
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
EOF
--------------------------------------------------
```

### 3.添加镜像加速
```
mkdir /etc/docker
--------------------------------------------------
cat > /etc/docker/daemon.json << EOF
{
"registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
--------------------------------------------------
```

### 4.检测rpm包依赖
```
rpm -qa containerd.io
rpm -qa device-mapper-libs 

ps：如果没有rpm的安装方法

安装
rpm -ivh containerd.io-1.2.6-3.3.el7.x86_64.rpm --nodeps
rpm -ivh device-mapper-libs-1.02.170-6.el7.x86_64.rpm --nodeps

这里可能有特殊情况,有一些环境会自带一些版本较低的依赖，导致版本冲突，就需要先手动卸载之前的rpm包
rpm -e containerd.io
```

### 5.启动docker服务
```
systemctl daemon-reload
systemctl start docker
systemctl enable docker

查看docker服务
systemctl status docker


启动成功返回
-----------------------------------show logs-----------------------------------
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since 三 2021-04-14 20:05:51 CST; 7s ago
     Docs: https://docs.docker.com
 Main PID: 11405 (dockerd)
   CGroup: /system.slice/docker.service
           ├─11405 /usr/bin/dockerd
           └─11413 containerd --config /var/run/docker/containerd/containerd.toml --log-level info

4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:50.854086675+08:00" level=info...rpc
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:50.854095317+08:00" level=info...rpc
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:50.854100903+08:00" level=info...rpc
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:50.880728349+08:00" level=info...t."
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:51.024190346+08:00" level=info...ss"
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:51.065931943+08:00" level=info...e."
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:51.132631963+08:00" level=info....15
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:51.132694626+08:00" level=info...on"
4月 14 20:05:51 k8s-master01 dockerd[11405]: time="2021-04-14T20:05:51.144233079+08:00" level=info...ck"
4月 14 20:05:51 k8s-master01 systemd[1]: Started Docker Application Container Engine.
Hint: Some lines were ellipsized, use -l to show in full.
----------------------------------------------------------------------
```

至此，我们的docker服务就部署完成啦！