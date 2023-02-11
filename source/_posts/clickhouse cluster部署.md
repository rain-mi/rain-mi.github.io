---
title: clickhouse cluster部署
date: 2022-04-03 15:30:05
categories: clickhouse
tags:
---



## 一 基础配置
### 1 配置/etc/hosts
必须在/etc/hosts配置主机名和ip地址映射关系， 否则副本之间的数据不能同步。
```
cat /etc/hosts
172.18.8.201    clickhouse1
172.18.8.202    clickhouse2
172.18.8.203    clickhouse3
172.18.8.204    clickhouse4
```

### 2 安装JDK
```rpm -ivh jdk-8u321-linux-x64.rpm```

设置jdk环境变量
```
cat <<EOF>>/etc/profile
##JDK##
JAVA_HOME=/usr/java/jdk1.8.0_321-amd64/
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export PATH
export CLASSPATH
EOF

使环境变量生效
source /etc/profile

查看Java版本
java -version
```



## 二 部署zookeeper集群
### 1 解压安装
```
cd /package/
tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz 
mv apache-zookeeper-3.6.3-bin /data/zookeeper
```

### 2 配置环境变量
```
vim /etc/profile
------------------------------------------------
##zookeeper##
export ZOOKEEPER_HOME=/data/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin

拷贝到其他服务器
scp /etc/profile 172.18.8.202:/etc/
scp /etc/profile 172.18.8.203:/etc/

source /etc/profile
```

### 3 配置zookeeper
```
cd /data/zookeeper/conf/
cp zoo_sample.cfg zoo.cfg 
vim zoo.cfg
-------------------------------------------------------
tickTime=2000		# 客户端心跳时间(毫秒)
initLimit=10		# 允许心跳间隔的最大时间
syncLimit=5		# 同步时限.
dataDir=/data/zookeeper/data	# 数据存储目录
dataLogDir=/data/zookeeper/logs 	# 数据日志存储目录
clientPort=2181		# 端口号

# 集群节点和服务端口配置
server.1=172.18.8.201:2888:3888
server.2=172.18.8.202:2888:3888
server.3=172.18.8.203:2888:3888

#以下为优化配置
maxClientCnxns=0	#服务器最大连接数，默认为10，改为0表示无限制
autopurge.snapRetainCount=3	# 快照数
autopurge.purgeInterval=1	# 快照清理时间，默认为0

A 创建zookeeper的数据存储目录和日志存储目录
mkdir -pv /data/zookeeper/{logs,data}

B 在data目录中创建一个文件myid，输入内容为1
echo "1" >> /data/zookeeper/data/myid
```

```
cd bin/
vim zkEnv.sh 
----------------------------------------
if [ "x${ZOO_LOG_DIR}" = "x" ]
then
   ZOO_LOG_DIR="$ZOOKEEPER_HOME/logs"
fi

if [ "x${ZOO_LOG4J_PROP}" = "x" ]
then
   ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
fi
```


```
vim conf/log4j.properties 
----------------------------------------
zookeeper.root.logger=INFO, ROLLINGFILE
----------------------------------------
```

同步到其他服务器
```
scp -r /data/zookeeper 172.18.8.202:/data/
scp -r /data/zookeeper 172.18.8.203:/data/
```

更改slave2、3的myid
```
echo "2" > /data/zookeeper/data/myid
echo "3" > /data/zookeeper/data/myid
```

启动zookeeper
```/data/zookeeper/bin/zkServer.sh start```

查看zookeeper状态
```/data/zookeeper/bin/zkServer.sh status```




## 三 部署clickhouse集群
### 1 安装clickhouse（所有节点）
```
设置clickhouse yum源
yum install -y yum-utils
yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo

安装clickhouse
yum install -y clickhouse-server clickhouse-client

启动clickhouse
/etc/init.d/clickhouse-server start

登录clickhouse
clickhouse-client
```

### 2 修改配置

```vim /etc/clickhouse-server/config.xml```

- 修改listen_host


```
    <!-- Default values - try listen localhost on ipv4 and ipv6: -->
    <!--
    <listen_host>::1</listen_host>
    <listen_host>127.0.0.1</listen_host>
    -->
    <listen_host>0.0.0.0</listen_host> <!-- 新增所有地址可以访问 -->

```

- 修改存储路径


```
<!-- Path to data directory, with trailing slash. -->
    <path>/data/clickhouse/</path>  <!-- 修改存储路径 -->

    <!-- Path to temporary data for processing hard queries. -->
    <tmp_path>/data/clickhouse/tmp/</tmp_path> 
```

- 添加集群配置

```
    <remote_servers>
        <bigdata> <!-- 集群名字，自定义 -->
            <shard> <!-- 定义一个分片 -->
                <!-- Optional. Shard weight when writing data. Default: 1. -->
                <weight>1</weight>
                <!-- Optional. Whether to write data to just one of the replicas. Default: false (write data to all replicas). -->
                <internal_replication>false</internal_replication>
                <replica> <!-- 这个分片的副本存在在哪些机器上 -->
                    <host>172.18.8.201</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>172.18.8.202</host>
                    <port>9000</port>
                </replica>
            </shard>
           <!--  
            <shard>
                <weight>2</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>172.18.8.203</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>172.18.8.204</host>
                    <port>9000</port>
                </replica>
            </shard>
            -->
        </bigdata>
    </remote_servers>  
```

- 添加zookeeper配置

```
    <zookeeper incl="zookeeper-servers" optional="true" />
    <zookeeper>
        <node index="1">
            <host>172.18.8.201</host>
            <port>2181</port>
        </node>
        <node index="2">
            <host>172.18.8.202</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>172.18.8.203</host>
            <port>2181</port>
        </node>
    </zookeeper>  
```

- 配置分片macros变量

```
    <macros incl="macros" optional="true" />
        <!-- 配置分片macros变量，在用client创建表的时候回自动带入 -->
        <macros>
        <shard>1</shard>
        <replica>172.18.8.201</replica> <!-- 这里指定当前集群节点的名字或者IP -->
    </macros>
```

### 3 启动集群
配置完成后， 在每个节点启动ClickHouse服务。
```systemctl restart clickhouse-server```

查看系统表
```select * from system.clusters where cluster='bigdata';```



### 4 测试

全部节点分别创建数据库：
```create database test1;```

一台建表建立数据

```
CREATE TABLE t1 ON CLUSTER bigdata
(
    `ts` DateTime,
    `uid` String,
    `biz` String
)
ENGINE = ReplicatedMergeTree('/ClickHouse/test1/tables/{shard}/t1', '{replica}')
PARTITION BY toYYYYMMDD(ts)
ORDER BY ts
SETTINGS index_granularity = 8192
######说明 {shard}自动获取对应配置文件的macros分片设置变量 replica一样  ENGINE = ReplicatedMergeTree，不能为之前的MergeTree
######'/ClickHouse/test1/tables/{shard}/t1' 是写入zk里面的地址，唯一，注意命名规范

INSERT INTO t1 VALUES ('2019-06-07 20:01:01', 'a', 'show');
INSERT INTO t1 VALUES ('2019-06-07 20:01:02', 'b', 'show');
INSERT INTO t1 VALUES ('2019-06-07 20:01:03', 'a', 'click');
INSERT INTO t1 VALUES ('2019-06-08 20:01:04', 'c', 'show');
INSERT INTO t1 VALUES ('2019-06-08 20:01:05', 'c', 'click');
```
其它几台机器查看数据，如果数据查询到了 ，并且一致，则成功