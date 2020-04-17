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

## 14、flink运行

### 1、job运行的环境

org.apache.flink.streaming.scala.examples.windowing.WindowWordCount示例入口

api job, 非sql job

1. ```scala
   
   val env = StreamExecutionEnvironment.getExecutionEnvironment//scala代码获取执行环境
   ```

   

2. ```scala
   def getExecutionEnvironment: StreamExecutionEnvironment = {
   	//scala的StreamExecutionEnvironment的构造器需要一个java类型的StreamExecutionEnvironment【JavaEnv是java类型的StreamExecutionEnvironment的别名】
       new StreamExecutionEnvironment(JavaEnv.getExecutionEnvironment)
     }
   ```

3. ```java
   //JavaEnv.getExecutionEnvironment：java代码获取执行环境
   
   public static StreamExecutionEnvironment getExecutionEnvironment() {
       return Utils.resolveFactory(threadLocalContextEnvironmentFactory, contextEnvironmentFactory)//如果设置了threadLocalContextEnvironmentFactory，则从此threadLocal中获取Factory；若未设置，则直接获取入参的contextEnvironmentFactory
           .map(StreamExecutionEnvironmentFactory::createExecutionEnvironment)//调用factory的createExecutionEnvironment获取执行环境
          .orElseGet(StreamExecutionEnvironment::createStreamExecutionEnvironment);//如果factory获取不到执行环境，则调用StreamExecutionEnvironment的createStreamExecutionEnvironment方法获取
   }
   ```

4. ```java
   //StreamExecutionEnvironment::createStreamExecutionEnvironment
   private static StreamExecutionEnvironment createStreamExecutionEnvironment() {
       ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
       if (env instanceof ContextEnvironment) {
           return new StreamContextEnvironment((ContextEnvironment) env);
       } else if (env instanceof OptimizerPlanEnvironment) {
           return new StreamPlanEnvironment(env);
       } else {
           return createLocalEnvironment();
       }
   }
   ```

5. ```java
   //ExecutionEnvironment.getExecutionEnvironment();
   public static ExecutionEnvironment getExecutionEnvironment() {
       return Utils.resolveFactory(threadLocalContextEnvironmentFactory, contextEnvironmentFactory)//CLI client中会设置为ContextEnvironmentFactory；web frontend中会调用org.apache.flink.client.program.OptimizerPlanEnvironment#setAsContext设置为可以返回OptimizerPlanEnvironment的匿名内部类
           .map(ExecutionEnvironmentFactory::createExecutionEnvironment)
           .orElseGet(ExecutionEnvironment::createLocalEnvironment);//local执行的时候未设置threadLocal，所有最后创建的是LocalEnvironment
   }
   ```

6. 总结：

   1. 如果在StreamExecutionEnvironment的threadLocalContextEnvironmentFactory设置了执行环境，则直接获取
   2. 否则通过获取一个执行环境：ExecutionEnvironment.getExecutionEnvironment()
      1. 如果ExecutionEnvironment的threadLocalContextEnvironmentFactory设置了执行环境，则直接获取
         1. CLI client中会设置为ContextEnvironmentFactory；【createExecutionEnvironment()返回ContextEnvironment】
         2. web frontend中会调用org.apache.flink.client.program.OptimizerPlanEnvironment#setAsContext设置为可以createExecutionEnvironment()返回OptimizerPlanEnvironment的匿名内部类
      2. 否则创建本地执行环境：LocalEnvironment
   3. 判断获取的ExecutionEnvironment
      1. ContextEnvironment，则创建StreamContextEnvironment
         1. 在CLI 
      2. OptimizerPlanEnvironment，则创建StreamPlanEnvironment
         1. 在web 
      3. 其他，则创建LocalEnvironmen
   4. **StreamExecutionEnvironment的子类重写了的execute()**
      1. 好像除了StreamContextEnvironment和ScalaShellStreamEnvironment的execute()有点不一样，
      2. RemoteStreamEnvironment只是捕获了异常
      3. LocalStreamEnvironment直接调用 父类方法

