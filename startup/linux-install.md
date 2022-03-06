---
layout: kafka
title: Linux安装 Kafka
parent: Kafka入门
nav_order: 4
---

准备一台linux机器，我自己创建了一个kafka用户，各位可以根据自己的需求，创建期望的linux用户。

## 第一步：下载安装包
```
[kafka@localhost ~]$ wget https://dlcdn.apache.org/kafka/3.0.0/kafka_2.13-3.0.0.tgz --no-check-certificate
```
你可以直接使用上面的命令下载安装包，也可以到官网下载安装包。我自己安装时是在 /home/kafka目录下安装的，你可以根据自己的需要选择一个合适的目录。然后就是解压
```
[kafka@localhost ~]$ tar -zxvf kafka_2.13-3.0.0.tgz 
```


## 第二步：运行 zookeeper. 

```
[kafka@localhost ~]$ cd kafka_2.13-3.0.0
[kafka@localhost kafka_2.13-3.0.0]$ ./bin/zookeeper-server-start.sh -daemon config/zookeeper.properties 

```
先通过 cd kafka_2.13-3.0.0 进入kafka所在目录，然后执行 ./bin/zookeeper-server-start.sh -daemon config/zookeeper.properties 命令启动zookeeper。为了方便调试，一开始可以先不加 -daemon 这个后台运行的参数。不加 -daemon就可以在命令行直接看到启动日志。 加了 -daemon 就表示直接后台运行。


## 第三步：运行 kafka-server

```
[kafka@localhost kafka_2.13-3.0.0]$ ./bin/kafka-server-start.sh -daemon config/server.properties 
```

到此为止，kafka就正常运行起来了，可以使用kafka客户端测试一下。

## 测试验证
我们先看看端口是否正常监听了，通过netstat -ntulp 命令
```
[kafka@localhost kafka_2.13-3.0.0]$ netstat -ntulp
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp6       0      0 :::9092                 :::*                    LISTEN      18815/java          
tcp6       0      0 :::2181                 :::*                    LISTEN      17837/java     
```
如上面的结果，如果你看到 9092端口（kafka默认端口)和2181端口(zookeeper默认端口)都是正常监听状态，就说明kafka已经正常运行了。如果你看不到 可以把上面第二步 和第三步的 -daemon 参数去掉，看看是否有报错。可能会有一些文件目录权限的错误。

接下来我们创建一个topic.
```
[kafka@localhost kafka_2.13-3.0.0]$ ./bin/kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --create --topic test-topic --partitions 1 --replication-factor 1
```
控制台打印出
```
Created topic test-topic.
```
这就说明 topic 创建成功了。

当然这个安装只是单机的安装，如果需要集群部署，还需要修改一些配置。这个就不在这篇文章里讨论了。