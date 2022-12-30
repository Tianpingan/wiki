参考：https://www.codedump.info/post/20180921-raft/
我见过讲的最好的文章了！！！
## 概述

Raft算法主要分成两部分：
- Leader节点的选举
- Leader节点处理一致性问题

Leader节点是Raft中处理一致性问题的关键，Leader节点接收来自客户端的请求日志数据，然后同步到集群中其它节点进行复制，当日志已经同步到超过半数以上节点的时候，Leader节点再通知集群中其它节点哪些日志已经被复制成功，可以提交到raft状态机中执行。

细分下来可以分成下面几个子问题：
- Leader选举：集群中必须存在一个leader节点。
- 日志复制：leader节点接收来自客户端的请求然后将这些请求序列化成日志数据再同步到集群中其它节点。
- 安全性：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引的另一条日志数据输入到raft状态机中执行。


Raft算法需要一直保持的几个属性。

-   选举安全性（Election Safety）：在一个任期内只能存在最多一个leader节点。
-   Leader节点上的日志为只添加（Leader Append-Only）：leader节点永远不会删除或者覆盖本节点上面的日志数据，leader节点上写日志的操作只可能是添加操作。
-   日志匹配性（Log Matching）：如果两个节点上的日志，在日志的某个索引上的日志数据其对应的任期号相同，那么在两个节点在这条日志之前的日志数据完全匹配。
-   leader完备性（Leader Completeness）：如果一条日志在某个任期被提交，那么这条日志数据在leader节点上更高任期号的日志数据中都存在。
-   状态机安全性（State Machine Safety）：如果某个节点已经将一条提交过的数据输入raft状态机执行了，那么其它节点不可能再将相同索引的另一条日志数据输入到raft状态机中执行。

## Raft算法基础
![](../../img/Pasted%20image%2020221211155046.png)

1.  start up：起始状态，节点刚启动的时候自动进入的是follower状态。
2.  times out, starts election：follower在启动之后，将开启一个选举超时的定时器，当这个定时器到期时，将切换到candidate状态发起选举。
3.  times out, new election：进入candidate 状态之后就开始进行选举，但是如果在下一次选举超时到来之前，都还没有选出一个新的leade，那么还会保持在candidate状态重新开始一次新的选举。
4.  receives votes from majority of servers：当candidate状态的节点，收到了超过半数的节点选票，那么将切换状态成为新的leader。
5.  discovers current leader or new term：candidate状态的节点，如果收到了来自leader的消息，或者更高任期号的消息，都表示已经有leader了，将切换回到follower状态。
6.  discovers server with higher term：leader状态下如果收到来自更高任期号的消息，将切换到follower状态。这种情况大多数发生在有网络分区的状态下。

从以上的描述可以看出，任期号在raft算法中更像一个“**逻辑时钟**（logic clock）”的作用，有了这个值，集群可以发现有哪些节点的状态已经过期了。每一个节点状态中都保存一个当前任期号（current term），节点在进行通信时都会带上本节点的当前任期号。**如果一个节点的当前任期号小于其他节点的当前任期号，将更新其当前任期号到最新的任期号**。如果一个candidate或者leader状态的节点发现自己的当前任期号已经小于其他节点了，那么将切换到follower状态。反之，**如果一个节点收到的消息中带上的发送者的任期号已经过期，将拒绝这个请求**。

## Leader选举
发起选举时，follower将递增它的任期号然后切换到candidate状态。然后通过向集群中其它节点发送RequestVote RPC请求来发起一次新的选举。一个节点将保持在该任期内的candidate状态下，直到以下情况之一发生。

-   该candidate节点赢得选举，即收到超过半数以上集群中其它节点的投票。
-   另一个节点成为了leader。
-   选举超时到来时没有任何一个节点成为leader。

下面来逐个分析以上几种情况。

