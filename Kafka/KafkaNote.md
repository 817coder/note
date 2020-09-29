# Kafka笔记

## 基本

### 初识Kafka

- **消息系统：** Kafka 和传统的消息系统（也称作消息中间件）都具备系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等功能。与此同时，Kafka 还提供了大多数消息系统难以实现的消息顺序性保障及回溯消费的功能。
- **存储系统：** Kafka 把消息持久化到磁盘，相比于其他基于内存存储的系统而言，有效地降低了数据丢失的风险。也正是得益于 Kafka 的消息持久化功能和多副本机制，我们可以把 Kafka 作为长期的数据存储系统来使用，只需要把对应的数据保留策略设置为“永久”或启用主题的日志压缩功能即可。
- **流式处理平台：** Kafka 不仅为每个流行的流式处理框架提供了可靠的数据来源，还提供了一个完整的流式处理类库，比如窗口、连接、变换和聚合等各类操作。

### 基本概念

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815172506863.png)

1. **Producer：** 生产者，也就是发送消息的一方。生产者负责创建消息，然后将其投递到 Kafka 中。
2. **Consumer：** 消费者，也就是接收消息的一方。消费者连接到 Kafka 上并接收消息，进而进行相应的业务逻辑处理。
3. **Broker：** 服务代理节点。对于 Kafka 而言，Broker 可以简单地看作一个独立的 Kafka 服务节点或 Kafka 服务实例。大多数情况下也可以将 Broker 看作一台 Kafka 服务器，前提是这台服务器上只部署了一个 Kafka 实例。一个或多个 Broker 组成了一个 Kafka 集群。一般而言，我们更习惯使用首字母小写的 broker 来表示服务代理节点。
4. **Topic**：主题，Kafka 中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题（发送到 Kafka 集群中的每一条消息都要指定一个主题），而消费者负责订阅主题并进行消费。
5. **Partition**：分区。主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题，很多时候也会把分区称为主题分区（Topic-Partition），在存储层面，分区就是一个**可追加的Log文件**，消息在被追加到分区日志文件的时候都会分配一个特定的偏移量（offset）。

### 分区

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173052193.png)

主题是一个逻辑上的概念，它还可以细分为多个分区，一个分区只属于单个主题，很多时候也会把分区称为主题分区（Topic-Partition）。同一主题下的不同分区包含的消息是不同的，分区在存储层面可以看作一个可追加的日志（Log）文件，消息在被追加到分区日志文件的时候都会分配一个特定的偏移量（offset）。

offset 是消息在分区中的唯一标识，Kafka 通过它来保证消息在分区内的顺序性，不过 offset 并不跨越分区，也就是说，Kafka 保证的是分区有序而不是主题有序。

如上图所示，主题中有4个分区，消息被顺序追加到每个分区日志文件的尾部。Kafka 中的分区可以分布在不同的服务器（broker）上，也就是说，一个主题可以横跨多个 broker，以此来提供比单个 broker 更强大的性能。

每一条消息被发送到 broker 之前，会根据分区规则选择存储到哪个具体的分区。如果分区规则设定得合理，所有的消息都可以均匀地分配到不同的分区中。如果一个主题只对应一个文件，那么这个文件所在的机器I/O将会成为这个主题的性能瓶颈，而分区解决了这个问题。在创建主题的时候可以通过指定的参数来设置分区的个数，当然也可以在主题创建完成之后去修改分区的数量，通过增加分区的数量可以实现水平扩展。

### 多副本

同一分区的不同副本中保存的是相同的消息（在同一时刻，副本之间并非完全一样），副本之间是“一主多从”的关系，其中 leader 副本负责处理读写请求，follower 副本只负责与 leader 副本的消息同步。副本处于不同的 broker 中，当 leader 副本出现故障时，从 follower 副本中重新选举新的 leader 副本对外提供服务。Kafka 通过多副本机制实现了故障的自动转移，当 Kafka 集群中某个 broker 失效时仍然能保证服务可用。

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173413230.png)

如上图所示，Kafka 集群中有4个 broker，某个主题中有3个分区，且副本因子（即副本个数）也为3，如此每个分区便有1个 leader 副本和2个 follower 副本。生产者和消费者只与 leader 副本进行交互，而 follower 副本只负责消息的同步，很多时候 follower 副本中的消息相对 leader 副本而言会有一定的滞后。

### AR、ISR、OSR

