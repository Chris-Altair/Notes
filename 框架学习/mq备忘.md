## 1.rabbitmq

rabbitmq消息确认机制

AcknowledgeMode.NONE：不确认
AcknowledgeMode.AUTO：自动确认
AcknowledgeMode.MANUAL：手动确认

手动确认时，生产者发送消息，消费者处理消息，处理成功则需调用basicAck发送确认，mq收到确认后才会删除该消息；

处理失败则应该调用basicReject(单条)或basicNack(批量)，可选择消息是否重新入队，是则重入，否则删除消息

rabbitmq默认消息不会持久化，重启则丢失

参考：https://www.huaweicloud.com/articles/9e6497a48dafd4e449386f8037510a78.html

@RabbitListener重试机制是针对方法抛异常，若方法抛出异常则按yml文件配置的方式重试（无论手动、自动）

当然若选择手动模式，并且catch掉所有异常，finally调用basicReject选择重新入队，则将陷入死循环。

```yaml
#参考springboot配置rabbitmq
spring:
  #消息队列
  rabbitmq:
    host: 121.196.162.145
    port: 5672
    virtual-host: /
    username: admin
    password: 123456789
    connection-timeout: 15000
    #CORRELATED值是发布消息成功到交换器后会触发回调方法
    publisher-confirm-type: CORRELATED
    #publisher-return模式可以在消息没有被路由到指定的queue时将消息返回，而不是丢弃
    publisher-returns: true
    publisher-retry: 3
    publisher-time: 30
    cache:
      connection:
        mode: channel
        size: 5
      channel:
        size: 10
    template:
      mandatory: true
      retry:
        enabled: true
    listener:
      simple:
        retry:
          enabled: true #开启重试机制
          max-attempts: 3 #消息最多消费3次
          initial-interval: 1s # 消息多次消费的间隔1秒
          max-interval: 16s #最大重试时间间隔
          multiplier: 2 #应用于上一重试间隔的乘数
        acknowledge-mode: manual
        concurrency: 2 #每个listener在初始化的时候设置的并发消费者的个数
        prefetch: 5 #每次从一次性从broker里面取的待消费的消息的个数
        default-requeue-rejected: false
```

## 2.rocketmq

官方文档：https://rocketmq.apache.org/zh/docs/featureBehavior/04transactionmessage

事务消息： 半事务消息 ，二次提交机制，保证生产者本地事务与消息发送状态的一致性。

保证若sender本地事务提交则保证消息发送成功（消息从事务存储系统提交到普通存储系统），等待下游消费；若sender本地事务回滚，则消息流程终止，不会被下游消费；若本地事务未知，则不处理，等待事务消息回查

（若服务端未收到发送者提交的二次确认结果，或服务端收到的二次确认结果为Unknown未知状态，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。）

使用注意：

- 创建事务topic
- 发送事务消息前，需要开启事务并关联本地的事务执行 
- 为保证事务一致性，在构建生产者时，必须设置事务检查器和预绑定事务消息发送的主题列表，客户端内置的事务检查器会对绑定的事务主题做异常状态恢复。

一般使用方式：参考：https://zhuanlan.zhihu.com/p/249233648

1. 实现自定义【本地事务执行器】，执行成功则提交本地事务并返回COMMIT，否则回滚本地事务并返回ROLLBACK
2. 实现自定义【本地事务检查器】，一般通过id检查
3. 消息生产者绑定【本地事务执行器】和【本地事务检查器】
4. 消息生产者发送事务消息
