### Redis

#### 字符串扩容

少于1M情况下翻倍扩容，多余1M，每次增加1M，字符串最大长度512M

#### 命令汇总

-------------------String-------------------

set key value

get key

exists key

del key

mset key1 value1 key2 value2  ....

mget key1 key2  

setex name 5 wang   # 等价于先写数据再设置超时时间

setnx name wang     # 如果name不存在执行创建

incr age

incrby age -5

-------------------List-------------------

lpush list1 v1 v2 v3.....

rpop list1

llen list1

lindex list1  # 慎用，效率低

lrange list1 0 -1

ltrim list1 2 3

-------------------Hash-------------------

hset key field value 

hgetall

hget

hlen

hmget

hmset

hincrby key feild 1

-------------------Set-------------------

sadd set1 value

smembers  set1

sismember set1 value

spop set1  # 弹出一个

scard set1   # 获取set长度

-------------------Zset-------------------

zadd key score value

zrange key 0 -1   # 按score排序列出

zrerange key 0 -1   # 按score倒序列出

zcard key    # 相当于count

zscore key value

zrank key value

zrangebyscore key score1 score2

zrem key value

-------------------公共-------------------

expire name 5       # 5s后过期

ttl name    # 剩余时间

keys *    # 按照正则获取所有的key

redis-cli -h 127.0.0.1 -p 6379 --bigkeys    # 大key分析与定位

object encoding key

info   # 当前redis状态信息

-------------------Sentinel-------------------

PING - 这个命令简单的返回 PONE。

SENTINEL masters - 展示监控的 master 清单和它们的状态。

SENTINEL master [master name] - 展示指定 master 的状态和信息。

SENTINEL slaves [master name] - 展示 master 的 slave 清单和它们的状态。

SENTINEL sentinels [master name] - 展示 master 的 sentinel 实例的清单和它们的状态。

SENTINEL get-master-addr-by-name [master name] - 返回 master 的 IP 和端口。如果故障转移在处理中或成功终止，返回晋升的 slave 的 IP 和端口。

SENTINEL reset [pattern] - 这个命令将重置所有匹配名字的 masters。参数是 blog 风格的。重置的过程清空 master 的所有状态，并移除已经发现和关联 master 的所有 slave 和 sentinel。

SENTINEL failover [master name] - 如果 master 不可到达，强制执行一个故障转移，而不征求其他 Sentinel 的同意。

#### 数据结构

##### list

底层采用ziplist存储，当数据量大时，采用quickList存储，quickList就是采用双向指针连接ziplist的存储方式，既节省了空间，又不损失效率

##### hash

和java类似，同样是采用了数组+链表的经典结构，但是当数组扩容时，采用渐进式hash的方式进行扩容，防止单线程阻塞

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200811195501014.png)



##### set

set的实现相当于hash结构，只不过hash的所value全部为null

##### zset

底层采用跳跃表+hash

#### 分布式锁

- redis分布式锁的原理： 

  从伪代码的角度来讲：首先通过setnx指令设置值，成功则代表获取锁成功，否则重试

  setnx key value

  expire key # 为了防止客户端故障没有手动释放锁

  del key # 释放锁

  上述的问题在于setnx key value、expire key不是原子操锁，更换为  ***set key value ex 5 nx***

- 超时问题：

  当线程1代码执行的时间超出数据过期的时间，其他线程（线程2）就会获取到这个锁，而当线程1执行del key时，会导致线程2持有的锁被动释放，其他线程获取锁成功，导致问题

  这个问题解决的方案就是每个线程set的value为一个随机值，删除key时对比一下是否为当前线程持有锁，而这个操作类似于get then del，不是一个原子操作，所以需要借助lua脚本

- 可重入锁思想：28页

#### Redlock ------- Redis Distrubute Lock

https://www.cnblogs.com/rgcLOVEyaya/p/RGC_LOVE_YAYA_1003days.html

#### 异步消息队列

