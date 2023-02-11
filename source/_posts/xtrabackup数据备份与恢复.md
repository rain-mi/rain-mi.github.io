---
title: xtrabackup数据备份与恢复
date: 2021-05-25 17:17:27
categories: MySQL
tags:
---
# 概述

xtrabackup是Percona公司CTO Vadim参与开发的一款基于InnoDB的在线热备工具，具有开源，免费，支持在线热备，备份恢复速度快，占用磁盘空间小等特点，并且支持不同情况下的多种备份形式。
xtrabackup的官方下载地址为http://www.percona.com/software/percona-xtrabackup

xtrabackup包含两个主要的工具，即xtrabackup和innobackupex，二者区别如下：
（1）xtrabackup只能备份innodb和xtradb两种引擎的表，而不能备份myisam引擎的表；
（2）innobackupex是一个封装了xtrabackup的Perl脚本，支持同时备份innodb和myisam，但在对myisam备份时需要加一个全局的读锁。还有就是myisam不支持增量备份。

  percona xtrabackup 8 概述：
  1.移除了innobackupex命令
  2.由于新的MySQL重做日志和数据字典格式，8.0版本只支持mysql8.0和percona8.0
  3.早于mysql8.0的版本需要使用xtrabackup2.4备份和恢复.

# 一.基本环境准备
## 1.安装使用xtrabackup
```
cd /package/
tar -zxvf percona-xtrabackup-2.4.22-Linux-x86_64.glibc2.12.tar.gz -C /usr/local/
mv /usr/local/percona-xtrabackup-2.4.22-Linux-x86_64.glibc2.12/ /usr/local/xtrabackup
echo "export PATH=\$PATH:/usr/local/xtrabackup/bin" >> /etc/profile
source /etc/profile
```

## 2.创建数据库测试数据
```
创建数据库
> create database client;

使用数据库
> use client

创建数据表
create table user_info (身份证 int(20),姓名 char(20),性别 char(2),用户ID号 int(110),资费 int(10));

插入数据到数据表
insert into user_info values ('000000001','孙空武','男','011','100');
insert into user_info values ('000000002','蓝凌','女','012','98');
insert into user_info values ('000000003','猪八戒','女','013','12');

查看
mysql> select * from user_info;
+-----------+-----------+--------+-------------+--------+
| 身份证     | 姓名      | 性别   | 用户ID号    | 资费   |
+-----------+-----------+--------+-------------+--------+
|         1 | 孙空武    | 男     |          11 |    100 |
|         2 | 蓝凌      | 女     |          12 |     98 |
|         3 | 姜文      | 女     |          13 |     12 |
+-----------+-----------+--------+-------------+--------+
```

# 二.完全备份与恢复
## 1.完全备份
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--user=root --password=root \
--backup /data/backup
```

## 2.故障模拟
```
删除数据库，然后恢复全备
drop database client;

恢复备份到mysql的数据文件目录，这一过程要先关闭mysql数据库，重命名或者删除原数据文件目录，再创建一个新的数据文件目录，将备份数据复制到新的数据文件目录下，赋权，修改权限，启动数据库
service mysqld stop
mv /data/db /data/db_bak
mkdir /data/db
```

## 3.xtrabackup创建恢复数据
```
innobackupex --apply-log /data/backup/2021-04-25_21-16-25/
```

## 4.将数据恢复到数据库
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--copy-back --rsync /data/backup/2021-04-25_21-16-25/

修改数据目录权限
chown -R mysql:mysql /data/db

启动MySQL
service mysqld start
```

## 5.查看数据恢复情况
```
查看数据库
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| client             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

查看表
mysql> select * from user_info;
+-----------+-----------+--------+-------------+--------+
| 身份证    | 姓名      | 性别   | 用户ID号    | 资费   |
+-----------+-----------+--------+-------------+--------+
|         1 | 孙空武    | 男     |          11 |    100 |
|         2 | 蓝凌      | 女     |          12 |     98 |
|         3 | 猪八戒    | 女     |          13 |     12 |
+-----------+-----------+--------+-------------+--------+
```


# 三.增量备份

## 1.创建备份路径
在进行增量备份时，首先要进行一次全量备份，第一次增量备份是基于全备的，之后的增量备份是基于上一次的增量备份，以此类推。
```
全备份放在/data/backup/full,增量备份放在/data/backup/incremental
mkdir -pv /data/backup/{full,incremental}
```

## 2.第一次全备
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--user=root --password=root \
--backup /data/backup/full
```

## 3.为了测试效果，我们在user_info表中插入数据
```
insert into user_info values ('000000004','关园','男','014','38');
insert into user_info values ('000000005','罗中昆','男','015','39');