分区中的所有副本统称为 AR（Assigned Replicas）。所有与 leader 副本保持一定程度同步的副本（包括 leader 副本在内）组成ISR（In-Sync Replicas），ISR 集合是 AR 集合中的一个子集。消息会先发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步，同步期间内 follower 副本相对于 leader 副本而言会有一定程度的滞后。

前面所说的“一定程度的同步”是指可忍受的滞后范围，这个范围可以通过参数进行配置。与 leader 副本同步滞后过多的副本（不包括 leader 副本）组成 OSR（Out-of-Sync Replicas），由此可见，AR=ISR+OSR。在正常情况下，所有的 follower 副本都应该与 leader 副本保持一定程度的同步，即 AR=ISR，OSR 集合为空。

leader 副本负责维护和跟踪 ISR 集合中所有 follower 副本的滞后状态，当 follower 副本落后太多或失效时，leader 副本会把它从 ISR 集合中剔除。如果 OSR 集合中有 follower 副本“追上”了 leader 副本，那么 leader 副本会把它从 OSR 集合转移至 ISR 集合。默认情况下，当 leader 副本发生故障时，只有在 ISR 集合中的副本才有资格被选举为新的 leader，而在 OSR 集合中的副本则没有任何机会（不过这个原则也可以通过修改相应的参数配置来改变）。

### HW、LEO

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173612551.png)

ISR 与 HW 和 LEO 也有紧密的关系。HW 是 High Watermark 的缩写，俗称高水位，它标识了一个特定的消息偏移量（offset），消费者只能拉取到这个 offset 之前的消息。

如上图所示，它代表一个日志文件，这个日志文件中有9条消息，第一条消息的 offset（LogStartOffset）为0，最后一条消息的 offset 为8，offset 为9的消息用虚线框表示，代表下一条待写入的消息。日志文件的 HW 为6，表示消费者只能拉取到 offset 在0至5之间的消息，而 offset 为6的消息对消费者而言是不可见的。

LEO 是 Log End Offset 的缩写，它标识当前日志文件中下一条待写入消息的 offset，上图中 offset 为9的位置即为当前日志文件的 LEO，LEO 的大小相当于当前日志分区中最后一条消息的 offset 值加1。分区 ISR 集合中的每个副本都会维护自身的 LEO，而 ISR 集合中最小的 LEO 即为分区的 HW，对消费者而言只能消费 HW 之前的消息。

### 复制机制

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173840366.png)

在消息写入 leader 副本之后，follower 副本会发送拉取请求来拉取消息3和消息4以进行消息同步。

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173854661.png)

在同步过程中，不同的 follower 副本的同步效率也不尽相同。如上图所示，在某一时刻 follower1 完全跟上了 leader 副本而 follower2 只同步了消息3，如此 leader 副本的 LEO 为5，follower1 的 LEO 为5，follower2 的 LEO 为4，那么当前分区的 HW 取最小值4，此时消费者可以消费到 offset 为0至3之间的消息。

写入消息（情形4）如下图所示，所有的副本都成功写入了消息3和消息4，整个分区的 HW 和 LEO 都变为5，因此消费者可以消费到 offset 为4的消息了。

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200815173916015.png)

由此可见，Kafka 的复制机制==**既不是完全的同步复制，也不是单纯的异步复制**==。事实上，同步复制要求所有能工作的 follower 副本都复制完，这条消息才会被确认为已成功提交，这种复制方式极大地影响了性能。而在异步复制方式下，follower 副本异步地从 leader 副本中复制数据，数据只要被 leader 副本写入就被认为已经成功提交。在这种情况下，如果 follower 副本都还没有复制完而落后于 leader 副本，突然 leader 副本宕机，则会造成数据丢失。Kafka 使用的这种 ISR 的方式则有效地权衡了数据可靠性和性能之间的关系。

### 配置

配置文件地址：$KAFKA_HOME/conf/server.properties

```
# broker的编号，如果集群中有多个broker，则每个broker的编号需要设置的不同
broker.id=0
# 该参数指明 broker 监听客户端连接的地址列表，即为客户端要连接 broker 的入口地址列表
# 配置格式为protocol1://hostname1:port1,protocol2://hostname2:port2
listeners=PLAINTEXT://localhost:9092
# 存放消息日志文件的地址，log.dir 和 log.dirs 都可以用来配置单个或多个根目录，log.dirs优先级更高
log.dirs=/tmp/kafka-logs
# Kafka所需的ZooKeeper集群地址，多个zookeeper localhost1:2181,localhost2:2181,localhost3:2181/kafka 推荐加/kafka否则使用的是zookeeper的根路径  
zookeeper.connect=localhost:2181
# 该参数用来指定 broker 所能接收消息的最大值，默认值为1000012（B），约等于976.6KB, 如果需要修改这个参数，那么还要考虑 max.request.size（客户端参数）、max.message.bytes（topic端参数）等参数的影响
message.max.bytes=1000012

```

