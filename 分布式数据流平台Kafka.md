# 分布式数据流平台Kafka

---

Kafka是一个分布式的基于分区的消息系统，支持消息持久化，可轻松水平扩展以支持处理巨大的信息流，且系统保持较高的吞吐量。

---

### **Kafka基本概念**
- **消息与批次**：Kafka的最小数据单元是消息，消息由字节数组组成，有一个可选的Key（供消息分区用）。批次就是一组消息，消息被分批次写入Kafka。
- **消息的Schema**：消息是字节数组，Schema用来描述消息的格式，如JSON、XML和Avro等，体现在消息的序列化和反序列化节点，可解耦消息的读写操作（向下兼容）。
- **主题Topic**：消息通过Topic分类，每个Topic之间互相独立。Topic可以理解为生产者与消费者之间的一种约定。
- **分区partition**：每个Topic可以划分为若干个分区，每个分区代表一个提交日志，消息以追加的方式写入分区。Kafka消息冗余，一个分区有多个不同的副本，其中一个为Leader。
![Kafka主题、分区、批次与消息][1]
<center>**Kafka主题、分区、批次与消息**</center>
- **生产者**：生产者负责发布消息，消息关联到特定的TOPIC上，默认情况下轮询写入到某个分区，但生产者可指定消息的key来将消息写入到（hash key）特定分区。
- **消费者**：消费者订阅一个或多个Topic，并按照消息的生成顺序（分区日志顺序）读取它们，通过检查消息的偏移量来区分是否已经读取过消息。
- **消费者群组**：Kafka通过划分消费者群组来读取消息，同一个Topic可以有多个组消费，保证每个组都消费到全量的消息数据。同一个组的消费者，共同消费一份全量数据。
- **Kafka服务器broker**：broker接收来自生产者的消息，为消息设置偏移量，并持久化消息。broker为消费者提供服务，对读取分区的请求作出响应，返回已经持久化的消息。
- **Kafka集群**：一个集群有多个broker，其中一个broker同时充当集群控制器的角色。控制器由zookeeper选出，负责管理整个集群的broker，包括broker间的分区分配等工作。
![Kafka架构示意][2]
<center>**Kafka架构示意图**</center>

---

### **Kafka生产者**

#### **基本配置参数**
- bootstrap.servers: 指定broker的地址清单，地址的格式为host:port
- key.serializer: 消息key的序列化方式，如StringSerializer
- value.serializer: 消息value的序列化方式，如StringSerializer

```java
Properties properties = new Properties();
properties.put("bootstrap.servers", "localhost:8088");
properties.put("client.id", "Producer");
properties.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
KafkaProducer<Integer, String> producer = new KafkaProducer<>(properties);
```

#### **KafkaProducer使用方式**
使用KafkaProducer的send方法异步发送消息记录，可调用返回结果Future的get方法来同步阻塞获取结果；也可使用Kafka的Callback机制来异步获取结果（Callback会挂载到相应的ProducerBatch上，当客户端收到消息批次的响应结果时会调用Callback的）。

- 发送但不关心是否到达（只发送不获取结果）：
```java
producer.send(new ProducerRecord<Integer, String>(topic, num, msg));
```

- 同步发送，阻塞发送方法直到得到发送响应。
```java
producer.send(new ProducerRecord<Integer, String>(topic, num, msg)).get();
```

- 异步发送，通过callback得到发送响应
```java
private class DemoProducerCallback implements Callback {
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if(e != null) {
            e.printStackTrace();
        }
    }
}
producer.send(new ProducerRecord<Integer, String>(topic, num, msg), new DemoProducerCallback());
```

#### **发送流程详解**
![Kafka消息发送流程][3]
<center>**Kafka消息发送流程**</center>
![KafkaProducer][4]
<center>**KafkaProducer**</center>
![KafkaProducerSender][5]
<center>**Sender**</center>

#### **生产者可靠性**

