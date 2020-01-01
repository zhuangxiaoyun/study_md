# kafka

## 1、topic

1. **partition**

   分区内有序

   负载均衡【producer】，提高并发【consumer】

2. **副本**

   副本的数量不能大于broker的数量
   partition没有限制

## 2、生产者---》topic---》partition【broker】

1. ​    分区策略
   ​        1、指定了分区
   ​        2、未指定分区，但是指定了消息的key,通过对key进行hash选出一个分区
   ​        3、未指定分区，未指定消息的key，则通过轮询选出分区
2. 数据可靠性保证
       1. ack应答机制
             1. ack=0，只管发送，leader不用应答
             2. ack=1，需要leader应答
             3. ack=-1，需要leader和跟随者都应答
                   1. 如果有一台副本挂了，则消息无法再生产
                   2. 优化策略：只要ISR中的副本完成了同步，则响应ack
                         1. 缺点：在除了leader外的所有副本都不在ISR时，退化为ack=1
           2. ISR
             1. ![1577027169780](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577027169780.png)
             2. replica.lag.time.max.ms：默认10s
             3. replica.lag.time.max.messages在0.9时被移除了
                   1. 因为producer是批次的，如果batch发送的数据量大于replica.lag.time.max.messages，则在producer发送数据给leader后，follower还没同步之前，所有的follower都会被踢出ISR，然后follower同步leader之后，又会加入到ISR
           3. HW和LEO
             1. ![1577112688342](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577112688342.png)
             2. HW: high watermark
                   1. 作用【消费一致性和存储一致性】
                      1. consumer可以读取到的最大的offset，即HW
                      2. leader重新选举后，其他的follower都截取到HW，然后跟leader同步其他数据
                      3. ![1577114000330](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577114000330.png)
             3. LEO: log end offset
           4. exactly once
             1. ack=0: at most once
             2. ack=-1: at least once
             3. **at least once + 幂等性 = exactly once**
                   1. ![1577283171491](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577283171491.png)
                   2. 启用幂等性：enable.idompotence=true【启动幂等性后，ack会被设置为-1】
                   3. **幂等性只能解决 单次会话，单个分区的数据重复问题**
                         1. producer重启后pid会变化，重启后不能保证数据不重复
                         2. 不同分区的数据不能保证不重复
3. 生产数据的流程
          1、producer从broker-list中获取partition的leader
          2、将数据发送给leader
          3、leader将数据写入到本地的log中
          4、follower从leader中pull数据
          5、follower将数据写入本地的log中，并向leader发送ack
          6、leader收到所有follower的ack后，向producer发送ack

## 3、存储数据

![1577025365717](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577025365717.png)





![1577025798467](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577025798467.png)



1. xxx.index
2. xxx.log
   1. 有效期：**7天**【log.retention.hours】或者**1G**【log.segment.bytes】
3. 文件命名
   1. index和log文件以当前segment的第一条消息的offset命名
4. 通过offset定位文件位置
   1. 先通过二分查找找到对应的index文件
   2. 从index查找到offset的位置，找到消息在log文件的偏移量和大小

## 4、消费者

​    消费者组【组内互斥，同一个消费组内，两个消费者不能同时消费同一个topic同一个分区的数据】

1. 消费方式：push【拉的方式】
2. 分区分配策略
   1. 消费者可以直接指定分区进行消费
   2. roundRobin【轮询】：基于group
      1. ![1577283811617](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577283811617.png)
      2. 将topic和partition组合起来求hashcode，然后将hashCode进行排序，最后轮询的方式分配给consumer
   3. range：基于topic
   4. 分区分配策略何时触发？
      1. 当消费者个数发生变化的时候
3. offset的维护
   1. 保存的唯一依据：消费者组+topic+partition
   2. 保存在zk
      1. ![img](https://images2018.cnblogs.com/blog/1228818/201805/1228818-20180508101652574-1613892176.png)
   3. 保存在broker
      1. 主题：__consumer_offsets
      2. 修改配置文件，支持读取kafka内部的topic，即__consumer_offsets
         1. exclude.internal.topics=false
         2. ![1577858234958](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577858234958.png)

## kafka特性

1. 高效读写
   1. 集群分布式【partition】
   2. 磁盘的顺序读写
   3. 零拷贝
2. zk在kafka中的作用
   1. topic的partition的leader的选举【topic的partition使用哪个broker作为leader，在replication-factor>1时，主要看ISR】
   2. broker选举controller
3. 事务
4. ![1577865025650](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577865025650.png)
   1. producer
      1. 跨会话跨分区的精准一次
   2. consumer

## 5、命令操作

1. **kafka-topics.sh**

   1. **list**
      1. bin/kafka-topics.sh --lisr --zookeeper zk_host:port 
   2. **describe**
      1. bin/kafka-topics.sh --describe --zookeeper zk_host:port --topic topic_name
   3. **create**
      1. bin/kafka-topics.sh --create --zookeeper zk_host:port --topic topic_name --partitions 2 --repication-factor 3 
   4. **delete**
      1. bin/kafka-topics.sh --delete --zookeeper zk_host:port --topic topic_name
   5. **alert**
      1. bin/kafka-topics.sh --alert --zookeeper zk_host:port --topic topic_name --partitions 2 --repication-factor 3 
      2. ==当前，Kafka 不支持减少一个 topic 的分区数。== 

2. 

3. 注意点

   1. Balancing leadership

      1. 每当一个 borker 停止或崩溃时，该 borker 上的分区的leader 会转移到其他副本。这意味着，在 broker 重新启动时，默认情况下，它将只是所有分区的跟随者，**这意味着它不会用于客户端的读取和写入**。

         ​		为了避免这种不平衡，Kafka有一个**首选副本**的概念。如果分区的副本列表为1,5,9，则节点1首选为节点5或9的 leader ，因为它在副本列表中较早。

         ​		您可以通过运行以下命令让Kafka集群尝试恢复已恢复副本的领导地位：`> bin``/kafka-preferred-replica-election``.sh --zookeeper zk_host:port``/chroot`由于运行此命令可能很乏味，

         ​		您也可以通过以下配置来自动配置Kafka：`auto.leader.rebalance.enable=true`

      2. 

## 6、API

### 1、producer

1. 发送数据采用的是**“异步发送”**

2. 两个线程：**“main线程”和“send线程”**，线程共享变量**“recordAccumulator”**

3. ![1577865535453](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1577865535453.png)

4. KafkaProducer、ProducerRecord、ProducerConfig、CommonClientConfigs

5. ```java
   Properties properties = new Properties();
           //        1、连接信息
           properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
           //        2、ack类型
           properties.put("acks", "all");
           //        3、重试次数
           properties.put("retries", 3);
           //        4、批次大小 16K
           properties.put("batch.size", 16384);
           //        5、等待时间 1ms
           properties.put("linger.ms", 1);
           //        6、recordAccumulator缓冲区大小 32M
           properties.put("buffer.memory", 33554432);
           //        key /value 的序列化
           properties.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
           properties.put(
               "value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
   
           KafkaProducer<String, String> producer = new KafkaProducer<String, String>(properties);
   
           for (int i = 0; i < 10; i++) {
               producer.send(new ProducerRecord<String, String>("first", "test+i"));
           }
           //        关闭资源
           producer.close();
   ```

   

### 2、consumer

1. KafkaConsumer、ConsumerRecords

2. ```java
   Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "test");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
   
   KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
   consumer.subscribe(Arrays.asList("foo", "bar"));
   while (true) {
       ConsumerRecords<String, String> records = consumer.poll(100);
       for (ConsumerRecord<String, String> record : records)
           System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
   }
   ```

   

