# quartz

## 1、组件

1. Job 

   表示一个工作，要执行的具体内容。此接口中只有一个方法，如下：

   ```
   void execute(JobExecutionContext context) 
   ```

2. **JobDetail** 表示一个具体的可执行的调度程序，Job 是这个可执行程调度程序所要执行的内容，另外 JobDetail 还包含了这个任务调度的方案和策略。 

3. **Trigger** 代表一个调度参数的配置，什么时候去调。 

4. **Scheduler** 代表一个调度容器，一个调度容器中可以注册多个 JobDetail 和 Trigger。当 Trigger 与 JobDetail 组合，就可以被 Scheduler 容器调度了。 

![1572059532003](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1572059532003.png)

### 1、job和jobDetail

- job
- job实例在quartz中的生命周期：每次调度器执行job时，它在调用execute方法前会**创建**一个新的job实例，当调用完成后，关联的job对象实例会被释放，释放的实例会被垃圾回收机制回收。
- jobDetail

### 2、jobExecutionContext

![1572061065231](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1572061065231.png)

### 3、JobDataMap

```java
//设置JobDataMap：在任务调度时设置
        JobDetail jobDetail = JobBuilder.newJob(MyJob.class)
            .withIdentity("job1", "group1")
            .usingJobData("userId","001")
            .usingJobData("userName","testName")
            .build();
//获取JobDataMap：在job类中获取
方法一：通过jobExecutionContext获取
		JobDetail jobDetail = context.getJobDetail();
        JobDataMap jobDataMap = jobDetail.getJobDataMap();
        String userId = jobDataMap.getString("userId");
方法二：在job类中定义对应的属性，并添加setter方法
    注意：方法二中若trigger中添加的JobDataMap的key与jobDetail中定义的相同，则trigger会覆盖jobDetail中的key
//trigger也可以设置JobDataMap
```

![1572070475278](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1572070475278.png)

### 4、有状态的job和无状态的job

- **@DisallowConcurrentExecution**：将该注解加到job类上，告诉Quartz不要并发地执行同一个job定义（这里指特定的job类）的多个实例 

- **@PersistJobDataAfterExecution**：将该注解加在job类上，告诉Quartz在成功执行了job类的execute方法后（没有发生任何异常），更新JobDetail中JobDataMap的数据，使得该job（即JobDetail）在下一次执行的时候，JobDataMap中是更新后的数据，而不是更新前的旧数据。 

### 5、trigger

**一个jobDetail可以有多个trigger，一个trigger只能对于一个jobDetail，即jobDetail与trigger是一对多的关系**

trigger的公共属性有：

- jobKey属性：当trigger触发时被执行的job的身份；
- startTime属性：设置trigger第一次触发的时间；该属性的值是java.util.Date类型，表示某个指定的时间点；有些类型的trigger，会在设置的startTime时立即触发，有些类型的trigger，表示其触发是在startTime之后开始生效。比如，现在是1月份，你设置了一个trigger–“在每个月的第5天执行”，然后你将startTime属性设置为4月1号，则该trigger第一次触发会是在几个月以后了(即4月5号)。
- endTime属性：表示trigger失效的时间点。比如，”每月第5天执行”的trigger，如果其endTime是7月1号，则其最后一次执行时间是6月5号。

### 6、Simple Trigger

 SimpleTrigger可以满足的调度需求是：

​		在具体的时间点执行一次，

​		或者在具体的时间点执行，并且以指定的间隔重复执行若干次。 

### 7、CronTrigger

#### 1、Cron Expressions

Cron-Expressions用于配置CronTrigger的实例。Cron Expressions是由七个子表达式组成的字符串，用于描述日程表的各个细节。这些子表达式用空格分隔，并表示：

1. Seconds

2. Minutes

3. Hours

4. Day-of-Month

5. Month

6. Day-of-Week

7. Year (optional field)

   **Cron表达式的格式：秒 分 时 日 月 周 年(可选)。**

| 字段名       允许的值          允许的特殊字符    |
| ------------------------------------------------ |
| 秒          0-59             , - * /             |
| 分          0-59             , - * /             |
| 小时         0-23             , - * /            |
| 日          1-31              , - * ? / L W C    |
| 月          1-12 or JAN-DEC    , - * /           |
| 周几         1-7 or SUN-SAT    , - * ? / L C #   |
| 年(可选字段)  empty            1970-2099 , - * / |

