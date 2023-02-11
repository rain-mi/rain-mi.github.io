---
title: jenkins之二：使用
date: 2021-05-24 11:50:34
categories: jenkins
tags:
---

# 一.基础环境准备
在linux服务器中安装git, maven配置git的公钥到你的github上，这些步骤是使用jenkins的前提。

推荐安装两个插件，在系统管理中可以安装插件：
Rebuilder
Safe Restart

## 1.安装git
yum install -y git
 

## 2.maven部署

### 1>. 安装jdk
rpm -ivh jdk-8u201-linux-x64.rpm
```
配置环境变量
cat > /etc/profile.d/java.sh << EOF
JAVA_HOME=/usr/java/jdk1.8.0_201-amd64/
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export PATH
export CLASSPATH
EOF

source /etc/profile.d/java.sh 
# java -version
java version "1.8.0_25"		//显示jdk版本
```

### 2>. 下载
```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
也可以在浏览器去maven官网下载需要的版本，这里安装的是二进制包，所以选择“-bin.tar.gz”结尾的包

```
### 3>. 解压
```
tar -zxvf apache-maven-3.8.1-bin.tar.gz -C /data && cd /data
mv apache-maven-3.8.1/ maven3.8
```

### 4>. 加入环境变量
```
在/etc/profile文件最下方加入新的一行export PATH=$PATH:/data/maven3.8/bin
echo "export PATH=$PATH:/data/maven3.8/bin" >> /etc/profile

使环境变量生效
source /etc/profile

验证：
执行which mvn
显示/data/maven3.8/bin/mvn就说明配置成功了
```

## 3.创建jenkins目录
```
mkdir -pv /data/jenkens
#用来存储拉取下来的项目代码
```

## 4.与远程仓库实现密钥连接
```
生成密钥：
ssh-keygen -t rsa -C "admin@abc.com"

把家目录中生成的公钥内容复制到github或其他仓库上。  
cd ~ && cat .ssh/id_rsa.pub 
```

# 二.Jenkins配置

## 1.将node服务器注册到Jenkins上
### 1>.在jenkins中选择系统管理->新建节点
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/%E6%96%B0%E5%BB%BA%E8%8A%82%E7%82%B9.png" style="zoom:30%;" /> 

### 2>.填写如下
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/%E9%85%8D%E7%BD%AE%E8%8A%82%E7%82%B9.png" style="zoom:30%;" /> 
远程工作目录即创建的jenkins目录。
在Credentials添加一个远程用户，输入远程机器用户名和密码保存。

### 3>.返回节点页面可以看到我们新建的node节点
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/node%E8%8A%82%E7%82%B9%E7%8A%B6%E6%80%81.png" style="zoom:30%;" /> 
``

``
## 2.配置自动化部署
### 1>.自动化部署原理
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%B5%81%E7%A8%8B.png" style="zoom:100%;" /> 
```
具体的创建Jenkins任务的过程为
a.创建jenkins任务
b.填写Server信息
c.配置git参数
d.填写构建语句（shell脚本）,实现自动部署。
```

### 2>.开启git
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/%E9%85%8D%E7%BD%AEgit.png" style="zoom:30%;" /> 

### 3>.在jenkins工作目录添加构建脚本
```
cat > deploy.sh << EOF
#!/bin/bash
# 项目路径, 在Execute Shell中配置项目路径, pwd 就可以获得该项目路径
export PROJ_PATH=/data/jenkins
# 输入你的环境上tomcat的全路径
export TOMCAT_APP_PATH=/data/tomcat
### base 函数
killTomcat()
{
    #pid=`ps -ef|grep tomcat|grep java|awk '{print $2}'`
    #echo "tomcat Id list :$pid"
    #if [ "$pid" = "" ]
    #then
    #  echo "no tomcat pid alive"
    #else
    #  kill -9 $pid
    #fi
    #上面注释的或者下面的
    cd $TOMCAT_APP_PATH/bin
    sh shutdown.sh
}
cd $PROJ_PATH/my-scrum
#mvn clean install
# 停tomcat
killTomcat
# 删除原有工程
rm -rf $TOMCAT_APP_PATH/webapps/ROOT
rm -f $TOMCAT_APP_PATH/webapps/ROOT.war
rm -f $TOMCAT_APP_PATH/webapps/my-scrum.war
# 复制新的工程到tomcat上
mkdir -p $TOMCAT_APP_PATH/webapps/ROOT
cp $PROJ_PATH/workspace/tomcat/my-java/* $TOMCAT_APP_PATH/webapps/ROOT
#cd $TOMCAT_APP_PATH/webapps/
#mv my-scrum.war ROOT.war
# 启动Tomcat
cd $TOMCAT_APP_PATH/
sh bin/startup.sh
EOF
```
### 4>新建任务
#### a.在jenkins上新建任务，填写任务名
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E6%96%B0%E5%BB%BA%E4%BB%BB%E5%8A%A1.png" style="zoom:30%;" /> 

#### b.选择节点
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E4%BB%BB%E5%8A%A1-%E9%80%89%E6%8B%A9%E8%8A%82%E7%82%B9.png" style="zoom:30%;" /> 

#### c.配置git仓库地址
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E4%BB%BB%E5%8A%A1-%E9%85%8D%E7%BD%AEgit.png" style="zoom:30%;" /> 

#### d.配置下载到jenkins路径下的指定目录
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E4%BB%BB%E5%8A%A1-%E9%85%8D%E7%BD%AE%E9%A1%B9%E7%9B%AE%E7%9B%AE%E5%BD%95.png" style="zoom:30%;" /> 

#### e.选择构建“运行shell”，并写入脚本
```
#当jenkins进程结束后新开的tomcat进程不被杀死
BUILD_ID=DONTKILLME
#加载变量
. /etc/profile
#配置运行参数
#PROJ_PATH为设置的jenkins目录的执行任务目录
export PROJ_PATH=/data/jenkins
#配置tomcat所在目录
export TOMCAT_APP_PATH=/data/tomcat
#执行写好的自动化部署脚本
sh /data/jenkins/deploy.sh
```
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E4%BB%BB%E5%8A%A1-%E6%9E%84%E5%BB%BA.png" style="zoom:30%;" /> 

# 三.执行构建
## 1.在jenkins页面单击构建按钮进行任务构建
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E4%BB%BB%E5%8A%A1-%E9%80%89%E6%8B%A9%E8%8A%82%E7%82%B9.png" style="zoom:30%;" /> 

## 2.打开浏览器访问自动部署到网站
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/jenkins%E6%9E%84%E5%BB%BA-%E6%B5%8B%E8%AF%95.png" style="zoom:50%;" />

**至此，我们已经完成了一次自动化的构建部署。** 
