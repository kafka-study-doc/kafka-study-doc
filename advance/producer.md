---
layout: kafka
title: 生产者
parent: Kafka进阶
nav_order: 11
---

我们先来看几个问题：
1. 创建kafka生产者时，有哪些必要的配置项？简单说明这些配置项的作用
2. 发送消息时如何实现消息压缩？
3. acks配置的作用是什么？它都有哪些值？默认值是什么？不同的值分别表示什么含义？
4. 哪些情况会导致消息发送失败？发送失败了如何处理？
5. 如何保证消息的顺序？

## 发送消息
我们还是先来看kafka生产者发送消息的代码

在pom.xml中添加依赖
```
 <dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.7.0</version>
</dependency>
```
创建一个java类
```
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;

public class Producer {
    public static void main(String[] args){
        Properties props = new Properties();
        props.put("bootstrap.servers", "192.168.0.166:9092,192.168.0.167:9092");
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
从上面的代码可以看出，我们是先创建一个 KafkaProducer 对象 producer. 然后创建一个 ProducerRecord 对象 record. 然后通过 producer.send(record) 方法，把消息发送到 kafka 服务器。

### 必要的配置项
上面的代码配置了生产者必须的三个配置项：

1. bootstrap.servers 表示 broker 服务地址，格式是host:port。如果是集群的多个 broker 地址之间用逗号分隔。清单里不需要包括所有broker的地址，生产者会从给定的broker里查找到其它broker的信息。为了保险起见，还是建议写上所有broker的地址。如果你只写了一个IP，而这个IP对应的主机又崩溃了，那就无法通过它去查看其它broker的信息了。

2. key.serializer 指定key使用的序列化器, broker收到的消息的键和值都是字节数组，我们需要指定具体的序列化类，通常情况下我们使用 org.apache.kafka.common.serialization.StringSerializer 来实现字符串的序列化。

3. value.serializer 指定value使用的序列化器

### 消息中key的作用
上面的示例代码里，我们使用
```
        ProducerRecord<String, String> record = new ProducerRecord<>("TEST1", message);
```
来创建 ProducerRecord 对象，并没有看到 key的存在。 上面的 TEST1 是表示主题。如果想发送带key的消息，可以这样写

```
ProducerRecord<String, String> record = new ProducerRecord<>("TEST1", "key1", message);
```
下面是 ProducerRecord 构造函数的定义
```
    public ProducerRecord(String topic, K key, V value) {
        this(topic, null, null, key, value, null);
    }    
    public ProducerRecord(String topic, V value) {
        this(topic, null, null, null, value, null);
    }
