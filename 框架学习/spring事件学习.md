[TOC]

spring支持事件机制，可在不引用消息队列的情况下完成事件通知；并且可以解耦代码逻辑

## 1. 自定义event类

需继承继承ApplicationEvent类

```java
public class MasterEvent extends ApplicationEvent {
    private String username;
    public MasterEvent(Object source) {
        super(source);
    }

    public MasterEvent(Object source, String username) {
        super(source);
        this.username = username;
    }

    public String getUsername() {
        return username;
    }
}
```

## 2. 事件发布

发布方服务需继承ApplicationEventPublisherAware接口或直接注入ApplicationEventPublisher，

发布事件applicationEventPublisher.publishEvent(new MasterEvent(this,username));

```java
@Service
@Slf4j
public class MasterService implements ApplicationEventPublisherAware {
    private ApplicationEventPublisher applicationEventPublisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    /**
     * spring事件默认同步执行
     * 事件发布后只要有一个出现异常则之后的其他监听器都不会执行
     * @param username
     */
    public void register(String username){
        log.info("执行master-register");
        applicationEventPublisher.publishEvent(new MasterEvent(this,username));
        log.info("master-register结束");
    }
}
///////////////////////////////////////////////////
@Service
@Slf4j
public class Master2Service {
    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    public void register(String username) {
        log.info("执行master-register");
        applicationEventPublisher.publishEvent(new MasterEvent(this,username));
        log.info("master-register结束");
    }
}
```

## 3. 事件监听

两种方式实现，一种通过注解@EventListener，一种通过接口ApplicationListener<Event>，**推荐注解**

```java
@Service
@Slf4j
@Async
public class Slave1Service implements ApplicationListener<MasterEvent> {
    @Override
    public void onApplicationEvent(MasterEvent event) {
        log.info("继承方式完成salve1:"+event.getUsername());
    }
}

@Service
@Slf4j
public class Slave2Service {

    /**
     * 重试5次，每次延时3秒
     * backoff
     *  delay：如果不设置的话默认是1秒
     *  maxDelay：最大重试等待时间
     *  multiplier：用于计算下一个延迟时间的乘数(大于0生效)
     *  random：随机重试等待时间(一般不用)
     * @param event
     */
    @EventListener
    @Retryable(maxAttempts=5,backoff = @Backoff(delay = 3000))
    @Async
    public void slave(MasterEvent event){
        log.info("注解方式完成salve2:"+event.getUsername());
        throw new RuntimeException("手动异常");
    }
    
    /**
     * 内部类能使用异步注解，会报错
     */
    @Component
    private static class MyListener {
        private ApplicationContext applicationContext;

        @EventListener(classes = ContextRefreshedEvent.class)
        public void applicationContextEvent(ContextRefreshedEvent event) {
            applicationContext = event.getApplicationContext();
        }

        @EventListener(classes = MasterEvent.class)
        public void slave3(MasterEvent event) {
            log.info("注解内部类方式完成salve3:" + event.getSource().toString());
        }
    }
}
```

## 4. 重试机制（不重要）

修改pom文件，启动类或@configation修饰的类上加@EnableRetry，监听器加入@Retryable，貌似依赖之间有冲突

```xml
<dependency>
     <groupId>org.springframework.retry</groupId>
     <artifactId>spring-retry</artifactId>
</dependency>
```

## 5. 开启异步

启动类或@configation修饰的类上加@EnableAsync，监听方法上加@Async

@Async可接线程池bean的名字，可使用该bean配置的线程池

```java
//@Async("asyncExecutor")可使用该线程池
@Configuration
@EnableAsync
public class EventAsyncConfig {
    @Bean
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        // 核心线程数        
        taskExecutor.setCorePoolSize(3);
        // 最大线程数
        taskExecutor.setMaxPoolSize(10);
        // 队列大小                                      
        taskExecutor.setQueueCapacity(15);
        // 线程名的前缀 
        taskExecutor.setThreadNamePrefix("event-thread-");
        taskExecutor.initialize();
        return taskExecutor;
    }
}
```

事件默认**同步**，同步模式若监听器抛出异常则**其余监听器及主方法**都不会继续执行，异步模式测试发现会抛出异常，多个监听器可以自由选择同步和异步

```bash
2021-01-29 10:44:25.282  INFO 22296 --- [           main] c.e.s.service.master.MasterService       : 执行master-register
2021-01-29 10:44:25.288  INFO 22296 --- [           main] c.e.s.service.master.MasterService       : master-register结束
2021-01-29 10:44:25.288  INFO 22296 --- [         task-2] c.e.s.service.slave.Slave1Service        : 继承方式完成salve1:Alice
2021-01-29 10:44:25.296  INFO 22296 --- [         task-1] c.e.s.service.slave.Slave2Service        : 注解方式完成salve2:Alice

2021-01-29 10:44:25.302 ERROR 22296 --- [         task-1] .a.i.SimpleAsyncUncaughtExceptionHandler : Unexpected exception occurred invoking async method: public void com.example.springevent.service.slave.Slave2Service.slave(com.example.springevent.service.MasterEvent)

java.lang.RuntimeException: 手动异常
```

## 6. 事务（扩展）

方法中修改数据后，在方法内**异步**执行任务，任务内发现数据未修改！

```java
@SneakyThrows
    @Override
    @Transactional(rollbackFor = Exception.class)
    public TUser add(TUser tUser) {
        tUserMapper.insert(tUser);
        new Thread(() -> {
            Integer count = tUserMapper.selectCount(new QueryWrapper<TUser>().lambda().eq(TUser::getId, tUser.getId()));
            System.out.println("count = " + count);//count一直为0 刚插入的数据，这里却查不到
        }).start();
        Thread.sleep(2000);
        return tUser;
    }
```

如何让事务提交后再异步执行？有两种方式

- 使用spring event，开启异步@EnableAsync，监听器加入@Async和@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)，phase根据事务状态决定是否执行

- 使用TransactionSynchronizationManager.registerSynchronization()方法，该方法默认同步执行，但会在方法执行完后再执行；若想异步可以加个线程池

  ```java
  TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
  //在事务提交之后执行的代码块（方法）  此处使用TransactionSynchronizationAdapter，其实在Spring5后直接使用接口也很方便了
      @Override
      public void afterCommit() {
              Integer count = tUserMapper.selectCount(new QueryWrapper<TUser>().lambda().eq(TUser::getId, tUser.getId()));log.info("count="+count);
      }
  });
  ```

>参考：https://cloud.tencent.com/developer/article/1497715

## 7. bean初始化（扩展）

> https://www.toutiao.com/i6809077726981390861/?tt_from=weixin&utm_campaign=client_share&wxshare_count=1&timestamp=1611894076&app=news_article&utm_source=weixin&utm_medium=toutiao_android&use_new_style=1&req_id=2021012912211601012903513719037FDF&share_token=6810c6e3-761b-45e2-a9bd-e3e0d7080514&group_id=6809077726981390861