### 脚本命令

- 启动：bin/kafka-server-start.sh config/server.properties

- 后台启动： bin/kafka-server-start.sh –-daemon config/server.properties

- 查看kafka是否启动：jps -l

- 创建主题：bin/kafka-topics.sh --zookeeper localhost: 2181/kafka --create --topic topic-demo --replication-factor 3 --partitions 4 --replica-assignment 2:0,0:1,1:2,2:1 --config cleanup.policy=compact --config max.message.bytes=10000

  replication-factor含义是副本因子，也就是针对每个分区创建多少个副本，副本数必须小于等于broker数量

  partitions指的是当前topic的分区数

  replica-assignment手动指定分区副本的分配方案

  config指定创建topic的配置参数

- 修改主题：bin/kafka-topics.sh --zookeeper localhost:2181/ --alter --topic topic-config --partitions 3

  修改主题可以变更主题的分区，变更config信息，删除config信息等

  这里注意的是修改主题 ，如果生产者指定了key，就会影响key本身的存储分区，从而可能影响到消费者消费、

  另外分区修改只支持新增，不支持减少

- 删除主题：bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --delete --topic topic-delete

- 查看主题信息：bin/kafka-topics.sh --zookeeper localhost: 2181/ --describe --topic topic1,topic2

  通过额外的参数topics-with-overrides、under-replicated-partitions 和 unavailable-partitions 这三个参数来增加一些附加功能。

  具体见https://juejin.im/book/6844733793220165639/section/6844733793639612424

- 查看主题列表：bin/kafka-topics.sh --zookeeper localhost:2181/ --list

- 生产消息：bin/kafka-console-producer.sh --broker-list localhost:9092 --topic topic-demo

- 消费消息：bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic topic-demo

- 配置管理：https://juejin.im/book/6844733793220165639/section/6844733793639596039

### KafkaAdminClient

可以用来替换脚本，可以在代码里操作kafka

### kafka.admin.TopicCommand

kafka-topics.sh 内部使用TopicCommand创建主题，所以可以手动调用TopicCommand创建主题

```java
//使用TopicCommand创建主题
public static void createTopic(){
    String[] options = new String[]{
            "--zookeeper", "localhost:2181/kafka",
            "--create",
            "--replication-factor", "1",
            "--partitions", "1",
            "--topic", "topic-create-api"
    };
    kafka.admin.TopicCommand.main(options);
}
```

### 分区副本的分配

#### 消费者端

消费者端的分区副本分配是指为消费者指定其可以消费消息的分区

#### 生产者端

生产者端的分区副本分配是指为集群制定创建主题时的分区副本分配方案，即在哪个 broker 中创建哪些分区的副本。

### Topic命名规范

1. 尽量不包含 . _
2. 不以 __ 双下划线开头，因为kafka会认为是内部主题。_
3. 主题的名称必须由大小写字母、数字、点号“.”、连接线“-”、下画线“_”组成，不能为空，不能只有点号“.”，也不能只有双点号“..”，且长度不能超过249。

## 生产者客户端

### 客户端API

```java
Properties properties = new Properties();
properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                       StringSerializer.class.getName());
properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
               StringSerializer.class.getName());
properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
try (KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(properties)) {
  // doSomething
  ProducerRecord<String, String> producerRecord = new ProducerRecord<>(topic, "hello", "hello Kafka!" + i);
  kafkaProducer.send(producerRecord);
} catch (Exception e) {
  e.printStackTrace();
}
```

**kafkaProducer**有多个实例化重载方法

**ProducerRecord**也有多个实例化重载方法

### 客户端发送消息的三种模式

kafkaProducer.send方法返回值并不是void，而是Future\<RecordMetadata>类型对象，所以通过不同的代码实现不同的消息发送模式

- 发后即忘（fire-and-forget） 性能最高，可靠性也最差

  ```java
  try {
      kafkaProducer.send(producerRecord);
  } catch (ExecutionException | InterruptedException e) {
      e.printStackTrace();
  }
  ```