- 生产者发送确认模式
 1. acks=0，生产者只要发送消息出去就认为消息已成功写入
 2. acks=1，broker收到消息并把它写入到分区数据文件时，会给生产者返回响应
 3. acks=all，broker收到消息并写入后，会等待同步副本收到消息确认，才给生产者返回响应。
- 生产者重试发送消息，有时候重试会带来重复消息的风险（即生产者由于网络问题没有收到broker的成功响应，而又重新发送一个相同的消息），这就需要消费端过滤或幂等。

### **Kafka消费者**
消费者通过向被指派为群组协调者的broker（不同群组可以有不同的协调器）发送心跳来维持它们和群组的从属关系以及它们对分区的所有权关系。消费者会在轮询消息或提交偏移量时发送心跳。

当群组里的消费者发生变化（新加入的消费者，或被迫关闭的消费者），或Topic分区数量发生变化时，分区的所有权会从一个消费者转移另一个消费者，即发生**分区再均衡**。
> 
- 当消费者加入群组时，它会向群组协调者发送一个JoinGroup请求。
- 第一个加入群组的消费者将成为群主。群主从协调者那里获得群组的成员列表，并负责给每一个消费者分配分区。
- 分配完毕后，群主把分配情况列表发送给协调者，协调者再把这些信息发送给所有消费者。这个过程会在再均衡时重复发生。


![Kafka消费者与群组协调者][6]
<center>**Kafka消费者与群组协调者**</center>

#### **基本配置参数**
- bootstrap.servers: 指定broker的地址清单，地址的格式为host:port
- key.serializer: 消息key的序列化方式，如StringSerializer
- value.serializer: 消息value的序列化方式，如StringSerializer
- enable.auto.commit: 是否自动提交偏移量。默认true
- auto.commit.interval.ms：自动提交间隔。默认5s
- session.timeout.ms: 指定消费者被认为死亡前可以与服务器断开连接的时间。默认是3s

```java
Properties properties = new Properties();
properties.put("bootstrap.servers", "localhost:8088");
properties.put("client.id", "Consumer");
properties.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
properties.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
properties.put("enable.auto.commit", "true");
properties.put("auto.commit.interval.ms", "1000");
properties.put("session.timeout.ms", "30000");
KafkaConsumer<Integer, String> consumer = new KafkaConsumer<>(properties);
```

#### **KafkaConsumer使用方式**
消费者通过轮询向服务器请求数据。分区再均衡和心跳也是在轮询时发生。

```java
try {
    consumer.subscribe(Collections.singletonList(topic));
    while(true) {
        ConsumerRecords<Integer, String> records = consumer.poll(100);
        for(ConsumerRecord<Integer, String> record : records) {
            // do something
            System.out.println(String.format("topic=%s, partition=%s, key=%s, value=%s",
            record.topic(), record.partition(), record.key(), record.value()));
        }
    }
} finally {
    consumer.close();
}
```

#### **消费者轮询过程详解**
![KafkaConsumer.png-132.6kB][7]
<center>**KafkaConsumer**</center>

#### **提交偏移量**
Kafka消费者通过维护分区偏移量来标识当前读取的消息在分区中的位置。偏移量的提交同样维护在Kafka消息队列中，消费者通过往_cusumer_offset主题发送偏移量消息，来维护每个分区的偏移量。

若提交的偏移量小于客户端已处理的偏移量，那么两个偏移量之间的消息就会被**重复处理**。若提交的偏移量大于客户端已处理的偏移量，那么两个偏移量之间的消息就会**丢失**。**提交偏移量的方式：**

- **自动提交**
默认情况下，消费者会自动提交消息偏移量。在每次轮询时会检查是否需要提交上一次的偏移量（提交间隔）。**但若在提交时间间隔内，发生分区再平衡，消息就有可能重复消费。**

