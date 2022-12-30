参考：http://blog.mrcroxx.com/posts/code-reading/etcdraft-made-simple/4-log/

https://www.codedump.info/post/20180922-etcd-raft/#raft%E6%B6%88%E6%81%AF%E7%BB%93%E6%9E%84%E4%BD%93



raft库对外提供一个Node的Interface。这也是应用层唯一需要与这个raft库直接打交道的结构体，简单的来看看Node接口需要实现的函数：
![](../../img/Pasted%20image%2020221211181235.png)
主要关心Ready结构体。
raftexample中，首先在main.go中创建了两个channel：

1.  proposeC：用于提交写入的数据。
2.  confChangeC：用于提交配置改动数据。

然后分别启动如下核心的协程：

1.  启动HTTP服务器，用于接收用户的请求数据，最终会将用户请求的数据写入前面的proposeC/confChangeC channel中。
2.  启动raftNode结构体，该结构体中有上面提到的raft/node.go中的node结构体，也就是通过该结构体实现的Node接口与raft库进行交互。同时，raftNode还会启动协程监听前面的两个channel，收到数据之后通过Node接口的函数调用raft库对应的接口。

以上的交互流程就很清楚了，HTTP服务负责接收用户数据，再写入到两个核心channel中，而raftNode负责监听这两个channel：

1.  如果收到proposeC channel的消息，说明有数据提交，则调用Node.Propose函数进行数据的提交。
2.  如果收到confChangeC channel的消息，说明有配置变更，则调用Node.ProposeConfChange函数进行配置变更。
3.  设置一个定时器tick，每次定时器到时时，调用Node.Tick函数。
4.  监听Node.Ready函数返回的Ready结构体channel，有数据变更时根据Ready结构体的不同数据类型进行相应的操作，完成了之后需要调用Node.Advance函数进行收尾。

将以上流程用伪代码实现如下：

```Go
// HTTP server
HttpServer主循环:
  接收用户提交的数据：
    如果是PUT请求：
      将数据写入到proposeC中
    如果是POST请求：
      将配置变更数据写入到confChangeC中

// raft Node
raftNode结构体主循环：
  如果proposeC中有数据写入：
    调用node.Propose向raft库提交数据
  如果confChangeC中有数据写入：
    调用node.Node.ProposeConfChange向raft库提交配置变更数据
  如果tick定时器到期：
    调用node.Tick函数进行raft库的定时操作
  如果node.Ready()函数返回的Ready结构体channel有数据变更：
    依次处理Ready结构体中各成员数据
    处理完毕之后调用node.Advance函数进行收尾处理
```

到了这里，已经对raft的使用有一个基本的概念了，即通过node结构体实现的Node接口与raft库进行交互，涉及数据变更的核心数据结构就是Ready结构体，接下来可以进一步来分析该库的实现了。


Ready结构体如下：
![](../../img/Pasted%20image%2020221211182027.png)

## Raft库日志存储
```go
type raftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage

	// unstable contains all unstable entries and snapshot.
	// they will be saved into storage.
	unstable unstable

	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	committed uint64
	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	applied uint64

	logger Logger
}
```


持久化与snapshot的辨析：
- 持久化是指将内存中的数据写到磁盘中
- snapshot是指将多个Log Entry变成一个状态机的快照
- Log Entry 和 Snapshot都可以持久化或者放在内存中
- 持久化的原因：应付另一种失败模式就是**崩溃**
- Snapshot的原因：Log Entry太多啦，用一个快照省地方
- 持久化的时机：需要持久化的数据发生变化时
- Snapshot的时机：Log Entry 过长，或者上层应用主动调用Shanpshot




## Raft 消息结构体



