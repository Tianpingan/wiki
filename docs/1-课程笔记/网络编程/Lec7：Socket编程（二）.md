![](../../img/Pasted%20image%2020220902203517.png)
## C/S模型

![](../../img/Pasted%20image%2020220902204001.png)

## Socket函数
![](../../img/Pasted%20image%2020220902204202.png)

为主动模式，可以直接发送连接。

## bind函数
![](../../img/Pasted%20image%2020220902204702.png)

## listen函数
![](../../img/Pasted%20image%2020220902204746.png)
其中`backlog`值为队列的大小，建议设置为`SOMAXCONN`这个宏。
![](../../img/Pasted%20image%2020220902204847.png)

调用`listen`之后，该套接字就从主动套接字变成了被动套接字。
- 被动套接字
	- 接受连接的，`accept`
- 主动套接字
	- 发起连接的，`connect`

一般来说，使用socket函数创建的socket默认是主动socket，这意味着一个主动的socket可以调用connect跟一个被动socket建立一个连接，对主动socket来说，这叫主动打开。

被动socket是一个通过调用listen函数监听要发起连接的socket，当被动socket接受一个连接通常称为被动打开。

在大多数网络程序中，服务端会作为被动socket被动接受连接，而客户端会作为主动socket主动发起连接。

服务端通过socket函数创建的socket是主动socket，而listen函数就是把这个还未接受连接的主动socket转换为被动socket，因为服务端只需要被动接受客户端的连接请求。

![](../../img/Pasted%20image%2020220902205112.png)
![](../../img/Pasted%20image%2020220902205159.png)

## accept函数
![](../../img/Pasted%20image%2020220902205229.png)

要注意，`addrlen`需要有初始值，否则会失败。
`socklen_t addrlen = sizeof(sockaddr_in)`

将返回一个已连接的套接字，并且该套接字是主动套接字。

## connect函数
![](../../img/Pasted%20image%2020220902210111.png)