\* ：代表所有可能的值。因此，“*”在Month中表示每个月，在Day-of-Month中表示每天，在Hours表示每小时

\- ：表示指定范围。

**,** ：表示列出枚举值。例如：在Minutes子表达式中，“5,20”表示在5分钟和20分钟触发。

**/** ：被用于指定增量。例如：在Minutes子表达式中，“0/15”表示从0分钟开始，每15分钟执行一次。"3/20"表示从第三分钟开始，每20分钟执行一次。和"3,23,43"（表示第3，23，43分钟触发）的含义一样。

**?** ：用在Day-of-Month和Day-of-Week中，指“没有具体的值”。当两个子表达式其中一个被指定了值以后，为了避免冲突，需要将另外一个的值设为“?”。例如：想在每月20日触发调度，不管20号是星期几，只能用如下写法：0 0 0 20 * ?，其中最后以为只能用“?”，而不能用“*”。

**L** ：用在day-of-month和day-of-week字串中。它是单词“last”的缩写。它在两个子表达式中的含义是不同的。

**W** ：“Weekday”的缩写。只能用在day-of-month字段。用来描叙最接近指定天的工作日（周一到周五）。例如：在day-of-month字段用“15W”指“最接近这个月第15天的工作日”，即如果这个月第15天是周六，那么触发器将会在这个月第14天即周五触发；如果这个月第15天是周日，那么触发器将会在这个月第 16天即周一触发；如果这个月第15天是周二，那么就在触发器这天触发。注意一点：这个用法只会在当前月计算值，不会越过当前月。“W”字符仅能在 day-of-month指明一天，不能是一个范围或列表。也可以用“LW”来指定这个月的最后一个工作日，即最后一个星期五。

\# ：只能用在day-of-week字段。用来指定这个月的第几个周几。例：在day-of-week字段用"6#3" or "FRI#3"指这个月第3个周五（6指周五，3指第3个）。如果指定的日期不存在，触发器就不会触发。

### 8、监听器

#### 1、JobListeners

```java
public interface JobListener {

    public String getName();

    public void jobToBeExecuted(JobExecutionContext context);

    public void jobExecutionVetoed(JobExecutionContext context);

    public void jobWasExecuted(JobExecutionContext context,
            JobExecutionException jobException);

}
```



#### 2、TriggerListeners

```java
public interface TriggerListener {

    public String getName();

    public void triggerFired(Trigger trigger, JobExecutionContext context);

    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context);

    public void triggerMisfired(Trigger trigger);

    public void triggerComplete(Trigger trigger, JobExecutionContext context,
            int triggerInstructionCode);
}
```

注册监听器：

```java
//局部监听器
scheduler.getListenerManager()).addJobListener(myJobListener，KeyMatcher.jobKeyEquals(new JobKey("myJobName"，"myJobGroup")));

//添加对两个特定组的所有job感兴趣的JobListener：
scheduler.getListenerManager().addJobListener(myJobListener, 		            or(jobGroupEquals("myJobGroup"), jobGroupEquals("yourGroup")));

//全局监听器
scheduler.getListenerManager().addJobListener(myJobListener, allJobs());
```



#### 3、SchedulerListeners

```java
public interface SchedulerListener {

    public void jobScheduled(Trigger trigger);

    public void jobUnscheduled(String triggerName, String triggerGroup);

    public void triggerFinalized(Trigger trigger);

    public void triggersPaused(String triggerName, String triggerGroup);

    public void triggersResumed(String triggerName, String triggerGroup);

    public void jobsPaused(String jobName, String jobGroup);

    public void jobsResumed(String jobName, String jobGroup);

    public void schedulerError(String msg, SchedulerException cause);

    public void schedulerStarted();

    public void schedulerInStandbyMode();

    public void schedulerShutdown();

    public void schedulingDataCleared();
}
```



```java
//添加SchedulerListener
scheduler.getListenerManager().addSchedulerListener(mySchedListener);

//删除SchedulerListener
scheduler.getListenerManager().removeSchedulerListener(mySchedListener);

```

## 2、源码

#### 1、流程图：

![1572158581584](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1572158581584.png)

#### 2、trigger

![1572159596227](C:\Users\zxy\AppData\Roaming\Typora\typora-user-images\1572159596227.png)

