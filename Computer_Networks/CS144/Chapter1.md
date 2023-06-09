# 1.1 A day in the life of an aplication

## 网络应用
---

- 程序可以和网络中的其他程序交换数据（通信）
	- 方式：双向可靠的字节流（Byte Stream）
		- 将整个网络抽象成了一个简单的读/写关系
	- 任何一方都可以随时关闭连接

## 万维网（客户端-服务器模型）
---

- 使用HTTP协议（超文本传输协议）进行通信
	- 以文档为中心的程序通信方式
	- 客户端可以向服务器发送请求，服务器对请求进行响应
		- 常见的请求：GET、PUT、DELETE、INFO等
		- 常见的响应：200OK、400ERROR
	- 使用ASCII码

## BitTorrent（p2p模式）（peer to peer）
---

- 允许用户共享和交换大文件
- 一个用户可以并行的向其他用户发送请求
- 将文件分成不同的片段（piece）
	- 用户可以从其他拥有该片段的用户处下载此片段
	- 这些合作的用户的集合成为群（swarms）
	- 用户想要下载文件时，需要先找到一个叫做torrent的文件，它会为追踪器（Tracker）提供信息（需要追踪哪个用户）![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/IMG_0275.jpeg)

## Skype（以上二者的结合）
---

- 问题1：客户端需要通过NAT（网络地址转换器）接入互联网
	- 路由器就食一个NAT
	- 自由连接外界，外界难以连接
- 解决方法：集合服务器（Rendezvous）
	- 客户端与服务器保持着连接，当其他客户端想要进行连接时，则可以通过服务器向目标客户端发送请求，产生反向连接（连接方向和请求方向相反）![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/IMG_0276.jpeg)
- 问题2:连接双方的客户端都位于NAT之后
- 解决方法：中继（Relay）
	- 双方客户端都与中继进行连接，一方发送的数据通过中继转交给另一方![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/IMG_0277.jpeg)

# 1.2 The four layer internet model

## 互联网的四层模型
---

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/Pasted%20image%2020230404151023.png)

### 链接层

- 互联网由终端主机、链接和路由器组成
- 数据以数据包（package）的形式传递
	- 数据包由标头和数据组成（标头包括了包的来源的去向）![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/Pasted%20image%2020230404151534.png)

### 网络层

- 负责从源端通过网络端到端地传递数据包到目的地
	- 网络层将数据报（datagram）发送给链接层，链接层再对外发送![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/IMG_0282.jpeg)
	- 当我们将数据包发送到互联网上时，必须使用IP协议（Internet Protocol）
		- IP将会尽力将数据包发送到另一端
		- IP数据包可能会丢失，无法按照顺序传递或损坏

### 传输层

- TCP（Transmission Control Protocol）
	- TCP/IP代表应用程序同时使用了TCP和IP
	- TCP确保互联网上一端的应用程序发送的数据以正确的顺序连接到另一端的应用程序
	- 如果网络层丢弃了一些数据报，TCP将重新传输它们
	- 如果网络层传输的顺序错误，TCP会将它们变成正确的顺序
	- 用途：邮件等需要保证正确传输的信息（反例：电话、在线会议等则会选择UDP）
- UDP（User Datagram Protocol）
	- 捆绑应用程序的数据，然后将其交给网络层进行传输
	- 传输没有保证

### 应用层

- 将输入通过API发送给传输层

# 1.3 The IP service model

## 数据报的发送
---

- 再每一层之间传输时都会多"包一层"![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/IMG_0285.jpeg)

## IP服务模型
---

- 以数据报为基础
- 不可靠
- 尽最大努力
- 无连接的（Connectionless）
- IP服务设计的如此简单的原因
	1. 使网络保持简单、愚蠢（？）和最小化：更快，降低了建立和维护的成本
	2. 端到端原则：如果可以能在端点上正确地实现功能的话，那就这样做（而不是在网络上进行实现）
	3. 允许在顶部构建各种可靠（或不可靠）的服务
		- IP可以让程序选择对应的协议来服务其需求
	4. 可以工作在任何链接层：IP对链接层的限制很少（比如：可以是有线的也可以是无线的）

### 细节

1. IP尝试阻止数据包无限循环（当某个节点转发出现错误时就可能会发生）
	- IP会在每个数据报的标头中添加一个字段（称为生存时间或TTL（Time to Live字段），通过每个路由器时发生递减，达到0时，IP会将它视为陷入环路中，并将其丢弃
2. 如果数据包太长，IP会对其进行分段
3. IP使用标头来校验以减少将数据报传递到错误目的地的机会
4. 不同版本的IP
	- IPv4，32位地址
	- IPv6，128位地址
5. IP允许将新的字段添加到数据报的标头当中

# 1.4 Lift of a Packet