1. 第一种方法：通过rpush + lpop指令混合使用可以实现单消费者的消息队列功能，为了防止cpu空转，当lpop值为nil，线程休眠1s

2. 第二种方法：对于上述方案采用 **rpush + blpop**是最佳方案，但是注意blpop长时间阻塞，redis会主动断开连接，所以应该通过try捕获异常，并重试

#### 延迟中心

1. 存储：采用zset，value存储的是消息的json化字符串数据，score存储的是当前消息delay的时间

2. 轮询：通过如下代码，zrangeByScore(key, 0, currTime, 0, 1) 每次获取一个value，反序列化

   ```
   Set<String> zrangeByScore(final String key, final double min, final double max,
       final int offset, final int count)
   ```

3. 处理value事件， 34页

#### 位图

存储bool型数据，例如一个用户365天内每天是否登陆的数据，底层数据结构就是简单的String，也就是byte数组

setbit s 1

getbit s

bitcount s   # 统计位图中1的个数

bitpos s start stop  # 第一个1的位置，stop和start可选

bitfield   # 魔法指令

#### HyperLogLog

- 使用场景：主要用于数据统计，可以用于统计页面uv，具有去重功能，统计的误差<0.81%

- 指令

  pfadd

  pfcount

  pfmerge

- 实现原理 todo

#### 布隆过滤器 Bloom Filter

- 特点和用途：专门用来解决去重的问题，也就是判断数据存在与否，例如：当前的数据在数据库中是否存在；当前新闻用户是否浏览过；垃圾邮件

- 布隆过滤器说不存在的值，一定不存在；布隆过滤器说存在的值，可能不存在，所以针对不存在的值，存在很小概率的误判，但是能节省90%的存储空间

- redis4.0以上版本需要安装布隆过滤器 https://blog.csdn.net/ChenMMo/article/details/93615438

- 命令

  bf.add

  bf.exists

   bf.madd

  bf.mexists

- 布隆过滤器误差和降低误差

  **bf.reserve key error_rate initial_size**  # 采用bf.reserve手动创建布隆过滤器，指定误判率和预计数据量，默认的误判率为0.01，预计数据量size为100，size的指定根据真实的数据情况指定，不要小于真实数据，误差率可以指定为 0.001来降低误判率，不过误差越小，占用存储空间越大

- 原理：![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200812113148061.png)

  添加key：通过几个不同的hash函数计算hash值，通过根据数组长度取模，映射到数组上置为1

  判断key：通过上述的hash函数计算hash值，如果有一个hash函数算出的值为0，就可以判定当前的key不存在，否则则可能存在

#### redis滑动窗口简单限流

参数为userId，actionKey用户的行为，period时间窗大小，maxCount时间窗内操作次数

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200812134907097.png)

#### 漏斗限流算法

单机漏斗限流算法思路：73页

- capacity漏斗容量， leakingRate流速，leftQuota剩余容量，leakingTs上一次makeSpace时间

- watering()方法，注水，注水时，首先通过makeSpace释放漏斗空间，判断剩余时间是否满足本次注水容量

- makeSpace()方法，释放空间，根据leftQuota和leakingTs释放空间

分布式漏斗算法：

- 将capacity、 leakingRate、leftQuota、leakingTs存储在redis中，采用分布式锁的方式，获取值计算限流逻辑，将值写回redis
- 缺点：引入了分布式锁，提升了代码复杂度和维护的成本，损失了性能和用户体验

redis-cell

- 这个找一下资料，再看一下

#### GeoHash

对于附近的人类似的功能可以采用GeoHash的数据结构，GeoHash底层采用zset存储，value为元素信息，score为GeoHash的52位整数，对于地理坐标来讲，越近的坐标，GeoHash的score越相似，也就是在zset中离得越近

命令：

- geoadd key longitude latitude member   # 存数据
- geodist key member1 menber2  km|m|etc...    # 计算距离
- geopos key member    # 获取坐标
- geohash key member   # 获取geoHash值
- georadiusbymember key member 20 km 3 asc    # 按照redis中存储数据计算距离当前member最近的三个members
- georadius key longitude latitude 20 km withdist 3 asc    # 按照地理坐标计算距离当前member最近的三个members