```
我们可以看到 没有传key时，key的值就是null。

那么key有什么用呢？

在默认的情况下，如果指定了 key, 那么Kafka会通过key,取它的哈希值，再对分区总数取模，分配到对应的分区。因此对于同样的key, 在分区数量不变的情况下，总是会被分配到同一个分区。所以如果你需要把某一类消息分配到同一个分区，就可以使用key。

### 可靠性配置

**acks** 是跟可靠性相关的生产者重要配置，它控制发送的消息的持久性。 允许以下设置：

* acks=0 如果设置为零，则生产者根本不会等待来自服务器的任何确认。 该记录将立即添加到套接字缓冲区并被视为已发送。 在这种情况下不能保证服务器已经收到记录，并且重试配置不会生效（因为客户端通常不会知道任何失败）。 如果你要的是吞吐量，对于部分消息的丢失能接受，可以考虑这样设置。比如web服务的访问日志。

* acks=1 这意味着Leader节点会将记录写入其本地日志，但会在不等待所有追随者的完全确认的情况下做出响应。 在这种情况下，如果Leader节点在确认记录后但在追随者复制它之前立即失败，那么记录将丢失。

* acks=all 这意味着Leader节点将等待完整的同步副本集来确认记录。 这保证了只要至少一个同步副本保持活动状态，记录就不会丢失。 这是最有力的保证。 

> 默认情况下：acks=all, 也就是会采用最可靠的配置。这一节写了这么多，其实就是告诉大家，acks这个参数你啥也不干就是最可靠的。

### 发送异常处理
在讲发送异常处理前，先简单讲述一下消息发送的过程。kafka为了提高吞吐量，并不是一条条消息从producer发送到broker的。而是一批批的发送的。下面按时间顺序简单描述发送过程
```
1. new KafkaProducer, 这个时候kafka client 会创建一个io线程，专门用来发送消息
2. producer.send(record)，这个时候 kafka client 会把这个消息放进一个消息池子里，存在于内存中
3. 当时间窗口到了之后，就会发送一批数据到broker
```
这种模式跟快递的处理类似，快递小哥总是会把快递一件件装车后，再开车把一批快递送到网点。

前面我们发送消息时，关键的就是一行代码
```
producer.send(record);
```
那么问题来了，如果消息发送失败了会发生什么呢？比如网络故障了，producer到broker之间的网络不通，那肯定会失败。为了解析清楚这个细节，我们需要了解消息发送的几个方式：

1. 发送并忘记，我们把消息发送给服务器，但并不关它是否正常到达。大多数情况下，消息会正常到
达，因为kafka是高可用的，而且生产者会自动尝试重发。不过，使用这种方式有时候也会丢失一些消息。我们前面使用的 producer.send(record) 就是 发送并忘记。

2. 同步发送，使用send()方法发送消息，它会返回一个Future对象，调用 get() 方法进行等待就可以实现同步发送。
```
producer.send(record).get();
```

3. 异步发送，调用 send()方法，并指定一个回调函数，服务器在返回响应时调用该函数。
```
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata metadata, Exception e) {
        if(e!=null){
            // do something
        }
    }
});
```
> 注意：如果只是按上面的示例直接使用 producer.send(record); 则可能会因为send方法内抛出的RuntimeException而中断后续的运行。所以如果你期望出现运行时异常后，程序能继续运行的话，最好是处理这个运行时异常。比如：

```
try {
    producer.send(record);
}catch (RuntimeException e){
    e.printStackTrace();
}
```

### 提高吞吐量
有些场景对吞吐量要求很高，比如web的访问日志，或者接口的请求记录。这个时候我们需要对可靠性和吞吐量之间作一个取舍。为了提高吞吐量可以考虑以下方法：
1. 水平扩展
    * 1.1 客户端侧 增加生产者的数量
    * 1.2 服务器侧 增加broker数量，增加分区数量
2. 牺牲可靠性
    * 2.1 修改acks参数，acks参数设置为1或0
3. 性能优化
    * 3.1 消息压缩，减少带宽的消耗
    * 3.2 提高磁盘吞吐量，使用固态硬盘
    * 3.3 检查网络流量，确保网络带宽有剩余，某些机房用的还是百兆交换机，容易跑满，这种情况可以考虑使用千兆或者万兆交换机
    * 3.4 配置linger.ms, 支持批量发送

### 消息压缩

**compression.type** 是用来配置消息是否压缩的，默认情况下是不压缩的。该参数可以设置为 snappy, gzip, lz4, zstd。snappy 压缩算法占用较少的 CPU ，却能提供较好的性能和相当可观的压缩比，如果较关注性能和网络带宽，可以使用这种算法。gzip 压缩算法一般会占用较多的CPU ，但会提供更高的压缩比，所以如果网络带宽比较有限，可以使用这种算法。使用压缩可以降低网络传输开销和存储开销，而这往往是向 Kafka 发送消息的瓶颈所在。

### 批次发送

producer发送消息到broker，是一批批发送的，跟批量发送相关的生产者配置主要有以下两个：

#### 1. batch.size 

这个配置控制默认批量大小（以字节为单位）默认值为16384，也就是16KB。所以这里控制的是字节数，不是消息个数。一个非常大的批处理大小可能会更浪费内存，因为我们总是会分配一个指定批处理大小的缓冲区以预期额外的记录。

> 注意：这个配置项设定了要发送的批量大小的上限。如果我们为此分区累积的字节数少于这个数量，我们将在 linger.ms 时间内“逗留”等待更多记录出现。这个 linger.ms 设置默认为 0，这意味着即使累积的批量大小在这个 batch.size 设置之下，我们也会立即发送一条记录。

#### 2. linger.ms 
这个配置项控制消息批量发送的等待时间，单位：毫秒，默认为 0（即无延迟）。在某些情况下，客户端也可能希望减少请求的数量。通过添加少量人为延迟来实现生产者不会立即发送一条记录，而是等待给定的延迟以允许发送其他记录，以便可以将发送一起批处理。例如，设置 linger.ms=5 会产生减少发送请求数量的效果，但会给在没有负载的情况下发送的记录增加最多 5ms 的延迟。

> 所以默认的情况下,kafka牺牲了吞吐量，为了获得更好的消息实时性。

### 消息的顺序保证
Kafka 可以保证同一个分区里的消息是有序的。也就是说，如果生产者按照一定的顺序发送消息， broker 就会按照这个顺序把它们写入分区，消费者按照同样的顺序读取它们。在某些情况下 顺序是非常重要的。例如，往一个账户存入 1OO 元再取出来，这个与先取钱再存钱是截然不同的！不过，有些场景对顺序不是很敏感。

如果把retries设为非零整数，同时把 max.in.flight.requests.per.connection 设为比1大的数，如果第1个批次消息写入失败，而第2个批次写入成功， broker 会重试写入第1个批次。如果此时第1个批次也写入成功，那么两个批次的顺序就反过来了。

一般来说，如果某些场景要求消息是有序的，那么消息是否写入成功也是很关键的，所以不建议把 retries 设为 0。可以把max.in.flight.requests.per.connection 设为 1 ，这样在生产者尝试发送第1批悄息时，就不会有其他的消息发送给 broker 。不过这样会严重影响生产者的吞吐量 ，所以只有在对消息的顺序有严格要求的情况下才能这么做。

所以如果要求kafka的某个主题，全局有序。那么这个主题只能1个分区。因为不同分区的先后顺序无法保证。同时配置 max.in.flight.requests.per.connection 为 1. max.in.flight.requests.per.connection 的默认值是5。因此在你什么也不管的情况下，kafka牺牲了顺序保证，确保了吞吐量。

### 消息发送失败后的重试

默认情况下retries的配置是2147483647，也就是kafka会尽可能的重试，确保消息发送成功。跟重试相关的另一个配置是 delivery.timeout.ms，它的默认值是120000，也就是2分钟。这意味着Kafka会在2分钟内，尽可能的把消息发送到broker. 但如果超过了这个时间，它就会放弃发送了。

## 问答
我们回过头来看看前面的问题，答案已经在上面的正文里给出来，这里简要的答复一下：

1. 创建kafka生产者时，有哪些必要的配置项？简单说明这些配置项的作用
> 答：必要的配置项包括：bootstrap.servers，key.serializer 和 value.serializer。bootstrap.servers 表示 broker 服务地址, key.serializer 指定key使用的序列化器， value.serializer 指定value使用的序列化器。

2. 发送消息时如何实现消息压缩？
> 答：通过 compression.type 配置来控制

3. acks配置的作用是什么？它都有哪些值？默认值是什么？不同的值分别表示什么含义？
> 答：acks控制发送的消息的持久性，它有0,1和all三个值，默认值all。
> * acks=0 如果设置为零，则生产者根本不会等待来自服务器的任何确认。 该记录将立即添加到套接字缓冲区并被视为已发送。 
> * acks=1 这意味着Leader节点会将记录写入其本地日志，但会在不等待所有追随者的完全确认的情况下做出响应。 在这种情况下，如果Leader节点在确认记录后但在追随者复制它之前立即失败，那么记录将丢失。
> * acks=all 这意味着Leader节点将等待完整的同步副本集来确认记录。 这保证了只要至少一个同步副本保持活动状态，记录就不会丢失。 这是最有力的保证。 

4. 哪些情况会导致消息发送失败？发送失败了如何处理？
> 答：网络异常或者集群状态异常都可能导致消息发送失败，发送失败了kafka默认会有重试发送的处理。

5. 如何保证消息的顺序？
> 答：Kafka 可以保证同一个分区里的消息是有序的，需要设置 max.in.flight.requests.per.connection 为 1。如果要求全局有顺，可以只使用一个分区。如果要求局部有序，比如同一个用户的消息要求有序，则可以通过key把这个用户相关的所有消息都分配到同一个分区。

## 总结
生产消息的过程是重要的一个环节，了解它的工作原理和工作模式对于问题的排查，和特殊场景的配置还是很有帮助的。用一首诗来总结kafka的默认行为。
```
顺序诚宝贵，
吞吐价更高，
若为实时故，
二者阶可抛!
```
但不同的业务场景，还是有不同的需要的，所以要根据具体的业务要求，使用不同的配置。

### 参考资料
1. kafka-clients 2.7.0 源代码
2. 《kafka权威指南》
3. 官方文档
