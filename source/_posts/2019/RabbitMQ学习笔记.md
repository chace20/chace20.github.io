---
title: RabbitMQ学习笔记
date: 2018-09-13 19:38:56
tags: rabbitMQ
---

## 1. 安装
```
apt install rabbitmq-server
```

## 2. 基本原理
一篇比较好的原理介绍文章：[消息队列之 RabbitMQ](https://www.jianshu.com/p/79ca08116d57)

![Rabbit架构图](https://upload-images.jianshu.io/upload_images/5015984-367dd717d89ae5db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/554)

![路由过程](https://upload-images.jianshu.io/upload_images/5015984-7fd73af768f28704.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/484)

Exchange分发消息时根据类型的不同分发策略有区别，目前共四种类型：direct、fanout、topic、headers 。headers 匹配 AMQP 消息的 header 而不是路由键，此外 headers 交换器和 direct 交换器完全一致，但性能差很多，目前几乎用不到了。

direct：发送到路由键完全匹配的队列
fanout: 发送到所有队列
topic: 基于模式，比如route key的usa.news, usa.weather都发送到binding key的usa.#

也可以启动集群，用NODE_NAME来区分。内存节点和磁盘节点，必须有一个磁盘节点。

## 3. 工作队列

带ACK的消息
``` java
// consume的时候发送ack=false
boolean autoAck = false;
channel.basicConsume(QUEUE_NAME, autoAck, consumer);

// 当完成任务后发送ACK
channel.basicAck(envelope.getDeliveryTag(), false);

# 以下命令查看未ack的消息
rabbitmqctl list_queues name message_ready message_unacknowledged
```

持久化
``` java
channel.queueDeclare(QUEUE_NAME, durable, false, false, null);
channel.basicPublish("", QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
```

公平调度
``` java
// 如果work没有发送ACK，则不再发送新消息
int prefetchCount = 1;
channel.basicQos(1);
```

## 4. 发布/订阅模式
Exchange
``` java
// 声明一个exchange，有direct、fanout、topic、headers四种模式
channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
// publish的时候指定exchange name即可
channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
```

Bindings
``` java
// 获取一个临时队列
String queueName = channel.queueDeclare().getQueue();
// 绑定队列到exchange上
channel.queueBind(queueName, EXCHANGE_NAME, "");

# 以下命令查看绑定
rabbitmqctl list_bindings
```

## 5. 路由
``` java
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
// 指定routing key和binding key即可
channel.basicPublish(EXCHANGE_NAME, "error", null, message.getBytes());
channel.queueBind(queueName, EXCHANGE_NAME, "error");
```

## 6. topic
``` java
* 代表一个word
# 代表一个或多个word
使用上面的两个符号来进行匹配，中间用"."隔开。比如"*.kern.error"。

如果bindingKey="#"，那就跟fanout模式一样；
如果bindingKey不包含*或#，那就跟direct一样。

channel.exchangeDeclare(EXCHANGE_NAME, "topic");
channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
channel.queueBind(queueName, EXCHANGE_NAME, bingingKey);
```

## 7. RPC
用replyTo和correlationId在两者之间建立联系
![RPC](https://www.rabbitmq.com/img/tutorials/python-six.png)
When the Client starts up, it creates an anonymous exclusive callback queue.
For an RPC request, the Client sends a message with two properties: replyTo, which is set to the callback queue and correlationId, which is set to a unique value for every request.
The request is sent to an rpc_queue queue.
The RPC worker (aka: server) is waiting for requests on that queue. When a request appears, it does the job and sends a message with the result back to the Client, using the queue from the  replyTo field.
The client waits for data on the callback queue. When a message appears, it checks the correlationId property. If it matches the value from the request it returns the response to the application.
