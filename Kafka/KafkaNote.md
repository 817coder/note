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

Kafka 消费端也具备一定的容灾能力。Consumer 使用拉（Pull）模式从服务端拉取消息，并且保存消费的具体位置，当消费者宕机后恢复上线时可以根据之前保存的消费位置重新拉取需要的消息进行消费，这样就不会造成消息丢失。

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

### 一些重要的服务端参数

#### 1. zookeeper.connect

该参数指明 broker 要连接的 ZooKeeper 集群的服务地址（包含端口号），没有默认值，且此参数为必填项。可以配置为 localhost:2181，如果 ZooKeeper 集群中有多个节点，则可以用逗号将每个节点隔开，类似于 localhost1:2181,localhost2:2181,localhost3:2181 这种格式。最佳的实践方式是再加一个 chroot 路径，这样既可以明确指明该 chroot 路径下的节点是为 Kafka 所用的，也可以实现多个 Kafka 集群复用一套 ZooKeeper 集群，这样可以节省更多的硬件资源。包含 chroot 路径的配置类似于 localhost1:2181,localhost2:2181,localhost3:2181/kafka 这种，如果不指定 chroot，那么默认使用 ZooKeeper 的根路径。

#### 2. listeners

该参数指明 broker 监听客户端连接的地址列表，即为客户端要连接 broker 的入口地址列表，配置格式为 protocol1://hostname1:port1,protocol2://hostname2:port2，其中 protocol 代表协议类型，Kafka 当前支持的协议类型有 PLAINTEXT、SSL、SASL_SSL 等，如果未开启安全认证，则使用简单的 PLAINTEXT 即可。hostname 代表主机名，port 代表服务端口，此参数的默认值为 null。比如此参数配置为 PLAINTEXT://198.162.0.2:9092，如果有多个地址，则中间以逗号隔开。如果不指定主机名，则表示绑定默认网卡，注意有可能会绑定到127.0.0.1，这样无法对外提供服务，所以主机名最好不要为空；如果主机名是0.0.0.0，则表示绑定所有的网卡。

与此参数关联的还有 advertised.listeners，作用和 listeners 类似，默认值也为 null。不过 advertised.listeners 主要用于 IaaS（Infrastructure as a Service）环境，比如公有云上的机器通常配备有多块网卡，即包含私网网卡和公网网卡，对于这种情况而言，可以设置 advertised.listeners 参数绑定公网IP供外部客户端使用，而配置 listeners 参数来绑定私网IP地址供 broker 间通信使用。

#### 3. broker.id
该参数用来指定 Kafka 集群中 broker 的唯一标识，默认值为-1。如果没有设置，那么 Kafka 会自动生成一个。

#### 4. log.dir和log.dirs

Kafka 把所有的消息都保存在磁盘上，而这两个参数用来配置 Kafka 日志文件存放的根目录。一般情况下，log.dir 用来配置单个根目录，而 log.dirs 用来配置多个根目录（以逗号分隔），但是 Kafka 并没有对此做强制性限制，也就是说，log.dir 和 log.dirs 都可以用来配置单个或多个根目录。log.dirs 的优先级比 log.dir 高，但是如果没有配置 log.dirs，则会以 log.dir 配置为准。默认情况下只配置了 log.dir 参数，其默认值为 /tmp/kafka-logs。

#### 5. message.max.bytes

该参数用来指定 broker 所能接收消息的最大值，默认值为1000012（B），约等于976.6KB。如果 Producer 发送的消息大于这个参数所设置的值，那么（Producer）就会报出 RecordTooLargeException 的异常。如果需要修改这个参数，那么还要考虑 max.request.size（客户端参数）、max.message.bytes（topic端参数）等参数的影响。为了避免修改此参数而引起级联的影响，建议在修改此参数之前考虑分拆消息的可行性。

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

KafkaProducer 是线程安全的，可以在多个线程中共享单个 KafkaProducer 实例，也可以将 KafkaProducer 实例进行池化来供其他线程调用。

### 消息对象 ProducerRecord

ProducerRecord 类的定义如下（只截取成员变量）：
```java
public class ProducerRecord<K, V> {
    private final String topic; //主题
    private final Integer partition; //分区号
    private final Headers headers; //消息头部
    private final K key; //键
    private final V value; //值
    private final Long timestamp; //消息的时间戳
    //省略其他成员方法和构造方法
}
```

其中 topic 和 partition 字段分别代表消息要发往的主题和分区号。headers 字段是消息的头部，Kafka 0.11.x 版本才引入这个属性，它大多用来设定一些与应用相关的信息，如无需要也可以不用设置。key 是用来指定消息的键，它不仅是消息的附加信息，还可以用来计算分区号进而可以让消息发往特定的分区。前面提及消息以主题为单位进行归类，而这个 key 可以让消息再进行二次归类，同一个 key 的消息会被划分到同一个分区中。

有 key 的消息还可以支持日志压缩的功能。value 是指消息体，一般不为空，如果为空则表示特定的消息—墓碑消息。timestamp 是指消息的时间戳，它有 CreateTime 和 LogAppendTime 两种类型，前者表示消息创建的时间，后者表示消息追加到日志文件的时间。



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

- 同步（sync）直到消息发送成功，或者发生异常。如果发生异常，那么就需要捕获异常并交由外层逻辑处理，可靠，性能不高。这里需要注意的是针对异常的捕获，针对异常要区分可重试异常和不可重试异常。常见的可重试异常有：NetworkException、LeaderNotAvailableException、UnknownTopicOrPartitionException、NotEnoughReplicasException、NotCoordinatorException 等。比如 NetworkException 表示网络异常，这个有可能是由于网络瞬时故障而导致的异常，可以通过重试解决；又比如 LeaderNotAvailableException 表示分区的 leader 副本不可用，这个异常通常发生在 leader 副本下线而新的 leader 副本选举完成之前，重试之后可以重新恢复。不可重试的异常，比如第2节中提及的 RecordTooLargeException 异常，暗示了所发送的消息太大，KafkaProducer 对此不会进行任何重试，直接抛出异常。对于可重试的异常，如果配置了 retries 参数，那么只要在规定的重试次数内自行恢复了，就不会抛出异常。retries 参数的默认值为0，配置方式参考如下：示例中配置了10次重试。如果重试了10次之后还没有恢复，那么仍会抛出异常，进而发送的外层逻辑就要处理这些异常了。

  ```java
  props.put(ProducerConfig.RETRIES_CONFIG, 10);
  ```
  同步发送消息

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
  
对于同一个分区而言，如果消息 record1 于 record2 之前先发送（参考上面的示例代码），那么 KafkaProducer 就可以保证对应的 callback1 在 callback2 之前调用，也就是说，回调函数的调用也可以保证分区有序。

通常，一个 KafkaProducer 不会只负责发送单条消息，更多的是发送多条消息，在发送完这些消息之后，需要调用 KafkaProducer 的 close() 方法来回收资源。下面的示例中发送了100条消息，之后就调用了 close() 方法来回收所占用的资源：

```java
int i = 0;
while (i < 100) {
    ProducerRecord<String, String> record =
            new ProducerRecord<>(topic, "msg"+i++);
    try {
        producer.send(record).get();
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
    }
}
producer.close();
```
close() 方法会阻塞等待之前所有的发送请求完成后再关闭 KafkaProducer。与此同时，KafkaProducer 还提供了一个带超时时间的 close() 方法，具体定义如下：
```java
public void close(long timeout, TimeUnit timeUnit)
```
如果调用了带超时时间 timeout 的 close() 方法，那么只会在等待 timeout 时间内来完成所有尚未完成的请求处理，然后强行退出。在实际应用中，一般使用的都是无参的 close() 方法。

### 重要的生产者参数

- **bootstrap.servers **该参数用来指定生产者客户端连接 Kafka 集群所需的 broker 地址清单

- **key.serializer** 和 **value.serializer** broker 端接收的消息必须以字节数组（byte[]）的形式存在。key.serializer 和 value.serializer 这两个参数分别用来指定 key 和 value 序列化操作的序列化器，这两个参数无默认值。注意这里必须填写序列化器的全限定名

- **client.id ** 这个参数用来设定 KafkaProducer 对应的客户端id，默认值为"",如果客户端不设置，则 KafkaProducer 会自动生成一个非空字符串，内容形式如“producer-1”、“producer-2”，即字符串“producer-”与数字的拼接。

- **acks**：这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的

  ```
  properties.put("acks", "0");
  # 或者
  properties.put(ProducerConfig.ACKS_CONFIG, "0");
  ```

  - ack = 1   默认值即为1。生产者发送消息之后，只要分区的 leader 副本成功写入消息，那么它就会收到来自服务端的成功响应。如果消息无法写入 leader 副本，比如在 leader 副本崩溃、重新选举新的 leader 副本的过程中，那么生产者就会收到一个错误的响应，为了避免消息丢失，生产者可以选择重发消息。如果消息写入 leader 副本并返回成功响应给生产者，且在被其他 follower 副本拉取之前 leader 副本崩溃，那么此时消息还是会丢失，因为新选举的 leader 副本中并没有这条对应的消息。acks 设置为1，是消息可靠性和吞吐量之间的折中方案。
  - ack = 0   生产者发送消息之后不需要等待任何服务端的响应，如果在消息从发送到写入 Kafka 的过程中出现某些异常，导致 Kafka 并没有收到这条消息，那么生产者也无从得知，消息也就丢失了。在其他配置环境相同的情况下，acks 设置为0可以达到最大的吞吐量。
  - ack = -1  或  ack = all：生产者在消息发送之后，需要等待 ISR 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。在其他配置环境相同的情况下，acks 设置为 -1（all） 可以达到最强的可靠性。但这并不意味着消息就一定可靠，因为ISR中可能只有 leader 副本，这样就退化成了 acks=1 的情况。要获得更高的消息可靠性需要配合 ==min.insync.replicas== 等参数的联动