总之一个原则：现用现看

#### scan

1. keys的问题：keys指令没有提供limit和offset的功能并且时间复杂度为O(n)，由于单线程响应的架构容易导致redis服务卡顿，其他的请求指令响应延后

2. scan的特性

   - 复杂度是O(n)，但是通过游标分步执行，不会阻塞线程
   - cursor在客户端维护，redis服务端不需要维护
   - 返回的值可能会重复，需要手动去重
   - 返回数据条数为0不代表结束，只有返回的游标为0才代表scan结束
   - 遍历过程中数据修改不能保证获取到

3. 命令：scan cursor match regex count 10  （注意count不是每次期望获取的数据个数，而是scan每次遍历字典的槽位数量）

   zscan hscan sscan等指令针对不同的存储结构进行扫描

#### 渐进式hash

和java中的hashMap类似，同样是扩容翻倍

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200812151033715.png)

和java不一样的是redis采用了渐进式的rehash，他会同时保留旧数组和新数组，然后在定时任务中以及后续的新指令渐渐的将数据从旧的数组中迁移到新数组中；那么查询时，先在旧的数组中查询，查询不到再到新的数组中查；scan对于rehash的数组，同时扫描新旧两个数组的槽，将数据汇总返回客户端

#### 大key

大key是指redis实例中很大的对象，大key在集群数据迁移、扩容以及删除时可能会导致系统卡顿

大key分析与定位：redis-cli -h 127.0.0.1 -p 6379 --bigkeys

#### 线程IO模型

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200812171831519.png)

对于redis来讲，redis采用单线程+多路复用的方式实现事件轮询，以下为redis处理请求的流程

- [redis服务器端绑定server socket事件]
- [acceptCommonHandler接受客户端的请求]
- [为客户端连接创建一个上下文]
- [接收客户端传来的数据]
- [整理客户端传来的数据：验证、识别、判断如何处理]processCommand)
- [执行客户端所指定的命令]
- [返回数据至客户端]

按照我的理解应该是：

1. 首先向selector注册accept事件，当有其他线程请求建立连接之后，向selector注册read事件，我redis要读取客户端传来的数据了！
2. 当有selector中有read事件准备就绪(也就是用户的请求kernal读取完毕之后)，读取请求指令，将当前客户端的指令放到当前客户端关联的指令列表中，依次处理
3. 将处理结果以此放到响应队列中，将文件描述符重新放入write_fds中，将数据写给客户端

#### redis如何顺序处理请求呢？

##### 指令队列

Redis会将每个客户端的套接字关联一个指令队列，按照指令顺序排队依次处理

##### 响应队列

Redis同样会将每个客户端的套接字关联一个响应队列，如果队列为空，就将客户端描述符从write_fds中暂时移除，等到有数据了，再将描述符放进去

#### Redis高性能的原因

1. 单线程避免了线程切换和竞态产生的消耗
2. 非阻塞I/O，Redis使用epoll作为I/O多路复用技术的实现，在加上Redis自身的事件处理模型将epoll中的链接、读写、关闭都转换为事件，不在网络I/O上浪费过多的时间；
3. 纯内存访问，Redis将所有数据放在内存中，内存的响应时间大约为100纳秒，这时Redis达到每秒万级别访问的重要基础；

#### 定时任务

redis单线程当select阻塞，会导致定时任务无法准时执行，redis的解决方案 是将任务存在最小堆的结构中，其中堆的顶点为需要执行的定时任务，同时redis在执行完定时任务之后，计算下一个需要执行的任务时间，依照这个时间计算出timeout，这个timeout就是select阻塞的时间

#### 通信协议

redis的作者认为数据库的性能瓶颈一般不在网络层面，所以才用了浪费流量的**文本协议**

这个点就不细看了

