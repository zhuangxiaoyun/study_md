# websocket日志



## 1、victini的实现

### 1、日志是通过定时器，每1s就输出到websocket中去【目标：/logger/userId】

```java
@Scheduled(cron = "0/1 * * * * ?")
    public void sendLogMsg() {
        while (!LoggerQueue.getInstance().isEmpty()) {
            LoggerMessage loggerMessage = LoggerQueue.getInstance().poll();
            msgTemplate.convertAndSend(WebSocketConfigurer.LOGGER_BROKER.concat("/").concat(String.valueOf(loggerMessage.getUserId())), loggerMessage);
        }
    }
```



### 2、LinkedBlockingQueue的api  ![img](https://img2018.cnblogs.com/blog/1212358/201904/1212358-20190422225207519-878576228.png) 

### 3、父子线程之间的日志

```java
FuturePartitionRun partitionRun = new FuturePartitionRun(partition, runParam);
FutureTask<Void> futureTask = new FutureTask<>(partitionRun);
RUN_EXECUTOR.submit(futureTask);
```

1. FuturePartitionRun继承了BaseSocketLogCallable
2. BaseSocketLogCallable在调用构造器的时候，会从当前线程中获取TraceContext.Value对象
3. BaseSocketLogCallable启动的时候会把实例属性TraceContext.Value的值放到当前线程中【与构造线程不在同一个线程中】

1. BaseSocketLogCallable

   ```java
   @Override
       public V call() throws Exception {
           TraceContext.set(value);
           V v = socketLogCall();
           TraceContext.remove();
           return v;
       }
   ```

   

2. FuturePartitionRun

   ```java
   @Override
           public Void socketLogCall() {
               RequestContextHolder.setRequestAttributes(requestAttributes);
               partition.run(runParam);
               return null;
           }
   ```

   1. RequestContextHolder.setRequestAttributes(requestAttributes);是为了在调用atlas接口时不会出问题？

3. SocketLogAppender

   ```java
   protected void append(ILoggingEvent event) {
           if (StringUtils.isNoneBlank(TraceContext.getTraceId())) {
               //被过滤掉
               if (LogMsgFilter.isFiltered(event.getLoggerName(), event.getFormattedMessage())) {
                   return;
               }
               LoggerMessage loggerMessage = LoggerMessage.builder()
                       .body(LogMsgFilter.replace(event.getFormattedMessage()))
                       .timestamp(DateFormat.getDateTimeInstance().format(new Date(event.getTimeStamp())))
                       .className(event.getLoggerName())
                       .threadName(event.getThreadName())
                       .level(event.getLevel().levelStr)
                       .userId(TraceContext.getUserId())
                       .traceId(TraceContext.getTraceId())
                       .traceName(TraceContext.getName())
                       .uniqueSign(TraceContext.getUniqueSign())
                       .requestTime(TraceContext.getRequestTime())
                       .build();
               try {
                   LoggerQueue.getInstance().push(loggerMessage);
               } catch (IllegalStateException e) {
                   log.warn("socket log 队列已满");
               }
           }
       }
   ```

   

### 4、总结

1. SocketLogAppender会将每行日志都存在LoggerQueue队列中【条件如下】
   1. 若TraceContext.getTraceId()不为空，即**TraceContext.begin()方法被调用过**
      1. 请求结束后，拦截器SocketLogInterceptor会调用TraceContext.remove();清空ThreadLocal的数据
   2. 需排除RequestLogAop和RedisIdWorker这两个类
2. SocketLogScheduler定时器会每一秒就将日志输出到websocket中
   1. **当LoggerQueue队列不空时**
3. 问题：
   1. 如果同时有多个分区的任务启动，怎么区分不同分区的日志啊？
      1. 每次请求的traceID不同
      2. uniqueSign设置为分区编码，前端通过分区编码分成多个tag展示
   2. 如果界面有两个请求都过来了，那么日志会混合在一起吧？
   3. 为啥要基于队列呢？LoggerQueue
      1. 现在这个模式，只有开启了websocket的线程的日志才会被收集，其他线程的日志是会被过滤的
         1. ThreadLocal
      2. 发送消息的目标地址是：/logger/userId
   4. 现在使用的是userid, 而不是使用sessionID，如果一个用户打开了多个浏览器界面，那么可能会收到两个浏览器各自的消息？但是有TracId?
