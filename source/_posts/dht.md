---
title: DHT
date: 2018-04-17 21:03:24
tags: p2p dht
---

## 1. 背景
BT作为历史上最成功的P2P文件系统，已经在全球实现了彻底的去中心化。BT的P2P网络，不依赖任何机构或者中心依然能够很好的运行。
这里面，最关键的设计是引入了DHT。这里我们主要介绍一下为什么要引入DHT，以及DHT的基本原理。

BT作为一种P2P系统，面临***资源索引***问题，简单来说***资源索引***维护资源(resource)当前有哪些节点(peer)也在同时下载的关系。

```
res0 -> peerA,peerB,...
res1 -> peerC,peerD,...

                         ------------------------------
              Index ---> | res0 | res1 | ...          |
                         ------------------------------
                            |      |
                           \|/    \|/
                         ------- -------
                         |peerA| |peerC|
                         ------- -------
                            |      |
                         ------- -------
                         |peerB| |peerD|
                         ------- -------

```


## 2. 中心化

实际上BT下载的传输本身是去中心化的，我们每一个peer会同时从网络上搜索到的其它peer进行数据传输。
但是***资源索引***一开始还是属于中心化的解决方案，就是Tracker。虽然网络上有很多提供Tracker的服务，每一个客户端也可以设置若干个Tracker，但是Tracker依然是一个中心化的服务。

Tracker提供了上报和查询服务，如下图：正在下载某一个资源(res)的BT客户端peer2把资源上报给Tracker。下载资源的BT客户端(peer1)可以到Tracker查询时下载这个资源的peer2。


```

                --------------    -----------------------------------
           |--->| Tracker    | -- |           Index                 |
           |    --------------    -----------------------------------
           |     |         /|\   
           |     |          |
           |     |          |
     query |     |resp      |
           |     |(peer2)   |
           |     |          |    report res
           |     |          |----------------
           |    \|/                         |
        -----------                     ---------
        |  peer1  |                     | peer2 |
        -----------                     ---------
```

Tracker提供了一种简单高效的方案，解决了资源索引的问题。缺点是一旦Tracker不可用，BT的P2P网络也就随之瘫痪(我们可以参考海盗湾事件)。这是所有中心化服务必将面临的问题，为了让整个P2P网络更加健壮，需要探索一个去中心化的实现。

## 3. 去中心化

考虑没有中心化的Tracker，我们需要一个去中心化的方式，把资源的索引存储到全网的节点，并且能高效的查询。

### 3.1 全量
最简单的办法，就是让每一个Node(节点)都维护一个全量的索引，相当于每个Node都是一个Tracker节点。

> 注意：我们需要区分一下Node和Peer的概念，以下是DHT的papper的描述
> A "peer" is a client/server listening on a TCP port that implements the BitTorrent protocol. A "node" is a client/server listening on a UDP port implementing the distributed hash table protocol. 

```
                --------------    -----------------------------------
                | Node0      | -- |           Index                 |
                --------------    -----------------------------------

                --------------    -----------------------------------
                | Node1      | -- |           Index                 |
                --------------    -----------------------------------
                    
                    ...

                --------------    -----------------------------------
                | NodeN      | -- |           Index                 |
                --------------    -----------------------------------
```

这种方案有几个明显的问题

1. 每个索引的变更需要扩散到全网，效率太差
2. 海量的索引，每个单点都需要全量存储，代价太大

### 3.2 Sharding
基本的思路是，我们需要把索引数据按照***一定规则***分布(sharding)到全网的节点。


```
        -----------       -----------     -----------                   -----------
        |  Node0  |       |  Node1  |     |  Node2  |       ...         |  NodeN  |
        -----------       -----------     -----------                   -----------
           /|\                 /|\            /|\                            /|\
            |--------           |              |                              |
                     |          |              |                              |
                -----------------------------------------------------------------------
    Index ----> |  shard0    |  shard1    |   shard2     |  ...      |    shardN      |
                -----------------------------------------------------------------------

```