#### Redis持久化

##### RDB：内存的二进制快照

redis在持久化时，会fork出一个子进程，快照持久化交给子进程执行。子进程做持久化只是简单的将内存中的页遍历读取，然后将序列化的数据写到磁盘。当有主进程接收客户端请求对内存的数据结构进行不断的修改，会利用系统的Copy On Write机制来进行数据页的分离，在分离出来的页面进行修改。这样子进程共享主进程的数据页完全没变化，所以就相当于一个不会变的快照。

##### AOF

AOF日志记录的是redis内存数据修改的指令记录文本，AOF只记录对内存进行修改的指令记录。Redis在接收到客户端的请求指令时，首先验证参数，如果没问题，立即将指令文本存储到AOF日志中，也就是先存到磁盘，然后再执行命令，这样即使redis突然宕机，也可以通过AOF日志指令重放就可以恢复。

##### AOF重写----->瘦身

开辟出一个子进程对内存遍历转换成一系列的Redis操作指令，并且将操作指令序列化到一个新的AOF文件中，序列化完成后将操作期间发生的数据增量AOF日志追加到新的AOF日志，并用新的AOF日志替换原先的日志，完成AOF瘦身。

##### fsync

当程序对AOF日志执行写操作时，实际上是将数据写到了内核为文件描述符提供的内存缓冲中，然后内核会异步将脏数据刷回磁盘。linux提供fsync(int fd)可以将指定文件的内容强制从内存缓存刷到磁盘。redis提供三种模式，默认everysec，另外两种always和no分别为每次和系统自动刷，一般不用。

##### 混合持久化 Redis4.0

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200813103330859.png)

将RDB数据和AOF日志文件放在一起，AOF不存储全量日志，而是持久化开始到结束这段时间发生的增量日志，于是redis重启先加载RDB再加载AOF就可以完成数据的重放

#### Pipline   108页

对于write和read操作，write是将数据写到操作系统的发送缓冲，几乎不耗时

read操作反而是要等到RedisServer处理完数据将数据通过服务器内核经过网卡和网线传到目标机器的网卡，内核读取完毕才能read，很耗时

所以redis的pipline就是redis client提供的优化，调整了读写的顺序，将写命令一股脑的写给内核，然后阻塞等待read，相当于只阻塞了第一个read的时间

### Redis缓存数据一致性

通过缓存可以提升系统的性能，但是会带来缓存数据与数据库的一致性问题

- 场景1:更新数据库成功，更新缓存失败，数据不一致；
- 场景2:更新缓存成功，更新数据库失败，数据不一致；
- 场景3:更新数据库成功，清除缓存失败，数据不一致；
- 场景4:清除缓存成功，更新数据库失败，数据弱一致；

造成这样的结果，原因有两个方面：一是写操作中更新数据库与更新缓存是两个操作，而不是一个原子操作；二是读操作中读取数据库和写入缓存两个操作不是原子的。要解决这个问题，需要做一些修改，引入分布式锁：

写操作	
1.清除缓存；若失败则返回错误信息（本次写操作失败）。
2.对key加分布式锁。
3.更新数据库；若失败则返回错误信息（本次写操作失败）同时释放锁，此时数据弱一致。
4.更新缓存，即使失败也返回成功，同时释放锁，此时数据弱一致。	

读操作
1.查询缓存，命中则直接返回结果。
2.对key加分布式锁。
3.查询数据库，将结果直接写入缓存，返回结果，同时释放锁。

[https://ouyblog.com/2017/04/Redis%E7%BC%93%E5%AD%98%E6%95%B0%E6%8D%AE%E4%B8%80%E8%87%B4%E6%80%A7](

#### 事务

1. 命令

   multi

   exec

   discard

2. redis事务的特点

   exec 执行事务，但是不同于MySql，redis的执行不保证所谓的原子性，当中间指令执行失败不影响其他指令，但是满足隔离性，中间不会有其他指令插入到事务中间执行

   discard 回滚事务，中间的指令全部不执行

3. 事务一般是伴随着pipline一起使用，否则会导致多次的网络IO开销

#### Watch

watch一个变量，在接下来的multi事务中，如果watch的变量变化之后，multi会执行失败，返回null

将watch理解为乐观锁，分布式锁理解为悲观锁就OK了

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200813112648639.png)

