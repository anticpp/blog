layout: udp_about
title: udp包的问题
date: 2016-11-14 19:51:27
tags: 技术
---

最近研究了一下udp相关的东西，做下笔记。
这里有相关的[测试代码](https://www.github.com/anticpp/experiments)。


# udp包一般多少适合
协议层的限制，理论上我们可以发送接收65535(0xFFFF)大小的udp包。为了避免IP层分片，udp包在ethernet上一般不超过1472(1500-20-8)，1500为ethernet的链路MTU。
实际上很多应用（例如DNS）的udp包都不超过512，为什么呢？

这是Stack Overflow上面的一句话
    The maximum safe UDP payload is 508 bytes. This is a packet size of 576, minus the maximum 60-byte IP header and the 8-byte UDP header. Any UDP payload this size or smaller is guaranteed to be deliverable over IP (though not guaranteed to be delivered). Anything larger is allowed to be outright dropped by any router for any reason.
按照字面理解的意思是，大于576的udp包在路由链路保证不了一定传输，难道是路由器的实现潜规则？

- 更新At 2016-11-14
似乎RFC的IPV4标准里面有定义。

RFC 791 excerpt:
...
All hosts must be prepared to accept datagrams of up to 576 octets ( whether they arrive whole or in fragments ). It is recommended that hosts only send datagrams larger than 576 octets if they have assurance that the destination is prepared to accept the larger datagrams.
The number 576 is selected to allow a reasonable sized data block to be transmitted in addition to the required header information. For example, this size allows a data block of 512 octets plus 64 header octets to fit in a datagram. The maximal internet header is 60 octets, and a typical internet header is 20 octets, allowing a margin for headers of higher level protocols.
...

既然这样，另外一个问题来了，为什么TCP的MSS一般不受这个限制。
是因为TCP有做链路MTU探测，以及稳定的重传机制吗？


# udp socket缓冲区
1. 发送缓冲区
经常有一个误解是udp没有发送缓冲区，实际udp也有发送缓冲区。只不过udp并不是流协议，发送缓冲区只会缓存一个包，然后立刻就被拷贝到内核协议栈。
如果发送一个超过缓冲区大小的udp包，大多数的实现是直接丢弃。

2. 接收缓冲区
接收缓冲区跟发送缓冲区有点不一样，所有的udp包会按照先后顺序缓存，应用层每次读会返回最早的包。实际上接收缓冲区就是FIFO队列。
如果接收缓冲区大小不够，大多数的实现是直接丢弃整个包。