- **同步提交**
通过commitSync()方法提交由poll返回的最新偏移量。采用同步提交的方式，在broker对提交请求响应前，应用程序会一直阻塞（并重试）。
```java
try {
	while(true) {
		ConsumerRecords<String, String> records = consumer.poll(100);
		// 消费消息
		for(...) {
			...
		}
		// 提交偏移量
		try {
			consumer.commitSync();
		} catch(CommitFailedException e) {
			...
		}
	}
} finally {
	consumer.close();
}
```

- **异步提交**
异步提交不会阻塞应用程序，也不会重试提交。因为重试提交有可能覆盖后续一个更大偏移量的提交。
```java
try {
	while(true) {
		ConsumerRecords<String, String> records = consumer.poll(100);
		// 消费消息
		for(...) {
			...
		}
		// 提交偏移量
		consumer.commitAsync();
	}
} finally {
	consumer.close();
}
```

#### **消费者可靠性**
消费者主要是通过准确维护偏移量来保证可靠性。有以下几个关键配置：

- auto.offset.reset：无效偏移量时的处理，latest or earliest
- enable.auto.commit：是否自动提交偏移量。
- auto.commit.interval.ms：自动提交偏移量间隔，越频繁重复处理消息概率越小，但系统开销越大。

消费者也可以显示提交来准确维护偏移量。

### **Kafka存储层**

Kafka是一个分布式的、分区的、多副本的、基于提交日志的流平台。

#### **分区复制**
Kafka使用Topic组织数据，每个Topic被分为若干分区，每个分区有多个副本**副本有两种类型：**

- **leader副本**：每个分区都有一个Leader副本，所有生产者和消费者的请求都会经过这个副本。
- **follower副本**：follower副本不处理来自客户端的请求，本质上也是一个消费者，消费Leader的消息来同步副本。。当Leader发生崩溃时，其中一个follower会被选为Leader。

通过查看每个follower请求的最新偏移量，leader就会知道每个follower的复制进度，如果follower在10s（replica.lag.time.max.ms）内没有请求最新消息，那么它就会被认为是不同步的。一个不同步的副本是无法被选为leader的。由集群控制器来负责分区Leader选举。

![分区复制][8]
<center>**分区复制**</center>

#### **分区日志**
Kafka的基本单元是分区。分区无法在多个broker间进行细分，也无法在同一个broker的多个磁盘上细分。

**每个分区对应一个日志目录，分区目录下有多个日志片段。**每个日志片段由一个数据文件和一个索引文件组成，数据文件存储信息的内容，索引文件存储消息的索引（因此可通过偏移量快速定位到消息）。在broker往分区写入数据时，如果达到片段大小上限，就会关闭当前文件并打开一个新的文件作为新片段。

Kafka为每个Topic设置数据保留期限，规定数据被删除前可以保留多长时间或清理数据前可以保留的数据量大小。当前正在写入数据的片段叫做活跃片段，活跃片段永远不会被删除。
![分区日志][9]
<center>**分区日志**</center>

#### **broker数据可靠性**
- Kafka可以保证分区消息的顺序，即消息A先于B写入分区，消费者也会先读取到A。
- 只有当消息被写入分区的所有同步副本时，它才会被认为是已提交。
- 只要还有一个副本是活跃的，那么已经提交的消息就不会丢失。
- 消费者只能读取已经提交的消息。

分区leader是同步副本，而对于follower副本来说，只有满足以下条件才能被认为是同步：

- 与Zookeeper之间有一个活跃的会话，即过去6秒有心跳
- 过去10秒有从leader获取最新消息，几乎零延迟

### **Kafka集群控制器**
Kafka控制器负责在Topic创建后为broker分配分区，以及为各个分区选出leader副本和follower副本。另外，控制器还需要在broker节点变化（新上线或故障）时调整分区。

Kafka 使用Zookeeper来维护集群成员的信息。除此之外，控制器负责维护各个Topic的分区状态，以及各个分区的副本状态，根据变化及时维护Zookeeper上的集群元信息，并通知其他broker更新元信息（broker会本地缓存元信息）。

