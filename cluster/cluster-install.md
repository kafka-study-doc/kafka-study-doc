---
layout: kafka
title: 集群的安装配置
parent: Kafka集群
nav_order: 21
---
本文记录并介绍kafka集群的安装过程。

## 准备工作
1. 准备3台Linux机器，可以是虚拟机,我用的是 CentOS 7

服务器名 | IP | 说明
--|--|--
broker-1 | 192.168.1.85 | Kafka节点1
broker-2 | 192.168.1.86 | Kafka节点2
broker-3 | 192.168.1.87 | Kafka节点3

2. 三台机器都安装好JDK，我用的是JDK8

## 安装Zookeeper

### 1. 下载zookeeper
```
cd /usr/local
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz
```
解压
```
tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz
```
### 2. 编辑配置文件
```
cd apache-zookeeper-3.6.3-bin
cd conf
cp zoo_sample.cfg zoo.cfg
```
然后使用 vi zoo.cfg 命令编辑配置文件
```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# ...
PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true

server.1=192.168.1.85:2888:3888
server.2=192.168.1.86:2888:3888
server.3=192.168.1.87:2888:3888

```
在文件后面加上 server.1, server.2, server.3 这三行 。 IP地址根据你的实际情况修改


### 3. 编辑myid. 
根据上面配置的 dataDir 配置项。到对应的目录里写入myid的值。比如上面的server.1 那么这台机器的 myid时就写1.

确保dataDir对应的目录已经存在了，我本地 /tmp/zookeeper还不存在。所以我先创建了这个目录

```
mkdir /tmp/zookeeper
```

1. 在server.1 也就是192.168.1.85上输入 echo '1'>/tmp/zookeeper/myid
2. 在server.2 也就是192.168.1.86上输入 echo '2'>/tmp/zookeeper/myid
3. 在server.3 也就是192.168.1.87上输入 echo '3'>/tmp/zookeeper/myid

### 4.启动zookeeper
启动
```
# ./bin/zkServer.sh start
```
然后控制台会输出以下结果：
```
/bin/java
ZooKeeper JMX enabled by default
Using config: /usr/local/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

查看状态

```
# ./bin/zkServer.sh status
```
然后控制台会输出以下结果：
```
ZooKeeper JMX enabled by default
Using config: /usr/local/apache-zookeeper-3.6.3-bin/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```
这里输出的Mode会有一个节点是 leader，其它两个是 follower。 

要停止zookeeper就运行 
```
# ./bin/zkServer.sh stop
```

## 安装kafka

### 1. 下载kafka

创建目录，及下载安装包。可以到官网 https://kafka.apache.org/ 查看最新的下载地址。我这里用的是3.1.0版本。
```
cd /usr/local
wget https://dlcdn.apache.org/kafka/3.1.0/kafka_2.13-3.1.0.tgz
```
下载成功后运行命令解压：
```
tar -zxvf kafka_2.13-3.1.0.tgz 
```

进入 kafka_2.13-3.1.0 目录
```
cd kafka_2.13-3.1.0
```

### 2. 编辑配置文件
```
vi config/server.properties
```
broker1
```
broker.id=1
listeners=PLAINTEXT://192.168.1.85:9092
zookeeper.connect=192.168.1.85:2181,192.168.1.86:2181,192.168.1.87:2181
```
broker2
```
broker.id=2
listeners=PLAINTEXT://192.168.1.86:9092
zookeeper.connect=192.168.1.85:2181,192.168.1.86:2181,192.168.1.87:2181
```
broker3
```
broker.id=2
listeners=PLAINTEXT://192.168.1.87:9092
zookeeper.connect=192.168.1.85:2181,192.168.1.86:2181,192.168.1.87:2181
```

> 注意，上面的说明是要找到对应的配置项去修改。配置文件里其它的配置项不动就可以了。配置文件还有其它的配置。

### 3. 启动kafka
在三台机器上依次输入启动命令
```
./bin/kafka-server-start.sh -daemon config/server.properties 
```
如果不是很有把握可以先把上面启动命令里的 -daemon去掉。这样控制台会有日志输出。能正常启动后再加上-daemon启动。

如果需要停止kafka, 可以运行 
```
./bin/kafka-server-stop.sh
```

## 测试Kafka

### 创建topic
我们在broker1上创建一个 test-topic 主题。我们指定3个副本，一个分区
```
./bin/kafka-topics.sh --create --bootstrap-server 192.168.1.85:9092 --replication-factor 3 --partitions 1 --topic test-topic
```
### 查看Topic

我们可以通过命令列出指定Broker的topic
```
./bin/kafka-topics.sh --list --bootstrap-server 192.168.1.85:9092
```

## 总结

一开始我想用 kafka自带的zookeeper来搭建集群。但是发现少了 ./bin/zkServer.sh status 这个查看状态对应的脚本。所以稳妥起见还是下载了独立的zookeeper。

