

# easyScheduler

1、easyscheduler的基本工作流程
2、原理
3、扩展

## 1、后端部署文档

### 命令：

​	sh ./bin/stop-all.sh
​	sh ./bin/escheduler-daemon.sh start master-server

### 服务：

​	MasterServer         ----- master服务
​	WorkerServer         ----- worker服务
​	LoggerServer         ----- logger服务
​	ApiApplicationServer ----- api服务
​	AlertServer          ----- alert服务

### 目录结构：

​	├── bin : 基础服务启动脚本
​    ├── conf : 项目配置文件
​    |—— lib : 项目依赖jar包，包括各个模块jar和第三方jar
​    |—— script : 集群启动、停止和服务监控启停脚本
​    |—— sql : 项目依赖sql文件
​    |—— install.sh : 一键部署脚本



## 2、zookeeper存储的地址信息：

| 配置                                       | path                              | 描述                                                |
| ------------------------------------------ | --------------------------------- | --------------------------------------------------- |
| zookeeper.escheduler.lock.masters          | /escheduler/lock/masters          | master在zk中**锁**的位置                            |
| zookeeper.escheduler.lock.workers          | /escheduler/lock/workers          | worker在zk中**锁**的位置                            |
| zookeeper.escheduler.masters               | /escheduler/masters               | **master**服务在zk==注册==的位置                    |
| zookeeper.escheduler.workers               | /escheduler/workers               | **worker**服务在zk==注册==的位置                    |
| zookeeper.escheduler.dead.servers          | /escheduler/dead-servers          | 已停止的服务的注册位置                              |
| zookeeper.escheduler.lock.failover.masters | /escheduler/lock/failover/masters | master stop后，在zk中添加已停止的服务和发送告警的锁 |
| zookeeper.escheduler.lock.failover.workers | /escheduler/lock/failover/workers |                                                     |
| zookeeper.escheduler.root                  | /escheduler                       | /escheduler/tasks_queue队列在zk中存储位置           |

==zk中队列存储的路径名称==

```java
${processInstancePriority}_${processInstanceId}_${taskInstancePriority}_${taskId}

${processInstancePriority}_${processInstanceId}_${taskInstancePriority}_${taskId}_${task executed by ip1},${ip2}...
```

==优先级==

1. 从队列中获取任务的时候，先给任务排序
   1. 排序的策略：${processInstancePriority} ${processInstanceId} ${taskInstancePriority} ${taskId}

```
@Override
public int compare(String o1, String o2) {

    String s1 = o1;
    String s2 = o2;
    String[] s1Array = s1.split(Constants.UNDERLINE);
    if (s1Array.length > 4) {
        // warning: if this length > 5, need to be changed
        s1 = s1.substring(0, s1.lastIndexOf(Constants.UNDERLINE));
    }

    String[] s2Array = s2.split(Constants.UNDERLINE);
    if (s2Array.length > 4) {
        // warning: if this length > 5, need to be changed
        s2 = s2.substring(0, s2.lastIndexOf(Constants.UNDERLINE));
    }

    return s1.compareTo(s2);
}
```

master在zk中的操作

- 注册：cn.escheduler.server.zk.ZKMasterClient#registMaster()
  - **存储的地址**【最后一个是自增序列】：
    - /escheduler/masters/192.168.103.132_**0000000039**
  - **HeartBeatInfo**【master的心跳信息】：
    - host,port,cpuUsage,memoryUsage,createTime, lastHeartbeatTime
    - 心跳的作用是更新服务器在zk注册节点的信息，即：CPU和内存使用情况
- 监听master服务的启停和变更：cn.escheduler.server.zk.ZKMasterClient#listenerMaster()
- master和worker与zk断开连接后，**zkMasterClient**是会发送告警 【zkWorkerClient不会发送告警的】
  - zk的监听器会在监听到节点变更的时候发送告警
  - 在最后一台master挂了的时候，进程会在结束的时候发送告警

## 3、代码分析

### 1、master

cn.escheduler.server.master.runner.MasterExecThread#run