- 同步（sync）直到消息发送成功，或者发生异常。如果发生异常，那么就需要捕获异常并交由外层逻辑处理，可靠，性能不高。这里需要注意的是针对异常的捕获，针对异常要==区分可重试异常==和==不可重试异常==

  ```java
  try {
      Future<RecordMetadata> future = producer.send(record);
    	// 通过get实现同步发送
      RecordMetadata metadata = future.get();
      System.out.println(metadata.topic() + "-" +
              metadata.partition() + ":" + metadata.offset());
  } catch (ExecutionException | InterruptedException e) {
      e.printStackTrace();
  }
  ```

- 异步（async）对于同一个分区而言，如果消息 record1 于 record2 之前先发送（参考上面的示例代码），那么 KafkaProducer 就可以保证对应的 callback1 在 callback2 之前调用，也就是说，回调函数的调用也可以保证分区有序。

  ```java
  producer.send(record, new Callback() {
      @Override
      public void onCompletion(RecordMetadata metadata, Exception exception) {
          if (exception != null) {
              exception.printStackTrace();
          } else {
              System.out.println(metadata.topic() + "-" +
                      metadata.partition() + ":" + metadata.offset());
          }
      }
  });
  ```

### 重要的生产者参数

- **bootstrap.servers **该参数用来指定生产者客户端连接 Kafka 集群所需的 broker 地址清单

- **key.serializer** 和 **value.serializer** broker 端接收的消息必须以字节数组（byte[]）的形式存在。key.serializer 和 value.serializer 这两个参数分别用来指定 key 和 value 序列化操作的序列化器，这两个参数无默认值。注意这里必须填写序列化器的全限定名

- **client.id ** 这个参数用来设定 KafkaProducer 对应的客户端id，默认值为“”

- **acks**：这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的

  ```
  properties.put("acks", "0");
  # 或者
  properties.put(ProducerConfig.ACKS_CONFIG, "0");
  ```

  - ack = 1   当分区的leader副本写入数据成功，就会接收到来自服务器端的响应。如果消息写入 leader 副本并返回成功响应给生产者，且在被其他 follower 副本拉取之前 leader 副本崩溃，那么此时消息还是会丢失，因为新选举的 leader 副本中并没有这条对应的消息。acks 设置为1，是消息可靠性和吞吐量之间的折中方案。
  - ack = 0   生产者发送消息之后不需要等待任何服务端的响应，这种方案可以达到最大的性能吞吐量
  - ack = -1  或  ack = all：生产者在消息发送之后，需要等待 ISR 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。在其他配置环境相同的情况下，acks 设置为 -1（all） 可以达到最强的可靠性。但这并不意味着消息就一定可靠，因为ISR中可能只有 leader 副本，这样就退化成了 acks=1 的情况。要获得更高的消息可靠性需要配合 ==min.insync.replicas== 等参数的联动

- **max.request.size**：这个参数用来限制生产者客户端能发送的消息的最大值，默认值为1048576B，即1MB。使用时应该注意一些联动的配置，比如 broker 端的 message.max.bytes 参数等，否则会导致异常。

- **retries和retry.backoff.ms**：retries—>重试次数，默认为0；retry.backoff.ms—> 重试间隔

- **compression.type**：压缩方式，默认为“none”不压缩。该参数还可以配置为“gzip”“snappy”和“lz4”。对消息进行压缩可以极大地减少网络传输量、降低网络I/O，从而提高整体的性能。消息压缩是一种使用时间换空间的优化方式，如果对时延有一定的要求，则不推荐对消息进行压缩。

- **connections.max.idle.ms**：这个参数用来指定在多久之后关闭闲置的连接，默认值是540000（ms），即9分钟。

- **linger.ms**:这个参数用来指定生产者发送 ProducerBatch 之前等待更多消息（ProducerRecord）加入 ProducerBatch 的时间，默认值为0。生产者客户端会在 ProducerBatch 被填满或等待时间超过 linger.ms 值时发送出去。增大这个参数的值会增加消息的延迟，但是同时能提升一定的吞吐量。这个 linger.ms 参数与 TCP 协议中的 Nagle 算法有异曲同工之妙。

- **receive.buffer.bytes**: 这个参数用来设置 Socket 接收消息缓冲区（SO_RECBUF）的大小，默认值为32768（B），即32KB。如果设置为-1，则使用操作系统的默认值。如果 Producer 与 Kafka 处于不同的机房，则可以适地调大这个参数值。

- **send.buffer.bytes**:这个参数用来设置 Socket 发送消息缓冲区（SO_SNDBUF）的大小，默认值为131072（B），即128KB。与 receive.buffer.bytes 参数一样，如果设置为-1，则使用操作系统的默认值。