- **max.request.size**：这个参数用来限制生产者客户端能发送的消息的最大值，默认值为1048576B，即1MB。参数还涉及一些其他参数的联动，比如 broker 端的 message.max.bytes 参数，如果配置错误可能会引起一些不必要的异常。比如将 broker 端的 message.max.bytes 参数配置为10，而 max.request.size 参数配置为20，那么当我们发送一条大小为15B的消息时，生产者客户端就会报出如下的异常：org.apache.kafka.common.errors.RecordTooLargeException: The request included a message larger than the max message size the server will accept.

- **retries和retry.backoff.ms**：retries 参数用来配置生产者重试的次数，默认值为0，即在发生异常的时候不进行任何重试动作。消息在从生产者发出到成功写入服务器之前可能发生一些临时性的异常，比如网络抖动、leader 副本的选举等，这种异常往往是可以自行恢复的，生产者可以通过配置 retries 大于0的值，以此通过内部重试来恢复而不是一味地将异常抛给生产者的应用程序。如果重试达到设定的次数，那么生产者就会放弃重试并返回异常。不过并不是所有的异常都是可以通过重试来解决的，比如消息太大，超过 max.request.size 参数配置的值时，这种方式就不可行了。

重试还和另一个参数 retry.backoff.ms 有关，这个参数的默认值为100，它用来设定两次重试之间的时间间隔，避免无效的频繁重试。在配置 retries 和 retry.backoff.ms 之前，最好先估算一下可能的异常恢复时间，这样可以设定总的重试时间大于这个异常恢复时间，以此来避免生产者过早地放弃重试。

Kafka 可以保证同一个分区中的消息是有序的。如果生产者按照一定的顺序发送消息，那么这些消息也会顺序地写入分区，进而消费者也可以按照同样的顺序消费它们。

对于某些应用来说，顺序性非常重要，比如 MySQL 的 binlog 传输，如果出现错误就会造成非常严重的后果。如果将 retries 参数配置为非零值，并且 max.in.flight.requests.per.connection 参数配置为大于1的值，那么就会出现错序的现象：如果第一批次消息写入失败，而第二批次消息写入成功，那么生产者会重试发送第一批次的消息，此时如果第一批次的消息写入成功，那么这两个批次的消息就出现了错序。一般而言，在需要保证消息顺序的场合建议把参数 max.in.flight.requests.per.connection 配置为1，而不是把 retries 配置为0，不过这样也会影响整体的吞吐。

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

如果 key 不为 null，那么计算得到的分区号会是所有分区中的任意一个；如果 key 为 null 并且有可用分区时，那么计算得到的分区号仅为可用分区中的任意一个，注意两者之间的差别。

### 分区的策略

分区器如果我们指定了partition，那么就不需要分区器的作用，否则需要，Kafka 中提供的默认分区器是 org.apache.kafka.clients.producer.internals.DefaultPartitioner。如果 key 不为 null，那么默认的分区器会对 key 进行哈希（采用 MurmurHash2 算法，具备高运算性能及低碰撞率），最终根据得到的哈希值来计算分区号，拥有相同 key 的消息会被写入同一个分区。如果 key 为 null，那么消息将会以轮询的方式发往主题内的各个可用分区。

### 注意⚠️
于 KafkaProducer 而言，它是线程安全的，我们可以在多线程的环境中复用它，而对于下面要讲解的消费者客户端 KafkaConsumer 而言，它是非线程安全的，因为它具备了状态。

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

消费组一共有 Dead、Empty、PreparingRebalance、CompletingRebalance、Stable 这几种状态，正常情况下，一个具有消费者成员的消费组的状态为 Stable。

### 客户端基础用法、反序列化



### 消费位移提交

背景：当我们不指定auto.offset.reset配置时，采用默认的消费策略--->返回没有被消费过的数据

要做到上述，必须在服务器端记录上一次消费的offset，并且这个数据必须做持久化。

在旧消费者客户端中，消费位移是存储在 ZooKeeper 中的。而在新消费者客户端中，消费位移存储在 Kafka 内部的主题__consumer_offsets 中。这里把将消费位移存储起来（持久化）的动作称为“提交”，消费者在消费完消息之后需要执行消费位移的提交。

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

综上：针对这三个变量的关系就是，last consumed offset是我消费的消息的位移；committed offset是我消费之后通过consumer.commitSync()手动提交的位移；position是我下一次需要拉取数据的位移；并且 position 和 committed offset 并不会一直相同！如果我们开启自动提交，position可以理解为处理完成的消息的offset+1，committed offset可以理解为是拉取批次最大位移的offset+1，在整批消息没有消费完成之前，那么这两个值会不相同，也就是说如果5s自动提交一次，那在这5s内，position 和 committed offset可能会不同。

在 Kafka 中默认的消费位移的提交方式是自动提交，这个由消费者客户端参数 enable.auto.commit 配置，默认值为 true。当然这个默认的自动提交不是每消费一条消息就提交一次，而是定期提交，这个定期的周期时间由客户端参数 auto.commit.interval.ms 配置，默认值为5秒，此参数生效的前提是 enable.auto.commit 参数为 true。在代码清单8-1中并没有展示出这两个参数，说明使用的正是默认值。

在默认的方式下，消费者每隔5秒会将拉取到的每个分区中最大的消息位移进行提交。自动位移提交的动作是在 poll() 方法的逻辑里完成的，在每次真正向服务端发起拉取请求之前会检查是否可以进行位移提交，如果可以，那么就会提交上一次轮询的位移。

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

```java
finally {
      consumer.commitSync();
      consumer.close();
  }
```

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

我们采用while循环来实现持续的消费，可以使用 while(isRunning.get()) 的方式，这样可以通过在其他地方设定 isRunning.set(false) 来退出 while 循环。还有一种方式是调用 KafkaConsumer 的 wakeup() 方法，wakeup() 方法是 KafkaConsumer 中唯一可以从其他线程里安全调用的方法（KafkaConsumer 是非线程安全的），调用 wakeup() 方法后可以退出 poll() 的逻辑，并抛出 WakeupException 的异常，我们也不需要处理 WakeupException 的异常，它只是一种跳出循环的方式。

当跳出循环之后，手动执行资源的关闭==consumer.close()==

```java
public void close()
public void close(Duration timeout)
@Deprecated
public void close(long timeout, TimeUnit timeUnit)
```

一个相对完整的消费程序的逻辑可以参考下面的伪代码：

```java
consumer.subscribe(Arrays.asList(topic));
try {
    while (running.get()) {
        //consumer.poll(***)
        //process the record.
        //commit offset.
    }
} catch (WakeupException e) {
    // ingore the error
} catch (Exception e){
    // do some logic process.
} finally {
    // maybe commit offset.
    consumer.close();
}
```

### 指定位移消费

一个新的消费组建立的时候，它根本没有可以查找的消费位移。或者消费组内的一个新消费者订阅了一个新的主题，它也没有可以查找的消费位移。当 __consumer_offsets 主题中有关这个消费组的位移信息过期而被删除后，它也没有可以查找的消费位移。

在 Kafka 中每当消费者查找不到所记录的消费位移时，就会根据消费者客户端参数 auto.offset.reset 的配置来决定从何处开始进行消费，这个参数的默认值为“latest”，表示从分区末尾开始消费消息。如果将 auto.offset.reset 参数配置为“earliest”，那么消费者会从起始处，也就是0开始消费。auto.offset.reset 参数还有一个可配置的值—“none”，配置为此值就意味着出现查到不到消费位移的时候，既不从最新的消息位置处开始消费，也不从最早的消息位置处开始消费，此时会报出 NoOffsetForPartitionException 异常.

seek() 方法中的参数 partition 表示分区，而 offset 参数用来指定从分区的哪个位置开始消费。seek() 方法只能重置消费者分配到的分区的消费位置，而分区的分配是在 poll() 方法的调用过程中实现的。也就是说，在执行 seek() 方法之前需要先执行一次 poll() 方法，等到分配到分区之后才可以重置消费位置。这里需要注意如果我们将seek之前的 poll() 方法的参数设置为0，即这一行替换为：consumer.poll(Duration.ofMillis(0));在此之后，会发现 seek() 方法并未有任何作用。因为当 poll() 方法中的参数为0时，此方法立刻返回，那么 poll() 方法内部进行分区分配的逻辑就会来不及实施。也就是说，消费者此时并未分配到任何分区。所以seek方法的使用示例：

```java
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList(topic));
Set<TopicPartition> assignment = new HashSet<>();
while (assignment.size() == 0) {//如果不为0，则说明已经成功分配到了分区
    consumer.poll(Duration.ofMillis(100));
    assignment = consumer.assignment();
}
for (TopicPartition tp : assignment) {
    consumer.seek(tp, 10);
}
while (true) {
    ConsumerRecords<String, String> records =
            consumer.poll(Duration.ofMillis(1000));
    //consume the record.
}
```

seek方法中的partation参数表示分区的信息，offset指定从那个位置开始消费，seek方法只能重置消费者消费的位置，再次之前需要调用poll方法实现消费者分区的分配。

常用的seek位移方法：

```java
// consumer获取endOffset
public Map<TopicPartition, Long> endOffsets(
            Collection<TopicPartition> partitions)
public Map<TopicPartition, Long> endOffsets(
            Collection<TopicPartition> partitions,
            Duration timeout)
// consumer获取begainOffset            
public Map<TopicPartition, Long> beginningOffsets(
            Collection<TopicPartition> partitions)
public Map<TopicPartition, Long> beginningOffsets(
            Collection<TopicPartition> partitions,
            Duration timeout)
// consumer从头或者尾开始消费
public void seekToBeginning(Collection<TopicPartition> partitions)
public void seekToEnd(Collection<TopicPartition> partitions)
// consumer从指定时间开始消费
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(
            Map<TopicPartition, Long> timestampsToSearch)
public Map<TopicPartition, OffsetAndTimestamp> offsetsForTimes(
            Map<TopicPartition, Long> timestampsToSearch, 
            Duration timeout)
```

位移越界：通过seek方法指定消费位置时 ，如果超出原本分区的offset，也会根据auto.offset.reset 参数的默认值来将拉取位置重置

### 再均衡

再均衡是指分区的所属权从一个消费者转移到另一消费者的行为，它为消费组具备高可用性和伸缩性提供保障，使我们可以既方便又安全地删除消费组内的消费者或往消费组内添加消费者。不过在再均衡发生期间，消费组内的消费者是无法读取消息的。也就是说，在再均衡发生期间的这一小段时间内，消费组会变得不可用。

