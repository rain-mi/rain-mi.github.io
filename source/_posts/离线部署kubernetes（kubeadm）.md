---
title: 离线部署kubernetes（kubeadm）
date: 2021-08-08 22:33:26
categories: kubernetes
tags:
---


## 一. 环境初始化（全部节点）
### 1）设置防火墙为iptables并设置空规则
```
systemctl stop firewalld && systemctl disable firewalld
```

### 2）关闭selinux
```
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
mkdir /package
```

### 3）域名解析
```
cat <<EOF>>/etc/hosts
192.168.0.151   k8s-master01
192.168.0.161   k8s-node01
192.168.0.162   k8s-node02
EOF
```

### 4）检测虚拟内存是否关闭,因为防止容器运行再虚拟内存
```
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 5）将桥接的 IPv4 流量传递到 iptables 的链
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

#让系统生效
sysctl --system   
```
 
### 6）时间同步
```
###master01节点
yum install chrony -y
vim /etc/chrony.conf
--------------------------
#修改下面2行
server ntp.aliyun.com iburst
allow 192.168.3.0/24

重启服务
systemctl restart chronyd
systemctl enable chronyd

###node01、02节点
yum install chrony -y
vim /etc/chrony.conf
--------------------------
#server修改成控制节点的ip或者主机名：
server 192.168.3.151 iburst

重启服务
systemctl restart chronyd
systemctl enable chronyd

手动同步
chronyc makestep

查看时间主机
chronyc sources
clock -w
```

### 7）必备工具安装
```
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y
```

### 8）内核参数修改
```
vim /etc/security/limits.conf
# 末尾添加如下内容
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

### 9）所有节点升级系统并重启，此处跳过升级内核
```
yum update -y --exclude=kernel* && reboot
```


## 二. kubernetes1.20以后版本需要升级内核（全部节点）
### 1）下载离线升级包
```
cd /package
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm
wget http://193.49.22.109/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm
```

### 2）从master01节点传到其他节点：
```
for i in k8s-node01 k8s-node02;\
do scp kernel-ml-4.19.12-1.el7.elrepo.x86_64.rpm \
kernel-ml-devel-4.19.12-1.el7.elrepo.x86_64.rpm \
$i:/package/ ; done
```

### 3）所有节点安装内核
```
cd /package && yum localinstall -y kernel-ml*
```

### 4）所有节点更改内核启动顺序
```
grub2-set-default  0 && grub2-mkconfig -o /etc/grub2.cfg
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
```

### 5）检查默认内核是不是4.19
```
grubby --default-kernel
```

### 6）所有节点重启，然后检查内核是不是4.19
```
uname -a
```

## 三. 所有节点安装ipvsadm
```
yum install ipvsadm ipset sysstat conntrack libseccomp -y
所有节点配置ipvs模块，在内核4.19+版本nf_conntrack_ipv4已经改为nf_conntrack， 4.18以下使用nf_conntrack_ipv4即可：
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

vim /etc/modules-load.d/ipvs.conf 
# 加入以下内容
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
然后执行
systemctl enable --now systemd-modules-load.service
```



## 四. 部署节点docker(3台节点相同）

### 1）安装源码包
```
##下载地址：
https://download.docker.com/linux/static/stable/x86_64/docker-19.03.14.tgz
```

### 2）解压安装包
```
tar -zxvf docker-19.03.14.tgz && cd docker/
mv docker dockerd docker-init docker-proxy /usr/bin/
```

### 3）配置systemd管理
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

### 4）添加镜像加速
```
mkdir /etc/docker
--------------------------------------------------
cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
--------------------------------------------------
```

### 5）检测rpm包依赖
```
rpm -qa containerd.io
rpm -qa device-mapper-libs 

ps：如果没有rpm的安装方法
#这里提供2个源
https://pkgs.org/     #这个比较全 
https://mirrors.aliyun.com/centos/7/os/x86_64/Packages/  #阿里的源,常见的都有

#获取rpm包
wget https://download.docker.com/linux/centos/7/x86_64/test/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/device-mapper-libs-1.02.170-6.el7.x86_64.rpm

#安装
rpm -ivh containerd.io-1.2.6-3.3.el7.x86_64.rpm --nodeps
rpm -ivh device-mapper-libs-1.02.170-6.el7.x86_64.rpm --nodeps

#这里可能有特殊情况,有一些环境会自带一些版本较低的依赖，导致版本冲突，就需要先手动卸载之前的rpm包
#rpm -e containerd.io
```

### 6）启动docker服务
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
-------------------------------------------------------------------------------
```