- **request.timeout.ms** 这个参数用来配置 Producer 等待请求响应的最长时间，默认值为30000（ms）。请求超时之后可以选择进行重试。注意这个参数需要比 broker 端参数 replica.lag.time.max.ms 的值要大，这样可以减少因客户端重试而引起的消息重复的概率。

- **transactional.id**： 设置事务id，必须唯一

详见：https://juejin.im/book/6844733793220165639/section/6844733793627013134

### 序列化器、分区器、拦截器

实现kafka提供的特定接口，实现序列化器、分区器、拦截器。消息在通过 send() 方法发往 broker 的过程中，有可能需要经过拦截器（Interceptor）、序列化器（Serializer）和分区器（Partitioner）的一系列作用之后才能被真正地发往 broker。

序列化器是必须的，kafka针对常见类型提供了各种序列化器，如果不满足要求，可以自定义。

拦截器不是必须的，有需要可以自定义拦截器实现类似于数据统计、日志记录等需求。

分区器如果我们指定了partition，那么就不需要分区器的作用，否则需要，Kafka 中提供的默认分区器是 org.apache.kafka.clients.producer.internals.DefaultPartitioner。如果 key 不为 null，那么默认的分区器会对 key 进行哈希（采用 MurmurHash2 算法，具备高运算性能及低碰撞率），最终根据得到的哈希值来计算分区号，拥有相同 key 的消息会被写入同一个分区。如果 key 为 null，那么消息将会以轮询的方式发往主题内的各个可用分区。

### 分区的策略

分区器如果我们指定了partition，那么就不需要分区器的作用，否则需要，Kafka 中提供的默认分区器是 org.apache.kafka.clients.producer.internals.DefaultPartitioner。如果 key 不为 null，那么默认的分区器会对 key 进行哈希（采用 MurmurHash2 算法，具备高运算性能及低碰撞率），最终根据得到的哈希值来计算分区号，拥有相同 key 的消息会被写入同一个分区。如果 key 为 null，那么消息将会以轮询的方式发往主题内的各个可用分区。

### 生产者客户端原理分析---重要

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200817145101409.png)

- ProducerRecord：主线程将消息包装称为ProducerRecord，定义的一条消息

- RecordAccumulator：消息收集器，KafkaProducer生产消息，将消息存入RecordAccumulator，sender线程从RecordAccumulator中获取，实现消息发送；buffer.memory RecordAccumulator的缓存大小，默认值为 33554432B，即32MB；max.block.ms 当缓存达到上限的阻塞时间，超出时间抛出异常

- ProducerBatch：双端队列，主线程将ProducerRecord存入队列的尾部，sender线程从头部读取数据发送。ProducerBatch 中可以包含一至多个 ProducerRecord，是一个批次消息的概念，这样可以使字节的使用更加紧凑，减少网络通信的次数，提升系统吞吐量。

- Sender：从RecordAccumulator的分区缓存中获取缓存的消息，sender将<分区, Deque< ProducerBatch>> 的保存形式转变成 <Node, List< ProducerBatch> 的形式，由于一个partition对应一个broker（Node节点）的partition Leader，所以这里是做一个应用逻辑层面到网络I/O层面的转换。在转换成 <Node, List> 的形式之后，Sender 还会进一步封装成 <Node, Request> 的形式，这样就可以将 Request 请求发往各个 Node 了。

- InFlightRequests：请求在从 Sender 线程发往 Kafka 之前还会保存到 InFlightRequests 中，InFlightRequests 保存对象的具体形式为 Map<NodeId, Deque>，它的主要作用是缓存了已经发出去但还没有收到响应的请求（NodeId 是一个 String 类型，表示节点的 id 编号）。与此同时，InFlightRequests 还提供了许多管理类的方法，并且通过配置参数还可以限制每个连接（也就是客户端与 Node 之间的连接）最多缓存的请求数。这个配置参数为 max.in.flight.requests. per. connection，默认值为5，即每个连接最多只能缓存5个未响应的请求，超过该数值之后就不能再向这个连接发送更多的请求了，除非有缓存的请求收到了响应（Response）。通过比较 Deque 的 size 与这个参数的大小来判断对应的 Node 中是否已经堆积了很多未响应的消息，如果真是如此，那么说明这个 Node 节点负载较大或网络连接有问题，再继续向其发送请求会增大请求超时的可能。

