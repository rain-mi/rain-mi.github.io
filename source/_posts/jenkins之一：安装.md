---
title: jenkins之一：安装
date: 2021-05-23 11:50:34
categories: jenkins
tags:
---
## Jenkins工作方式
<img src="https://s2.loli.net/2023/02/11/rcYBRxvKghmFdVt.png" style="zoom:80%;" />





## 环境准备
docker已部署

## 开始安装
### 1.启动docker，下载Jenkins镜像文件
```
docker pull jenkinsci/blueocean
```

### 2.创建Jenkins挂载目录并授权权限

我们在服务器上先创建一个jenkins工作目录 /var/jenkins_mount，赋予相应权限，稍后我们将jenkins容器目录挂载到这个目录上，这样我们就可以很方便地对容器内的配置文件进行修改。 如果我们不这样做，修改容器配置文件将会有点麻烦，因为虽然我们可以使（docker exec -it --user root 容器id /bin/bash）命令进入容器，但是容器中连简单的vi命令都没有安装。
```
mkdir -p /var/jenkins_mount
chmod 777 /var/jenkins_mount
```

### 3.创建并启动Jenkins容器
```
docker run -d -p 8081:8080 -p 50001:50000 \
-v /var/jenkins_mount:/var/jenkins_home \
-v /etc/localtime:/etc/localtime \
--name myjenkins jenkinsci/blueocean

命令详解：
　　-d 后台运行镜像
　　-p 8081:8080		将镜像的8080端口映射到服务器的8081端口。
　　-p 50001:50000	将镜像的50000端口映射到服务器的50001端口
　　-v /var/jenkins_mount:/var/jenkins_hone	/var/jenkins_home目录为容器jenkins工作路径，将其挂载到硬盘上的/var/jenkins_mount路径
　　-v /etc/localtime:/etc/localtime	让容器使用和服务器同样的时间设置。
　　--name myjenkins	给容器起一个别名
```

### 4.查看jenkins是否启动成功，如下图出现端口号，就为启动成功了
```
docker ps -l
```

### 5.查看docker容器日志
```
docker logs myjenkins
```

### 6.配置镜像加速
```
cd /var/jenkins_mount/
修改 vim hudson.model.UpdateCenter.xml
修改前
<sites>
  <site>
    <id>default</id>
    <url>https://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
将 url 修改为 清华大学官方镜像：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

修改后
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>
  </site>
</sites>
```


### 7.WEB页面配置
#### 1> 等待10分钟左右，浏览器访问：http://192.168.3.151:8081/  

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E8%A7%A3%E9%94%81.png" style="zoom:30%;" />  

<font color="red">注意：</font>
这里显示的路径是容器内的路径，我们不进入容器查看，而是到挂载的本地路径去查看。
容器路径 /var/jenkins_home/secrets/initialAdminPassword
本地路径 /var/jenkins_mount/secrets/initialAdminPassword

**使用此密码登录安装**
cat /var/jenkins_mount/secrets/initialAdminPassword 
3e59c41fc1f74e6c8bbd02c3bfd35f8b

#### 2> 自定义Jenkins安装插件（选择推荐安装）
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E8%87%AA%E5%AE%9A%E4%B9%89%E5%AE%89%E8%A3%85.png" style="zoom:30%;" /> 

#### 3> Jenkins进入安装程序，这里等待10分钟左右
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E5%BC%80%E5%A7%8B%E5%AE%89%E8%A3%85.png" style="zoom:30%;" />

                        出现如下页面证明Jenkins安装成功，我们点击“重启”按钮
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E5%88%9B%E5%BB%BA%E5%AE%8C%E6%88%90%E9%87%8D%E5%90%AF.png" style="zoom:30%;" />

#### 4> Jenkins安装完成，我们接续浏览器访问：http://192.168.3.151:8081/ 可登录Jenkins管理平台
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E7%99%BB%E5%BD%95.png" style="zoom:30%;" />

**到这里我们的 Jenkins 就已经安装完成了，在下节我们讲述Jenkins的使用。**

