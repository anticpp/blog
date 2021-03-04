---
title: message-queue
date: 2021-03-04 10:15:47
tags:
---


不同的消息队列的消息语义

## nsq

```
                                 Consumer Group
                                   --------
  topic --        ------------------- C0  |
         |       \|/               |      |
         --> channel0 <-------------- C1  |
         |       /|\               |      |
         |        ------------------- C2  |
         --> channel1              |      |
         |                         --------
         --> channel2        

```

- 分发

每一个topic会分发到多个channel，每个channel是独立的队列。不同的Consumer Group消费不同的channel。

- 流控

每一个消费者实例，通过RDY告知nsq自己的处理窗口大小，消费者通过RDY来控制消息的处理吞吐。同一个channel的消息，只会分发给Consumer Group的一个实例。


- 并发

channel对同一个消费者实例，会并发通知多个消息，取决于消费者实例的RDY。channel对于多个消费者实例，也会并行通知。也就是如果想提高消费的吞吐，可以提高每一个消费者实例的RDY大小，也可以增加消费者实例。

- 有序性

由于channel的通知并行性，以及消息的REQUEUE语义，消息的有序性不保证

## kafka

```
  topic --                       Consumer Group
         |                         ---------
         --> partition0 <------------- C0  |
         |                         |       |                          
         --> partition1 <------------- C1  |
         |                         |       |
         --> partition2 <------------- C2  |
                                   |       |
                                   |   C3  |
                                   ---------
```

- 分发

kafka内部会维护每一个Consumer Group对topic的消费状态(offset)，不同的Consumer Group可以单独订阅消费，互相不影响。

- 流控

消费者主动拉取消息，并且自行调整消息offset

- 并发

1. 同一个消费者实例可以同时拉取多个消息进行并行消费，然后一次性调整offset
2. topic可以进行多个partition，多个partition可以有多个消费者实例并行进行消费


- 有序性

kafka保证topic的每一个partition是顺序的

由于channel的通知并行性，以及消息的REQUEUE语义，消息的有序性不保证


## pulsar