if (processInstance.isComplementData() &&  Flag.NO == processInstance.getIsSubProcess()){
	// sub process complement data
	executeComplementProcess();
}else{
	// execute flow
	executeProcess();
}	

#### master主要的线程：

简介：

1. MasterSchedulerThread只有一个，里面包含了MasterExecThread的线程池【最大100】
   1. 一个master最多有100个正在运行的流程实例
2. MasterExecThread包含了MasterTaskExecThread的线程池【最大20】
   1. 一个流程实例最多与20个正在运行的任务实例
   2. 流程实例执行结束后，任务实例线程池也会shutdown()

作用：

1. MasterExecThread处理processInstance
   1. 根据流程定义创建任务实例
   2. 更新processInstance的状态
2. MasterTaskExecThread处理taskInstance
   1. 将taskInstance提交到zk的队列中
   2. 更新taskInstance的状态



##### 1、MasterSchedulerThread

##### 1、MasterSchedulerThread

```
1、master如果运行中，则一直循环
2、获取锁
3、调用scanCommand
	3.1、获取一条Command【条件：流程定义的release_state是已上线，flag是可用】
	3.2、constructProcessInstance():组装
	3.3、判断processInstance是否为空
		3.3.1、processInstance不为空，且资源充足，则
			保存ProcessInstance
			setSubProcessParam
			删除Command

```

问题：==如果master挂了之后，ProcessInstance没有启动的任务应该不会启动了？==

##### 2、MasterExecThread

​	作用：处理processInstance，创建TaskInstance

​	注意：线程 **结束循环** 的点：**流程实例已停止 ：processInstance.IsProcessInstanceStop()**

​	流程： 

MasterExecThread的run()

```
1、processInstance为空，则线程结束
2、processInstance的状态为已完成，则线程结束
3、processInstance的类型是补数且不是子流程，则调用executeComplementProcess()
4、否则调用executeProcess()，改方法有三个子方法
	prepareProcess();
    runProcess();
    endProcess();
5、finally，执行：taskExecService.shutdown();
```



1. prepareProcess

2. runProcess

   ```markdown
   0、submit start node
   1、如果流程没结束，则一直循环
   2、判断流程是否已超时，超时则告警【只告警一次】（超时条件：设置了超时时间，且当前时间 - 启动时间超过了超时分钟）
   3、循环已经提交的任务线程【activeTaskNode】
   	3.1：判断线程是否执行完成，未执行完成则continue
   	3.2：activeTaskNode移除已完成的线程
   	3.3：从线程中获取任务实例对象，若任务实例对象为空，则this.taskFailedSubmit = true;【这个值与失败策略一起判断流程是否失败，之后updateProcessInstanceState的时候用于判断流程状态】
   	3.4：判断任务实例是否为成功，若成功则
   		completeTaskList.put(task.getName(), task);//完成列表中添加当前任务实例
           submitPostNode(task.getName());//执行下一个节点的任务
           continue
       3.5：任务实例状态是否失败【FAILURE或者NEED_FAULT_TOLERANCE】
       	若任务实例状态是NEED_FAULT_TOLERANCE，则添加到recoverToleranceFaultTaskList
       	若任务实例可重试，则添加到readyToSubmitTaskList
       	若任务实例不可重试，则添加到errorTaskList，completeTaskList
       		若流程的失败策略是停止，则调用kill()【activeTaskNode未完成的线程调用kill方法】
       			将taskExecThread的cancel设置为true
      		continue
   	3.6:任务实例是其他状态： other status stop/pause，则添加到completeTaskList
   4、recoverToleranceFaultTaskList不为空，则发送告警worker alert fault tolerance，清空recoverToleranceFaultTaskList
   5、errorTaskList不为空，则将completeTaskList中状态为PAUSE改为KILL
   6、判断是否可以提交任务【校验master是否有资源】，有资源则提交任务
   7、休眠一秒
   8、更新流程的状态
   ```

   kill()：任务实例执行失败，不可重试，且流程失败策略为停止，调用kill()

   ​	MasterBaseTaskExecThread.kill()，将taskExecThread的cancel设置为true

   

   

