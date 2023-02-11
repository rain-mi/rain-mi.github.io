---
title: rhel6安装oracle11gr2之一：安装
date: 2021-12-08 22:26:13
categories: oracle
tags:
---

# 一、安装基础依赖包
```
# yum install -y compat-libcap1 binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel gcc gcc-c++ glibc glibc-common glibc-devel glibc-headers libaio libaio-devel libgcc libstdc++ libstdc++-devel make numactl-devel sysstat unixODBC-2.2.14-14.el6.x86_64 unixODBC-2.2.14-14.el6.i686 unixODBC-devel-2.2.14-14.el6.x86_64 unixODBC-devel-2.2.14-14.el6.i686
# rpm -ivh pdksh-5.2.14-37.el5_8.1.x86_64.rpm
```
```
Oracle官方文档中确定要安装的包如下:
    binutils-2.17.50.0.6
    compat-libstdc++-33-3.2.3
    compat-libstdc++-33-3.2.3 (32 bit)
    elfutils-libelf-0.125
    elfutils-libelf-devel-0.125
    gcc-4.1.2
    gcc-c++-4.1.2
    glibc-2.5-24
    glibc-2.5-24 (32 bit)
    glibc-common-2.5
    glibc-devel-2.5
    glibc-devel-2.5 (32 bit)
    glibc-headers-2.5
    ksh-20060214
    libaio-0.3.106
    libaio-0.3.106 (32 bit)
    libaio-devel-0.3.106
    libaio-devel-0.3.106 (32 bit)
    libgcc-4.1.2
    libgcc-4.1.2 (32 bit)
    libstdc++-4.1.2
    libstdc++-4.1.2 (32 bit)
    libstdc++-devel 4.1.2
    make-3.81
    numactl-devel-0.9.8.x86_64
    sysstat-7.0.2
    unixODBC-2.2.14-12.el6_3.i686.rpm
    unixODBC-2.2.14-12.el6_3.x86_64.rpm
    unixODBC-devel-2.2.14-12.el6_3.i686.rpm
    unixODBC-devel-2.2.14-12.el6_3.x86_64.rpm
    
    注:
    (1)安装glibc-2.17-55.el7.i686.rpm时，因为依赖包相互依赖，需要与依赖包同时安装:
    rpm -ivh glibc-2.17-55.el7.i686.rpm nss-softokn-freebl-3.15.4-2.el7.i686.rpm
    (2)RHEL7中缺少包compat-libstdc++-33，可以在RHEL6中找到.
    (3)在Oracle数据库软件安装时,ksh实际是用的pdksh，但ksh也是可以使用的，只是安装时会有警告，pdksh是一个早期的软件包，可以在以下网站找到.
    http://rpm.pbone.net/
```

# 二、系统参数配置

## 2.1 系统语言
```
vim /etc/sysconfig/i18n 
LANG="zh_CN.UTF-8"
LANG="en_US.UTF-8"
SYSFONT="latarcyrheb-sun16"
```

## 2.2 内核参数调优
```
vim /etc/sysctl.conf 
-------------------------------
# for oracle
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 2001117184
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586
-------------------------------
使内核生效
# /sbin/sysctl -p
```

```
# vim /etc/security/limits.conf
--------------------------------------
# for oracle
oracle              soft    nproc   2047
oracle              hard    nproc   16384
oracle              soft    nofile  1024
oracle              hard    nofile  65536
oracle              soft    stack   10240
--------------------------------------
```

```
# vim /etc/pam.d/login(limits.conf是pam_limits.so的配置文件)
session    required     /lib/security/pam_limits.so
```

## 2.3 创建oracle用户与组
```
groupadd oinstall
groupadd dba
useradd -g oinstall -G dba oracle
echo "oracle" | passwd --stdin oracle
```

## 2.4 配置/etc/hosts文件
```
# vim /etc/hosts

添加如下
192.168.8.12    sq1
```

