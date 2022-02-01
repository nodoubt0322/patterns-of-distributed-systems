# 复制日志（Replicated Log）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/replicated-log.html

通过使用复制到所有集群节点的预写日志，保持多个节点的状态同步。

**2022.1.11**

## 问题

当多个节点共享一个状态时，该状态就需要同步。所有的集群节点都需要对同样的状态达成一致，即便某些节点崩溃或是断开连接。这需要对每个状态变化请求达成共识。

但仅仅在单个请求上达成共识是不够的。每个副本还需要以相同的顺序执行请求，否则，即使它们对单个请求达成了共识，不同的副本会进入不同的最终状态。

## 解决方案

集群节点维护了一个[预写日志（Write-Ahead Log）](write-ahead-log.md)。每个日志项都存储了共识所需的状态以及相应的用户请求。这些节点通过日志项的协调建立起了共识，这样一来，所有的节点都拥有了完全相同的预写日志。然后，请求按照日志的顺序进行执行。因为所有的集群节点在每条日志项都达成了一致，它们就是以相同的顺序执行相同的请求。这就确保了所有集群节点共享相同的状态。

使用 [Quorum](quorum.md) 的容错共识建立机制需要两个阶段。
* 一个阶段负责建立[世代时钟（Generation Clock）](generation-clock.md)，了解在前一个 [Quorum](quorum.md) 中复制的日志项。
* 一个阶段负责在所有集群节点上复制请求。

每次状态变化的请求都去执行两个阶段，这么做并不高效。所以，集群节点会在启动时选择一个领导者。领导者会在选举阶段建立起[世代时钟（Generation Clock）](generation-clock.md)，然后检测上一个 [Quorum](quorum.md) 所有的日志项。（前一个领导者或许已经将大部分日志项复制到了大多数集群节点上。）一旦有了一个稳定的领导者，复制就只由领导者协调了。客户端与领导者通信。领导者将每个请求添加到日志中，并确保其复制到到所有的追随者上。一旦日志项成功地复制到大多数追随者，共识就算已经达成。按照这种方式，当有一个稳定的领导者时，对于每次状态变化的操作，只要执行一个阶段就可以达成共识。

### 多 Paxos 和 Raft

[多 Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1920s) 和 [Raft](https://raft.github.io/) 是最流行的实现复制日志的算法。多 Paxos 只在学术论文中有模糊的描述。[Spanner](https://cloud.google.com/spanner) 和 [Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/introduction) 等云数据库采用了[多 Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1920s)，但实现细节却没有很好地记录下来。Raft 非常清楚地记录了所有的实现细节，因此，它成了大多数开源系统的首选实现方式，尽管 Paxos 及其变体在学术界得到了讨论得更多。

下面的章节描述了如何实现复制日志。