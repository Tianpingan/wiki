![](../../img/Pasted%20image%2020220903062709.png)
## REUSEADDR
当服务器关闭后，在一段时间内是无法重启的，因为之前的地址已经被绑定过了，结束后还处于`TIME_WAIT`状态。可以使用`REUSEADDR`来解决这个问题。
![](../../img/Pasted%20image%2020220903063609.png)
```C
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
```

这样设置后，就允许地址还在`TIME_WAIT`状态下就重启。

## 处理多客户连接
每来一个连接，就创建个进程去处理该连接上的读与写。

## 点对点聊天程序