### 7）拷贝文件到node01、node02
```
scp /package/docker-19.03.14.tgz root@192.168.0.161:/package
scp /package/*.rpm root@192.168.0.161:/package
scp -pr /etc/docker/ root@192.168.0.161:/etc/
scp /usr/lib/systemd/system/docker.service root@192.168.0.161:/usr/lib/systemd/system/
```

### 8）node01、node02安装服务
```
cd /package/
tar -zxvf docker-19.03.14.tgz && cd docker/
mv docker dockerd docker-init docker-proxy /usr/bin/

cd /package/
rpm -ivh containerd.io-1.2.6-3.3.el7.x86_64.rpm --nodeps
rpm -ivh device-mapper-libs-1.02.170-6.el7.x86_64.rpm --nodeps
```

### 9）node01、node02启动docker服务
```
systemctl daemon-reload
systemctl start docker
systemctl enable docker

查看docker服务
systemctl status docker
```





## 五. 部署cni网络(3台节点相同）
### 1）下载地址
```
https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz
```

### 2）解包
```
cd /package/
mkdir /opt/cni/bin -p
tar -zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
```
### 3）配置环境变量
```
echo 'export PATH=$PATH:/opt/cni/bin' > /etc/profile.d/cni.sh
source /etc/profile.d/cni.sh
```

## 六. 部署cri(3台节点相同）
```
cd /package/
tar -zxvf crictl-v1.16.0-linux-amd64.tar.gz -C /usr/bin/
```




## 七. 部署kubeadm
### 1）安装二进制文件
```
##下载地址：
https://dl.k8s.io/v1.19.12/kubernetes-server-linux-amd64.tar.gz
```

### 2）解压安装包
```
tar -zxvf kubernetes-server-linux-amd64.v1.19.12.tar.gz
mv kubernetes/server/bin/{kubeadm,kubelet,kubectl} /usr/bin/
chmod +x /usr/bin/{kubeadm,kubelet,kubectl}
```

### 3）配置systemd管理
```
添加到systemd管理
--------------------------------------------------
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
--------------------------------------------------
```

### 4）添加配置文件
```
mkdir -pv /usr/lib/systemd/system/kubelet.service.d
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
----------------------------------------------------------------------------------------------------
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
----------------------------------------------------------------------------------------------------
```

### 5）拷贝二进制文件到node1、node2节点
```
scp /usr/bin/{kubeadm,kubelet,kubectl} root@192.168.0.161:/usr/bin/
scp /usr/lib/systemd/system/kubelet.service root@192.168.0.161:/usr/lib/systemd/system/
scp -pr /usr/lib/systemd/system/kubelet.service.d root@192.168.0.161:/usr/lib/systemd/system/
```

### 6）设置开机自启动
```
systemctl enable kubelet.service
```

### 7）安装依赖包
```
yum -y install socat conntrack-tools
```




## 八. 部署kubernetes
```
kubeadm init \
--apiserver-advertise-address=192.168.0.151 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.19.12 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```



## 九. 部署网络插件
## 方案一：安装网络插件flannel组件
### 1）下载版本相对应的kube-flannel.yml文件
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 2）修改镜像地址：(默认quay.io镜像国内访问过慢，修改为中科大镜像地址。修改第169、183两行。)
```
vim kube-flannel.yml
169         image: quay.mirrors.ustc.edu.cn/coreos/flannel:v0.14.0
183         image: quay.mirrors.ustc.edu.cn/coreos/flannel:v0.14.0
```

### 3）创建网络插件
```
kubectl apply -f kube-flannel.yml
```

### 4）查看集群的node状态，安装完网络工具之后，所有节点全部都Ready之后才能继续后面的操作
```
# kubectl get nodes
NAME           STATUS   ROLES    AGE    VERSION
k8s-master01   Ready    master   18m   v1.19.12
k8s-node01     Ready    <none>   18m   v1.19.12
k8s-node02     Ready    <none>   96m   v1.19.12

# kubectl get pod -n kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-7ff77c879f-2jc7b               1/1     Running   0          110m
coredns-7ff77c879f-s7qdp               1/1     Running   0          110m
etcd-k8s-master01                      1/1     Running   0          111m
kube-apiserver-k8s-master01            1/1     Running   0          111m
kube-controller-manager-k8s-master01   1/1     Running   0          111m
kube-flannel-ds-qp968                  1/1     Running   0          74m
kube-flannel-ds-tqwnk                  1/1     Running   0          74m
kube-flannel-ds-trrrg                  1/1     Running   0          74m
kube-proxy-2xwlx                       1/1     Running   0          97m
kube-proxy-bt4rz                       1/1     Running   0          97m
kube-proxy-fxq7n                       1/1     Running   0          110m
kube-scheduler-k8s-master01            1/1     Running   0          111m

注意：
只有全部都为1/1则可以成功执行后续步骤，如果flannel需检查网络情况，重新进行如下操作
kubectl delete -f kube-flannel.yml

然后重新wget，然后修改镜像地址，然后
kubectl apply -f kube-flannel.yml
```


