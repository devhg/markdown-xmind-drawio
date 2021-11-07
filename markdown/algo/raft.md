# [浅谈分布式一致性算法raft](https://www.cnblogs.com/wyq178/p/13899534.html)

> https://www.cnblogs.com/wyq178/p/13899534.html



前言：在分布式的系统中,存在很多的节点,节点之间如何进行协作运行、高效流转、主节点挂了怎么办、如何选主、各节点之间如何保持一致，这都是不可不面对的问题,此时raft算法应运而生,专门 用来解决上述问题。对于分布式的一致性算法,著名的有paxos,zookeeper基于paxos提出了zab协议, paxos是出名的晦涩难懂.而raft的设计初衷就是容易理解和简单、高效,本篇博客我们就来循序渐进的看看raft到底是什么？它的运行原理是什么样的？

本篇博客的目录：

# 一：raft的状态

raft的集群角色分为3种,不同的节点在运行环境中处于不同的角色，任何节点的任何一个时刻都处于以下三种角色之一,不同的角色具有不同的功能，所承担的职责也不一样：

## ①：**follower**

follwer是集群的初始状态,所有的节点在刚开始加入到集群中,默认是follower的角色,也就是从节点~

## ②：**candidate**

candidate的含义可以理解为候选人,这是专门用于在follower在进行选举的时候,被投票者的称谓,这是一个中间角色,followerA会发起投票到followerB,此时followerA的角色就是candidate

## ③：**leader**

有了从节点之后就必须有主节点了,所以接下来伴随的是选主的过程,选主之后的节点称之为leader,也就是主节点，主节点只有一个,由leader接收用户的请求,每一次请求都被被记录到log entry中

 三个角色可以这样理解：试想新学期开学第一天,所有的同学都是普通学生的身份(**follower**)，新学期开始需要挑选出班长(leader),需要进行投票,大家先选出几个候选人,候选人可以自己投票给自己,此时要选中成为班长的这几个人就是**candidate,**最后经过选举出来的班长就是***\*leader\****

### ***任期：***

  任期简称Term,是raft里面非常重要的概念,每个任期可以是任意时长，任期用连续的整数进行标号,每个节点都维护着当前的任期值,每个节点在检测到自己的任期值低于其他节点会更新自己的任期值，设置为检测到的较高值。当leader和candidate发现自己的任期低于别的节点，则会立即把自己转换为follower

***\*下面是几种角色的流转图：\****

