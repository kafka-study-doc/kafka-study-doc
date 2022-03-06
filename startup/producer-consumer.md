---
layout: kafka
title: 生产者与消费者示例代码
parent: Kafka入门
nav_order: 5
---

本文档旨在用最简单的方法，把kafka的生产者和消费者的示例代码，运行起来，并做相应的讲解。读者需要有一定的java基础，会用maven,会使用一款java开发工具。

代码清单包括：
1. 依赖项说明 pom.xml
2. 日志配置 src/main/resources/logback.xml
3. 生产者类 src/main/cn/genlei/Producer.java
4. 消费者类 src/main/cn/genlei/Consummer.java

## 依赖说明

在pom.xml中添加依赖

```
 <dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.7.0</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.7</version>
</dependency>
```

kafka-clients是kafka的依赖包，logback-classic是为了看kafka-client日志的，虽然logback-classic不是必须的，但是为了能看到相关的日志显示，建议还是加上。如果你的项目里已经有 log4j了，可以不用logback.

## 日志配置
创建一个日志的配置文件src/main/resources/logback.xml，内容如下：

```
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```
这里把root的 log level 设成了 info, 实现在 kafka的 debug 日志太多了。

## 生产者类
创建一个java类，我这里把文件放到了 src/main/cn/genlei/Producer.java, 你可以根据自己的需要，修改java文件的包名。
```
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;

public class Producer {
    public static void main(String[] args){
        Properties props = new Properties();
        props.put("bootstrap.servers", "10.20.200.166:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);

        String message = "message " + System.currentTimeMillis();
        ProducerRecord<String, String> record = new ProducerRecord<>("TEST1", message);
        producer.send(record);

        System.out.println(message);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
1. 前面的三行import就是导入相关用到的类
2. 第5，6行是 声明 类和默认的main方法，便于测试
3. 7-11行是创建KafkaProducer对象，这个是生产者的核心对象。配置项中的 bootstrap.servers 需要根据你自己的实际情况修改，通常这个kafka服务器地址应该写到配置文件里
4. 13-15行是 发送消息
5. 17行是打印一下消息内容，便于调试
6. 18-22行只是让主线程等待一下，好让kafka的发送线程能顺利把内容发送到kafka服务器，在正式的项目里，这个sleep是没有必要的。因为真实项目一般都是web项目类的，一直运行的。这里是因为发送消息后，进程直接结束了。因为kafka producer是异步发送的。如果不等待一点时间，会导致消息发不出去。

## 消费者类
创建一个java类，我这里把文件放到了 src/main/cn/genlei/Consummer.java, 你可以根据自己的需要，修改java文件的包名。







```
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Properties;

public class Consummer {
    static Logger logger = LoggerFactory.getLogger(Consummer.class);

    public static void main(String[] args){
        Properties props = new Properties();
        props.put("bootstrap.servers", "10.20.200.166:9092");
        props.put("group.id", "my-group");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);

        List<String> list = new ArrayList<>();
        list.add("TEST1");
        consumer.subscribe(list);

        while (true){
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            for (ConsumerRecord<String, String> record : records) {
                String topicName = record.topic();
                String val = record.value();
                logger.info("{},{}",topicName,val);
            }
        }
    }
}
```

1. 1-9行是引入相关的类
2. 12 行是获取一个logger对象，用于打印日志
3. 15-20行，是创建一个KafkaConsumer对象，这里的group.id是一个关键配置，group.id表一个消费组。一个消息只会被一个消费组消费一次。 key.deserializer 需要和生产者的 key.serializer相匹配，这样数据才能正确的被解释。同样的value.deserializer也需要和生产者的 value.serializer相匹配。
4. 22-24行是订阅TEST1 这个topic。为后面的poll做准备
5. 26-33 就是循环消费消息。 

## 调试与运行
代码写好后就可以运行查看效果了。需要说明的是如果你的kafka服务器没有开启topic的自动创建，就需要手动先创建一个topic. 可以参考下面的命令：
```
./bin/kafka-topics.sh --bootstrap-server 127.0.0.1:9092 --create --topic TEST1 --partitions 1 --replication-factor 1
```

可以先运行 Consummer , 然后再运行 Producer . 每运行一次 Producer 后 ，Consummer都应该能收到一条消息。