#### PubSub

发布订阅，比较简单，发布订阅模式，和消息队列不同的是，消息订阅支持多播

#### 内存策略

##### 小对象优化

##### 内存回收

按页回收，如果key删掉，key所在的页不为空，页不会回收，需要手动执行flushDb

##### 内存分配

采用第三方的库管理内存

#### redis主从

#### Redis的CAP

一句话，当主从发生网络分区redis选择放弃一致性，保证可用性，redis主从采用最终一致性，从节点会努力追赶主节点，实现数据的最终一致性。

#### 主从复制

##### 主从同步过程

当在从库的client端或者配置文件通过==slaveof==指令设置主从关系时，过程如下：

1. 从库向主库发送sync请求
2. 主库执行bgsave指令，生成一个RDB文件，并且采用一个buffer保存从现在开始的所有写命令
3. 将RDB发送给从库，从服务器接收并且加载RDB文件数据
4. 主服务器将缓冲数据发送给从服务器，从服务器加载完成同步

##### 同步方式

1. 增量同步：redis的同步指的是指令流，主节点会将自己的状态记录在本地的内存buffer中，然后异步的将buffer的指令同步给从节点，从节点一边执行同步的指令流，一边反馈自身的同步状态。

   因为redis中内存的buffer是有限的定长环形数组，当buffer满了之后，增量数据会覆盖环的头节点数据，这时候就不能采用增量同步的主从同步方式，而采用第二种，快照同步

2. 快照同步的方式是：主节点立即执行一次bgsave指令，将数据全量写入磁盘，然后将快照文件的内容发送给从节点，从节点加载快照文件数据，将自身数据全部删除，重新加载数据，完成同步

3. 当redis增加新节点之后，从节点必须执行一次快照同步，后续的操作采用增量同步

4. 无盘复制：redis2.8之后提供了无盘复制，主服务器直接通过套接字将数据发送到从节点，主节点遍历数据，一边遍历一边发送数据给从节点。

5. wait指令：redis的主从同步默认是异步执行的，wait指令可以强制同步执行，实现数据的强一致性。  wait 1 0。 1表示从库数量为1，0表示无限时间将数据同步给从库，如果发生了网络分区，wait将永久阻塞，损失可用性

6. 主库给从库法数据时会带上自身的偏移量，从库加载完数据更新偏移量来判断主从之间的数据是否一致



#### Sentinel哨兵

##### Sentinel的作用

当节点发生故障，自动实现主从切换，实现系统的高可用。Sentinel集群监控主节点的状态，当主节点发生故障，自动切换主节点，客户端来连接集群时，会首先连接Sentinel集群，Sentinel集群会告诉客户端主节点的地址，当主节点发生故障，会重新向Sentinel要地址，所以客户端无需重启实现节点切换。

##### Sentinel的数据丢失问题

Sentinel不能保证数据不丢失，因为主节点挂掉，当增量数据所有从节点没有同步，就会导致少量数据丢失

通过min-slaves-to-write 1 （表示必须有一个从节点正常主从复制，牺牲可用性）和min-slaves-max-lag 10（表示10s没接到任何反馈，意味着从节点同步不正常） 两个参数表示同步是否正常，不正常停止对外提供服务。

##### Sentinel集群搭建

1. 编写配置文件
2. 启动sentinel 命令： redis-server sentinel-my.conf --sentinel
3. 命令，见命令汇总

##### Sentinel检测服务器状态

