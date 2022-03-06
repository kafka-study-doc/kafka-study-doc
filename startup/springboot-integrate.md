---
layout: kafka
title: SpringBoot集成Kafka
parent: Kafka入门
nav_order: 6
---

在Springboot中使用kafka通常有两种式，一种是直接使用kafka-client, 自己做简单封装；另一种是使用spring-kafka的封装。本文介绍的前一种，这样我们只需要更多的了解 kafka本身就可以了。

## 目录结构
我们先看示例工程的目录结构：
```
|   pom.xml
+---src
|   \---main
|       +---java
|       |   \---cn.genlei
|       |           Application.java
|       |           KafkaConfig.java
|       |           KafkaService.java
|       |           TestController.java
|       \---resources
|               application.yml
```
## pom.xml
我们在pom.xml中引入了spring boot 项目通常都包含的 spring-boot-starter-web 和 lombok. 然后引入了 kafka-clients。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.7.0</version>
</dependency>
```

## Application.java
Application就是启动Springboot的启动入口，没有额外的配置
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args){
        SpringApplication.run(Application.class);
    }
}
```
##  KafkaConfig.java

```
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.List;
import java.util.Properties;

@Configuration
@Slf4j
public class KafkaConfig {

    //kafka 服务器IP地址和端口
    @Value("${kafka.host}")
    private String kafkaHost;
    // 消费组
    @Value("${kafka.groupid}")
    private String groupid;
    // 监听的主题列表，多个主题用,分隔
    @Value("#{'${kafka.topics}'.split(',')}")
    List<String> topics;

    /**
     * 初始化一个 KafkaProducer 对象，并作为 Bean 注入到 Spring 管理的对象中
     * @return KafkaProducer 实例
     */
    @Bean
    KafkaProducer kafkaProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", kafkaHost);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        log.info("KafkaProducer init finished.");
        return kafkaProducer;
    }

    /**
     * 初始化一个 kafkaConsumer 对象，并作为 Bean 注入到 Spring 管理的对象中
     * @return kafkaConsumer 实例
     */
    @Bean
    KafkaConsumer kafkaConsumer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", kafkaHost);
        props.put("group.id", groupid);

        //配置自动提交offset
        props.put("enable.auto.commit", "true");
        //配置自动提交offset的时间间隔为1秒
        props.put("auto.commit.interval.ms", "1000");

        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(topics);
        log.info("KafkaConsumer  init finished. topics:{}",topics);
        return consumer;
    }
}

```
这里的示例创建了两个对象实例，一个 kafkaConsumer, 一个 KafkaProducer。如果你的工程只需要生产者，那么把 kafkaConsumer 相关的部分删除掉就可以了。如果你的工程只需要消费者，那么把 KafkaProducer 相关的部分删除掉就可以了。

## KafkaService.java
```
import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Service;
import java.time.Duration;

@Service
@Slf4j
public class KafkaService implements CommandLineRunner {

    @Autowired
    KafkaConsumer kafkaConsumer;

    @Override
    public void run(String... args) throws Exception {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true){
                    ConsumerRecords<String, String> records = kafkaConsumer.poll(Duration.ofMillis(100));
                    for (ConsumerRecord<String, String> record : records) {
                        log.info("received message:{},{}",record.topic(),record.value());
                    }
                }
            }
        });
        thread.setName("kafkaConsumer");
        thread.start();
    }
}
```
KafkaService 这里是一个 消费者的示例，通过实现 CommandLineRunner 接口，让这部分代码在springboot 初始化后，执行。在这里启动一独立的线程去消费是为了 避免主线程阻塞在这里。

## TestController.java

```
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

    @Autowired
    KafkaProducer kafkaProducer;

    @RequestMapping("/test")
    public String send(@RequestParam String msg){
        ProducerRecord<String, String> record = new ProducerRecord<>("TEST1", msg);
        kafkaProducer.send(record);
        return "send ok. msg:" + msg;
    }
}
```
TestController 是一个简单的发送示例。把用户通过浏览器发送的消息写入 kafka 的 TEST1 这个主题中。各位测试的时候要注意看自己的kafka服务器中，是否有 TEST1 这个主题，如果没有的话，可能需要手动创建一下。

## application.yml
这里配置文件，在 KafkaConfig 中引用 
```
kafka:
  host: 127.0.0.1:9092
  groupid: test-group
  topics: TEST1

```

## 启动测试
经过前面的准备，现在就可以启动测试了。测试前需要确认：
1. kafka的ip地址和端口是否已经改成了你本地可用的地址和端口
2. kafka 服务器中是否已经有了 TEST1 主题，如果没有可以手动创建。

然后就可以运行 Application 开始你的测试。服务起来后，可以通过浏览器访问
```
http://localhost:8080/test?msg=hello123
```
如果你看到控制台里输出
```
received message:TEST1,hello123
```
这样的log，就说明集成成功了。

总的来说 KafkaConfig.java这个类是关键的集成部分，其它的其实都是辅助测试用的类，在实际项目中需要根据实际情况来修改。