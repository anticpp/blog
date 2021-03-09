---
title: Message queue
date: 2021-03-04 10:15:47
tags: 
    - mq
---

不同的消息队列中间件对比

# 消息语义

## 通行概念解释

- Consumer Group

通常指一个消费者的多个消费者实例，同一个Consumer Group的多个实例消费同一个队列，不同Consumer Group并行消费(订阅)同一个队列不互相影响。不同的消息队列对Consumer Group的定义方式有差别，但是语义上基本一致。

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

pulsar在Producer和Consumer端支持不同的模式，并且Producer和Consumer的模式互相是独立的，通过组合这些模式可以达到非常丰富的消息模型(例如可以组合出nsq/kafka相关的模型)。先说明一下这些模式

- Subscription mode(Consumer)

![pulsar-topic](/images/pulsar-topic.png)


Exclusive: 一个topic对于一个Consumer Group同时只能有一个消费者实例进行消费，如果有多个消费者实例，多余的消费者实例订阅的时候会失败。

Failover: 相比于Exclusive模式，可以有多个消费者实例同时订阅一个topic。但是Pulsar会选择其中一个消费者实例做为master(这个粒度可以是topic，也可以是每个topic的partition，取决于topic本身是non-partitioned还是partitioned，这个后面Producer模式会涉及)。如果Master断开了，Pulsar会选择下一个消费者实例做为Master。

Shared: 这个是最好理解的方式，一个Consumer Group的多个消费者实例共同消费一个topic，消息分发在Consumer Group多个实例间round robin模式负载均衡。这个模式跟`nsq`的订阅模型一样。

Key\_Shared: 跟Shared模式类似，只是消息会根据消息的Key进行消费者选择，同一个Key的消息只会由一个消费者实例进行消费。

- Partioned topics/Routing mode(Producer)

![pulsar-partition](/images/pulsar-partition.png)

跟kafka一样，一个topic会进行多个partition，每一个partition的消息在队列里面也是有序的。partition会分配给一个broker，这样做的目的主要考量是topic在多个broker之间做系统负载均衡。一个topic的消息发布的时候，会分发到不同的partition，这个分发策略叫Routing mode，有以下几种

RoundRobinPartition: 消息通过Round robin的方式轮询发布到多个partition。如果消息有指定Key，消息会根据Key的hash规则找到指定的partition发布。

SinglePartition: 随机找到一个partition，把所有消息都发布到这个partition。如果消息有指定Key，消息会根据Key的hash规则找到指定的partition发布。

CustomPartition: 可以自定义每一个消息的发布routing规则，具体客户sdk有相应的接口。


接下来我们说明一下几个消息语义

- 推拉模型

跟kafka一样采取客户端拉去模型，并且也可以批量拉取

- 确认机制

支持单个消息确认(individual ack)，也支持类似kafka的累积确认(cumulative ack)。但是，cumulative只能在Subscription mode位Exclusive和Failover使用，因为这2种模式能保证同一个partition的消息在同一个消费者实例消费。

- 并发模型

消费者端通过Subscription mode位Shared和Key\_Shared都能做到多个消费者实例并行消费，这个方式类似`nsq`的模式。另外一种做法是，通过topic多partition，结合Subscription mode为Failover(因为Failover的选master粒度是每一个partition)，可以实现跟kafka同样的语义。

- 有序性

跟kafka一样，topic的同一个partition是有序的


# 分布式集群

## nsq

nsq本身不提供完整的集群方案，每一个nsq是一个独立的状态数据。由Producer自己决定把topic发布到哪个nsq实例，Consumer通过订阅多个nsq实例进行消费。nsq提供了nsq\_lookupd来支持topic的路由，来简化应用的集群构建。消费者可以通过nsq\_lookupd找到nsq集群地址，以及topic的路由信息，而不需要单独去找每一个nsq地址。

![nsq-cluster-architecture](/images/nsq_cluster_architecture.png)

## kafka

![kafka-cluster-architecture](/images/kafka_cluster_architecture.jpg)

## pulsar

![pulsar-cluster-architecture](/images/pulsar_cluster_architecture.png)