4. 

## 2、websocket的原理

### [1、spring官网](https://docs.spring.io/spring/docs/5.1.13.RELEASE/spring-framework-reference/web.html#websocket-stomp-handle-annotations)

### 2、controller

1. [`@MessageMapping`](https://docs.spring.io/spring/docs/5.1.13.RELEASE/spring-framework-reference/web.html#websocket-stomp-message-mapping)
2. [`@SubscribeMapping`](https://docs.spring.io/spring/docs/5.1.13.RELEASE/spring-framework-reference/web.html#websocket-stomp-subscribe-mapping)
3. [`@MessageExceptionHandler`](https://docs.spring.io/spring/docs/5.1.13.RELEASE/spring-framework-reference/web.html#websocket-stomp-exception-handler)

### 3、服务端发送消息

1. 注解：@SendTo和@SendToUser

   1. sendTo ：是广播
   2. sendToUser ：是对应登录的用户名称，即登录成功后放置在Principal的name
      1. 前端：
         如果只监听服务器发给自己的信息，则在订阅的主题前需要加上/user
         spring webscoket能识别带”/user”的订阅路径并做出处理，例如，如果浏览器客户端，订阅了’/user/topic/greetings’这条路径，就会被spring websocket利用UserDestinationMessageHandler进行转化成”/topic/greetings-usererbgz2rq”,”usererbgz2rq”中，user是关键字，erbgz2rq是sessionid，这样子就把用户和订阅路径唯一的匹配起来了
      2. 后端：
         spring webscoket在使用@SendToUser广播消息的时候,
         /topic/greetings”会被UserDestinationMessageHandler转化成”/user/role1/topic/greetings”,role1是用户的登录帐号，这样子就把消息唯一的推送到请求者的订阅路径中去，这时候，如果一个帐号打开了多个浏览器窗口，也就是打开了多个websocket session通道，这时，spring webscoket默认会把消息推送到同一个帐号不同的session，你可以利用broadcast = false把避免推送到所有的session中
      3. ==注意==：@SendToUser在没有登录的情况下也是可以使用的，只是点对点的通信【即请求和响应】
         1. @SendToUser：在登录的情况下，是可以通过登录的Principal的name来发送响应的【不一定是请求方收到响应】，而且是同一个用户的多个websocket连接都可以收到响应，也可通过 `broadcast=false` 参数，只可请求方收到响应

2. SimpMessagingTemplate

   1. ```java
      /*
      直接在bean中注入SimpMessagingTemplate，即可使用
      However, you can also qualify it by its name (brokerMessagingTemplate), if another bean of the same type exists.
      */
      private SimpMessagingTemplate template;
      ```

### 4、ChannelInterceptor

1. ChannelInterceptor：
   Message被发送到线程池，在发送动作执行前（后）拦截，发生在当前线程
2. ExecutorChannelInterceptor：
   Message被发送到线程池后，在线程池持有的新线程中，在MessageHandler处理前（后）拦截。

### 5、WebSocket Scope

 As with any custom scope, Spring initializes a new `MyBean` instance the first time it is accessed from the controller and stores the instance in the WebSocket session attributes. The same instance is subsequently returned until the session ends. WebSocket-scoped beans have all Spring lifecycle methods invoked, as shown in the preceding examples. 

```
websocketScope中设置的属性，是当前websocketsession中特有的，在websocketSession会话过期后失效
```

```java
@Component
@Scope(scopeName = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MyBean {

    @PostConstruct
    public void init() {
        // Invoked after dependencies injected
    }

    // ...

    @PreDestroy
    public void destroy() {
        // Invoked when the WebSocket session ends
    }
}

@Controller
public class MyController {

    private final MyBean myBean;

    @Autowired
    public MyController(MyBean myBean) {
        this.myBean = myBean;
    }

    @MessageMapping("/action")
    public void handle() {
        // this.myBean from the current WebSocket session
    }
}
```