7. 杂记：

   1. orElse()和orElseGet()相同点和区别

      1. 相同点：都是在optional中值为空的时候返回的值

      2. 区别

         1. orElse()的入参数是数据T【对应的类型值】，orElseGet()的入参是方法 Supplier<? extends T> other

         2. 无论optional中是否有值，orElse()中若是调用了方法，此方法都会被调用

         3. 只有在optional中无值，orElseGet()中的方法才会被调用

            ![1585475760597](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1585475760597.png)

### 2、job执行过程

1. api job 

   1. ```scala
      eg: env.execute("WindowWordCount")
      def execute(jobName: String) = javaEnv.execute(jobName)//执行任务
      ```

   2. ```java
      public JobExecutionResult execute(String jobName) throws Exception {
      		Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");
      
      		return execute(getStreamGraph(jobName));//StreamExecutionEnvironment的部分子类重写了此方法，即不同的子类会有不同的启动流程
      	}
      ```

      ==getStreamGraph(jobName)；//todo  后续需要详细了解==

      

   3. ```java
      //StreamExecutionEnvironment的execute
      JobClient jobClient = executeAsync(streamGraph);
      
      //executeAsync方法
      final PipelineExecutorFactory executorFactory =
      			executorServiceLoader.getExecutorFactory(configuration);
      //获取执行器工厂executorFactory
      /**
      1、ServiceLoader.load(PipelineExecutorFactory.class)：使用SPI获取PipelineExecutorFactory的子类【从classpath路径下的/META-INFO/services文件夹中加载文件名为org.apache.flink.core.execution.PipelineExecutorFactory中配置的所有类】
      2、factory != null && factory.isCompatibleWith(configuration)
      3、如果查询出多个factory，则抛异常
      */
      		.....
      		CompletableFuture<JobClient> jobClientFuture = executorFactory
      			.getExecutor(configuration)//从执行器工厂获取执行器
      			.execute(streamGraph, configuration);//执行streamGraph
      ```

      PipelineExecutorFactory的所有子类：

      ![1585491429929](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1585491429929.png)

   4. ```java
      //LocalExecutor的执行过程
      public CompletableFuture<JobClient> execute(Pipeline pipeline, Configuration configuration) throws Exception {
          checkNotNull(pipeline);
          checkNotNull(configuration);
      
          // we only support attached execution with the local executor.
          checkState(configuration.getBoolean(DeploymentOptions.ATTACHED));
      //TODO 后续需要了解如何从streamGraph转变成jobGraph的
          final JobGraph jobGraph = getJobGraph(pipeline, configuration);
          //本地启动最小集群
          final MiniCluster miniCluster = startMiniCluster(jobGraph, configuration);
          //获取最小集群客户端
          final MiniClusterClient clusterClient = new MiniClusterClient(configuration, miniCluster);
      	//集群客服端提交jobGraph
          CompletableFuture<JobID> jobIdFuture = clusterClient.submitJob(jobGraph);
      	
          jobIdFuture
              .thenCompose(clusterClient::requestJobResult)//submitJob和requestJobResult进行合并
              .thenAccept((jobResult) -> clusterClient.shutDownCluster());//集群客户端停止集群
      
          return jobIdFuture.thenApply(jobID ->
                                       new ClusterClientJobClientAdapter<>(() -> clusterClient, jobID));
      }
      ```

   5. ```java
      //RemoteExecutor extends AbstractSessionClusterExecutor
      
      public CompletableFuture<JobClient> execute(@Nonnull final Pipeline pipeline, @Nonnull final Configuration configuration) throws Exception {
          final JobGraph jobGraph = ExecutorUtils.getJobGraph(pipeline, configuration);
      
          try (final ClusterDescriptor<ClusterID> clusterDescriptor = clusterClientFactory.createClusterDescriptor(configuration)) {
              final ClusterID clusterID = clusterClientFactory.getClusterId(configuration);
              checkState(clusterID != null);
      
              final ClusterClientProvider<ClusterID> clusterClientProvider = clusterDescriptor.retrieve(clusterID);
              ClusterClient<ClusterID> clusterClient = clusterClientProvider.getClusterClient();
              return clusterClient
                  .submitJob(jobGraph)
                  .thenApplyAsync(jobID -> (JobClient) new ClusterClientJobClientAdapter<>(
                      clusterClientProvider,
                      jobID))
                  .whenComplete((ignored1, ignored2) -> clusterClient.close());
          }
      }
      ```

      

      **clusterClientFactory的子 	类：远程执行有多种方式：k8s，yarn，standalone**

      ![1585493586485](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1585493586485.png)

   6. 总结：

      1. executor()获取JobClient ：JobClient jobClient = super.executeAsync(streamGraph);
      
         1. 使用SPI获取PipelineExecutorFactory的子类
      
         2. 通过factory获取PipelineExecutor
      
         3. 执行streamGraph，获取jobClient
      
            1. LocalExecutor
      
               1. 在本地启动最小集群MiniCluster
               2. 获取最小集群客户端clusterClient
               3. 提交任务
      
            2. remoteExecutor、KubernetesSessionClusterExecutor、YarnJobClusterExecutor和YarnSessionClusterExecutor都是AbstractSessionClusterExecutor的子类
      
               1. ```java
                  public RemoteExecutor() {
                      super(new StandaloneClientFactory());
                  }
                  //即：clusterClientFactory = new StandaloneClientFactory()
                  
                  public KubernetesSessionClusterExecutor() {
                      super(new KubernetesClusterClientFactory());
                  }
                  
                  public YarnJobClusterExecutor() {
                      super(new YarnClusterClientFactory());
                  }
                  
                  public YarnSessionClusterExecutor() {
                      super(new YarnClusterClientFactory());
                  }
                  ```
      
                  ```java
                  //也就是说remoteExecutor执行clusterClient就是new RestClusterClient
                  public ClusterClientProvider<StandaloneClusterId> retrieve(StandaloneClusterId standaloneClusterId) throws ClusterRetrieveException {
                      return () -> {
                          try {
                              return new RestClusterClient<>(config, standaloneClusterId);
                          } catch (Exception e) {
                              throw new RuntimeException("Couldn't retrieve standalone cluster", e);
                          }
                      };
                  }
                  ```
      
                  
      
            3. 
      
               
      
   7. 杂记：

      1. thenCompose：合并两个CompletableFuture【类似flatMap】
      2. thenAccept：转换CompletableFuture【类似于map】