- leastLoadedNode：对于InFlightRequests来讲，缓存最少的节点就是leastLoadedNode，也就是负载最小的节点。如下图中的Node1

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200817150751169.png)

- 元数据：元数据是指 Kafka 集群的元数据，这些元数据具体记录了集群中有哪些主题，这些主题有哪些分区，每个分区的 leader 副本分配在哪个节点上，follower 副本分配在哪些节点上，哪些副本在 AR、ISR 等集合中，集群中有哪些节点，控制器节点又是哪一个等信息。当客户端中没有需要使用的元数据信息时，比如没有指定的主题信息，或者超过 metadata.max.age.ms 时间没有更新元数据都会引起元数据的更新操作。客户端参数 metadata.max.age.ms 的默认值为300000，即5分钟。元数据的更新操作是在客户端内部进行的，对客户端的外部使用者不可见。当需要更新元数据时，会先挑选出 leastLoadedNode，然后向这个 Node 发送 MetadataRequest 请求来获取具体的元数据信息。这个更新操作是由 Sender 线程发起的，在创建完 MetadataRequest 之后同样会存入 InFlightRequests，之后的步骤就和发送消息时的类似。元数据虽然由 Sender 线程负责更新，但是主线程也需要读取这些信息，这里的数据同步通过 synchronized 和 final 关键字来保障。

### 消息的有序性

和消息有序性相关的参数：retries和retry.backoff.ms、max.in.flight.requests.per.connection

retries和retry.backoff.ms的重试机制会导致如果第一批次消息写入失败，而第二批次消息写入成功，那么生产者会重试发送第一批次的消息，此时如果第一批次的消息写入成功，那么这两个批次的消息就出现了错序。

所以避免消息乱序的方案是：**max.in.flight.requests.per.connection 配置为1，而不是把 retries 配置为0 ** 也就是针对每一个node，缓存中只能最多有一个请求等待效应，这样实现了同步顺序执行的目的，类似MySql的serializer



## 消费者客户端

### 消费者与消费组

消费者组是kafka中的概念，一个消费者组消费topic时，消费者组中的消费者共同消费topic中的消息。不同的消费者消费自己的分区数据，不同的消费者组互不影响。通过不同的策略可以实现point2point和发布订阅模式。

- 如果所有的消费者都隶属于同一个消费组，那么所有的消息都会被均衡地投递给每一个消费者，即每条消息只会被一个消费者处理，这就相当于点对点模式的应用。
- 如果所有的消费者都隶属于不同的消费组，那么所有的消息都会被广播给所有的消费者，即每条消息会被所有的消费者处理，这就相当于发布/订阅模式的应用。

这个概念比较好理解，具体：https://juejin.im/book/6844733793220165639/section/6844733793627013133

### 客户端基础用法、反序列化



### 消费位移提交

背景：当我们不指定auto.offset.reset配置时，采用默认的消费策略--->返回没有被消费过的数据

要做到上述，必须在服务器端记录上一次消费的offset，并且这个数据必须做持久化。

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200817195054821.png)

首先必须掌握三个变量的概念：

1. ==last consumed offset==上图的x，代表上一次消费的位置
2. ==position==消费者需要提交的位置，x+1，也就是下一条需要拉取的位移
3. ==committed offset==消费者提交的数据

KafkaConsumer 类提供了 position(TopicPartition) 和 committed(TopicPartition) 两个方法来分别获取上面所说的 position 和 committed offset 的值。这两个方法的定义如下所示。

```
public long position(TopicPartition partition)
public OffsetAndMetadata committed(TopicPartition partition)
```

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200817195935681.png)

综上：针对这三个变量的关系就是，last consumed offset是我消费的消息的位移；committed offset是我消费之后通过consumer.commitSync()手动提交的位移；position是我下一次需要拉取数据的位移；并且 position 和 committed offset 并不会一直相同！如果我们开启自动提交，自动提交会每5s自动提交一次，那在这5s内，position 和 committed offset可能会不同

### 消息丢失和重复消费

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200817195935681.png)

自动提交消费位移的方式非常简单，但是随之而来的问题就是消息丢失和重复消费

重复消费：如上图，位移提交的动作是在消费完所有拉取到的消息之后才执行的，那么当消费 x+5 的时候遇到了异常，在故障恢复之后，我们重新拉取的消息是从 x+2 开始的。也就是说，x+2 至 x+4 之间的消息又重新消费了一遍，故而又发生了重复消费的现象。

