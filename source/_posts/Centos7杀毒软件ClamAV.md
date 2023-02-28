---
title: Centos7杀毒软件ClamAV
date: 2023-02-11 12:06:36
categories: ClamAV
tags:
---

##  安装
1.1 安装基础依赖

```
yum groupinstall -y "Development Tools"
yum install -y openssl-devel sendmail-devel libcurl-devel check-devel
```



1.2 源码编译安装

```
tar -zxvf clamav-0.103.7.tar.gz 
cd clamav-0.103.7/
./configure '--build=x86_64-redhat-linux-gnu' \
'--host=x86_64-redhat-linux-gnu' \
'--target=x86_64-redhat-linux-gnu' \
'--prefix=/usr' \
'--bindir=/usr/bin' \
'--sbindir=/usr/sbin' \
'--libexecdir=/usr/libexec' \
'--sysconfdir=/etc' \
'--localstatedir=/var' \
'--libdir=/usr/lib64' \
'--includedir=/usr/include' \
'--datadir=/usr/share' \
'--infodir=/usr/share/info' \
'--localedir=/usr/share/locale' \
'--mandir=/usr/share/man' \
'--docdir=/usr/share/doc/clamav-0.103.7' \
'--exec-prefix=/usr' \
'--sharedstatedir=/var/lib' \
'--program-prefix=' \
'--enable-milter' \
'--disable-clamav' \
'--disable-static' \
'--disable-zlib-vcheck' \
'--disable-unrar' \
'--enable-id-check' \
'--enable-dns' \
'--with-dbdir=/var/lib/clamav' \
'--with-group=clamav' \
'--with-user=clamav' \
'--with-zlib=/usr' \
'--enable-ltdl-convenience' \
'--enable-check' \
'--with-systemdsystemunitdir=no' \
'build_alias=x86_64-redhat-linux-gnu' \
'host_alias=x86_64-redhat-linux-gnu' \
'target_alias=x86_64-redhat-linux-gnu' \
'CXXFLAGS=-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic' \
'LDFLAGS= -Wl,-z,relro' \
'CFLAGS=-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic -fPIC' \
'PKG_CONFIG_PATH=:/usr/lib64/pkgconfig:/usr/share/pkgconfig'

make -j8
make install
```



##  配置数据更新

2.1 修改配置文件

```
cp /etc/freshclam.conf.sample /etc/freshclam.conf

## 修改如下
vim /etc/freshclam.conf
--------------------------------
# Example
DatabaseDirectory /var/lib/clamav
UpdateLogFile /var/log/clamav/freshclam.log
LogSyslog yes
DatabaseOwner clamav
DatabaseMirror database.clamav.net
```



2.2 根据配置创建所需的运行用户

```
groupadd  -g 498 clamav
useradd -u 498 -g 498 -d /var/lib/clamav -s /sbin/nologin -c "Clam Anti Virus Checker" clamav
```



2.3 创建所需的目录或修改已有目录的权限

```
mkdir /var/log/clamav
chown clamav:clamav /var/lib/clamav/ /var/log/clamav
chmod 775 -R /var/lib/clamav/ /var/log/clamav
```



2.4 设置selinux

```
setsebool -P antivirus_can_scan_system 1
```



2.5 测试更新

```
freshclam
```



2.6 配置自动更新

```
crontab -e
---------------------
47  *  *   *    *  /usr/bin/freshclam --quiet
```





##  测试扫描

```
-r 递归扫描子目录
-i 只显示发现的病毒文件
clamscan  -r  /

## 扫描文件 
clamscan ~

## 递归扫描home目录，并且记录日志
clamscan -r -i /home -l /var/log/clamav/clamscan.log 

## 递归扫描home目录，将病毒文件删除，并且记录日志 
clamscan -r -i /home --remove -l /var/log/clamav/clamscan.log 

## 扫描指定目录，然后将感染文件移动到指定目录，并记录日志(生产环境建议使用此方法)
mkdir /opt/infected
clamscan -r -i /home --move=/opt/infected -l /var/log/clamav/clamscan.log
```