另外，当一个分区被重新分配给另一个消费者时，消费者当前的状态也会丢失。比如消费者消费完某个分区中的一部分消息时还没有来得及提交消费位移就发生了再均衡操作，之后这个分区又被分配给了消费组内的另一个消费者，原来被消费完的那部分消息又被重新消费一遍，也就是发生了重复消费。一般情况下，应尽量避免不必要的再均衡的发生。

kafka提供了再均衡监听器用来设定发生再均衡动作前后的一些准备或收尾的动作。ConsumerRebalanceListener 是一个接口，在回调接口ConsumerRebalanceListener中定义了两个回调方法，具体的释义如下：

1. void onPartitionsRevoked(Collection partitions) 这个方法会在再均衡开始之前和消费者停止读取消息之后被调用。可以通过这个回调方法来处理消费位移的提交，以此来避免一些不必要的重复消费现象的发生。参数 partitions 表示再均衡前所分配到的分区。
2. void onPartitionsAssigned(Collection partitions) 这个方法会在重新分配分区之后和消费者开始读取消费之前被调用。参数 partitions 表示再均衡后所分配到的分区。

```java
// 再均衡监听器的用法
Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();
consumer.subscribe(Arrays.asList(topic), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync(currentOffsets);
	        currentOffsets.clear();
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        //do nothing.
    }
});

try {
    while (isRunning.get()) {
        ConsumerRecords<String, String> records =
                consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            //process the record.
            currentOffsets.put(
                    new TopicPartition(record.topic(), record.partition()),
                    new OffsetAndMetadata(record.offset() + 1));
        }
        consumer.commitAsync(currentOffsets, null);
    }
} finally {
    consumer.close();
}
```

代码中将消费位移暂存到一个局部变量 currentOffsets 中，这样在正常消费的时候可以通过 commitAsync() 方法来异步提交消费位移，在发生再均衡动作之前可以通过再均衡监听器的 onPartitionsRevoked() 回调执行 commitSync() 方法同步提交消费位移，以尽量避免一些不必要的重复消费。如果担心commitAsync会覆盖重平衡的提交，可以采用版本号的方式防止异步覆盖之前的commit。

再均衡监听器还可以配合外部存储使用。在下面代码中，我们将消费位移保存在数据库中，这里可以通过再均衡监听器查找分配到的分区的消费位移，并且配合 seek() 方法来进一步优化代码逻辑。

```java
consumer.subscribe(Arrays.asList(topic), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        //store offset in DB （storeOffsetToDB）
    }
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        for(TopicPartition tp: partitions){
            consumer.seek(tp, getOffsetFromDB(tp));//从DB中读取消费位移
        }
    }
});
```

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

下面使用消费者拦截器来实现一个简单的消息 TTL（Time to Live，即过期时间）的功能。在代码中，自定义的消费者拦截器 ConsumerInterceptorTTL 使用消息的 timestamp 字段来判定是否过期，如果消息的时间戳与当前的时间戳相差超过10秒则判定为过期，那么这条消息也就被过滤而不投递给具体的消费者。

```java
// 自定义的消费者拦截器
public class ConsumerInterceptorTTL implements 
        ConsumerInterceptor<String, String> {
    private static final long EXPIRE_INTERVAL = 10 * 1000;

    @Override
    public ConsumerRecords<String, String> onConsume(
            ConsumerRecords<String, String> records) {
        long now = System.currentTimeMillis();
        Map<TopicPartition, List<ConsumerRecord<String, String>>> newRecords 
                = new HashMap<>();
        for (TopicPartition tp : records.partitions()) {
            List<ConsumerRecord<String, String>> tpRecords = 
            records.records(tp);
            List<ConsumerRecord<String, String>> newTpRecords = new ArrayList<>();
            for (ConsumerRecord<String, String> record : tpRecords) {
                if (now - record.timestamp() < EXPIRE_INTERVAL) {
                    newTpRecords.add(record);
                }
            }
            if (!newTpRecords.isEmpty()) {
                newRecords.put(tp, newTpRecords);
            }
        }
        return new ConsumerRecords<>(newRecords);
    }

    @Override
    public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets) {
        offsets.forEach((tp, offset) -> 
                System.out.println(tp + ":" + offset.offset()));
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

### 消费者多线程实现

KafkaProducer 是线程安全的，然而 KafkaConsumer 却是非线程安全的。KafkaConsumer 中定义了一个 acquire() 方法，用来检测当前是否只有一个线程在操作，若有其他线程正在操作则会抛出 ConcurrentModifcationException 异常。

acquire() 方法和我们通常所说的锁（synchronized、Lock 等）不同，它不会造成阻塞等待，我们可以将其看作一个轻量级锁，它仅通过线程操作计数标记的方式来检测线程是否发生了并发操作，以此保证只有一个线程在操作。

KafkaConsumer 非线程安全并不意味着我们在消费消息的时候只能以单线程的方式执行。如果生产者发送消息的速度大于消费者处理消息的速度，那么就会有越来越多的消息得不到及时的消费，造成了一定的延迟。除此之外，由于 Kafka 中消息保留机制的作用，有些消息有可能在被消费之前就被清理了，从而造成消息的丢失。

多线程的实现方式有多种，第一种也是最常见的方式：线程封闭，即为每个线程实例化一个 KafkaConsumer 对象。这种实现方式的并发度受限于分区的实际个数，当消费线程的个数大于分区数时，就有部分消费线程一直处于空闲的状态。

第二种方式是多个消费线程同时消费同一个分区，这个通过 assign()、seek() 等方法实现，这样可以打破原有的消费线程的个数不能超过分区数的限制，进一步提高了消费的能力。不过这种实现方式对于位移提交和顺序控制的处理就会变得非常复杂，实际应用中使用得极少。

考虑第三种实现方式，将处理消息模块改成多线程的实现方式，模型如下图：

<img width="604" alt="image" src="https://user-images.githubusercontent.com/41377703/160269960-506c031d-14e2-4e3a-8d36-49599ef2cb9c.png">

```java
public class ThirdMultiConsumerThreadDemo {
    public static final String brokerList = "localhost:9092";
    public static final String topic = "topic-demo";
    public static final String groupId = "group.demo";

    //省略initConfig()方法，具体请参考代码清单14-1
    public static void main(String[] args) {
        Properties props = initConfig();
        KafkaConsumerThread consumerThread = 
                new KafkaConsumerThread(props, topic,
                Runtime.getRuntime().availableProcessors());
        consumerThread.start();
    }

    public static class KafkaConsumerThread extends Thread {
        private KafkaConsumer<String, String> kafkaConsumer;
        private ExecutorService executorService;
        private int threadNumber;

        public KafkaConsumerThread(Properties props, 
                String topic, int threadNumber) {
            kafkaConsumer = new KafkaConsumer<>(props);
            kafkaConsumer.subscribe(Collections.singletonList(topic));
            this.threadNumber = threadNumber;
            executorService = new ThreadPoolExecutor(threadNumber, threadNumber,
                    0L, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<>(1000),
                    new ThreadPoolExecutor.CallerRunsPolicy());
        }