消息丢失：拉取线程A不断地拉取消息并存入本地缓存，比如在 BlockingQueue 中，另一个处理线程B从缓存中读取消息并进行相应的逻辑处理。假设目前进行到了第 y+1 次拉取，以及第m次位移提交的时候，也就是 x+6 之前的位移已经确认提交了，处理线程B却还正在消费 x+3 的消息。此时如果处理线程B发生了异常，待其恢复之后会从第m此位移提交处，也就是 x+6 的位置开始拉取消息，那么 x+3 至 x+6 之间的消息就没有得到相应的处理，这样便发生消息丢失的现象。

### 手动提交

```java
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
```

- commitSync()

  ```java
  public void commitSync();
  public void commitSync(final Map<TopicPartition, OffsetAndMetadata> offsets)
  ```

  常见的用法：针对每个分区进行消费，然后partition层面提交

  ```java
  try {
      while (isRunning.get()) {
          ConsumerRecords<String, String> records = consumer.poll(1000);
          for (TopicPartition partition : records.partitions()) {
              List<ConsumerRecord<String, String>> partitionRecords =
                      records.records(partition);
              for (ConsumerRecord<String, String> record : partitionRecords) {
                  //do some logical processing.
              }
              long lastConsumedOffset = partitionRecords
                      .get(partitionRecords.size() - 1).offset();
              consumer.commitSync(Collections.singletonMap(partition,
                      new OffsetAndMetadata(lastConsumedOffset + 1)));
          }
      }
  } finally {
      consumer.close();
  }
  ```

- commitAsync()

  ```java
  public void commitAsync()
  public void commitAsync(OffsetCommitCallback callback)
  public void commitAsync(final Map<TopicPartition, OffsetAndMetadata> offsets,
              OffsetCommitCallback callback)
  ```

  如果采用异步提交，就会存在提交失败的问题。如果引入了重试机制，就有第一次失败的提交重复执行覆盖了后面提交的结果。解决这个问题的方案可以采用乐观锁的思想，为此我们可以设置一个递增的序号来维护异步提交的顺序，每次位移提交之后就增加序号相对应的值。在遇到位移提交失败需要重试的时候，可以检查所提交的位移和序号的值的大小，如果前者小于后者，则说明有更大的位移已经提交了，不需要再进行本次重试；如果两者相同，则说明可以进行重试提交。除非程序编码错误，否则不会出现前者大于后者的情况。这种方式提高了系统的复杂度。

  除此之外，可以采用try finally语句保证在消费者正常退出或者消费者再均衡时手动同步提交

### 控制和关闭消费

KafkaConsumer 提供了对消费速度进行控制的方法，在有些应用场景下我们可能需要暂停某些分区的消费而先消费其他分区，当达到一定条件时再恢复这些分区的消费。KafkaConsumer 中使用 pause() 和 resume() 方法来分别实现暂停某些分区在拉取操作时返回数据给客户端和恢复某些分区向客户端返回数据的操作。这两个方法的具体定义如下：

```java
public void pause(Collection<TopicPartition> partitions)
public void resume(Collection<TopicPartition> partitions)
```

KafkaConsumer 还提供了一个无参的 paused() 方法来返回被暂停的分区集合，此方法的具体定义如下：

```java
public Set<TopicPartition> paused()
```



我们采用while循环来实现持续的消费，可以使用 while(isRunning.get()) 的方式，这样可以通过在其他地方设定 isRunning.set(false) 来退出 while 循环。还有一种方式是调用 KafkaConsumer 的 wakeup() 方法，wakeup() 方法是 KafkaConsumer 中唯一可以从其他线程里安全调用的方法（KafkaConsumer 是非线程安全的，可以通过14节了解更多细节），调用 wakeup() 方法后可以退出 poll() 的逻辑，并抛出 WakeupException 的异常，我们也不需要处理 WakeupException 的异常，它只是一种跳出循环的方式。

当跳出循环之后，手动执行资源的关闭==consumer.close()==

```java
public void close()
public void close(Duration timeout)
@Deprecated
public void close(long timeout, TimeUnit timeUnit)
```

### 指定位移消费

```java
public void seek(TopicPartition partition, long offset)
```

seek方法中的partation参数表示分区的信息，offset指定从那个位置开始消费，seek方法只能重置消费者消费的位置，再次之前需要调用poll方法实现消费者分区的分配。

常用的seek位移方法：