第一种情况，如果收到了集群中半数以上节点的投票，那么此时candidate节点将成为新的leader。每个节点在一个任期中只能给一个节点投票，而且遵守“先来后到”的原则。这样就保证了，每个任期最多只有一个节点会赢得选举成为leader。但并不是每个进行选举的candidate节点都会给它投票，在后续的“选举安全性”一节中将展开讨论这个问题。当一个candidate节点赢得选举成为leader后，它将发送心跳消息给其他节点来宣告它的权威性以阻止其它节点再发起新的选举。

第二种情况，当candidate节点等待其他节点时，如果收到了来自其它节点的AppendEntries RPC请求，同时做个请求中带上的任期号不比candidate节点的小，那么说明集群中已经存在leader了，此时candidate节点将切换到follower状态；但是，如果该RPC请求的任期号比candidate节点的小，那么将拒绝该RPC请求继续保持在candidate状态。

第三种情况，一个candidate节点在选举超时到来的时候，既没有赢得也没有输掉这次选举。这种情况发生在集群节点数量为偶数个，同时有两个candidate节点进行选举，而两个节点获得的选票数量都是一样时。当选举超时到来时，如果集群中还没有一个leader存在，那么candidate节点将继续递增任期号再次发起一次新的选举。这种情况理论上可以一直无限发生下去。
以上过程用伪代码来表示如下。

```markdown
节点刚启动，进入follower状态，同时创建一个超时时间在150-300毫秒之间的选举超时定时器。

follower状态节点主循环：
  如果收到leader节点心跳：
    心跳标志位 置1

  如果选举超时到期：
    没有收到leader节点心跳（心跳标志位 为空）：
      任期号term+1，换到candidate状态。
    如果收到leader节点心跳：
      心跳标志位 置空

  如果收到选举消息：
    如果当前没有给任何节点投票过 或者 消息的任期号大于当前任期号：
      投票给该节点
    否则：
      拒绝投票给该节点

candidate状态节点主循环：
  向集群中其他节点发送RequestVote请求，请求中带上当前任期号term

  收到AppendEntries消息：
    如果该消息的任期号 >= 本节点任期号term：
      说明已经有leader，切换到follower状态
    否则：
      拒绝该消息

  收到其他节点应答RequestVote消息：
    如果数量超过集群半数以上，切换到leader状态
    
  如果选举超时到期：
    term+1，进行下一次的选举
```

## 日志复制
日志复制的流程大体如下：

1.  每个客户端的请求都会被重定向发送给leader，这些请求最后都会被输入到raft算法状态机中去执行。
2.  leader在收到这些请求之后，会首先在自己的日志中添加一条新的日志条目。
3.  在本地添加完日志之后，leader将向集群中其他节点发送AppendEntries RPC请求同步这个日志条目，当这个日志条目被成功复制之后（什么是成功复制，下面会谈到），leader节点将会将这条日志输入到raft状态机中，然后应答客户端。

Raft日志的组织形式如下图所示。
![](../../img/Pasted%20image%2020221211160216.png)
每个日志条目包含以下成员。

1.  index：日志索引号，即图中最上方的数字，是严格递增的。
2.  term：日志任期号，就是在每个日志条目中上方的数字，表示这条日志在哪个任期生成的。
3.  command：日志条目中对数据进行修改的操作。

一条日志如果被leader同步到集群中超过半数的节点，那么被称为“成功复制”，leader节点在收到半数以上的节点应答某条日志之后，就会提交该日志，此时日志这个日志条目就是“已被提交（committed）”。如果一条日志已被提交，那么在这条日志之前的所有日志条目也是被提交的，包括之前其他任期内的leader提交的日志。如上图中索引为7的日志条目之前的所有日志都是已被提交的日志。

