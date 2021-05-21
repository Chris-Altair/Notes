rabbitmq消息确认机制

AcknowledgeMode.NONE：不确认
AcknowledgeMode.AUTO：自动确认
AcknowledgeMode.MANUAL：手动确认

手动确认时，生产者发送消息，消费者处理消息，处理成功则需调用basicAck发送确认，mq收到确认后才会删除该消息；

处理失败则应该调用basicReject(单条)或basicNack(批量)，可选择消息是否重新入队，是则重入，否则删除消息

rabbitmq默认消息不会持久化，重启则丢失

参考：https://www.huaweicloud.com/articles/9e6497a48dafd4e449386f8037510a78.html

@RabbitListener重试机制是针对方法抛异常，若方法抛出异常则按yml文件配置的方式重试（无论手动、自动）

当然若选择手动模式，并且最后basicReject选择重新入队，则将陷入死循环。