3. endProcess

##### 3、MasterTaskExecThread

​	作用：处理taskInstance，提交节点到队列tasks_queue

​	注意：线程 **结束循环** 的点：**任务实例的状态是完成**

​	流程：

```markdown
1、提交任务
2、如果任务实例未完成，则调用waitTaskQuit
	2.1：从数据库重新查询任务实例
	2.2：获取任务超时参数
	2.3：若任务实例有超时参数且超时策略是WARN、WARNFAILED，则需检查是否超时
	2.4：master是否还在运行中，则一直循环
		2.4.1：任务实例状态是cancel或者流程状态是READY_STOP，则调用cancelTaskInstance()
			2.4.1.1、往队列tasks_kill中添加节点【taskInstanceHost-taskInstanceId】
		2.4.2：任务实例状态是完成，则**跳出循环**
		2.4.3：checkTimeout为TRUE，则需检查是否超时
			2.4.3.1：判断是否超时：超时则发送任务超时告警
                条件一：当前时间 - taskInstance.getStartTime()
                条件二：StartTime > SubmitTime
		2.4.4：查询并替换线程对象taskInstance
		2.4.5：查询并替换线程对象processInstance
	
3、任务实例设置结束时间
4、更新任务实例
```



#### master的执行过程：

MasterServer implements **CommandLineRunner** 【CommandLineRunner、ApplicationRunner 接口是在容器启动成功后的最后一步回调（类似开机自启动）。】

cn.escheduler.server.master.MasterServer#run(java.lang.String...)
cn.escheduler.server.master.MasterServer#run(cn.escheduler.dao.ProcessDao)

```java
MasterSchedulerThread masterSchedulerThread = new MasterSchedulerThread(
					zkMasterClient,
					processDao,conf,
					masterExecThreadNum);
// submit master scheduler thread
masterSchedulerService.execute(masterSchedulerThread);
```
cn.escheduler.server.master.runner.**MasterSchedulerThread**#run

```java
masterExecService.execute(new MasterExecThread(processInstance));
```

cn.escheduler.server.master.runner.**MasterExecThread**#run
cn.escheduler.server.master.runner.MasterExecThread#executeProcess
cn.escheduler.server.master.runner.MasterExecThread#runProcess
cn.escheduler.server.master.runner.MasterExecThread#submitStandByTask
cn.escheduler.server.master.runner.MasterExecThread#submitTaskExec

```java
if(taskInstance.isSubProcess()){
    abstractExecThread = new SubProcessTaskExecThread(taskInstance, processInstance);
}else {
    abstractExecThread = new MasterTaskExecThread(taskInstance, processInstance);
}
Future<Boolean> future = taskExecService.submit(abstractExecThread);
```

cn.escheduler.server.master.runner.MasterBaseTaskExecThread#call
cn.escheduler.server.master.runner.MasterTaskExecThread#submitWaitComplete

```java
// 将任务提交到zk中
this.taskInstance = submit();
//
if(!this.taskInstance.getState().typeIsFinished()) {
    result = waitTaskQuit();
}
```

1、==task timeout warn告警信息保存在数据库中==

​	cn.escheduler.server.master.runner.MasterTaskExecThread#waitTaskQuit
​	cn.escheduler.dao.AlertDao#sendTaskTimeoutAlert

2、==将任务提交到zk中==

​	cn.escheduler.server.master.runner.MasterBaseTaskExecThread#submit

​	cn.escheduler.dao.ProcessDao#submitTask

​	cn.escheduler.dao.ProcessDao#submitTaskToQueue

```java
	taskQueue.add(SCHEDULER_TASKS_QUEUE, taskZkInfo(task));
```



```java
Future<Boolean> future = taskExecService.submit(abstractExecThread);activeTaskNode.putIfAbsent(abstractExecThread, future);return abstractExecThread.getTaskInstance();
```

### 2、worker

#### worker主要的线程

##### 1、FetchTaskThread

