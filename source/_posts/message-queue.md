---
title: Message queue
date: 2021-03-04 10:15:47
tags: 
    - mq
---


对比不同的消息队列中间件的消息语义

## 通行概念解释

- Consumer Group

通常指一类消费者的多个消费者实例，同一个Consumer Group的多个实例消费同一个队列，不同Consumer Group并行消费同一个队列不互相影响。不同的消息队列对Consumer Group的定义方式有差别，但是语义上基本一致。


## nsq


- 订阅模型

![nsq-topic](/images/nsq-topic.gif)

消费者订阅一个topic需要指定channel，所有订阅同一个topic/channel的消费者实例，构成一个Consumer Group。不同的Consumer Group需要订阅不同的channel。

- 推拉模型

nsq采取消息推的模型，并且，消费者实例通过通告RDY告知nsq server同时可消费的消息最大值来控制消息推送速度。

- 确认模型

消费者对每一个消息单独确认，或者REQUEUE。

- 并发模型

一个channel对Consumer Group的多个消费者实例，消息并行通知。

- 有序性

不保证有序性，因为一个channel对多个消费者实例会并行通知，并且消息单独确认(or REQUEUE)设计，nsq不保证消息的有序性。

## kafka


- 订阅模型

![kafka-topic](/images/kafka-topic.png)

每一个消费者需要指定Consumer Group ID，ID相同的消费者实例构成一个Consumer Group。Consumer Group可以对一个topic进行订阅，kafka内部会维护不同的Consumer Group的消费状态(offset)，互相不影响。

- 推拉模型

kafka采取消费者拉取模型，由消费者决定一次拉取多少个

- 确认机制

offset的批量确认机制，消费者自行调整offset对已经消费的消息进行确认

- 并发模型

kafka通过partion支持topic的多消费者并发。topic进行多个partion，同一个partion只能同时由一个消费者实例进行消费，多个partion由多个消费者实例并行消费。partion和消费者是N:1的关系，一个partion最多有一个消费者，一个消费者同时可以处理多个partion。

- 有序性

kafka保证topic的每一个partition是顺序的，但是多个partion之间的消息不保证有序。

## pulsar