mysql> select * from user_info;
+-----------+-----------+--------+-------------+--------+
| 身份证    | 姓名      | 性别   | 用户ID号    | 资费   |
+-----------+-----------+--------+-------------+--------+
|         1 | 孙空武    | 男     |          11 |    100 |
|         2 | 蓝凌      | 女     |          12 |     98 |
|         3 | 猪八戒    | 女     |          13 |     12 |
|         4 | 关园      | 男     |          14 |     38 |
|         5 | 罗中昆    | 男     |          15 |     39 |
+-----------+-----------+--------+-------------+--------+
```

## 4.增量备份
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--user=root --password=root \
--incremental /data/backup/incremental/ \
--incremental-basedir=/data/backup/full/2021-04-25_21-37-12/ --parallel=2

查看增量备份的大小以及文件内容
--全
du -sh /data/backup/full/2021-04-25_21-37-12/
26M     /data/backup/full/2021-04-25_21-37-12/

--增
du -sh /data/backup/incremental/2021-04-25_21-41-59/
3.3M    /data/backup/incremental/2021-04-25_21-41-59/
#看见增量备份的数据很小吧，就是备份改变的数据而已。


查看增量备份信息
# pwd
/data/backup/incremental/2021-04-25_21-41-59

# cat xtrabackup_checkpoints 
backup_type = incremental
from_lsn = 2574907
to_lsn = 2578906
last_lsn = 2578915
compact = 0
recover_binlog_info = 0
flushed_lsn = 2578915
```

## 5.我们再次在user_info表中插入数据
```
insert into user_info values ('000000006','关晓彤','女','016','38');
insert into user_info values ('000000007','刘亦菲','女','017','39');

mysql> select * from user_info;
+-----------+-----------+--------+-------------+--------+
| 身份证    | 姓名      | 性别   | 用户ID号    | 资费   |
+-----------+-----------+--------+-------------+--------+
|         1 | 孙空武    | 男     |          11 |    100 |
|         2 | 蓝凌      | 女     |          12 |     98 |
|         3 | 猪八戒    | 女     |          13 |     12 |
|         4 | 关园      | 男     |          14 |     38 |
|         5 | 罗中昆    | 男     |          15 |     39 |
|         6 | 关晓彤    | 女     |          16 |     38 |
|         7 | 刘亦菲    | 女     |          17 |     39 |
+-----------+-----------+--------+-------------+--------+
```

## 6.第二次增量备份
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--user=root --password=root \
--incremental /data/backup/incremental/ \
--incremental-basedir=/data/backup/incremental/2021-04-25_21-41-59/ --parallel=2
```

## 7.查看备份文件
```
# ls -ltr /data/backup/full/
drwxr-x--- 6 root root 237 4月  25 21:37 2021-04-25_21-37-12

# ls -ltr /data/backup/incremental/
drwxr-x--- 6 root root 263 4月  25 21:42 2021-04-25_21-41-59
drwxr-x--- 6 root root 263 4月  25 21:55 2021-04-25_21-55-41
```

# 四.增量备份恢复

增量备份的恢复大体为3个步骤
*恢复完全备份
*恢复增量备份到完全备份（开始恢复的增量备份要添加--redo-only参数，到最后一次增量备份去掉--redo-only参数）
*对整体的完全备份进行恢复，回滚那些未提交的数据

## 1.恢复完全备份
```
注意这里一定要加--redo-only参数，该参数的意思是只应用xtrabackup日志中已提交的事务数据，不回滚还未提交的数据
innobackupex --apply-log --redo-only /data/backup/full/2021-04-25_21-37-12/
```

## 2.将增量备份1应用到完全备份
```
innobackupex --apply-log --redo-only /data/backup/full/2021-04-25_21-37-12/ \
--incremental-dir=/data/backup/incremental/2021-04-25_21-41-59/
```

## 3.将增量备份2应用到完全备份
```
注意恢复最后一个增量备份时需要去掉--redo-only参数，回滚xtrabackup日志中那些还未提交的数据
innobackupex --apply-log /data/backup/full/2021-04-25_21-37-12/ \
--incremental-dir=/data/backup/incremental/2021-04-25_21-55-41/
```

## 4.故障模拟
```
drop database client;
service mysqld stop
mv /data/db /data/db_bak2
mkdir /data/db
```

## 5.把恢复完的备份反应到数据库中
```
innobackupex --defaults-file=/data/mysql/etc/my.cnf \
--copy-back --rsync /data/backup/full/2021-04-25_21-37-12/
```

## 6.赋权，然后启动mysql数据库
```
修改数据目录权限
chown -R mysql:mysql /data/db
service mysqld start
```

## 7.检测数据正确性
```
查看数据
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| client             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+

查看表
mysql> select * from user_info;
+-----------+-----------+--------+-------------+--------+
| 身份证    | 姓名      | 性别   | 用户ID号    | 资费   |
+-----------+-----------+--------+-------------+--------+
|         1 | 孙空武    | 男     |          11 |    100 |
|         2 | 蓝凌      | 女     |          12 |     98 |
|         3 | 猪八戒    | 女     |          13 |     12 |
|         4 | 关园      | 男     |          14 |     38 |
|         5 | 罗中昆    | 男     |          15 |     39 |
|         6 | 关晓彤    | 女     |          16 |     38 |
|         7 | 刘亦菲    | 女     |          17 |     39 |
+-----------+-----------+--------+-------------+--------+
```

**至此，我们已经完成XtraBackup完全备份与恢复、增量备份与恢复**