```markdown
1、检查系统资源和线程数
2、判断队列tasks_queue是否有节点
3、如果有节点，则获取锁
4、从队列tasks_queue中读取一定数量的节点【这个数量可通过参数配置】
	4.1、查询出所有的节点
	4.2、判断节点是否配置在此worker中执行【节点名称包含了执行worker的ip】
	4.3、排序后，取出前n个
		4.3.1、排序内容：${processInstancePriority}_${processInstanceId}_${taskInstancePriority}_${taskId}
		
5、从节点名称中获取taskInstanceId
6、检查该任务是否配置在此worker中执行
7、将任务节点从队列tasks_queue中移除
8、生成本地执行路径
	8.1、String execLocalPath = FileUtils.getProcessExecDir(processDefine.getProjectId(), processDefine.getId(), processInstance.getId(), taskInstance.getId());
9、查询此任务执行的租户信息
10、创建TaskScheduleThread线程执行任务

```

任务执行的路径executePath：

**/tmp/escheduler/exec**/**process**/processId/processDefineId/processInstanceId/taskInstanceId

任务执行日志路径logPath：

System.getProperty("user.dir") + /logs/processDefineId/processInstanceId/taskInstanceId.log

##### 2、TaskScheduleThread

```
1、更新任务的状态，启动时间，host,executePath,logPath
	1.1、如果任务类型是SQL,PROCEDURE，则executePath为空
2、获取流程实例中自定义的参数，放置到allParamMap
3、调用createProjectResFiles()，获取项目资源文件【并不是创建啊】
	3.1、根据任务类型，将任务参数json字符串转成子类参数对象
	3.2、获取子类参数对象的资源文件名称
4、从hdfs中将资源文件下载到executePath路径
5、taskProps设置信息
6、获取租户信息：租户信息为空，则任务设置为失败
7、taskProps中设置超时信息
	7.1、判断任务是否配置了超时参数，若未配置则任务的超时时间为Integer.MAX_VALUE
	7.2、设置taskProps的超时策略
	7.3、设置taskProps的超时时间
		7.3.1、如果超时策略是告警：则任务的超时时间为Integer.MAX_VALUE【master会处理】
		7.3.2、如果超时策略是“失败”或者“告警且失败”：则设置超时时间
	
8、创建TaskLogger
	taskAppId=TASK_processDefineId_processInstanceId_taskInstanceId
9、创建task对象
10、task初始化
11、task执行
12、判断执行的状态


```

##### 3、killProcessThread

```markdown
1、从队列tasks_kill获取所有节点taskInfoSet
2、如果worker没有停止，则一直循环
3、遍历taskInfoSet
	3.1、判断任务的host是否是当前worker【即这个任务是不是分配给当前worker处理的】
	3.2、查询taskInstance，ProcessInstance
	3.3、判断 taskInstance的类型
		3.3.1、如果是依赖节点，则将taskInstance的状态更新为KILL
		3.3.2、如果是flink节点，则cancel flink任务
    3.4、其他类型的任务，则调用ProcessUtils.kill(taskInstance);
    	3.4.1、如果taskInstance.getPid()为0，则不做任何处理【就是任务还没有启动】
    	3.4.2、执行shell命令kill
    	3.4.3、生成sh文件，kill yarn任务
    3.5、将taskInfo移除队列tasks_kill
	
```



#### 代码

cn.escheduler.server.worker.WorkerServer#run

```java
FetchTaskThread fetchTaskThread = new FetchTaskThread(taskNum,zkWorkerClient, processDao,conf, taskQueue);

        // submit fetch task thread
        fetchTaskExecutorService.execute(fetchTaskThread);
```

cn.escheduler.server.worker.runner.FetchTaskThread#run

