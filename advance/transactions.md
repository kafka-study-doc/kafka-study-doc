---
layout: kafka
title: 事务使用示例
parent: Kafka进阶
nav_order: 14
---

kafka从0.2.11版本开始支持事务，本文档对kafka事务作一个简单的说明，同时给出java代码示例，并对代码做一些简单的说明，同时说明相关的注意事项。希望能对需要使用kafka事务的朋友有帮助。

2017年6月28日，Kafka官方发布了0.11.0.0的版本，从这个版本开始，kafka支持了事务。那么，什么是kafka中的事务呢？

kafka事务支持生产者能够将一组消息作为单个事务发送，该事务要么原子地成功要么失败。举个例子，用户支付了某个订单，订单支付后，需要通知库存模块去减少库存，同时需要通知优惠券模型去扣减优惠券，这两个消息你需要让它们要么都成功，要么都失败。这个时候就可以使用kafka事务。

我们先看代码。
第一步：引入依赖，在pom.xml中增加kafka client的依赖
```
 <dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.7.0</version>
</dependency>
```
第二步：发送消息相关代码



```
Properties props = new Properties();
props.put("bootstrap.servers", "10.20.200.166:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("transactional.id", "my-transactional-id");
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

producer.initTransactions();

int c =0;
while (true) {
    if(c++>=3){
        break;
    }
    try {
        producer.beginTransaction();

        ProducerRecord<String, String> record = new ProducerRecord<>("TEST1", "message a " + c);
        producer.send(record);
        System.out.println("send TEST1 " + c);

        ProducerRecord<String, String> record2 = new ProducerRecord<>("TEST2", "message b " + c);
        producer.send(record2);
        System.out.println("send TEST2 " + c);

        producer.commitTransaction();
    }catch (RuntimeException e){
        System.out.println(e.getMessage());
        producer.abortTransaction();
    }
}
```

1. 前面6行，是为了构建一个 KafkaProducer 实例，在spring boot的工程里，这个总分通常会封装成一个方法，然后通过@Bean 完成对象的注入管理。其中 bootstrap.servers 对应你的kafka服务器的地址，需要按你本地的实际情况修改，这一类地址通常也应该放到配置文件里。
2. 第8行 producer.initTransactions(); 表示对kafka事务进行初始化，这个方法对于 每个producer 调用一次就可以了。
3. 接下来是一个循环控制，这里是想表达 可以重复使用 producer对象，发送多次事务消息。对于每一次的 事务 以 producer.beginTransaction() 表示一次事务的开始，producer.send(record) 表示这一次事务里，需要发送的消息，每个事务里，send方法可以多次调用，具休取决于业务需求，然后通过 producer.commitTransaction() 提交事务，如果发生了异常，则通过 producer.abortTransaction() 来取消事务。

>注意： transactional.id 的取值是不能重复的，如果你的环境里，只有单一节点，那这个值直接用一个固定的字符串就可以了。但是如果你的程序需要支持横向扩展，比如：同时有两个或者更多服务器同时运行你的代码，这个时候就会出问题。对于同样的transactional.id, 在一个新的KafkaProducer 调用 initTransactions 后，原来的进程就会报错。只有最新的进程能正常工作。所以这个时候，你需要保证不同的节点运行时，取到的transactional.id的值是不一样的。你可以使用 UUID.randomUUID().toString() 来生成一个保证不重复的随机ID，或者 直接在不同的实例的服务器里配置不同的transactional.id。 而对于同一个节点的运行，多次事务，是可以使用同一个KafkaProducer的，也就可以使用一样的 transactional.id。

参考文档：
https://www.confluent.io/blog/transactions-apache-kafka/