#### **控制器选举**
broker通过创建自己ID的临时节点来注册到Zookeeper（/brokers/ids），监听此目录可知道目前可用的broker。

集群里第一个往Zookeeper创建/controller临时节点的broker会成为集群控制器。监听这个节点可知道当前集群的控制器变化，一个集群只能有一个控制器。

由于控制器创建的ZK节点为临时节点，因此控制器出现故障时，临时节点会被删除。这时候所有broker都会尝试重新创建/controller节点来选取出新控制器。

#### **Kafka集群（关键）元信息**
Kafka的元信息非常重要，整个集群的运行都依赖metaData的更新，不管是broker间的交互，还是生产者和消费者。一般来说，所有broker都有一份最新的元信息，而客户端定时向broker请求最新的元信息。

- /controller：控制器
- /brokers/ids：Brokers节点 
- /brokers/topics/topic1：topic1分布在不同broker的分区信息
- /brokers/topics/topic1/partitions/0/state：topic1的分区0保存的各个副本

![Kafka集群元信息][10]
<center>**Kafka集群元信息**</center>

#### **控制器管理Kafka集群**

![Kafka控制器 (1).png-53.1kB][11]
<center>**Kafka控制器**</center>

### **其他**

#### **Kafka Connect**

#### **延迟操作与（不支持的）定时异步消息——时间轮算法**

#### **回顾**
- 控制器——Zookeeper——broker集群——Topic——分区
- 协调器——消费者群组——分区副本

#### **参考**
- 《Kafka权威指南》
- 《Kafka技术内幕》

  [1]: http://static.zybuluo.com/yongdagan/rwahtuoh6qyz4hxvd11w31o5/Kafka%E4%B8%BB%E9%A2%98%E3%80%81%E5%88%86%E5%8C%BA%E3%80%81%E6%89%B9%E6%AC%A1%E4%B8%8E%E6%B6%88%E6%81%AF%20%281%29.png
  [2]: http://static.zybuluo.com/yongdagan/cuwci9tz2u8p42jekkuepf09/Kafka%E6%9E%B6%E6%9E%84%E7%A4%BA%E6%84%8F.png
  [3]: http://static.zybuluo.com/yongdagan/cypn1u9csndpyz7fyihv3khe/Kafka%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81%E6%B5%81%E7%A8%8B.png
  [4]: http://static.zybuluo.com/yongdagan/lgk159m0wpcyj2ug9csfmhk5/KafkaProducer.png
  [5]: http://static.zybuluo.com/yongdagan/ggs3adbwos431hwjnvxbcggk/KafkaProducerSender.png
  [6]: http://static.zybuluo.com/yongdagan/9sf6q8d2k2wj1m9ljiwyh9i8/Kafka%E6%B6%88%E8%B4%B9%E8%80%85%E4%B8%8E%E7%BE%A4%E7%BB%84%E5%8D%8F%E8%B0%83%E8%80%85%20%281%29.png
  [7]: http://static.zybuluo.com/yongdagan/cmc5t5r19m416tafiahet74c/KafkaConsumer.png
  [8]: http://static.zybuluo.com/yongdagan/vtbbcb0o9gwdh42jmhzd0d91/%E5%88%86%E5%8C%BA%E5%A4%8D%E5%88%B6.png
  [9]: http://static.zybuluo.com/yongdagan/nsfc6oyvpe5gkr2455i98p1b/%E5%88%86%E5%8C%BA%E6%97%A5%E5%BF%97.png
  [10]: http://static.zybuluo.com/yongdagan/1mri63u0984w9fqhb9r4ghut/Kafka%E9%9B%86%E7%BE%A4%E5%85%83%E4%BF%A1%E6%81%AF.png
  [11]: http://static.zybuluo.com/yongdagan/dxwzb4t539uyhh0y9oj02g48/Kafka%E6%8E%A7%E5%88%B6%E5%99%A8%20%281%29.png