## 方案二：安装网络插Calico组件
```
We test Calico v3.20 against the following Kubernetes versions.
v1.19
v1.20
v1.21
```
### 1）下载Calicov3.20配置文件
```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
vim calico.yaml
修改
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
与kubeadm初始化时
--pod-network-cidr相同
```

### 2） 安装calico组件
```
kubectl apply -f calico.yaml
```

### 3）查看容器状态
```
kubectl get pods -n kube-system
```



## 扩展一：yum安装kubeadm方法
### 1）查看官方kube版本
```
yum list --showduplicates | grep 'kubeadm\|kubectl\|kubelet'
```

### 2）配置国内yum源
```
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3）安装kubeadm
```
yum install -y kubelet-1.19.12 kubeadm-1.19.12 kubectl-1.19.12

```



### 扩展二：离线镜像
#### kubernetes核心镜像
```
##打标签
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.19.12 registry.aliyuncs.com/google_containers/kube-proxy:v1.19.12
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.12 registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.12
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.12 registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.12
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.12 registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.12
docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0 registry.aliyuncs.com/google_containers/etcd:3.4.13-0
docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0 registry.aliyuncs.com/google_containers/coredns:1.7.0
docker tag registry.aliyuncs.com/google_containers/pause:3.2 registry.aliyuncs.com/google_containers/pause:3.2

##保存本地
docker save -o kube-proxy:v1.19.12.tar registry.aliyuncs.com/google_containers/kube-proxy:v1.19.12
docker save -o kube-controller-manager:v1.19.12.tar registry.aliyuncs.com/google_containers/kube-controller-manager:v1.19.12
docker save -o kube-apiserver:v1.19.12.tar registry.aliyuncs.com/google_containers/kube-apiserver:v1.19.12
docker save -o kube-scheduler:v1.19.12.tar registry.aliyuncs.com/google_containers/kube-scheduler:v1.19.12
docker save -o etcd:3.4.13-0.tar registry.aliyuncs.com/google_containers/etcd:3.4.13-0
docker save -o coredns:1.7.0.tar registry.aliyuncs.com/google_containers/coredns:1.7.0
docker save -o pause:3.2.tar registry.aliyuncs.com/google_containers/pause:3.2

##导入docker镜像
docker load -i kube-webhook-certgen:v1.5.0.tar
docker load -i kube-proxy:v1.19.12.tar
docker load -i kube-controller-manager:v1.19.12.tar
docker load -i kube-apiserver:v1.19.12.tar
docker load -i kube-scheduler:v1.19.12.tar
docker load -i etcd:3.4.13-0.tar 
docker load -i coredns:1.7.0.tar
docker load -i pause:3.2.tar
```


#### Calico镜像
```
##打标签
docker tag calico/node:v3.20.0 calico/node:v3.20.0
docker tag calico/pod2daemon-flexvol:v3.20.0 calico/pod2daemon-flexvol:v3.20.0
docker tag calico/cni:v3.20.0 calico/cni:v3.20.0
docker tag calico/kube-controllers:v3.20.0 calico/kube-controllers:v3.20.0

 ##保存本地
docker save -o node:v3.20.0.tar calico/node:v3.20.0
docker save -o pod2daemon-flexvol:v3.20.0.tar calico/pod2daemon-flexvol:v3.20.0
docker save -o cni:v3.20.0.tar calico/cni:v3.20.0
docker save -o kube-controllers:v3.20.0.tar calico/kube-controllers:v3.20.0

##导入docker镜像
docker load -i node:v3.20.0.tar
docker load -i pod2daemon-flexvol:v3.20.0.tar
docker load -i cni:v3.20.0.tar
docker load -i kube-controllers:v3.20.0.tar
```


#### flannel镜像
```
docker tag quay.io/coreos/flannel:v0.14.0-rc1 quay.io/coreos/flannel:v0.14.0-rc1
docker save -o flannel:v0.14.0-rc1.tar quay.io/coreos/flannel:v0.14.0-rc1
docker load -i flannel:v0.14.0-rc1.tar
```

#### 推送镜像示例
```
docker push ${repository}/{仓库名}/{镜像名}:{版本号}
```









