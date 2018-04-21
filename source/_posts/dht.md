---
title: DHT
date: 2018-04-17 21:03:24
tags: p2p dht
---

## 背景
BT作为历史上最成功的P2P文件系统，已经在全球实现了彻底的去中心化。整个BT的P2P网络，不依赖任何机构或者中心依然能够很好的运行。
这里面，最关键的设计是引入了DHT。我们这里通过简单的方式，介绍一下为什么要引入DHT，以及DHT的基本原理。

BT下载的一个核心问题是***资源索引***的问题，就是我们需要下载一个资源的时候，怎么能知道还有其它客户端也在同时下载这个资源，这样我就可以通过其它客户端给我提供P2P加速。

```
res1 -> peer1,peer2,...
res2 -> peer3,peer4,...
```

## 中心化

最早的BT下载，是通过中心化的Tracker解决资源索引的问题。

Tracker提供了上报和查询服务，正在下载资源的BT客户端可以把资源上报给Tracker，需要下载资源的BT客户端可以到Tracker查询其它同时下载这个资源的Peer。

![](/images/bt-tracker.png)

Tracker提供了一种简单高效的方案，解决了资源索引的问题。缺点是一旦Tracker被封了或者挂了，BT的P2P网络也就挂了。这是所有中心化服务必将面临的问题，这时候需要一个去中心化的资源索引方案。

## 去中心化

考虑没有中心化的Tracker，我们需要一个去中心化的方式，把资源的索引数据存储到全网的Peer节点，并且能高效的查询。

### 方案1 - 全量
最简单的办法，就是让每一个Node(节点)都维护一个全量的索引，相当于每个Node都是一个Tracker节点。

这种方案有2个明显的问题

1. 每个索引的变更需要扩散到全网，效率太差
2. 海量的索引，每个单点都需要全量存储，代价太大

### 方案2 - 一致性Hash
基本的思路是，我们需要把索引数据按照***一定规则(hash<key>)***分布到全网的节点。一致性Hash是一种解决办法，首先把Node散列(hash<NodeID>)到一个整数的空间，这样每一个Node负责一个范围段的索引。

例如，NodeA负责范围`[i, j)`的范围，那么如果索引按照分布规则在范围内，那么该索引就由NodeA负责存储。

```
// NodeA covers [i, j).
// Use hash(res.key) to caculate resource.
if i < hash(res.key) < j {
    set(res, NodeA) 
}
```

这里盗用一个网上的一致性Hash图片，具体的算法这里就不重复介绍了。

![](/images/dht-constant-hash.jpg)

一致性Hash依然遇到一些问题，去中心化的网络里面，没有人去负责维护Hash环(分布规则)。如果Node变更（新增or删除），没办法在全网更新这个Hash环

## DHT
DHT(Distributed Hash Table)，从名称上讲就是分布式的Hash table。因为我们是要把一个大的Hash table（Tracker一般是通过Hash table维护索引数据，Hash table也是我们比较常用高效的索引结构）分布到网络的不同节点上，所以叫分布式Hash table。

所以一致性Hash，也是一种分布式Hash table的实现，DHT可以看作是一致性Hash的基础上的改进。一致性Hash算法的问题主要是Hash环没有人去维护，实际上只要我们能设计出一种***读写共识***机制，我们并不需要去维护一个Hash环。就是说，我们能让索引的写策略和读策略(分布策略)，能够保持一致并且高效，我们就可以保持无状态（不需要固定的Hash环）。

DHT本质上设计了这样一种***共识机制***。

DHT引入了2个概念:

1. hash

DHT网络中每个节点的Node ID就是一个hash值，每个资源也有一个infohash

2. 距离

用来衡量2个hash的距离，Node ID之间可以有距离，infohash和Node Id也可以计算距离。要注意的是，这个距离并不是实际意义上的距离，只是一个算法上的需要。

> 注意：Node和Peer的概念是区分的，以下是DHT的papper的描述
> A "peer" is a client/server listening on a TCP port that implements the BitTorrent protocol. A "node" is a client/server listening on a UDP port implementing the distributed hash table protocol. 

DHT网络中每个Node都维护一个Routing table（路由表），保存它和网络中一小部分节点信息。每一个节点，都能够提供查询接口，并且能够返回距离目标hash更近的新节点。这样，可以通过一个***收敛的递归查询***，不断的趋近目标。

这个过程实际上是一个Peer Routing的过程，[IPFS peer routing](https://github.com/libp2p/specs/blob/master/4-architecture.md#41-peer-routing)对这块有一个比较好的整理。

一个典型的Peer Routing过程：
```
1 ) client需要查找目标hash，先到Node0进行查询。
2 ) Node0本地没有这个hash的索引，所以Node0从自己的Routing table返回离目标hash更近的节点Node1和Node2
3 ) client对Node1和Node2重复过程1的查询
4 ) client递归1-3的过程，不停的在网络中搜索目标hash

           1) query hash       -----------
client  -------------------->  |  Node0  |
        <-------------------   |         |
           2 ) Node1,Node2     -----------

           3 ) query hash      -----------
        -------------------->  |  Node1  |
                               -----------

           3 ) query hash      -----------
        -------------------->  |  Node2  |
                               -----------

```

更完整的DHT介绍，大家可以直接看一下原文Papper[DHT](http://www.bittorrent.org/beps/bep_0005.html)。

## Kademlia
DHT的规范并没有说明怎么去实现一个收敛Peer Routing，DHT的实现有很多种，我们这里介绍使用最广的Kademlia，简称KAD。
KAD能实现每一次查询折半的效率，也就是log2(N)的查询效率。假设1000000规模的网络，可以最多20次查询能够完成查询。

1. 距离

KAD使用XOR(按位异或)对两个hash进行距离计算，为什么要使用异或？我们后面结合k桶可以看到，这是经过精心设计的算法。

```
distance(A, B) = A xor B
```

2. Routing table

KAD使用k桶(k-bucket)实现一个Routing table，其实就是一个hash table，只是每一个bucket的大小不超过k(目的是控制Routing table的大小)。

```
                    -----------------------------------------
buckets  ----->     |       |       |       |       |       |
                    -----------------------------------------

```