![img](https://img2020.cnblogs.com/blog/1066538/202010/1066538-20201031214246248-1408062860.png)

# **二：raft的选主**

## 2.1：leader负责处理客户端的请求

所有对日志的添加或者状态变化的操作都是通过leader来完成,当leader接收请求之后会将日志分发到集群的所有follower节点,日志的数据流是从leader到其他的节点,而不会产生follower流向日志的情况，raft会保证流向所有follower节点的日志副本都是一致的：

选举leader发生在以下两种情况：

**①当一个raft集群初始化的时候** **②当选举出来的主节点宕机、崩溃的时候**

## 2.2：接下来谈一谈raft是如何选主的： 

  一个 `Raft`集群开始，集群中的节点所有的初始状态都是 `Follower,`然后设置任期(term为0),并启动计时器发起选举(Election),开始选举之后每个参与方都会有一个随机的超时时间（`Election Timeout`）,这里的关键就是随机 T`imeout（`150ms 到 300ms之间`）`，最先走完`timeout时间的一个节点开始`发起投票，向还在 `timeout` 中的另外节点请求投票(Reuest Vote)并等待回复，此时它就只能投给自己，然后raft会统计得票数,在计数器时间内,得票最多的会成为leader.这样的结果就是最先发起投票的节点会有大概率成为主节点,选出 `Leader` 后，term值会+1，并且`Leader`通过定期向所有 `Follower` 发送心跳信息(官方称之为：Append Entries,Append Entries是一种RPC协议)保持连接。

![img](https://img2020.cnblogs.com/blog/1066538/202010/1066538-20201031000602991-112895272.png)



### 两个节点同时发起选举

因为follwer节点的超时时间是随机的,所以可能会存在两个节点正好随机到相同的random time，并且拥有相同的term,此时raft会如何处理呢？raft会在相同的random time out时间同时发起leader选举,因为两个Candidate存在相同的term和timeout，并且同时发起投票,最终他们得到的votes是相同的。这个时候raft会等待下一轮的重试,下一轮两个节点的time out可能会不同,重试直到选举出leader

## 2.3：当选举完成之后考虑以下几个情形：

### 情形一：leader宕机

每次当leader对所有的followe发出Append Entries的时候,follower会有一个随机的超时时间,如果再超时时间内收到了leader的请求就会重置超时时间,如果没有收到超过超时时间,follower没有收到 `Leader`的心跳，follower会认为 `Leader` 可能已经挂了，此时第一个超时的follower会发起投票,注意这个时候它依然会向宕机的原leader发出Reuest Vote,但原leader不会回复。raft设计的

请求投票都是幂等的,会检测状态。当收到集群超过一半的节点的RequestVote reply后,此时的follower会成为leader

![img](https://img2020.cnblogs.com/blog/1066538/202010/1066538-20201031002702815-202104583.png)

 ps:后期leader恢复正常之后,加入到raft集群,初始化的角色是**follower**，而并非leader。因为任何时刻leader只有一个,如果是两个,就会发生"脑裂"问题

### 情形二：follower宕机

  follower宕机对整个集群影响不大,最多的影响是leader发出的Append Entries无法被收到,但是leader还会继续一直发送,直到follower恢复正常。raft会保证发送AppendEntries request的rpc消息是幂等的，如果follower已经接受到了消息，但是leader又让它再次接受，follower会直接忽略

# 三：raft如何保证集群的一致性

`3.1:Raft` 协议由leader节点负责接收客户端的请求,leader会将请求包装成log entry分发到从节点,所以集群强依赖 `Leader`节点的可用性,以确保集群 数据的一致性。数据的流向只能从 `Leader` 节点向 `Follower` 节点转移，这个过程叫做日志复制(**Log Replication)**：

① 当 Client 向集群 Leader 节点 提交数据 后，Leader 节点 接收到的数据 处于 未提交状态（Uncommitted）。

② 接着 Leader 节点会并发地向所有Follower节点复制数据并等待接收响应ACK

③ leader会等待集群中至少超过一半的节点已接收到数据后， Leader 再向 Client 确认数据 已接收。

④ 一旦向 Client 发出数据接收 Ack 响应后，表明此时 数据状态 进入 已提交（Committed），Leader 节点再向 Follower 节点发通知告知该数据状态已提交

⑤ follower开始commit自己的数据,此时raft集群达到主节点和从节点的一致

## ![img](https://img2020.cnblogs.com/blog/1066538/202010/1066538-20201031175644753-1690451960.png)3.2:在进行一致性复制的过程中,假如出现了异常情况，raft都是如何处理的呢？

1.数据到达 Leader 节点前，这个阶段 Leader 挂掉不影响一致性

2.数据到达 Leader 节点，但未复制到 Follower 节点。这个阶段 `Leader` 挂掉，数据属于 未提交状态，`Client` 不会收到 `Ack` 会认为 超时失败 可安全发起 重试。

3.数据到达 Leader 节点，成功复制到 Follower 所有节点，但 Follower 还未向 Leader 响应接收。这个阶段 `Leader` 挂掉，虽然数据在 `Follower` 节点处于 未提交状态（`Uncommitted`），但是 保持一致 的。重新选出 `Leader` 后可完成 数据提交。

4.数据到达 Leader 节点，成功复制到 Follower 的部分节点，但这部分 Follower 节点还未向 Leader 响应接收。这个阶段 `Leader` 挂掉，数据在 `Follower` 节点处于 未提交状态（`Uncommitted`）且 不一致。

`Raft` 协议要求投票只能投给拥有 最新数据 的节点。所以拥有最新数据的节点会被选为 `Leader`，然后再 强制同步数据 到其他 `Follower`，保证 数据不会丢失并 最终一致。

5.数据到达 Leader 节点，成功复制到 Follower 所有或多数节点，数据在 Leader 处于已提交状态，但在 Follower 处于未提交状态。

这个阶段 `Leader` 挂掉，重新选出 新的 `Leader` 后的处理流程和阶段 `3` 一样。

6.数据到达 Leader 节点，成功复制到 Follower 所有或多数节点，数据在所有节点都处于已提交状态，但还未响应 Client。这个阶段 `Leader` 挂掉，集群内部数据其实已经是 一致的，`Client` 重复重试基于幂等策略对 一致性无影响。

# 四：如何解决脑裂问题

  当raft在集群中遇见网络分区的时候,集群就会因此而相隔开,在不同的网络分区里会因为无法接收到原来的leader发出的心跳而超时选主,这样就会造成多leader现象,见下图：在网络分区1和网络分区2中，出现了两个leaderA和D,假设此时要更新分区2的值,因为分区2无法得到集群中的大多数节点的ACK,会复制失败。而网络分区1会成功,因为分区1中的节点更多,leaderA能得到大多数回应

当网络恢复的时候,集群不再是双分区,raft会有如下操作：

①: leaderD发现自己的Term小于LeaderA,会自动下台(step down)成为follower,leaderA保持不变依旧是集群中的主leader角色

②: 分区中的所有节点会回滚roll back自己的数据日志,并匹配新leader的log日志,然后实现同步提交更新自身的值。通知旧leaderA也会主动匹配主leader节点的最新值,并加入到follower中

③: 最终集群达到整体一致，集群存在唯一leader（节点A）

![img](https://img2020.cnblogs.com/blog/1066538/202010/1066538-20201031210601798-546846443.png)

#  五：总结

  本篇博客从整体上讲了下raft的状态角色、如何选举出leader、如何保证一致性、以及如何处理网络分区时的脑裂问题,整理较为粗略，raft实现起来更为复杂和细致，所以这里只是浅谈一下。理解raft的主要目的在于分布式环境中,对于集群之间的节点交互、宕机后如何处理如何保证高可用、高一致性有一定的理解。