        @Override
        public void run() {
            try {
                while (true) {
                    ConsumerRecords<String, String> records =
                            kafkaConsumer.poll(Duration.ofMillis(100));
                    if (!records.isEmpty()) {
                        executorService.submit(new RecordsHandler(records));
                    }  ① 
		    synchronized (offsets) {
    	                if (!offsets.isEmpty()) {
                            kafkaConsumer.commitSync(offsets);
                            offsets.clear();
                        }
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                kafkaConsumer.close();
            }
        }

    }

    public static class RecordsHandler extends Thread{
        public final ConsumerRecords<String, String> records;

        public RecordsHandler(ConsumerRecords<String, String> records) {
            this.records = records;
        }

        @Override
        public void run(){
            for (TopicPartition tp : records.partitions()) {
                List<ConsumerRecord<String, String>> tpRecords = records.records(tp);
                //处理tpRecords.
                long lastConsumedOffset = tpRecords.get(tpRecords.size() - 1).offset();
                synchronized (offsets) {
                    if (!offsets.containsKey(tp)) {
                        offsets.put(tp, new OffsetAndMetadata(lastConsumedOffset + 1));
                    }else {
                        long position = offsets.get(tp).offset();
                        if (position < lastConsumedOffset + 1) {
                            offsets.put(tp, new OffsetAndMetadata(lastConsumedOffset + 1));
                        }
                    }
                }
            }
        }
    }
}
```

可以细想一下这样实现是否万无一失？其实这种位移提交的方式会有数据丢失的风险。对于同一个分区中的消息，假设一个处理线程 RecordHandler1 正在处理 offset 为0～99的消息，而另一个处理线程 RecordHandler2 已经处理完了 offset 为100～199的消息并进行了位移提交，此时如果 RecordHandler1 发生异常，则之后的消费只能从200开始而无法再次消费0～99的消息，从而造成了消息丢失的现象。这里虽然针对位移覆盖做了一定的处理，但还没有解决异常情况下的位移覆盖问题。

对此就要引入更加复杂的处理机制，这里再提供一种解决思路，参考下图，总体结构上是基于滑动窗口实现的。对于第三种实现方式而言，它所呈现的结构是通过消费者拉取分批次的消息，然后提交给多线程进行处理，而这里的滑动窗口式的实现方式是将拉取到的消息暂存起来，多个消费线程可以拉取暂存的消息，这个用于暂存消息的缓存大小即为滑动窗口的大小，总体上而言没有太多的变化，不同的是对于消费位移的把控。

<img width="589" alt="image" src="https://user-images.githubusercontent.com/41377703/160270119-cf62d9f7-74e6-4f5e-9286-de62f439dd20.png">

如上图所示，每一个方格代表一个批次的消息，一个滑动窗口包含若干方格，startOffset 标注的是当前滑动窗口的起始位置，endOffset 标注的是末尾位置。每当 startOffset 指向的方格中的消息被消费完成，就可以提交这部分的位移，与此同时，窗口向前滑动一格，删除原来 startOffset 所指方格中对应的消息，并且拉取新的消息进入窗口。滑动窗口的大小固定，所对应的用来暂存消息的缓存大小也就固定了，这部分内存开销可控。

方格大小和滑动窗口的大小同时决定了消费线程的并发数：一个方格对应一个消费线程，对于窗口大小固定的情况，方格越小并行度越高；对于方格大小固定的情况，窗口越大并行度越高。不过，若窗口设置得过大，不仅会增大内存的开销，而且在发生异常（比如 Crash）的情况下也会引起大量的重复消费，同时还考虑线程切换的开销，建议根据实际情况设置一个合理的值，不管是对于方格还是窗口而言，过大或过小都不合适。

如果一个方格内的消息无法被标记为消费完成，那么就会造成 startOffset 的悬停。为了使窗口能够继续向前滑动，那么就需要设定一个阈值，当 startOffset 悬停一定的时间后就对这部分消息进行本地重试消费，如果重试失败就转入重试队列，如果还不奏效就转入死信队列。真实应用中无法消费的情况极少，一般是由业务代码的处理逻辑引起的，比如消息中的内容格式与业务处理的内容格式不符，无法对这条消息进行决断，这种情况可以通过优化代码逻辑或采取丢弃策略来避免。如果需要消息高度可靠，也可以将无法进行业务逻辑的消息（这类消息可以称为死信）存入磁盘、数据库或 Kafka，然后继续消费下一条消息以保证整体消费进度合理推进，之后可以通过一个额外的处理任务来分析死信进而找出异常的原因。

### 消费者端配置

- **enable.auto.commit**   默认为true，开启自动提交，当然这个默认的自动提交不是每消费一条消息就提交一次，而是定期提交，这个定期的周期时间由客户端参数 auto.commit.interval.ms 配置，默认值为5秒，此参数生效的前提是 enable.auto.commit 参数为 true
- **auto.offset.reset** 参数的默认值时latest，如果将 auto.offset.reset 参数配置为“earliest”，那么消费者会从起始处，也就是0开始消费。

- **fetch.min.bytes** 每次poll拉取最小数据量，默认1B，如果返回给 Consumer 的数据量小于这个参数所配置的值，那么它就需要进行等待，直到数据量满足这个参数的配置大小。可以通过增大该参数提升系统吞吐量，但是会造成延迟增加
- **fetch.max.bytes** 每次poll拉取最大数据量，默认50M，并且为了防止消息无法消费，如果发现第一条大于50M，正常消费
- **fetch.max.wait.ms** 配合fetch.min.bytes使用，如果消息小于fetch.min.bytes，拉取数据最长等待时间，默认500ms。
- **max.partition.fetch.bytes** 这个参数用来配置从每个分区里返回给 Consumer 的最大数据量，默认值为1048576（B），即1MB。
- **max.poll.records** 每次拉取数据最多条数，默认500条
- **connections.max.idle.ms** 用来设置多久关闭空闲连接，默认9分钟
- request.timeout.ms 这个参数用来配置 Consumer 等待请求响应的最长时间，默认值为30000（ms）。
- metadata.max.age.ms 这个参数用来配置元数据的过期时间，默认值为300000（ms），即5分钟。如果元数据在此参数所限定的时间范围内没有进行更新，则会被强制更新，即使没有任何分区变化或有新的 broker 加入。
- reconnect.backoff.ms 这个参数用来配置尝试重新连接指定主机之前的等待时间（也称为退避时间），避免频繁地连接主机，默认值为50（ms）。这种机制适用于消费者向 broker 发送的所有请求。
- retry.backoff.ms 这个参数用来配置尝试重新发送失败的请求到指定的主题分区之前的等待（退避）时间，避免在某些故障情况下频繁地重复发送，默认值为100（ms）。
- isolation.level 这个参数用来配置消费者的事务隔离级别。字符串类型，有效值为“read_uncommitted”和“read_committed”，表示消费者所消费到的位置，如果设置为“read_committed”，那么消费者就会忽略事务未提交的消息，即只能消费到LSO（LastStableOffset）的位置，默认情况下为“read_uncommitted”，即可以消费到 HW（High Watermark）处的位置。

更多的：https://juejin.im/book/6844733793220165639/section/6844733793635401736

### 优先副本的选举

分区使用多副本机制来提升可靠性，但只有 leader 副本对外提供读写服务，而 follower 副本只负责在内部进行消息的同步。如果一个分区的 leader 副本不可用，那么就意味着整个分区变得不可用，此时就需要 Kafka 从剩余的 follower 副本中挑选一个新的 leader 副本来继续对外提供服务。虽然不够严谨，但从某种程度上说，broker 节点中 leader 副本个数的多少决定了这个节点负载的高低。

在创建主题的时候，该主题的分区及副本会尽可能均匀地分布到 Kafka 集群的各个 broker 节点上，对应的 leader 副本的分配也比较均匀。比如我们使用 kafka-topics.sh 脚本创建一个分区数为3、副本因子为3的主题 topic-partitions，创建之后的分布信息如下：

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-partitions
Topic:topic-partitions	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: topic-partitions	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: topic-partitions	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: topic-partitions	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```

可以看到 leader 副本均匀分布在 brokerId 为0、1、2的 broker 节点之中。针对同一个分区而言，同一个 broker 节点中不可能出现它的多个副本，即 Kafka 集群的一个 broker 中最多只能有它的一个副本，我们可以将 leader 副本所在的 broker 节点叫作分区的 leader 节点，而 follower 副本所在的 broker 节点叫作分区的 follower 节点。

随着时间的更替，Kafka 集群的 broker 节点不可避免地会遇到宕机或崩溃的问题，当分区的 leader 节点发生故障时，其中一个 follower 节点就会成为新的 leader 节点，这样就会导致集群的负载不均衡，从而影响整体的健壮性和稳定性。当原来的 leader 节点恢复之后重新加入集群时，它只能成为一个新的 follower 节点而不再对外提供服务。比如我们将 brokerId 为2的节点重启，那么主题 topic-partitions 新的分布信息如下：

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-partitions
Topic:topic-partitions	PartitionCount:3	ReplicationFactor:3	Configs: 
    Topic: topic-partitions	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2
    Topic: topic-partitions	Partition: 1	Leader: 0	Replicas: 2,0,1	Isr: 0,1,2
    Topic: topic-partitions	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```

可以看到原本分区1的 leader 节点为2，现在变成了0，如此一来原本均衡的负载变成了失衡：节点0的负载最高，而节点2的负载最低。

为了能够有效地治理负载失衡的情况，Kafka 引入了优先副本（preferred replica）的概念。所谓的优先副本是指在AR集合列表中的第一个副本。比如上面主题 topic-partitions 中分区0的AR集合列表（Replicas）为[1,2,0]，那么分区0的优先副本即为1。理想情况下，优先副本就是该分区的leader 副本，所以也可以称之为 preferred leader。Kafka 要确保所有主题的优先副本在 Kafka 集群中均匀分布，这样就保证了所有分区的 leader 均衡分布。如果 leader 分布过于集中，就会造成集群负载不均衡。

所谓的优先副本的选举是指通过一定的方式促使优先副本选举为 leader 副本，以此来促进集群的负载均衡，这一行为也可以称为“分区平衡”。

需要注意的是，分区平衡并不意味着 Kafka 集群的负载均衡，因为还要考虑集群中的分区分配是否均衡。更进一步，每个分区的 leader 副本的负载也是各不相同的，有些 leader 副本的负载很高，比如需要承载 TPS 为30000的负荷，而有些 leader 副本只需承载个位数的负荷。也就是说，就算集群中的分区分配均衡、leader 分配均衡，也并不能确保整个集群的负载就是均衡的，还需要其他一些硬性的指标来做进一步的衡量，这个会在后面的章节中涉及，本节只探讨优先副本的选举。

在 Kafka 中可以提供分区自动平衡的功能，与此对应的 broker 端参数是 auto.leader. rebalance.enable，此参数的默认值为 true，即默认情况下此功能是开启的。如果开启分区自动平衡的功能，则 Kafka 的控制器会启动一个定时任务，这个定时任务会轮询所有的 broker 节点，计算每个 broker 节点的分区不平衡率（broker 中的不平衡率=非优先副本的 leader 个数/分区总数）是否超过 leader.imbalance.per.broker.percentage 参数配置的比值，默认值为10%，如果超过设定的比值则会自动执行优先副本的选举动作以求分区平衡。执行周期由参数 leader.imbalance.check.interval.seconds 控制，默认值为300秒，即5分钟。

不过在生产环境中不建议将 auto.leader.rebalance.enable 设置为默认的 true，因为这可能引起负面的性能问题，也有可能引起客户端一定时间的阻塞。因为执行的时间无法自主掌控，如果在关键时期（比如电商大促波峰期）执行关键任务的关卡上执行优先副本的自动选举操作，势必会有业务阻塞、频繁超时之类的风险。前面也分析过，分区及副本的均衡也不能完全确保集群整体的均衡，并且集群中一定程度上的不均衡也是可以忍受的，为防止出现关键时期“掉链子”的行为，笔者建议还是将掌控权把控在自己的手中，可以针对此类相关的埋点指标设置相应的告警，在合适的时机执行合适的操作，而这个“合适的操作”就是指手动执行分区平衡。

Kafka 中 kafka-perferred-replica-election.sh 脚本提供了对分区 leader 副本进行重新平衡的功能。优先副本的选举过程是一个安全的过程，Kafka 客户端可以自动感知分区 leader 副本的变更。下面的示例演示了 kafka-perferred-replica-election.sh 脚本的具体用法：

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181/kafka
 
Created preferred replica election path with topic-demo-3,__consumer_offsets-22, topic-config-1,__consumer_offsets-30,__bigdata_monitor-12,__consumer_offsets-8,__consumer_offsets-21,topic-create-0,__consumer_offsets-4,topic-demo-1,topic-partitions-1,__consumer_offsets-27,__consumer_offsets-7,__consumer_offsets-9,__consumer_offsets-46,(…省略若干)

[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-partitions
Topic:topic-partitions	PartitionCount:3	ReplicationFactor:3	Configs: 
    Topic: topic-partitions	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2
    Topic: topic-partitions	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 0,1,2
    Topic: topic-partitions	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```

可以看到在脚本执行之后，主题 topic-partitions 中的所有 leader 副本的分布已经和刚创建时的一样了，所有的优先副本都成为 leader 副本。

上面示例中的这种使用方式会将集群上所有的分区都执行一遍优先副本的选举操作，分区数越多打印出来的信息也就越多。leader 副本的转移也是一项高成本的工作，如果要执行的分区数很多，那么必然会对客户端造成一定的影响。如果集群中包含大量的分区，那么上面的这种使用方式有可能会失效。在优先副本的选举过程中，具体的元数据信息会被存入 ZooKeeper 的/admin/preferred_replica_election 节点，如果这些数据超过了 ZooKeeper 节点所允许的大小，那么选举就会失败。默认情况下 ZooKeeper 所允许的节点数据大小为1MB。

kafka-perferred-replica-election.sh 脚本中还提供了 path-to-json-file 参数来小批量地对部分分区执行优先副本的选举操作。通过 path-to-json-file 参数来指定一个 JSON 文件，这个 JSON 文件里保存需要执行优先副本选举的分区清单。

举个例子，我们再将集群中 brokerId 为2的节点重启，不过我们现在只想对主题 topic- partitions 执行优先副本的选举操作，那么先创建一个JSON文件，文件名假定为 election.json，文件的内容如下：

```
{
        "partitions":[
                {
                        "partition":0,
                        "topic":"topic-partitions"
                },
                {
                        "partition":1,
                        "topic":"topic-partitions"
                },
                {
                        "partition":2,
                        "topic":"topic-partitions"
                }
        ]
}
```

然后通过 kafka-perferred-replica-election.sh 脚本配合 path-to-json-file 参数来对主题 topic-partitions 执行优先副本的选举操作，具体示例如下：

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181/kafka --path-to-json-file election.json
Created preferred replica election path with topic-partitions-0,topic-partitions-1, topic-partitions-2
Successfully started preferred replica election for partitions Set(topic- partitions-0, topic-partitions-1, topic-partitions-2)

[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --describe --topic topic-partitions
Topic:topic-partitions	PartitionCount:3	ReplicationFactor:3	Configs:
    Topic: topic-partitions	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,0,2 
    Topic: topic-partitions	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 0,1,2
    Topic: topic-partitions	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```

读者可以自行查看一下集群中的其他主题是否像之前没有使用 path-to-json-file 参数的一样也被执行了选举操作。

在实际生产环境中，一般使用 path-to-json-file 参数来分批、手动地执行优先副本的选举操作。尤其是在应对大规模的 Kafka 集群时，理应杜绝采用非 path-to-json-file 参数的选举操作方式。同时，优先副本的选举操作也要注意避开业务高峰期，以免带来性能方面的负面影响。

### 分区重分配

当集群中的一个节点突然宕机下线时，如果节点上的分区是单副本的，那么这些分区就变得不可用了，在节点恢复前，相应的数据也就处于丢失状态；如果节点上的分区是多副本的，那么位于这个节点上的 leader 副本的角色会转交到集群的其他 follower 副本中。总而言之，这个节点上的分区副本都已经处于功能失效的状态，Kafka 并不会将这些失效的分区副本自动地迁移到集群中剩余的可用 broker 节点上，如果放任不管，则不仅会影响整个集群的均衡负载，还会影响整体服务的可用性和可靠性。

当要对集群中的一个节点进行有计划的下线操作时，为了保证分区及副本的合理分配，我们也希望通过某种方式能够将该节点上的分区副本迁移到其他的可用节点上。

当集群中新增 broker 节点时，只有新创建的主题分区才有可能被分配到这个节点上，而之前的主题分区并不会自动分配到新加入的节点中，因为在它们被创建时还没有这个新节点，这样新节点的负载和原先节点的负载之间严重不均衡。

为了解决上述问题，需要让分区副本再次进行合理的分配，也就是所谓的分区重分配。Kafka 提供了 kafka-reassign-partitions.sh 脚本来执行分区重分配的工作，它可以在集群扩容、broker 节点失效的场景下对分区进行迁移。 kafka-reassign-partitions.sh 脚本的使用分为3个步骤：首先创建需要一个包含主题清单的 JSON 文件，其次根据主题清单和 broker 节点清单生成一份重分配方案，最后根据这份方案执行具体的重分配动作。

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

分区重分配对集群的性能有很大的影响，需要占用额外的资源，比如网络和磁盘。在实际操作中，我们将降低重分配的粒度，分成多个小批次来执行，以此来将负面的影响降到最低，这一点和优先副本的选举有异曲同工之妙。 还需要注意的是，如果要将某个 broker 下线，那么在执行分区重分配动作之前最好先关闭或重启 broker。这样这个 broker 就不再是任何分区的 leader 节点了，它的分区就可以被分配给集群中的其他 broker。这样可以减少 broker 间的流量复制，以此提升重分配的性能，以及减少对集群的影响。

### 如何选择合适的分区数

kafka提供了性能压测工具，用于生产者性能测试的 kafka-producer- perf-test.sh 和用于消费者性能测试的 kafka-consumer-perf-test.sh。具体的使用可以参照教程。

#### 那么按照经验思考一个问题，分区数越多吞吐量就越高吗？

<img width="694" alt="image" src="https://user-images.githubusercontent.com/41377703/160290116-90412006-c5e8-43ec-bbe5-008712f4c567.png">

其实根据实际的压测结果，无论是生产者端还是消费者端随着分区数的增加，相应的吞吐量也会有所增长。一旦分区数超过了某个阈值之后，整体的吞吐量也是不升反降的，说明了分区数越多并不会使吞吐量一直增长。

#### 那么分区数存在上限吗？

试着在一台普通的 Linux 机器上创建包含10000个分区的主题，比如在下面示例中创建一个主题 topic-bomb：

```
[root@node1 kafka_2.11-2.0.0]# bin/kafka-topics.sh --zookeeper localhost:2181/ kafka --create --topic topic-bomb --replication-factor 1 --partitions 10000
Created topic "topic-bomb".
```
执行完成后可以检查 Kafka 的进程是否还存在（比如通过 jps 命令或 ps -aux|grep kafka 命令）。一般情况下，会发现原本运行完好的 Kafka 服务已经崩溃。此时或许会想到，创建这么多分区，是不是因为内存不够而引起的进程崩溃？为了分析真实的原因，我们可以打开 Kafka 的服务日志文件（$KAFKA_HOME/logs/ server.log）来一探究竟，会发现服务日志中出现大量的异常：

```
[2018-09-13 00:36:40,019] ERROR Error while creating log for topic-bomb-xxx in dir /tmp/kafka-logs (kafka.server.LogDirFailureChannel)
java.io.IOException: Too many open files 
     at java.io.UnixFileSystem.createFileExclusively(Native Method)
     at java.io.File.createNewFile(File.java:1012)
     at kafka.log.AbstractIndex.<init>(AbstractIndex.scala:54)
     at kafka.log.OffsetIndex.<init>(OffsetIndex.scala:53)
     at kafka.log.LogSegment$.open(LogSegment.scala:634)
     at kafka.log.Log.loadSegments(Log.scala:503)
     at kafka.log.Log.<init>(Log.scala:237)
```

异常中最关键的信息是“Too many open flies”，这是一种常见的 Linux 系统错误，通常意味着文件描述符不足，它一般发生在创建线程、创建 Socket、打开文件这些场景下。在 Linux 系统的默认设置下，这个文件描述符的个数不是很多，通过 ulimit 命令可以查看：

```
[root@node1 kafka_2.11-2.0.0]# ulimit -n
1024
[root@node1 kafka_2.11-2.0.0]# ulimit -Sn
1024
[root@node1 kafka_2.11-2.0.0]# ulimit -Hn
4096
```

ulimit 是在系统允许的情况下，提供对特定 shell 可利用的资源的控制。-H 和 -S 选项指定资源的硬限制和软限制。硬限制设定之后不能再添加，而软限制则可以增加到硬限制规定的值。如果 -H 和 -S 选项都没有指定，则软限制和硬限制同时设定。限制值可以是指定资源的数值或 hard、soft、unlimited 这些特殊值，其中 hard 代表当前硬限制，soft 代表当前软件限制，unlimited 代表不限制。如果不指定限制值，则打印指定资源的软限制值，除非指定了 -H 选项。硬限制可以在任何时候、任何进程中设置，但硬限制只能由超级用户设置。软限制是内核实际执行的限制，任何进程都可以将软限制设置为任意小于等于硬限制的值。

如何避免这种异常情况？对于一个高并发、高性能的应用来说，1024或4096的文件描述符限制未免太少，可以适当调大这个参数。比如使用 ulimit -n 65535命令将上限提高到65535，这样足以应对大多数的应用情况，再高也完全没有必要了。

```
[root@node1 kafka_2.11-2.0.0]# ulimit -n 65535
#可以再次查看相应的软硬限制数
[root@node1 kafka_2.11-2.0.0]# ulimit -Hn
65535
[root@node1 kafka_2.11-2.0.0]# ulimit -Sn
65535
```

也可以在/etc/security/limits.conf 文件中设置，参考如下：

```
#nofile - max number of open file descriptors
root soft nofile 65535
root hard nofile 65535
```

limits.conf 文件修改之后需要重启才能生效。limits.conf 文件与 ulimit 命令的区别在于前者是针对所有用户的，而且在任何 shell 中都是生效的，即与 shell 无关，而后者只是针对特定用户的当前 shell 的设定。在修改最大文件打开数时，最好使用 limits.conf 文件来修改，通过这个文件，可以定义用户、资源类型、软硬限制等。也可以通过在/etc/profile 文件中添加 ulimit 的设置语句来使全局生效。

#### 结论 - 考量因素

从吞吐量方面考虑，增加合适的分区数可以在一定程度上提升整体吞吐量，但超过对应的阈值之后吞吐量不升反降。如果应用对吞吐量有一定程度上的要求，则建议在投入生产环境之前对同款硬件资源做一个完备的吞吐量相关的测试，以找到合适的分区数阈值区间。

在创建主题之后，虽然我们还能够增加分区的个数，但基于 key 计算的主题需要严谨对待。当生产者向 Kafka 中写入基于 key 的消息时，Kafka 通过消息的 key 来计算出消息将要写入哪个具体的分区，这样具有相同 key 的数据可以写入同一个分区。Kafka 的这一功能对于一部分应用是极为重要的，比如日志压缩（Log Compaction）；再比如对于同一个 key 的所有消息，消费者需要按消息的顺序进行有序的消费，如果分区的数量发生变化，那么有序性就得不到保证。

在创建主题时，最好能确定好分区数，这样也可以省去后期增加分区所带来的多余操作。尤其对于与 key 高关联的应用，在创建主题时可以适当地多创建一些分区，以满足未来的需求。通常情况下，可以根据未来2年内的目标吞吐量来设定分区数。当然如果应用与 key 弱关联，并且具备便捷的增加分区数的操作接口，那么也可以不用考虑那么长远的目标。

有些应用场景会要求主题中的消息都能保证顺序性，这种情况下在创建主题时可以设定分区数为1，通过分区有序性的这一特性来达到主题有序性的目的。

当然分区数也不能一味地增加，参考前面的内容，分区数会占用文件描述符，而一个进程所能支配的文件描述符是有限的，这也是通常所说的文件句柄的开销。虽然我们可以通过修改配置来增加可用文件描述符的个数，但凡事总有一个上限，在选择合适的分区数之前，最好再考量一下当前 Kafka 进程中已经使用的文件描述符的个数。

分区数的多少还会影响系统的可用性。在前面章节中，我们了解到 Kafka 通过多副本机制来实现集群的高可用和高可靠，每个分区都会有一至多个副本，每个副本分别存在于不同的 broker 节点上，并且只有 leader 副本对外提供服务。在 Kafka 集群的内部，所有的副本都采用自动化的方式进行管理，并确保所有副本中的数据都能保持一定程度上的同步。当 broker 发生故障时，leader 副本所属宿主的 broker 节点上的所有分区将暂时处于不可用的状态，此时 Kafka 会自动在其他的 follower 副本中选举出新的 leader 用于接收外部客户端的请求，整个过程由 Kafka 控制器负责完成。分区在进行 leader 角色切换的过程中会变得不可用，不过对于单个分区来说这个过程非常短暂，对用户而言可以忽略不计。如果集群中的某个 broker 节点宕机，那么就会有大量的分区需要同时进行 leader 角色切换，这个切换的过程会耗费一笔可观的时间，并且在这个时间窗口内这些分区也会变得不可用。

分区数越多也会让 Kafka 的正常启动和关闭的耗时变得越长，与此同时，主题的分区数越多不仅会增加日志清理的耗时，而且在被删除时也会耗费更多的时间。对旧版的生产者和消费者客户端而言，分区数越多，也会增加它们的开销，不过这一点在新版的生产者和消费者客户端中有效地得到了抑制。

如何选择合适的分区数？从某种意思来说，考验的是决策者的实战经验，更透彻地说，是对 Kafka 本身、业务应用、硬件资源、环境配置等多方面的考量而做出的选择。在设定完分区数，或者更确切地说是创建主题之后，还要对其追踪、监控、调优以求更好地利用它。读者看到本节的内容之前或许没有对分区数有太大的困扰，而看完本节的内容之后反而困惑了起来，其实大可不必太过惊慌，一般情况下，根据预估的吞吐量及是否与 key 相关的规则来设定分区数即可，后期可以通过增加分区数、增加 broker 或分区重分配等手段来进行改进。如果一定要给一个准则，则建议将分区数设定为集群中 broker 的倍数，即假定集群中有3个 broker 节点，可以设定分区数为3、6、9等，至于倍数的选定可以参考预估的吞吐量。不过，如果集群中的 broker 节点数有很多，比如大几十或上百、上千，那么这种准则也不太适用。

### kafka如何实现数据的持久化

总的来说，Kafka 使用消息日志（Log）来保存数据，一个日志就是磁盘上一个只能追加写（Append-only）消息的物理文件。因为只能追加写入，故避免了缓慢的随机 I/O 操作，改为性能较好的顺序 I/O 写操作，这也是实现 Kafka 高吞吐量特性的一个重要手段。不过如果你不停地向一个日志写入消息，最终也会耗尽所有的磁盘空间，因此 Kafka 必然要定期地删除消息以回收磁盘。怎么删除呢？简单来说就是通过日志段（Log Segment）机制。在 Kafka 底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka 会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka 在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的。

### 生产者端和消费者端吞吐量
生产者端通过多个分区，提升吞吐量，消费者端则通过消费者组的多个消费者实现吞吐量的提升

#### kafka线上部署
- 服务器：选择Linux服务器进行部署，首先Linux下Kafka可以利用linux的epoll实现非阻塞I/O，性能更好。另外Linux提供了零拷贝，减少了内核态用户态的切换开销。在社区维护上，Linux版本的问题可以更好的支持

- 磁盘：使用常规的机械硬盘即可，因为kafka采用顺序读写操作，一定程度上规避了机械磁盘最大的劣势，即随机读写操作慢。通过分区的概念，Kafka 也能在软件层面自行实现负载均衡。

- 磁盘容量：从新增消息数、消息留存时间、平均消息大小、备份数、是否启用压缩几方面进行计算

- 带宽：对于千兆网络，做多使用带宽的70%，再多就有丢包的风险了，另外所有的带宽不能都由kafka占用，应该预留出2/3的资源用于其他的资源。所以对于千兆网络，kafka消息应该占用不超多 700 / 3。

#### kafka中zk的作用

它是一个分布式协调框架，负责协调管理并保存 Kafka 集群的所有元数据信息，比如集群都有哪些 Broker 在运行、创建了哪些 Topic，每个 Topic 都有多少分区以及这些分区的 Leader 副本都在哪些机器上等信息。

zookeeper.connect = zk1:2181,zk2:2181,zk3:2181/kafka1       后面的kafka1叫做chroot，可以用来区分多个kafka集群


#### 集群配置
-------Broker端-----------

存储信息

- log.dirs=/home/kafka1,/home/kafka2,/home/kafka3        通过指定多个存储路径，可以实现提升读写性能；另外推荐路径为多个挂载磁盘的路径，当某个磁盘挂了kafka可以实现failover

zk相关

- zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka1

连接相关

- listeners=PLAINTEXT://:9092       告诉外部连接者要通过什么协议访问指定主机名和端口开放的 Kafka 服务。

Topic 管理

- auto.create.topics.enable：是否允许自动创建 Topic。建议测试环境配置为false，线上环境配置为true

- unclean.leader.election.enable：是否允许 Unclean Leader 选举。默认为false，配置为true会造成数据的丢失，配置成false会降低kafka集群的可用性

- auto.leader.rebalance.enable：是否允许定期进行 Leader 选举。默认false，建议设置为false，否则一个运行良好的leader随时可能会被替换

数据保留方面

- log.retention.{hour|minutes|ms}：这是个“三兄弟”，都是控制一条消息数据被保存多长时间。从优先级上来说 ms 设置最高、minutes 次之、hour 最低。

- log.retention.bytes：这是指定 Broker 为消息保存的总磁盘容量大小。

- message.max.bytes：控制 Broker 能够接收的最大消息大小。

- log.segment.bytes=1073741824：topic的分区是以一堆segment文件存储的，这个控制每个segment的大小

-----------topic级别------------

topic级别的配置会覆盖上面的broker的公共配置

数据保存方面

- retention.ms：规定了该 Topic 消息被保存的时长。默认是 7 天，即该 Topic 只保存最近 7 天的消息。一旦设置了这个值，它会覆盖掉 Broker 端的全局参数值。

- retention.bytes：规定了要为该 Topic 预留多大的磁盘空间。和全局参数作用相似，这个值通常在多租户的 Kafka 集群中会有用武之地。当前默认值是 -1，表示可以无限使用磁盘空间。

- max.message.bytes：消息的大小

修改上面的topic级别的参数只能通过脚本的方式进行设置 bin/kafka-configs.sh--zookeeperlocalhost:2181--entity-typetopics--entity-nametransaction--alter--add-configmax.message.bytes=10485760

-------------jvm-------------------

通过下面的三步设置jvm的参数

- export KAFKA_HEAP_OPTS=--Xms6g  --Xmx6g

- export  KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true

- bin/kafka-server-start.sh config/server.properties

------------操作系统参数------------

- 文件描述符限制       ulimit -n 1000000， 避免出现too many connections

- 文件系统类型      选择更优的文件系统，例如XFS

- Swappiness       swap 的调优

- 提交时间        提交时间或者说是 Flush 落盘时间，向 Kafka 发送数据并不是真要等数据被写入磁盘才会认为成功，而是只要数据被写入到操作系统的页缓存（Page Cache）上就可以了，随后操作系统根据 LRU 算法会定期将页缓存上的“脏”数据落盘到物理磁盘上。这个定期就是由提交时间来确定的，默认是 5 秒。一般情况下我们会认为这个时间太频繁了，可以适当地增加提交间隔来降低物理磁盘的写操作。鉴于 Kafka 在软件层面已经提供了多副本的冗余机制，因此这里稍微拉大提交间隔去换取性能还是一个合理的做法。

#### 数据压缩

 props.put("compression.type", "gzip");
 
 如果CPU是系统瓶颈，就不应该再耗费CPU进行压缩，如果带宽是瓶颈，则应该开启压缩。


#### 如何确保消息不丢失

不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。

设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。

设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。

设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。

设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。

设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。

确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。

确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。

#### kafka生产者客户端如何管理TCP连接

KafkaProducer 实例创建时启动 Sender 线程，从而创建与 bootstrap.servers 中所有 Broker 的 TCP 连接。

KafkaProducer 实例首次更新元数据信息之后，还会再次创建与集群中所有 Broker 的 TCP 连接。

如果 Producer 端发送消息到某台 Broker 时发现没有与该 Broker 的 TCP 连接，那么也会立即创建连接。

如果设置 Producer 端 connections.max.idle.ms 参数大于 0，则步骤 1 中创建的 TCP 连接会被自动关闭；如果设置该参数 =-1，那么步骤 1 中创建的 TCP 连接将无法被关闭，从而成为“僵尸”连接。

#### 幂等和事务

消息交付可靠性保障，是指 Kafka 对 Producer 和 Consumer 要处理的消息提供什么样的承诺。常见的承诺有以下三种：

- 最多一次（at most once）：消息可能会丢失，但绝不会被重复发送。

- 至少一次（at least once）：消息不会丢失，但有可能被重复发送。

- 精确一次（exactly once）：消息不会丢失，也不会被重复发送。

kafka默认提供的是第二种，此时的配置是接收到broker的应答才认为成功发送，如果发送成功了，由于网络问题没有接收到回复消息，则会触发重试机制，这也就保证了第二个承诺。如果不重试，可以接受少量的消息丢失，就满足了第一个承诺。

如果满足第三个承诺，需要通过幂等性和事务。

幂等性：指定 Producer 幂等性的方法很简单，仅需要设置一个参数即可，即 props.put(“enable.idempotence”, ture)，或 props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG， true)。Kafka 自动帮你做消息的重复去重。底层具体的原理很简单，就是经典的用空间去换时间的优化思路，即在 Broker 端多保存一些字段。当 Producer 发送了具有相同字段值的消息后，Broker 能够自动知晓这些消息已经重复了，于是可以在后台默默地把它们“丢弃”掉。幂等性存在问题，首先，它只能保证单分区上的幂等性，即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，它无法实现多个分区的幂等性。其次，它只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了。

事务：事务型 Producer 能够保证将消息原子性地写入到多个分区中。这批消息要么全部写入成功，要么全部失败。另外，事务型 Producer 也不惧进程的重启。Producer 重启回来后，Kafka 依然保证它们发送消息的精确一次处理。设置事务型 Producer 的方法也很简单，满足两个要求即可：和幂等性 Producer 一样，开启 enable.idempotence = true。设置 Producer 端参数 transctional. id。最好为其设置一个有意义的名字。

```java
// 事务代码
producer.initTransactions();
try {
            producer.beginTransaction();
            producer.send(record1);
            producer.send(record2);
            producer.commitTransaction();
} catch (KafkaException e) {
            producer.abortTransaction();
}
```

Consumer 端，读取事务型 Producer 发送的消息也是需要一些变更的。修改起来也很简单，设置 isolation.level 参数的值即可。当前这个参数有两个取值：

read_uncommitted：这是默认值，表明 Consumer 能够读取到 Kafka 写入的任何消息，不论事务型 Producer 提交事务还是终止事务，其写入的消息都可以读取。很显然，如果你用了事务型 Producer，那么对应的 Consumer 就不要使用这个值。
read_committed：表明 Consumer 只会读取事务型 Producer 成功提交事务写入的消息。当然了，它也能看到非事务型 Producer 写入的所有消息。

总结一下：幂等性 Producer 和事务型 Producer 都是 Kafka 社区力图为 Kafka 实现精确一次处理语义所提供的工具，只是它们的作用范围是不同的。幂等性 Producer 只能保证单分区、单会话上的消息幂等性；而事务能够保证跨分区、跨会话间的幂等性。从交付语义上来看，自然是事务型 Producer 能做的更多。不过，切记天下没有免费的午餐。比起幂等性 Producer，事务型 Producer 的性能要更差，在实际使用过程中，我们需要仔细评估引入事务的开销，切不可无脑地启用事务。

#### 消费者组

Consumer Group 下可以有一个或多个 Consumer 实例。这里的实例可以是一个单独的进程，也可以是同一进程下的线程。在实际场景中，使用进程更为常见一些。
Group ID 是一个字符串，在一个 Kafka 集群中，它标识唯一的一个 Consumer Group。
Consumer Group 下所有实例订阅的主题的单个分区，只能分配给组内的某个 Consumer 实例消费。这个分区当然也可以被其他的 Group 消费。

Kafka 仅仅使用 Consumer Group 这一种机制，却同时实现了传统消息引擎系统的两大模型：如果所有实例都属于同一个 Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。

老版本的 Consumer Group 把位移保存在 ZooKeeper 中。在新版本的 Consumer Group 中，Kafka 社区重新设计了 Consumer Group 的位移管理方式，采用了将位移保存在 Kafka 内部主题的方法。这个内部主题就是让人既爱又恨的 __consumer_offsets。

#### 重平衡

Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅 Topic 的每个分区。Rebalance 的触发条件有 3 个。

- 组成员数发生变更。比如有新的 Consumer 实例加入组或者离开组，抑或是有 Consumer 实例崩溃被“踢出”组。

- 订阅主题数发生变更。Consumer Group 可以使用正则表达式的方式订阅主题，比如 consumer.subscribe(Pattern.compile(“t.*c”)) 就表明该 Group 订阅所有以字母 t 开头、字母 c 结尾的主题。在 Consumer Group 的运行过程中，你新创建了一个满足这样条件的主题，那么该 Group 就会发生 Rebalance。

- 订阅主题的分区数发生变更。Kafka 当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有 Group 开启 Rebalance。

重平衡的缺点是重平衡过程中服务会停止消费，另外当group内的consumer非常多的时候，重平衡耗费的时间很长，甚至几个小时，所以应该尽量避免重平衡

如何避免重平衡呢，由于后面两种发生重平衡都是运维的主动操作，所以我们开发能做的只是尽量减少第一种情况的重平衡。例如新启动一台机器进行消费，或者主动下掉某台机器和消费者被动下线，都会导致组成员数发生变化从而引发重平衡。被动下线包含心跳检测失败和Consumer 消费时间过长导致两次poll操作的时间超过了max.poll.interval.ms

针对消费者被动下线的原因有心跳检测失败，解决的方案是
设置 session.timeout.ms = 6s。
设置 heartbeat.interval.ms = 2s。
要保证 Consumer 实例在被判定为“dead”之前，能够发送至少 3 轮的心跳请求，即 session.timeout.ms >= 3 * heartbeat.interval.ms。

针对消费时间过长，解决的方案是预估业务最长耗时逻辑的时间，然后配置例如max.poll.interval.ms=5，这个5要大于业务耗时。


#### 位移主题

__consumer_offsets, 当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题。该主题的分区数是 50，副本数是 3。

新版本 Consumer 的位移管理机制其实也很简单，就是将 Consumer 的位移数据作为一条条普通的 Kafka 消息，提交到 __consumer_offsets 中。可以这么说，__consumer_offsets 的主要作用是保存 Kafka 消费者的位移信息。位移主题的 Key 中应该保存 3 部分内容：<Group ID，主题名，分区号 >。消息体存储了消费者的offset和位移提交的一些其他元数据，诸如时间戳和用户自定义的数据等。保存这些元数据是为了帮助 Kafka 执行各种各样后续的操作，比如删除过期位移消息等。

Kafka 提供了专门的后台线程定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据。这个后台线程叫 Log Cleaner。

#### 位移提交

Consumer 需要向 Kafka 汇报自己的位移数据，这个汇报过程被称为提交位移（Committing Offsets），Consumer 需要为分配给它的每个分区提交各自的位移数据。位移提交的语义保障是由你来负责的，Kafka 只会“无脑”地接受你提交的位移。从用户的角度来说，位移提交分为自动提交和手动提交；从 Consumer 端的角度来说，位移提交分为同步提交和异步提交。

自动提交的逻辑：auto.commit.interval.ms。它的默认值是 5 秒，表明 Kafka 每 5 秒会为你自动提交一次位移。自动提交的逻辑是在执行poll的时候，提交上一次poll的位移，所以自动提交不会导致消息丢失，但是会导致消息重复消费。在默认情况下，Consumer 每 5 秒自动提交一次位移。现在，我们假设提交位移之后的 3 秒发生了 Rebalance 操作。在 Rebalance 之后，所有 Consumer 从上一次提交的位移处继续消费，但该位移已经是 3 秒前的位移数据了，故在 Rebalance 发生前 3 秒消费的所有数据都要重新再消费一次。虽然你能够通过减少 auto.commit.interval.ms 的值来提高提交频率，但这么做只能缩小重复消费的时间窗口，不可能完全消除它。这是自动提交机制的一个缺陷。

手动提交对比自动提交增加了灵活性，但是如果采用了commitSync，那么提交的过程是同步阻塞的，这肯定会影响系统的TPS。所以可以使用commitAsync，那么代码如下：
```java
try {
        while (true) {
                    ConsumerRecords<String, String> records = 
                                consumer.poll(Duration.ofSeconds(1));
                    process(records); // 处理消息
                    commitAysnc(); // 使用异步提交规避阻塞
        }
} catch (Exception e) {
            handle(e); // 处理异常
} finally {
            try {
                        consumer.commitSync(); // 最后一次提交使用同步阻塞式提交
	} finally {
	     consumer.close();
}
}
```
更加精确的手动提交逻辑如下：
```java
private Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
int count = 0;
……
while (true) {
            ConsumerRecords<String, String> records = 
	consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record: records) {
                        process(record);  // 处理消息
                        offsets.put(new TopicPartition(record.topic(), record.partition()),
                                    new OffsetAndMetadata(record.offset() + 1)；
                        if（count % 100 == 0）
                                    consumer.commitAsync(offsets, null); // 回调处理逻辑是 null
                        count++;
	}
}
```

#### kafka的CommitFailedException 

所谓 CommitFailedException，顾名思义就是 Consumer 客户端在提交位移时出现了错误或异常，而且还是那种不可恢复的严重异常。如何避免出现CommitFailedException 

- 缩短单条消息处理的时间。比如，之前下游系统消费一条消息的时间是 100 毫秒，优化之后成功地下降到 50 毫秒，那么此时 Consumer 端的 TPS 就提升了一倍。

- 增加 Consumer 端允许下游系统消费一批消息的最大时长。这取决于 Consumer 端参数 max.poll.interval.ms 的值。在最新版的 Kafka 中，该参数的默认值是 5 分钟。如果你的消费逻辑不能简化，那么提高该参数值是一个不错的办法。值得一提的是，Kafka 0.10.1.0 之前的版本是没有这个参数的，因此如果你依然在使用 0.10.1.0 之前的客户端 API，那么你需要增加 session.timeout.ms 参数的值。不幸的是，session.timeout.ms 参数还有其他的含义，因此增加该参数的值可能会有其他方面的“不良影响”，这也是社区在 0.10.1.0 版本引入 max.poll.interval.ms 参数，将这部分含义从 session.timeout.ms 中剥离出来的原因之一。

- 减少下游系统一次性消费的消息总数。这取决于 Consumer 端参数 max.poll.records 的值。当前该参数的默认值是 500 条，表明调用一次 KafkaConsumer.poll 方法，最多返回 500 条消息。可以说，该参数规定了单次 poll 方法能够返回的消息总数的上限。如果前两种方法对你都不适用的话，降低此参数值是避免 CommitFailedException 异常最简单的手段。

- 下游系统使用多线程来加速消费。这应该算是“最高级”同时也是最难实现的解决办法了。具体的思路就是，让下游系统手动创建多个消费线程处理 poll 方法返回的一批消息。之前你使用 Kafka Consumer 消费数据更多是单线程的，所以当消费速度无法匹及 Kafka Consumer 消息返回的速度时，它就会抛出 CommitFailedException 异常。如果是多线程，你就可以灵活地控制线程数量，随时调整消费承载能力，再配以目前多核的硬件条件，该方法可谓是防止 CommitFailedException 最高档的解决之道。事实上，很多主流的大数据流处理框架使用的都是这个方法，比如 Apache Flink 在集成 Kafka 时，就是创建了多个 KafkaConsumerThread 线程，自行处理多线程间的数据消费。不过，凡事有利就有弊，这个方法实现起来并不容易，特别是在多个线程间如何处理位移提交这个问题上，更是极容易出错。在专栏后面的内容中，我将着重和你讨论一下多线程消费的实现方案。

除了上面的四种场景之外，还有一种standAlone的Consumer的groupID设置的和其他的消费者组的名称相同的情况下，同样会导致这个问题，使用的时候要特别注意一下。

#### Consumer端的TCP连接管理

和Producer不一样的是，Consumer端的TCP连接是在首次调用poll时候创建的，

1. 首先发起 FindCoordinator 请求，查看哪个Broker是管理他的Broker。Coordinator是协调者，驻留在 Broker 端的内存中，负责消费者组的组成员管理和各个消费者的位移提交管理。Consumer会向当前负载最小的Broker发出FindCoordinator请求。

2. 连接协调者：消费者知晓了真正的协调者后，会创建连向该 Broker 的 Socket 连接。只有成功连入协调者，协调者才能开启正常的组协调操作，比如加入组、等待组分配方案、心跳请求处理、位移获取、位移提交等。

3. 消费数据时：消费者会为每个要消费的分区创建与该分区领导者副本所在 Broker 连接的 TCP。举个例子，假设消费者要消费 5 个分区的数据，这 5 个分区各自的领导者副本分布在 4 台 Broker 上，那么该消费者在消费时会创建与这 4 台 Broker 的 Socket 连接。

创建的以上三种连接，针对第一种，当创建了第三种之后，第一种连接就会慢慢的被干掉，只保留第二种和第三种连接，这两种连接当kill -9或者手动KafkaConsumer.close或者连接失活之后会断开连接。

#### 消费者组消费进度监控


消费者lag监控有三种方式。

- 使用 Kafka 自带的命令行工具 kafka-consumer-groups 脚本。

- 使用 Kafka Java Consumer API 编程。

- 使用 Kafka 自带的 JMX 监控指标。

#### 副本机制

在 Kafka 中，副本分成两类：领导者副本（Leader Replica）和追随者副本（Follower Replica）。每个分区在创建时都要选举一个副本，称为领导者副本，其余的副本自动称为追随者副本。kafka的副本采用拉的形式同步数据，副本是不对外提供服务的，副本只做数据的冗余，它唯一的任务就是从领导者副本异步拉取消息，并写入到自己的提交日志中，从而实现与领导者副本的同步。当领导者副本挂掉了，或者说领导者副本所在的 Broker 宕机时，Kafka 依托于 ZooKeeper 提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个作为新的领导者。老 Leader 副本重启回来后，只能作为追随者副本加入到集群中。

Kafka 引入了 In-sync Replicas，ISR，副本是否在ISR中不取决于落后的消息数，而是取决于落后的消息时间， 这个配置是Broker 端参数 replica.lag.time.max.ms 参数值。默认情况下，当leader挂掉之后，只能从ISR中进行leader选举。可以通过配置开启Unclean 领导者选举（Unclean Leader Election）Broker 端参数 unclean.leader.election.enable 控制是否允许 Unclean 领导者选举，开启Unclean 领导者选举后如果ISR为空，也会选举出leader继续提供服务，但是代价是数据的丢失。这就是CAP中的A和C的选择问题。

#### broker的如何处理请求



![image](https://github.com/wangjunjie0817/note/blob/master/images/kafka1.png)

broker采用了多路复用的方式处理请求和响应，只不过名称和Reactor模型稍有差异，具体模型如下：

![image](https://github.com/wangjunjie0817/note/blob/master/images/kafka2.png)

Acceptor 线程和一个工作线程池，叫网络线程池。Kafka 提供了 Broker 端参数 num.network.threads，用于调整该网络线程池的线程数。其默认值是 3，表示每台 Broker 启动时会创建 3 个网络线程，专门处理客户端发送的请求。

当网络线程接收到请求后，它是怎么处理的呢？你可能会认为，它顺序处理不就好了吗？实际上，Kafka 在这个环节又做了一层异步线程池的处理，我们一起来看一看下面这张图。

![image](https://github.com/wangjunjie0817/note/blob/master/images/kafka3.png)

当网络线程拿到请求后，它不是自己处理，而是将请求放入到一个共享请求队列中。Broker 端还有个 IO 线程池，负责从该队列中取出请求，执行真正的处理。如果是 PRODUCE 生产请求，则将消息写入到底层的磁盘日志中；如果是 FETCH 请求，则从磁盘或页缓存中读取消息。

IO 线程池处中的线程才是执行请求逻辑的线程。Broker 端参数num.io.threads控制了这个线程池中的线程数。目前该参数默认值是 8，表示每台 Broker 启动后自动创建 8 个 IO 线程处理请求。你可以根据实际硬件条件设置此线程池的个数。

请求队列是所有网络线程共享的，而响应队列则是每个网络线程专属的。Purgatory 的组件，这是 Kafka 中著名的“炼狱”组件。它是用来缓存延时请求（Delayed Request）的。所谓延时请求，就是那些一时未满足条件不能立刻处理的请求。

在 Kafka 内部，除了客户端发送的 PRODUCE 请求和 FETCH 请求之外，还有很多执行其他操作的请求类型，比如负责更新 Leader 副本、Follower 副本以及 ISR 集合的 LeaderAndIsr 请求，负责勒令副本下线的 StopReplica 请求等。与 PRODUCE 和 FETCH 请求相比，这些请求有个明显的不同：它们不是数据类的请求，而是控制类的请求。也就是说，它们并不是操作消息数据的，而是用来执行特定的 Kafka 内部动作的。

#### kafka控制器

同一时刻，在kafka集群中，有且只有一个控制器，控制器的选择依赖于zookeeper，第一个写节点成功的broker节点成为控制器。

控制器有哪些功能？

1.主题管理（创建、删除、增加分区）

这里的主题管理，就是指控制器帮助我们完成对 Kafka 主题的创建、删除以及分区增加的操作。换句话说，当我们执行kafka-topics 脚本时，大部分的后台工作都是控制器来完成的。关于 kafka-topics 脚本，我会在专栏后面的内容中，详细介绍它的使用方法。

2.分区重分配

分区重分配主要是指，kafka-reassign-partitions 脚本（关于这个脚本，后面我也会介绍）提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。

3.Preferred 领导者选举

Preferred 领导者选举主要是 Kafka 为了避免部分 Broker 负载过重而提供的一种换 Leader 的方案。在专栏后面说到工具的时候，我们再详谈 Preferred 领导者选举，这里你只需要了解这也是控制器的职责范围就可以了。

4.集群成员管理（新增 Broker、Broker 主动关闭、Broker 宕机）

这是控制器提供的第 4 类功能，包括自动检测新增 Broker、Broker 主动关闭及被动宕机。这种自动检测是依赖于前面提到的 Watch 功能和 ZooKeeper 临时节点组合实现的。

比如，控制器组件会利用Watch 机制检查 ZooKeeper 的 /brokers/ids 节点下的子节点数量变更。目前，当有新 Broker 启动后，它会在 /brokers 下创建专属的 znode 节点。一旦创建完毕，ZooKeeper 会通过 Watch 机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker 作业。

侦测 Broker 存活性则是依赖于刚刚提到的另一个机制：临时节点。每个 Broker 启动后，会在 /brokers/ids 下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与 ZooKeeper 的会话结束，这个 znode 会被自动删除。同理，ZooKeeper 的 Watch 机制将这一变更推送给控制器，这样控制器就能知道有 Broker 关闭或宕机了，从而进行“善后”。

5.数据服务

控制器的最后一大类工作，就是向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 Broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

#### leader epoch

引用 Leader Epoch 机制后，Follower 副本 B 重启回来后，需要向 A 发送一个特殊的请求去获取 Leader 的 LEO 值。在这个例子中，该值为 2。当获知到 Leader LEO=2 后，B 发现该 LEO 值不比它自己的 LEO 值小，而且缓存中也没有保存任何起始位移值 > 2 的 Epoch 条目，因此 B 无需执行任何日志截断操作。这是对高水位机制的一个明显改进，即副本是否执行日志截断不再依赖于高水位进行判断。

#### kafka为什么性能高？

partition 并行处理

顺序写磁盘，充分利用磁盘特性

利用了现代操作系统分页存储 Page Cache 来利用内存提高 I/O 效率

采用了零拷贝技术

Producer 生产的数据持久化到 broker，采用 mmap 文件映射，实现顺序的快速写入

Customer 从 broker 读取数据，采用 sendfile，将磁盘文件读到 OS 内核缓冲区后，转到 NIO buffer进行网络发送，减少 CPU 消耗

https://zhuanlan.zhihu.com/p/183808742?utm_source=wechat_timeline