![log-replication-1](https://www.codedump.info/media/imgs/20180921-raft/log-replication-1.png "log replication 1")

在上图中，一个请求有以下步骤。

-   1.  客户端发送SET a=1的命令到leader节点上。
-   2.  leader节点在本地添加一条日志，其对应的命令为SET a=1。这里涉及到两个索引值，committedIndex存储的最后一条提交（commit）日志的索引，appliedIndex存储的是最后一条应用到状态机中的日志索引值，一条日志只有被提交了才能应用到状态机中，因此总有 committedIndex >= appliedIndex不等式成立。在这里只是添加一条日志还并没有提交，两个索引值还指向上一条日志。
-   3.  leader节点向集群中其他节点广播AppendEntries消息，带上SET a=1命令。

![log-replication-2](https://www.codedump.info/media/imgs/20180921-raft/log-replication-2.png "log replication 2")

接下来继续看，上图中经历了以下步骤。

-   4.  收到AppendEntries请求的follower节点，同样在本地添加了一条新的日志，也还并没有提交。
-   5.  follower节点向leader节点应答AppendEntries消息。
-   6.  当leader节点收到集群半数以上节点的AppendEntries请求的应答消息时，认为SET a=1命令成功复制，可以进行提交，于是修改了本地committed日志的索引指向最新的存储SET a=1的日志，而appliedIndex还是保持着上一次的值，因为还没有应用该命令到状态机中。

![log-replication-3](https://www.codedump.info/media/imgs/20180921-raft/log-replication-3.png "log replication 3")

当这个命令提交完成了之后，命令就可以提交给应用层了。

-   7.  提交命令完成，给应用层说明这条命令已经提交。此时修改appliedIndex与committedIndex一样了。
-   8.  leader节点在下一次给follower的AppendEntries请求中，会带上当前最新的committedIndex索引值，follower收到之后同样会修改本地日志的committedIndex索引。

需要说明的是，7和8这两个操作并没有严格的先后顺序，谁在前在后都没关系。

leader上保存着已被提交的最大日志索引信息，这个索引值被称为“nextIndex”（原文有误？应该是CommittedIndex？），在每次向follower节点发送的AppendEntries RPC请求中都会带上这个索引信息，这样follower节点就知道哪个日志已经被提交了，被提交的日志将会输入Raft状态机中执行。

Raft算法保持着以下两个属性，这两个属性共同作用满足前面提到的日志匹配（LogMatch）属性：

-   如果两个日志条目有相同的索引号和任期号，那么这两条日志存储的是同一个指令。
-   如果在两个不同的日志数据中，包含有相同索引和任期号的日志条目，那么在这两个不同的日志中，位于这条日志之前的日志数据是相同的。

## 新Leader与Follower同步数据
在Raft算法中，解决日志数据不一致的方式是Leader节点同步日志数据到follower上，覆盖follower上与leader不一致的数据。

为了解决与follower节点同步日志的问题，leader节点中存储着两个与每个follower节点日志相关的数据。

-   nextIndex存储的是下一次给该节点同步日志时的日志索引（需要复制的，不管是提交还是没提交）。
-   matchIndex存储的是该节点的最大日志索引。

从以上两个索引的定义可知，在follower与leader节点之间日志复制正常的情况下，nextIndex = matchIndex + 1。但是如果出现不一致的情况，则这个等式可能不成立。每个leader节点被选举出来时，将做如下初始化操作：

-   nextIndex为leader节点最后一条日志再加一（原文有误？）。
-   matchIndex为0。

这么做的原因在于：leader节点将从后往前探索follower节点当前存储的日志位置，而在不知道follower节点日志位置的情况下只能置空matchIndex了。

leader节点通过AppendEntries消息来与follower之间进行日志同步的，每次给follower带过去的日志就是以nextIndex来决定，其可能有两种结果：

-   如果follower节点的日志与这个值匹配，将返回成功；
-   否则将返回失败，同时带上本节点当前的最大日志ID（假设这个索引为hintIndex）或者是发生冲突的日志ID，方便leader节点快速定位到follower的日志位置以下一次同步正确的日志数据。

而leader节点在收到返回失败的情况下，将置`nextIndex = min(hintIndex+1,上一次append消息的索引)`，再次发出添加日志请求。

![](../../img/Pasted%20image%2020221211163044.png)

## 安全性