##  配置ClamAV守护进程

4.1 修改配置文件

```
cp /etc/clamd.conf.sample /etc/clamd.conf

## 修改如下
vim /etc/clamd.conf
---------------------------------------
# Example
LogFile /var/log/clamav/clamd.log
LogFileMaxSize 0
LogTime yes
LogSyslog yes
PidFile /var/run/clamav/clamd.pid
TemporaryDirectory /var/tmp
DatabaseDirectory /var/lib/clamav
LocalSocket /var/run/clamav/clamd.sock
FixStaleSocket yes
TCPSocket 3310
TCPAddr 127.0.0.1
MaxConnectionQueueLength 30
MaxThreads 50
ReadTimeout 300
User clamav
ScanPE yes
ScanELF yes
ScanOLE2 yes
ScanMail yes
ScanArchive yes
ArchiveBlockEncrypted no

```



4.2 创建所需的文件夹

```
mkdir /var/run/clamav
chown clamav:clamav /var/run/clamav
```



4.3 配置系统服务

```
vim /etc/init.d/clamd
---------------------------------------------
#!/bin/sh
#
# Startup script for the Clam AntiVirus Daemon
#
# chkconfig: - 61 39
# description: Clam AntiVirus Daemon is a TCP/IP or socket protocol \
#              server.
# processname: clamd
# pidfile: /var/run/clamav/clamd.pid
# config: /etc/clamd.conf

pidfile=/var/run/clamav/clamd.pid
sockfile=/var/run/clamav/clamd.pid
lockfile=/var/lock/subsys/clamd
config=/etc/clamd.conf
user=clamav
group=clamav

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

[ -x /usr/sbin/clamd ] || exit 0

# Local clamd config
test -f /etc/sysconfig/clamd && . /etc/sysconfig/clamd

# See how we were called.
case "$1" in
  start)
        echo -n "Starting Clam AntiVirus Daemon: "
        piddir=`dirname $pidfile`
        if [ ! -d $piddir ]; then
          mkdir -p $piddir
          chown $user:$group $piddir
        fi
        sleep 1
        daemon clamd -c $config
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch $lockfile
        ;;
  stop)
        echo -n "Stopping Clam AntiVirus Daemon: "
        killproc clamd
        rm -f $sockfile
        rm -f $pidfile
        RETVAL=$?
        echo
### heres the fix... we gotta remove the stale files on restart
        [ $RETVAL -eq 0 ] && rm -f $lockfile
        ;;
  status)
        status clamd
        RETVAL=$?
        ;;
  restart|reload)
        $0 stop
        $0 start
        RETVAL=$?
        ;;
  condrestart)
        [ -e $lockfile ] && $0 restart
        RETVAL=$?
        ;;
  *)
        echo "Usage: clamd {start|stop|status|restart|reload|condrestart}"
        exit 1
esac

exit $RETVAL

```



4.4 启动clamd

```
chmod +x /etc/init.d/clamd
/etc/init.d/clamd start
chkconfig clamd on
```



4.5 另外，如果遇到SELinux无法启动问题，可以使用如下命令反复调试，

```
ausearch -c 'clamd' --raw | audit2allow -M my-clamd
semodule -X 300 -i my-clamd.pp
```



4.6 或者，你可以选择关闭SELinux，

```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```





##  离线更新病毒库

```
## 主机不能上网，可以在能上网的主机更新再上传过来
## 或者直接下载病毒库文件放到 /var/lib/clamav 下

http://database.clamav.net/main.cvd
http://database.clamav.net/daily.cvd
http://database.clamav.net/bytecode.cvd
注：可以用windows主机上讯雷下载再传上来

/var/lib/clamav

## 重新加载病毒库
clamdscan --reload

## 查看病毒版本
clamdscan --version
```

