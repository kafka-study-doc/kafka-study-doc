---
layout: kafka
title: 消费者与消费者群组
parent: Kafka进阶
nav_order: 13
---

我们先来看几个问题：

1. 简述消费者与消费组之间的关系
2. 有哪些情形会造成重复消费？
3. 那些情景下会造成消息漏消费？
4. KafkaConsumer 是非线程安全的，那么怎么样实现多线程消费？
5. 我明明发了消息，但是为什么没收到？
6. 怎么知道现在消息是不是有堆积，消费不及时？
7. 怎么查看某个主题的某个分区分配给哪个消费者了？

上面的这些问题，或者是在面试时会被问到，或者是在实际项目中排查问题会遇到。希望能通过这篇文章为大家解答这些问题。

## 概念
消费者就是通过订阅一个或多个主题，消费消息的实体。消费者从属于消费者群组，一个群组内的多个消费者通过分区分配来接收不同分区的消息。下面是要点：

1. 对于同一个消费者群组，一条消息只会被其中一个消费者消费。
2. 一个消费者可以从多个分区中消费消息，但对于每一个消费者群组，每个分区在同一时刻只能被一个消费者消费。
3. 不同的消费者群组之间没有影响

示例一：假设有这样一个微服务，需要把一些配置信息加载到内存中，这些配置信息是在数据库里存储于一张配置表里的，同时提供了一个web界面去修改。要求在web界面修改后，内存中的缓存也要即时更新。这个时候如果我们只一个web节点，那是很简单的，就是当配置信息发生变更的时候，直接重新加载一下配置信息就可以了。但是在上规模的系统里，往往是要求水平扩展的，我们会有2个或者更多的web节点，这个时候，当操作人员在界面上操作配置变更时，只有其中一个web节点，知道配置变更了，其它节点并不知道。

为了解决这个问题，我们就会先把变更的消息发送到kafka, 然后每个web节点都来消费这个消息，**这种情况需要特别注意不能使用相同的消费者群组**，不然还是只有一个web节点能收到这个消息。

示例二：假设有一个电商系统，包括：订单微服务务，配送微服务。订单支持成功后，订单消息写入kafka, 通知配送微服务，后来由于业务扩展，需要新增积分微服务，这个时候积分微服务只需要使用一个新的消费者群组，就可以完成业务扩展，不影响原来的消费者群组。所以对于生产者来说，它并不知道有多少个消息者群组会来消费消息。也是这个特性使用kafka后，系统的可扩展性，会变得更好。完全可以实现，在现有微服务不变的前提下，增加新的微服务，完成业务的扩展。

## 消费过程及偏移量控制

我们先来看一段简单的消费代码：

```
Properties props = new Properties();
props.put("bootstrap.servers", "192.168.1.102:9092");
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
        logger.info("{},{}",record.topic(),record.value());
    }
}
```
上面的这段代码主要是三部分：
1. 创建一个 KafkaConsumer 对象
2. 订阅目标主题
3. 循环消费消息

如果只看上面的代码，我们找不到任何关于偏移量的相关代码。那么什么是偏移量？它有什么作用？

回答这个问题前，我们先来看几个业务场景：

* 场景一：不在乎过去，只关心从消费者开始消费后产生的消息。
* 场景二：尽可能往早去追溯，越早越好
* 场景三：从某一特定时间点开始追溯，比如：从今天0点开始

**偏移量（offset)** 是一个不断递增的整数值，在创建消息时，kafka会把它添加到消息里。在给定的分区里，每个消息的偏移量是唯一的。Kafka对于每一个消费者群组，记录其对应的每个分区最后消费的偏移量。因此偏移量是用于记录和控制消费进度的状态值。通过控制偏移量，可以消费指定偏移量后的消息，也可以实现消费从上次中断后产生的新消息。

**auto.offset.reset** 是一个关键的消费者配置项，它的值是字符串类型，有两个值 earliest 和 latest，默认值是latest。它表示消费者的偏移量控制策略。当kafka服务器找不到该消费者群组订阅的当前主题的当前分区的消费偏移量记录时，就会使用这个策略。比如一个新的消费者群组开始消费时，因为从来没有消费过，kafka服务器肯定是没有消费记录的。

* earliest: 自动重置到最早的偏移量，这种情况能消费到目前依然记录在服务器上的所有消息。
* latest:  自动重置到最新的偏移量，这种情况能消费到从消费者群组首次活跃后产生的消息。

