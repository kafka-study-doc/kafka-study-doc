---
layout: kafka
title: Windows安装 Kafka
parent: Kafka入门
nav_order: 3
---

通常使用kafka都是安装在linux系统的，但有时候为了方便学习和本地开发测试，我们手上又没有linux机器，就可以考虑直接在windows下安装一个kafka. 请看下面的安装步骤

## 第一步：下载安装包
我们到官网 https://kafka.apache.org/downloads 下载安装包，我写这篇文章的时候，最新的版本是3.0.0。我下载的是 kafka_2.13-2.8.1.tgz 这个安装包。
下载后解压，我把文件解压到了 c:\soft目录下，大家可以根据自己的需要存放

## 第二步：运行 zookeeper. 
1. 打开 windows 命令行，进入 c:\soft\kafka_2.13-2.8.1 目录
2. 执行 bin\windows\zookeeper-server-start.bat config\zookeeper.properties 命令启动zookeeper 
```
C:\soft\kafka_2.13-2.8.1>bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```

如果有错误，注意检查一下自己当前所在的目录。

## 第三步：运行 kafka-server
1. 打开 另一个 windows 命令行，进入 c:\soft\kafka_2.13-2.8.1 目录
2. 执行 bin\windows\kafka-server-start.bat config\server.properties 命令启动kafka-server
```
C:\soft\kafka_2.13-2.8.1>bin\windows\kafka-server-start.bat config\server.properties
```

到此为止，在windows下安装和启动kafka就算结束了，如果要验证安装是否正确，可以使用kafka客户端测试一下。

## 测试验证
接下来我们创建一个topic.
```
C:\soft\kafka_2.13-2.8.1>bin\windows\kafka-topics.bat --bootstrap-server 127.0.0.1:9092 --create --topic test-topic
```
控制台打印出
```
Created topic test-topic.
```
这就说明 topic 创建成功了。

## 小插曲
一开始我用的是kafka3.0.0的版本，我在执行这个命令时，报了一个错：
```
[2021-12-02 09:02:37,380] ERROR Failed to create or validate data directory C:\tmp\kafka-logs (kafka.server.LogDirFailureChannel)
java.nio.file.AccessDeniedException: C:\tmp
        at sun.nio.fs.WindowsException.translateToIOException(WindowsException.java:83)
        at sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:97)
        at sun.nio.fs.WindowsException.rethrowAsIOException(WindowsException.java:102)
        ...
        at kafka.server.KafkaServer.startup(KafkaServer.scala:254)
        at kafka.Kafka$.main(Kafka.scala:109)
        at kafka.Kafka.main(Kafka.scala)
```
按错误信息看,是一个目录没有权限。后来给目录加上了权限，还是报这个错。在网上查有人说是数据有问题，我就把 c:\tmp下的目录 全删了，再重启zookeeper 和 kafka-server。也还是不行，再查了一下发现有人说是版本有问题。3.0.0的版本会报错，用2.8.1就好了。我试了一下，果然如此。

最后说明一下，windows下使用kafka建议还是仅供本地开发测试。生产还是使用linux系统安装kafka。