1. Sentinel每1s向其他主服务器、从服务器、Sentinel实例发送PING，其他实例应该正常响应，如果在配置的min-slaves-max-lag时间内没有响应，则当前的sentinel节点主观下线失联的节点
2. 当一个sentinel对一个其他节点主观下线之后，会询问其他的sentinel，来确认是否节点真的已经下线，当足够多的sentinel认为节点已经下线了（quorum参数，如果配置2，则2个sentinel认为节点下线就OK）也就是节点的客观下线！

##### Sentinel Leader选举

当Sentinel对主节点客观下线时，需要选举出sentinel 的leader节点来完成本次的==故障转移==

原则：所有的sentinel节点都有成为leader的权利，每次选举之后节点的纪元epoch都会更新+1，每一个纪元，sentinel节点都有一个将某个sentinel设为leader的机会，并且在这个纪元中不能修改

1. 每个发现主节点客观下线的sentinel节点都会要求其他sentinel节点将自己设置为leader节点
2. sentinel采用先到先得的原则，sentinel节点会为最早收到的sentinel节点投上一票，后续的请求全部拒绝投票
3. 当发起选举的sentinel节点收到投票信息之后，和自身的epoch、运行ID进行比较，如果一致，票数+1
4. 如果有sentinel节点收到半数以上的票，那么选举结束，当选为leader
5. 如果在有限的时间内没有选举成功，那么延迟一段时间重新选举

##### 故障转移

当master下线达到用户指定的时间，那么Sentinel系统会察觉到master节点的下线，会对master节点进行故障转移

1. Sentinel挑选从节点替代主节点
2. Sentinel向剩余从节点发送复制的命令，让他们去复制新的主节点
3. 监控下线的主节点，当主节点连接上之后，让他去复制新的master节点

新master选举：

首先先过滤掉下线、断线的从节点，在过滤掉一些延迟比较高，短时间没有info响应的节点

过滤掉之后，首先对比从服务器的优先级，优先级高的当选master；如果优先级相同，采用偏移量offset大的节点；如果以上两个条件均相同，采用设备ID较小的节点

##### java使用Sentinel

https://segmentfault.com/a/1190000002690506 一系列的文章

简单总结一下就是new JedisSentinelPool对象，回去jedis实例，通过实例操作数据库

在JedisSentinelPool通过get-master-addr-by-name的命令获取主节点信息，其中参数master-name应该是在sentinel配置文件中配置的

并且启动线程通过订阅+switch-master的消息，实现主节点切换时间的监听

```java
//初始化Sentinel连接池，注意：这里名字是JedisSentinelPool只是为了区分它是sentinel方式连接，其内部还是连接master
JedisSentinelPool sentinelPool = new JedisSentinelPool(masterName,SentinelSet,poolConfig,timeout);

Jedis jedis = null;
try{
  jedis = sentinelPool.getResource();
  //这里执行jedis command
}catch(Exception e){
  logger.error(e.getMessage(),e);
}finally{
  if(jedis != null)
     //归还连接
     jedis.close();
}
```

#### Redis Cluster

##### 集群算法

https://zhuanlan.zhihu.com/p/92937061

##### 基本概念

Redis Cluster是Redis提供的集群方案，术语包括节点和槽，redis将数据划分为16384个slots，槽位的信息存储在每个节点中。当redis cluster来查找某个key时，可以直接定位到目标节点。除此之外呢，客户端会缓存节点信息，以方便与快速定位节点。

##### 槽位定位算法

采用crc32 hash算法计算hash值，对16384进行取模来定位slot。

##### 数据迁移

从原节点获取key的列表 ------> 依次迁移key，从原节点dump数据 ------> 目标节点restore ------> 原节点删除key完成数据迁移

当数据迁移之后，客户端获取数据之后定位slot和节点失败，会返回moved slot ip:port的数据，重新查询

可以通过redis-trib工具实现数据迁移

##### 集群容错

集群中的角色分为主节点和从节点，主节点负责槽数据的操作，从节点复制直接点数据。

当某个主节点断开连接，其他主节点会像sentinel一样的方式将失联的主节点客观下线，并广播下线的消息；