我们回过头来再看看上面的代码，没有对 auto.offset.reset 进行任何配置，也没有指定offset，因此默认值 latest 会生效，上面的代码对应了前面讲的场景一，会消费到从这个消费者群组首次启动后产生的消息。

那么如果要实现场景二，也很简单，加一行代码，修改 auto.offset.reset 的配置就好了。
```
props.put("auto.offset.reset", "earliest");
```

如果要实现场景三，那么代码会多一些，示例代码如下：
```
Properties props = new Properties();
props.put("bootstrap.servers", "192.168.1.102:9092");
props.put("group.id", "my-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);

List<String> list = new ArrayList<>();
list.add("TEST1");
consumer.subscribe(list);

long start = getStart(); // 按实际需要设置开始的时间戳
Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
for(TopicPartition topicPartition:consumer.assignment()){
    timestampsToSearch.put(topicPartition,start);
}

Map<TopicPartition, OffsetAndTimestamp> offsetAndTimestampMap = consumer.offsetsForTimes(timestampsToSearch);
for(Map.Entry<TopicPartition, OffsetAndTimestamp> entry:offsetAndTimestampMap.entrySet()){
    if(entry.getValue()!=null) {
        consumer.seek(entry.getKey(), entry.getValue().offset());
    }else{
        // make some log
    }
}

while (true){
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        logger.info("{},{}",record.topic(),record.value());
    }
}
```
从上面的代码可以看出，我们先通过 KafkaConsumer 的 offsetsForTimes(Map<TopicPartition, Long> timestampsToSearch)方法，要检索指定主题指定分区对应特定时间的偏移量。再通过 KafkaConsumer 的 seek 方法，来设置偏移量。

### 自动提交偏移量
**enable.auto.commit** 是一个消费者配置项，它的值是布尔类型，默认是true. 当它的值为true时，消息者会自动在后台周期性的提交偏移量。所以默认的情况下，kafka consumer就是会自动的提交偏移量。

**auto.commit.interval.ms** 是跟 enable.auto.commit 配套的配置，也是消费者的配置项，它表示 自动提交偏移量的时间间隔，单位是毫秒，默认值是5000，也就是5秒自动提交一次。

### 手动提交偏移量

如果你期望手动控制偏移量的提交，可以把 enable.auto.commit 设为 false. 这样你可以自己用代码手动控制偏移量的提交。
代码片断参考：
```
while(true){
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        logger.info("{},{}",record.topic(),record.value());
    }
    try {
        consumer.commitSync();
    }catch (RuntimeException e){
        logger.error("commit failed:{}",e);
    }
}
```
上述代码表示，消费到一批数据，进行处理后，手动提交偏移量。commitSync()可能会抛出异常，如果提交失败，打印日志记录，便于排查问题。commitSync()表示同步提交，这样会影响吞吐量，如果对性能要求较高，可以使用 commitAsync()进行异步提交。

### 重复消费
什么是重复消费？一条消息被再两次或多次消费，就是重复消费。按前面讲的偏移量的说明，如果由于某种原因导致消息消费了，但偏移量没有及时提交，那么下一次再消费消息时，就会导致重复消费。下面是一个按时间线的举例：
```
10:00:00 consumer拉取了新的2条消息，M1,M2
10:00:01 consumer完成M1的处理
10:00:02 consumer所在的主机停电。自动提交偏移量的时间窗口未到，偏移量未提交
...
10:01:00 consumer所在的主机电力系统恢复，重新消费消息
10:01:01 consumer再次拉取了消息，M1,M2。
10:01:02 M1被重复消费，由于前面的故障导致原来消费的偏移量没有提交，所以造成重复消费
```
对于重要的业务系统，幂等的设计是很有必要的，比如一个电商系统中有一个积分模块，定义每消费1元即可获得1个积分。如果出现了重复消费，就可能导致一个订单加了两次积分。为了避免这种情况的发生，我们需要有一张积分处理记录表，记录某个的订单ID对应的订单是否已经处理过积分了。这样重复消费时，发现该订单对应的积分已经处理过了，就不再对积分进行增加的操作。

