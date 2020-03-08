# flink

## 0.flink运行时组件(JM,RM,TM,dispatcher)

## Flink 允许子任务共享 slot

一个流程序的并行度，可以认为就是其所有算子中最大的并行度。
一个程序中，不同的算子可能具有不同的并行度
Stream 在算子之间传输数据的形式可以是 one-to-one(forwarding)的模式也可以
是 redistributing 的模式
任务链: 相同并行度的 one to one 操作，Flink 这样相连的算子链接在一起形成一个 task

## 1.并行度

1. ​	flink每个算子都可以设置并行度

## 2.flink的操作链

可以设置的

1. env.disableOperationchain
2. stream.disableChain
3. stream.startNewChain

## 3.UDF

1. sourceFunction自定义source
2. sinkFunction自定义sink

## 4.多流转换算子

1. split,select 
2. connect ,coMap
3. union

## 5.sink

1. kafkaSink
2. redisSink
3. esSink
4. jdbcSink

## 6.window

1.   两种类型：(timeWindow,countWindow)
2. windowAssigner：窗口分配器，用来划分窗口的
3. windowFunction
   1. windowFunction又可以分为(增量聚合函数、全量窗口函数)
4. trigger(): 触发器，定义窗口的关闭时机，可以提前触发的窗口的关闭(timeWindow默认以窗口结束时间到达时关闭窗口)
5. ==allowedLateness==
   1. 问题：
      1. allowedLateness和watermark的延迟的区别？
         1. watermark的延迟是用来定义什么时候关闭窗口
         2. allowedLateness可以在窗口触发后，再更新窗口
   2. 默认情况下，当watermark通过end-of-window之后，再有之前的数据到达时，这些数据会被删除。
   3. 为了避免有些迟到的数据被删除，因此产生了allowedLateness的概念。
   4. 简单来讲，**allowedLateness就是针对==event time==而言**，对于watermark超过end-of-window之后，还允许有一段时间（也是以event time来衡量）来等待之前的数据到达，以便再次处理这些数据。
   5. 注意：对于trigger是默认的EventTimeTrigger的情况下，**allowedLateness会再次触发窗口的计算**，而之前触发的数据，会buffer起来，**直到watermark超过end-of-window + allowedLateness（）**的时间，窗口的数据及元数据信息才会被删除。再次计算就是DataFlow模型中的Accumulating的情况。

## 7.如果使用processTime则不需要设置数据时间；如果使用eventTime则需要设置数据时间

processTime没有watermark
watermark的timestamAssigner有两种类型:周期性(默认是每200毫秒触发一次)，打点式（与数据相关）
watermark特点: 1.单调递增，2.与数据时间相关
watermark衡量eventTime的标志
eventTime是通过watermark推进的
watermark向下游发送时，取当前算子分区中最小的watermark为准
窗口–>时间语意–>watermark(eventTime需要指定)

## 8.processFunction,keyedProcessFunction

timeService，定时器(timer)

## 9.算子状态和健控状态

算子状态(列表状态，联合列表状态，广播状态)
健控状态(值状态，列表状态，map状态，...)

## 10.状态后端 （MemoryStateBackend,FsStateBackend,RocksDBStateBackend）

## 11.checkpoint(内部实现精确一次，但不保证外部是否精确一次)

JM触发checkpoint的
savepoint

## 12.状态一致性

内部:checkpoint实现exactly once
sink:幂等和事务(预写日志WAL，两段式提交2PC)



## 13、CEP

### 1.个体模式

​         单例和循环(量词->times(2))
​         条件(简单，组合，终止，迭代条件: where,or,until)

### 2.组合模式

​     模式序列(严格近邻[next]，宽松近邻[followedBy],非确定性宽松近邻[followedByAny])
​     否定模式序列(notNext,notFollowedBy)
​     限定时间: within
​     通过select或者flatSelect将patternStream转成dataStream

### 3.模式组

订单超时: 通过超时的侧输出流