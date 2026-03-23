---
{"dg-publish":true,"permalink":"/66.归档发布/04.并发/分布式/Gossip协议理解/","dg-note-properties":{"时间":"2026-03-15"}}
---

#分布式 #协议 #gossip

```ad-summary
title: 总结

- Gossip 是去中心化协议，节点间互相传播信息，最终收敛一致，没有单点故障
- 两种模式：反熵（全量同步，消除差异）和谣言传播（增量扩散，Redis Cluster 用这个）
- 缺点：最终一致，有收敛窗口；消息可能重复；不能容忍恶意节点
```

## 1. Gossip 是什么？

Gossip 是一种去中心化的通信协议，节点之间像传八卦一样互相传播信息，最终让整个集群的所有节点都拿到最新状态。

和主从模式不同，Gossip 没有中心节点，不存在单点故障，节点随时可以加入或离开，集群自己会收敛到一致状态。Redis Cluster 用它来传播节点状态和槽信息。

## 2. 两种传播模式

### 2.1 反熵（Anti-Entropy）

每隔一段时间，节点随机选一个其他节点，互相交换全量数据，消除差异。
![反熵模式](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/反熵模式.png)
- 推模式：我把数据推给你
- 拉模式：我来拉你的数据
- 推拉结合：互相同步，效率最高

缺点是传播的是全量数据，节点多了网络开销大。一般会设计成环形，避免重复传播。

### 2.2 谣言传播（Rumor-Mongering）

节点有了新数据就变成"活跃节点"，周期性地随机联系其他节点发送新数据，直到所有节点都收到为止。

只传播增量数据，适合节点数量多或节点频繁变动的场景。Redis Cluster 用的就是这种模式。
![gossip rumor mongering D0IpXnM4](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/gossip-rumor-mongering-D0IpXnM4.gif)
## 3. 优缺点

优点：
- 去中心化，没有单点故障
- 容错性强，节点随意增减、重启都没问题
- 节点多时传播速度比单主广播更快（多路并发扩散）

缺点：
- 最终一致性，各节点状态收敛需要时间，中间会有不一致窗口
- 消息可能重复，同一个节点可能收到多次相同消息
- 不能容忍恶意节点（拜占庭问题）

## 4. 在 Redis Cluster 中的应用

Redis Cluster 节点间通过 Gossip 协议传播：
- 节点上下线状态
- 槽的归属变更
- 集群拓扑信息

每个节点定期随机选几个节点发送 PING 消息，携带自己知道的部分节点状态，对方回 PONG 时也带上自己的状态，这样信息就在集群里扩散开了。

详见 [[Redis Cluster槽管理机制\|Redis Cluster槽管理机制]]。

## 相关链接

- [[Redis Cluster槽管理机制\|Redis Cluster槽管理机制]]
- [[Redis哨兵与集群\|Redis哨兵与集群]]
- [[66.归档发布/04.并发/分布式/Raft协议原理\|Raft协议原理]]