```java
// 获取endOffset
public Map<TopicPartition, Long> endOffsets(
            Collection<TopicPartition> partitions)
public Map<TopicPartition, Long> endOffsets(
            Collection<TopicPartition> partitions,
            Duration timeout)
// 获取begainOffset            
public Map<TopicPartition, Long> beginningOffsets(
            Collection<TopicPartition> partitions)
public Map<TopicPartition, Long> beginningOffsets(
            Collection<TopicPartition> partitions,
            Duration timeout)
// 从头或者尾开始消费
public void seekToBeginning(Collection<TopicPartition> partitions)
public void seekToEnd(Collection<TopicPartition> partitions)
// 从指定时间开始消费
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(
            Map<TopicPartition, Long> timestampsToSearch)
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(
            Map<TopicPartition, Long> timestampsToSearch, 
            Duration timeout)
```

位移越界：通过seek方法指定消费位置时 ，如果超出原本分区的offset，也会根据auto.offset.reset 参数的默认值来将拉取位置重置

### 再均衡

 在回调接口ConsumerRebalanceListener中定义了两个回调方法：

1. void onPartitionsRevoked(Collection partitions) 这个方法会在再均衡开始之前和消费者停止读取消息之后被调用。可以通过这个回调方法来处理消费位移的提交，以此来避免一些不必要的重复消费现象的发生。参数 partitions 表示再均衡前所分配到的分区。
2. void onPartitionsAssigned(Collection partitions) 这个方法会在重新分配分区之后和消费者开始读取消费之前被调用。参数 partitions 表示再均衡后所分配到的分区。

```java
consumer.subscribe(Arrays.asList("java-topic_test"), new ConsumerRebalanceListener() {
  public void onPartitionsRevoked(Collection<TopicPartition> collection) {
    consumer.commitAsync(); // 提交偏移量
  }
  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
    // 获取该分区下已消费的偏移量
    long commitedOffset = -1;
    for (TopicPartition topicPartition : partitions) {
      // 获取该分区下已消费的偏移量
      commitedOffset = consumer.committed(topicPartition).offset();
      // 重置偏移量到上一次提交的偏移量下一个位置处开始消费
      consumer.seek(topicPartition, commitedOffset + 1);
    }
  }
});
```



可以通过onPartitionsRevoked来实现offset的记录或者存储，通过onPartitionsAssigned 调用seek方法实现回溯消费，避免消息的重复和丢失

### 拦截器

拦截器比较简单，提供了三个方法，onConsume在消息poll方法之前被调用，onCommit在提交位移之后被调用，可以用来记录

- public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records)；
- public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets)；
- public void close()。

### 消费者端配置

- **enable.auto.commit**   默认为true，开启自动提交，当然这个默认的自动提交不是每消费一条消息就提交一次，而是定期提交，这个定期的周期时间由客户端参数 auto.commit.interval.ms 配置，默认值为5秒，此参数生效的前提是 enable.auto.commit 参数为 true
- **auto.offset.reset** 参数的默认值时latest，如果将 auto.offset.reset 参数配置为“earliest”，那么消费者会从起始处，也就是0开始消费。

- **fetch.min.bytes** 每次poll拉取最小数据量，默认1B，可以通过增大该参数提升系统吞吐量，但是会造成延迟增加
- **fetch.max.bytes** 每次poll拉取最大数据量，默认50M，并且为了防止消息无法消费，如果发现第一条大于50M，正常消费
- **fetch.max.wait.ms** 拉取数据最长等待时间，默认500ms， 如果拉取的数据小于fetch.min.bytes，等待fetch.max.wait.ms时间后返回
- **max.partition.fetch.bytes** 这个参数用来配置从每个分区里返回给 Consumer 的最大数据量，默认值为1048576（B），即1MB。
- **max.poll.records** 每次拉取数据最多条数，默认500条
- **connections.max.idle.ms** 用来设置多久关闭空闲连接

更多的：https://juejin.im/book/6844733793220165639/section/6844733793635401736

## 分区副本

### kafka重平衡

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-partitions
Topic:topic-partitions	PartitionCount:3	ReplicationFactor:3	Configs: 
    Topic: topic-partitions	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2
    Topic: topic-partitions	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 0,1,2
    Topic: topic-partitions	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```

对于上面的Topic info来讲，分区的Leader副本分配是均匀的，当假设broker1挂掉，之后又重连，此时会导致leader的分配不在均匀。

为此，我们可以使用kafka-perferred-replica-election.sh脚本对分区的leader进行重平衡。除此之外，可以指定 --path-to-json-file election.json，具体使用见掘金

```
bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181/kafka
```

上述的重平衡方式会严重影响kafka系统的性能
