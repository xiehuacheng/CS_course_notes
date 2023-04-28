
## 1.1 什么是Internet？
---

通过网络连成的网络，网络的网络，”网际“

松散的层次结构，互连的ISP

### 从构成的角度

#### 硬件

节点
- 主机及其上运行的应用程序
- 路由器、交换机等网络交换设备

边：通信链路（link）
- 接入网链路：主机连接到互联网的链路
- 主干链路：路由器间的链路

主机（host）=端系统（end system）

链路传输速率=带宽（单位bps/bit pre second）

分组交换设备：转发分组（packets）
- 路由器（router）和交换机（switch）

#### 协议

对等层的实体在通信过程中应当遵守的规则集合

协议可以分为不同的层次

定义了在两个或多个通信实体之间交换的**报文格式**和**次序**，以及在保温传输和/或接收或其他事件方面采取的**动作**

常见的协议：TCP、IP、HTTP、FTP、PPP

### 从服务的角度

#### 分布式的应用进程

运行在主机部分的应用层中

网络存在的理由

#### 为应用进程提供通信服务的基础设施

主机部分中应用层之下的部分以及发送主机和目标主机之间的网络部分

提供服务的方式：API（也称为Socket API）

服务的形式
1. 无连接不可靠服务（由UDP提供）
2. 面向连接的可靠服务（由TCP提供）

## 1.2 网络边缘(edge)
---

- 主机
- 应用程序（客户端和服务器）

### 两种通信模式

#### 客户端/服务器模式

服务器为主（先运行起来）

客户端发送请求，服务器接收请求

如：Web浏览器/浏览器；email客户端/服务器

可扩展性较差

#### 对等(peer-peer,p2p)模式

很少（甚至没有）专门的服务器

每个节点既是客户端也是服务器；每个节点可以向其他节点请求资源片段，也可以为其他节点提供资源片段

分布式文件分发系统

### 两种通信方式

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230425165320.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230425165442.png)

#### 面向连接的服务(TCP)

对可靠性需求较强的应用使用

#### 无连接的服务(UDP)

对及时性需求较强的应用使用

事务性的应用（例如：域名查询），一次通信即可完成

## 1.3 网络核心(core)
---

### 基本概念

由多个分布式系统连接而成，网络的网络

作用：数据交换

### 方式

#### 电路交换（circuit switch）

- 用于传统电话网

- 在两台主机之间直接建立单独的连接（需要耗费时间）

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427154053.png)

网络资源（如带宽）被分成片（piece）
- 为呼叫分配片
- 如果某个呼叫没有数据，则其资源片处于空闲状态（不共享）

分片方式：
1. 频分(FDM)
2. 时分(TDM)
3. 波分(WDM)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427155715.png)

#### 分组交换（package switch）

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427161340.png)

- 数据以包（package）的形式传输，每个节点对包进行存储再对其进行转发（为了链路的共享），所以每个节点需要耗费的时间更多（从原来一个bit的接收时间变成一个package的接收时间）

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427160858.png)

- 排队延迟由当前的网络状态来决定，当某个节点的到达速率/输出速率的比值（流量强度）接近1时，排队延迟将会暴涨

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427161256.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427161503.png)

- 可以看作是一个动态的复用方式（按需分配）

### 对比

- 同样的网络资源，分组交换允许更多的用户使用网络

分组交换的优缺点

适合于对突发式数据传输
- 资源共享
- 简单，不必建立呼叫

过度使用会造成==网络拥塞==：分组延时和丢失
- 对可靠地数据传输需要协议来约束：拥塞控制

### 分组交换的两种形式

分组的存储转发一段一段从源端传到目标端，按照有无网络层的连接，分成以下两种

#### 数据报网络

- 数据报(Datagram)，其中携带了目标主机的地址信息（类似于快递单）

- 分组的目标地址决定下一跳
- 在不同的阶段，路由可以改变，发往同一个目标主机的包可能会走不同的路径
- 类似：问路
- Internent

#### 虚电路网络

- 每个分组都带标签（虚电路标识 VC ID），标签决定下一跳
- 在呼叫建立时决定路径，在整个呼叫中路径保持不变，依靠信令来建立
- 路由器维持每个呼叫的状态信息
- X.25 和ATM

## 1.4 接入网(access)和物理媒体
---

物理媒体：有线或者无线通信链路

- 带宽（bps）
- 共享/专用

将边缘接入核心

### 住宅接入：modem（调制解调器）

- 利用已有的电话线
- 将上网数据调制加载到音频信号上，在局端将其中的数据解调出来；反之亦然
- 速率慢（56Kbps）
- 不能同时打电话和上网

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427165811.png)

- 电话和网络用的频段没有重叠，所以可以同时运行

### 线缆网络

- 采用有线电视信号线缆
- 和电话线原理类似，但是属于共享网络

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427170424.png)

### 企业接入网络(Ethernet，以太网)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427171359.png)

### 无线接入网络

各无线端系统共享无线接入网络（端系统到无线路由器）
- 通过基站或者叫接入点

### 物理媒体（介质）

- 导引性媒体（衰减小，传得更远）和非导引性媒体

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427171621.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427171650.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230427171752.png)

- 无线链路受到的干扰大，随距离的衰减快

## 1.5 Internet结构和ISP
---

### 互联网是网络的网络

- 端系统通过接入==ISPs (Internet Service Providers)==连接到互联网
	- 比如：住宅、公司、通信运营商、学校等
- 接入ISPs相应的必须是互联的
	- 任何2个端系统可相互发送分组到对方

- ISP之间的连接方式
	- 全连接的代价太大O(N^2)，不可扩展
	- 全局ISP，代价小，可扩展性强![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230428235839.png)
	- 竞争与合作![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230428235954.png)
	- 业务细分![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429000510.png)
	- ICP(Internet Content Provider)的专网（也是需要接入ISP中的）![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429000731.png)
		- 原因1：ICP收费太高
		- 原因2：无法为距离较远的地区提供优质的服务
		- 方式：在各地布设数据中心(DC)、机房以及线缆，相当于自己布设了专网
		- 连接若干local ISP和各级（包括一层）ISP,更加靠近用户

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429002120.png)

### 结构

- 松散的层次模型

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429001510.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429001752.png)

![](https://orange-pictures.oss-cn-guangzhou.aliyuncs.com/img/20230429001812.png)

## 1.6 分组延时、丢失和吞吐量
---

## 1.7 协议层次及服务模型
---

## 1.8 历史
---