一致性Hash是一种常用的办法，首先把Node散列(hash<NodeID>)到一个整数的空间，这样每一个Node负责一个范围段的索引。
同样，我们可以把资源也计算一个整数的hash。
例如，NodeA负责范围`[i, j)`的范围，那么如果索引按照分布规则在范围内，那么该索引就由NodeA负责存储。

```
// NodeA covers [i, j).
// Use hash(resource) to caculate resource.
if i < hash(resource) < j {
    store(resource, NodeA) 
}
```

这里盗用一个网上的一致性Hash图片，具体的算法这里就不重复介绍了。

![](/images/dht-constant-hash.jpg)

一致性Hash解决了sharding的问题，但是一致性Hash引入了Hash环状态。在系统中如何维护Hash状态数据，跟如何维护资源索引面临一样的问题。

我们需要彻底解决状态数据的问题，才能真正实现去中心化。

## 4. DHT
DHT(Distributed Hash Table)，就是分布式Hash table。实际上也是Sharding的一种实现方法，DHT可以看作是一致性Hash的基础上的改进。

一致性Hash算法的问题是Hash环的状态数据，只要我们能设计出一种关于索引分布规则的去中心化的***共识机制***，我们并不需要去维护一个Hash环。DHT本质上设计了这样一种去中心化的***共识机制***。

关于DHT的比较完整的介绍，可以参考[DHT Protocol](http://www.bittorrent.org/beps/bep_0005.html)。

### 4.1 Hash

DHT网络中每个节点的NodeID就是一个hash值，每个资源也有一个hash。

### 4.2 距离

用来计算2个hash的距离，Node和Node之间可以有距离，Node和资源之间也可以有距离。要注意的是，这个距离并不是实际意义上的距离，只是一个算法上的需要。

### 4.3 Routing table
DHT网络中每个Node都维护一个Routing table（路由表），保存它和网络中一小部分节点信息。每一个节点，都能够提供查询接口，并且能够返回距离目标hash更近的新节点。这样，可以通过一个***收敛的递归查询***，不断的趋近目标。

```
                          ---------------------------------------------
       Routing table -->  | NodeInfo0 | NodeInfo1 | ...   | NodeInfoN |
                          ---------------------------------------------

```

这个过程实际上是一个Peer Routing的过程，[IPFS peer routing](https://github.com/libp2p/specs/blob/master/4-architecture.md#41-peer-routing)对这个概念有比较好的总结。

一个典型的Query Resource过程：
```
1 ) client需要查找目标资源hash，先到Node0进行查询。
2 ) 如果Node0本地有这个hash的索引，则返回结果
3 ) 如果Node0本地没有这个hash的索引，Node0从自己的Routing table查找离目标hash更近的节点，例如Node1和Node2
4 ) client对Node1和Node2重复过程1的查询，直到有查询到有结果

           1) query hash       -----------
client  -------------------->  |         |
        <-------------------   |  Node0  |
           2 ) Resource        |         |
           3 ) Node1,Node2     -----------

           4 ) query hash      -----------
        -------------------->  |  Node1  |
                               -----------

           4 ) query hash      -----------
        -------------------->  |  Node2  |
                               -----------

```


## 5. Kademlia
DHT有很多不同的实现，这里主要介绍Kademlia，一般我们叫KAD DHT。
KAD使用k-bucket(k桶)实现Routing table，能实现log2(N)的查询效率。假设1000000节点的规模的网络，可以最多20次查询能够完成全网节点的查询。

### 5.1 距离

KAD使用XOR(按位异或)对两个hash进行距离计算，为什么要使用异或？我们后面结合k桶可以看到，这是经过精心设计的算法。

```
distance(A, B) = A xor B
```

### 5.2 Routing table

KAD使用k桶(k-bucket)实现一个Routing table，其实就是一个hash table，只是每一个bucket的大小不超过k(目的是控制Routing table的大小)。

```

```