## 2.5 创建oracle安装目录 
```
mkdir -pv /oracle/app
chmod 770 /oracle
chown -R oracle.oinstall /oracle
```
## 2.6 设置oracle用户环境变量
``` 
# su - oracle
$ vim .bash_profile 
ORACLE_BASE=/oracle/app
ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
ORACLE_SID=orcl
PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin
LD_LIBRARY_PATH=$ORACLE_HOME/lib:/usr/lib
LANG=en_US.UTF-8
export ORACLE_BASE ORACLE_HOME ORACLE_SID PATH LD_LIBRARY_PATH LANG

注意：
  ORACLE_SID=orcl
  orcl为oracle数据实例名
  
让环境变量生效
$ source .bash_profile 
```


# 三、oracle安装

## 3.1 解压oracle软件包
```
unzip oracle_database_linux32.zip 
```

## 3.2 执行安装命令
```
以oracle 用户登陆 进入解压缩目录  
./runInstaller
注意：这里一定要以oracle用户登录。
```
## 3.3 进入oracle图形安装界面,安装oracle软件

### 3.3.1 取消登录账号
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall1.png" style="zoom:35% ;" />

### 3.3.2 确认取消登录账号
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall2.png" style="zoom:35% ;" />

### 3.3.3 跳过软件升级
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall3.png" style="zoom:35% ;" />

### 3.3.4 Installtion Option
选择“Install database software only”，只安装数据库软件，点击Next继续
第一项为安装数据库软件并创建一个数据库实例
第二项为只安装数据库软件
第三项为升级已经存在的数据库
选择第二项，可以在安装数据库软件后，手工通过dbca来创建实例。

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall4.png" style="zoom:35% ;" />

### 3.3.5 Grid Options
选择第一项点击Next继续。对于单节点的数据库选择“Single instance database installation”进行安装，如果是安装集群数据库则要选择“Real Application Clusters database installation”这个选项进行安装。

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall5.png" style="zoom:35% ;" />

### 3.3.6 Product Languages
这个界面上选择支持的语言，在左面列表里面选择“Simplified Chinese”，添加到右面列表，点击Next继续

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall6.png" style="zoom:35% ;" />

### 3.3.7 Database Edition
这个界面选择oracle的版本，选择第一项“Enterprise Edition”企业版，点击Next继续。
Standard版（标准版）的oracle有很多功能上的限制，生产系统上务必选择Enterprise Edition（企业版）。

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall7.png" style="zoom:35% ;" />

### 3.3.8 Installation Location
选择安装位置，配置好oracle用户的.bash_profile文件后，安装程序会自动把ORACLE_BASE变量作为Oracle Base，软件安装目录Software Location取的是ORACLE_HOME变量。不用修改，直接在界面点击Next继续

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall8.png" style="zoom:35% ;" />

### 3.3.9 Create Inventory
选择Inventory的位置，保持默认，点击Next继续。这个位置是在ORACLE_BASE下创建oraInventory目录，用于注册ORACLE_HOME下安装的数据库的组件及其版本，存放oracle软件安装的目录信息。oracle数据库软件的升级、增删组件，都需要用到inventory。oracle OUI会创建一个有oraInst.ora文件指定的全局inventory。

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall9.png" style="zoom:35% ;" />

### 3.3.10 Operating System Groups
选择oralce使用的操作系统用户组，保持默认，点击Next继续

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall10.png" style="zoom:35% ;" />

### 3.3.11 Prerequiste Checks
Oracle安装程序用检查系统参数，以确定是否满足了安装oracle的前提条件，包括系统内核、虚拟内存和软件包等。安装3.3章节介绍的方法把包安装好之后，由于操作系统的包版本过高，会导致oracle提示如下包缺失，实际上已经安装好了。勾选右上角的“Ignore All”复选框，点击Next继续：

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall11.png" style="zoom:35% ;" />

### 3.3.12 Install Product
Oracle开始按照前面步骤的设定，开始安装：

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall13.png" style="zoom:35% ;" />

### 3.3.13 Finish
使用root用户执行系统提示的2个脚本，完成安装：
/oracle /oraInventory/orainstRoot.sh
/oracle/app/oracle/product/11.2.0/dbhome_1/root.sh

<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall14.png" style="zoom:35% ;" />

对于脚本中的提示，直接回车确认即可，执行完毕后，点击OK安装过程就完成了。

### 3.3.14 数据库软件成功安装
<img src="https://rainmi.coding.net/p/CODING-Pages-1306006895/d/images/git/raw/master/oracleinstall15.png" style="zoom:35% ;" />

至此：Oracle 11gR2 软件安装完毕！




