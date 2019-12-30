# kafka

## 1、topic

1. **partition**

   分区内有序

   负载均衡【producer】，提高并发【consumer】

2. **副本**

   副本的数量不能大于broker的数量
   partition没有限制

### 2、生产者---》topic---》partition【broker】

1. ​    生产数据分区的分配方式
   ​        1、指定了分区
   ​        2、未指定分区，但是指定了消息的key,通过对key进行hash选出一个分区
   ​        3、未指定分区，未指定消息的key，则通过轮询选出分区
2. ack应答机制
           1、ack=0，只管发送，leader不用应答
           2、ack=1，需要leader应答
           3、ack=all，需要leader和跟随者都应答
3. 生产数据的流程
           1、producer从broker-list中获取partition的leader
           2、将数据发送给leader
           3、leader将数据写入到本地的log中
           4、follower从leader中pull数据
           5、follower将数据写入本地的log中，并向leader发送ack
           6、leader收到所有follower的ack后，向producer发送ack

## 3、存储数据

## 4、消费者

​    消费者组【组内互斥，同一个消费组内，两个消费者不能同时消费同一个topic同一个分区的数据】