```java
//获取zk的任务列表
List<String> tasksQueueList = taskQueue.getAllTasks(Constants.SCHEDULER_TASKS_QUEUE);
//获取zk锁
String zNodeLockPath = zkWorkerClient.getWorkerLockPath();
                    mutex = new InterProcessMutex(zkWorkerClient.getZkClient(), zNodeLockPath);
                    mutex.acquire();

//获取任务实例id
String[] taskStringArray = taskQueueStr.split(Constants.UNDERLINE);
                                String taskInstIdStr = taskStringArray[3];

//提交任务
workerExecService.submit(new TaskScheduleThread(taskInstance, processDao));
```

cn.escheduler.server.worker.runner.TaskScheduleThread#call



### 3、告警信息

#### 分类

1、Process success/failed：sendProcessAlert

2、Process Timeout：sendProcessTimeoutAlert

3、Task Timeout：sendTaskTimeoutAlert

4、Worker Tolerance Fault：sendAlertWorkerToleranceFault

5、sendServerStopedAlert：sendServerStopedAlert

注意：

==只有sendServerStopedAlert是放在alertDao，其他的都是放在alertManager==



### 4、流程上线

### 5、问题：

定时的时间是怎么判断的？什么时候开始执行，什么时候结束呢？



1. quartz是怎么起效的？在流程实例创建的时候吗?
2. worker工作的时候获取zk的队列是先加锁了？处理完成后队列是怎么出列的？
   1. 查询task_queue是否有数据
   2. 如果有数据，则尝试获取锁
   3. 获取锁后，从task_queue中获取3条任务
   4. 校验任务是否存在
   5. 任务存在，则从task_queue中删除当前任务信息
   6. 从线程池中获取线程并执行任务
3. master和worker与zk断开连接后，zk是会发送告警吗？ 
   1. zk的监听器会在监听到节点变更的时候发送告警，
   2. 在最后一台master挂了的时候，进程会在结束的时候发送告警
4. 流程的上线
   1. quartz添加一个job
      1. job的类型是ProcessScheduleJob.class
      2. jobDataMap中包含projectId、scheduleId和schedule
   2. 当定时时间到了，ProcessScheduleJob的逻辑
      1. 
5. 调度计划
   1. 同一流程串行执行，不同流程并行执行
      1. 没有堆积，直接抛弃，没有告警
         1. 流程实例创建的过程：
            1. 先判断是否可以创建
               1. 是否存在未结束的ProcessInstance，存在则什么都不做
               2. 是否存在未执行的Command，存在则什么都不做
            2. 先创建Command：在ProcessScheduleJob中创建
            3. 再创建ProcessInstance：在MasterSchedulerThread中创建
            4. 然后创建TaskInstance：在MasterExecThread中创建
   2. 资源监控
      1. master资源监控
      2. worker资源监控
   3. 任务停止
      1. 流程停止
      2. 队列移除
      3. 流程实例停止
      4. 任务实例停止
   4. 优先级管理
      1. 是否有预留空间？





+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++



schedulerController在上线定时器的时候会，在quartz中添加/更新任务，以添加任务为例：

```
scheduler.addJob(jobDetail, false, true);
```



master启动的时候会启动quartz的调度器：

```
scheduler.start();
```





==================文档==========================

队列是在执行spark、mapreduce等程序，需要用到“队列”参数时使用的。
租户对应的是Linux的用户，用于worker提交作业所使用的用户。如果linux没有这个用户，worker会在执行脚本的时候创建这个用户。



下线工作流定义的时候，要先将定时管理中的定时任务下线，这样才能成功下线工作流定义

 

```
processDefinitionId: 52
scheduleTime: 2020-03-08 00:00:00,2020-03-08 00:00:00
failureStrategy: END
warningType: NONE
warningGroupId: 4
execType: COMPLEMENT_DATA
startNodeList: a
taskDependType: TASK_POST
runMode: RUN_MODE_SERIAL
processInstancePriority: HIGH
receivers: 
receiversCc: 

```

```
processDefinitionId: 52
scheduleTime: 
failureStrategy: END
warningType: SUCCESS
warningGroupId: 4
execType: 
startNodeList: a
taskDependType: TASK_POST
runMode: RUN_MODE_SERIAL
processInstancePriority: HIGH
receivers: 
receiversCc: 
workerGroupId: 
```