### 漏消费
漏消费正好跟重复消费相反，漏消费就是某条消息没有被正常处理，被漏掉了。当consumer已经拉取了某个消息，且偏移量正常提交了，但业务代码并没有正常处理这个消息。下面的时间线给出了一个示例：
```
11:00:00 consumer拉取了新的2条消息，M3,M4
11:00:01 consumer开始处理M3
...
11:00:05 consumer自动提交偏移量，此时kafka认为M3,M4均已被消费
11:00:06 consumer所在的主机停电。M3处理到一半，M4未处理
...
11:01:01 consumer所在的主机电力恢复。重新消费
11:01:02 consumer拉取了新的M5。 因为它以为M3,M4已处理
```
从上面的示例可以看出，当一个消息的处理时间比较长的时候，就容易导致漏消费。这种情况建议
1. 每次尽量拉取少量的消息，比如一次拉取1条，需要设置消费者配置 max.poll.records 为 1
2. 手动提交偏移量
3. 业务设计上，允许人工补录。比如允许客服对某个指定的订单进行积分手动触发结算

## 线程问题
上面的代码示例里，都是单线程来处理消息的，在同一个群组里，我们无法让一个线程运行多个消费者，也无法让多个线程安全地共享一个消费者。一个消费者使用一个线程。当然在做消息处理时，可以把业务处理的部分用多个线程来处理。

## 查看分配信息与消费延迟
```
# bin/kafka-consumer-groups.sh --bootstrap-server 192.168.1.102:9092 --describe --group my-group

GROUP          TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET    LAG             CONSUMER-ID                                     HOST            CLIENT-ID
my-group       TEST        0          563             563             0               consumer-1-215d937e-205e-4965-be82-fad8142d3830 /192.168.1.85   consumer-1
my-group       TEST        1          527             527             0               consumer-1-215d937e-205e-4965-be82-fad8142d3830 /192.168.1.85   consumer-1
```

如上面的示例，我们可能使用 kafka-consumer-groups.sh 脚本来查看某个消费者群组的分配情况及延迟情况。如上面的命令执行结果，LAG 列对应的就是延迟的情况，如果延迟比较多，就说明消费者已经处理不过来了。HOST列则是表示这个分区当前正分配给了哪个IP的消费者来处理。

如果在开发调试的过程中，特别是有团队内多个成员协作的情况下，发现发了消息但是没收到，或者说有的消息能收到，有点消息确又收不到，就可以通过上面的命令来查看。看看是不是所有分区都分配在你当前调试的机器上。有时候如果配置错误，会出现开发环境，测试环境错乱的情况，需要根据当前的分区实际分配情况，来排查问题。

## 问答
我们回过头来看看前面的问题，答案已经在上面的正文里给出来，这里简要的答复一下：

1. 简述消费者与消费组之间的关系
> 消费者就是通过订阅一个或多个主题，消费消息的实体。消费者从属于消费者群组，一个群组内的多个消费者通过分区分配来接收不同分区的消息。 对于同一个消费者群组，一条消息只会被其中一个消费者消费。

2. 有哪些情形会造成重复消费？
> 如果由于某种原因导致消息消费了，但偏移量没有及时提交，那么下一次再消费消息时，就会导致重复消费

3. 那些情景下会造成消息漏消费？
> 当consumer已经拉取了某个消息，且偏移量正常提交了，但业务代码并没有正常处理这个消息就崩溃了，就会导致漏消费

4. KafkaConsumer 是非线程安全的，那么怎么样实现多线程消费？
> 在同一个群组里，我们无法让一个线程运行多个消费者，也无法让多个线程安全地共享一个消费者。一个消费者使用一个线程。在做消息处理时，可以把业务处理的部分用多个线程来处理。

5. 我明明发了消息，但是为什么没收到？
> 通过 kafka-consumer-groups.sh 检查消费组的分区分配情况，检查是否有配置错误

6. 怎么知道现在消息是不是有堆积，消费不及时？
> 通过 kafka-consumer-groups.sh 检查消费组的情况，查看LAG的数值，如果LAG数值较大，说明有堆积

7. 怎么查看某个主题的某个分区分配给哪个消费者了？
> 通过 kafka-consumer-groups.sh 检查消费组的分区分配情况，查看HOST列

## 总结

消费是kafka的重要环节，了解其工作模式对于问题排查还是很有必要的。


### 参考资料
1. kafka-clients 2.7.0 源代码
2. 《kafka权威指南》
3. 官方文档