2. sql job

### 3、任务是如何运行的

1. 概要：

   1. 任务会分解成多个operator，然后组成图
      1. operator的属性userFunction为用户自定义的Function，例如自定义的sourceFunction、sinkFunction
   2. 并行度相同的one to one operator可以合并为operatorChain，一个operatorChain是封装成task
   3. task运行时，获取headOperator，调用run()，实际上会调用userFunction.run()，然后调用ctx.collect()
   4. ctx.collect()调用后，会获取operatorChain的算子，依次执行operator

### 4、akka在flink中的使用



# flink 主流程

1. 启动任务

   1. 提交任务到JM的过程

      1. 查找clusterClient的过程
      2. clusterClient提交任务到 JM 的过程
         1. JM 接收到请求后做了什么
         2. 

   2. 图变化【 http://wuchong.me/blog/2016/05/03/flink-internals-overview/ 和 https://www.jianshu.com/p/652f454ca64c 】

      1. streamGraph
      2. jobGraph
      3. ExecutionGraph 
      4. physicalGraph

      ```markdown
      1、StreamGraph：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。
      2、JobGraph：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
      3、ExecutionGraph：JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是- JobGraph的并行化版本，是调度层最核心的数据结构。
      4、物理执行图：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构
      ```

      

   3. jar包的处理

      1. jobClient上传jar
      2. TM拉取jar【还是TM分发jar呢？】

   4. 从savepoint中启动的过程

2. 中断任务

   1. 触发checkpoint
      1. checkpoint触发逻辑
      2. checkpoint的存储
   2. 触发savepoint
      1. savepoint触发逻辑
      2. savepoint的存储
   3. checkpoint和savepoint的区别
   4. JM和TM的通信
   5. 

3. 