当从节点接收到主节点下线的消息之后，从节点会对主节点进行故障转移；

新的主节点选举：

和sentinel选leader方案类似，从节点率先发起投票，并且设置epoch等数据，参与投票的是主节点，当从节点收到超出半数的票，则自动升级为主节点，发出广播；否则重新选举。

##### 主节点的可能下线与确定下线

由于Redis Cluster是去中心化的，所以当一个节点认为某个节点下线并不能代表节点下线。Redis采用Gossip协议来广播自己对集群状态变化信息。

1. 例如一个节点发现另外一个节点A失联了，首先会向整个集群广播节点下线信息，其他节点会收到节点下线的信息。

2. 当一个节点收到节点失联的数量已经达到了节点数量的大多数，则能标记该节点下线确认下线状态，并广播此消息，强迫其他节点接受A节点下线的消息。
3. 对失联的节点进行主从切换

##### 失联节点的从节点选举原理

1. 当从节点发现自己的主节点挂了，首先会延迟一下，DELAY = 500ms + random(0 ~ 500ms) + SLAVE_RANK * 1000ms

2. 率先DELAY完成的从节点发起选举，期望成为新的master节点，将自己记录的集群currentEpoch（选举轮次标记）加1，并广播信息给集群中其他节点
3. 其他节点收到该信息，只有master响应，判断请求者的合法性，并发送结果
4. 尝试选举的slave收集master返回的结果，收到超过半数master的统一后变成新Master
5. 广播Pong消息通知其他集群节点。
6. 如果本次选举不成功，重新选举

#### 诊断指令info

info

#### Redlock

#### Redis过期策略

1. redis会将设置过期时间的key放在一个独立的字典中，以后会定期遍历这个字典来删除到期的key，定时扫描的策略为一种贪心的策略：
   - 从过期字典中随机 20 个 key
   - 删除这 20 个 key 中已经过期的 key；
   - 如果过期的 key 比率超过 1/4，那就重复步骤 1
2. 使用惰性策略来删除过期的 key，所谓 惰性策略就是在客户端访问这个 key 的时候，redis 对 key 的过期时间进行检查，如果过期 了就立即删除

注意以上为主库的过期策略，从库的话通过主库在AOF日志写入的del命令实现过期数据删除

#### 数据淘汰策略

Redis提供了MaxMemory参数，来限制内存使用的最大容量。当存储数据导致内存超出限制，redis提供如下选项来指定key淘汰的策略

- neoviction   # 阻止写请求，允许读请求，为redis的默认策略
- volatile-lru    # 淘汰最少使用的key，可以采用链表每次将使用的key放在链表头，这样末尾的key就是最少使用的key，但是redis采用了近似lru算法，每次从redis中随机取出n个key，删除最近一次访问时间戳最小的key
- volatile-ttl   # 剩余过期时间
- volatile-random   # 随机
- allkeys-lru   # 不区分key是否过期，一般用于全缓存redis
- allkeys-ttl

#### Redis的异步线程

redis在执行耗时操作时做了优化，采用异步线程+任务队列的形式提升主线程的效率

1. unlink    当redis删除超大对象时，由于内存回收会导致主线程阻塞，所以redis采用异步线程实现回收，将大对象放到线程安全的异步队列中
2. flushdb|flushall async    当清空redis时采用异步的方式

![image](https://github.com/wangjunjie0817/note/raw/master/images/image-20200814135053339.png)

#### 保护redis

##### 危险的命令

rename-command keys abckeysabc # 将命令重命名，保护线上数据库

rename-command flushall ""

##### 设置密码

server：requirepass pass

client：redis-cli -p 6379 -a password 

##### SSL安全

可以考虑使用spiped工具，具体有需要的话

#### 注意事项

1. 当字符串设置了expire time，当对key重新赋值写入，过期时间被删除
2. 如果redis当成缓存数据库来用，那么挂掉重启就好了；如果当成数据库来用，那么必须认真对待主从，保证数据安全

#### todo

redlock
