# 从一条网线到tcp/ip协议

## 1. 通过一条电线发送信息

![](./image/网线.png)

我们把电线中的0v和5v当做是两种不同的状态,0v代表数字0,5v代表1。我们就可以用电线来传递数字了

但是我们如何传递文字呢？英文？中文？

我们就可以使用编码，ASCII 来用数字表示字符

![](./image/电信号光信号.png)

## 2. 光纤和射频编码

网络的传递介质有很多，除了网线（电缆）之外，我们还可能有光纤（光缆）、无线电（wifi）

![](./image/光纤.png)

光纤的开和关可以表示0和1

![](./image/无线电.png)

简单的说你可以理解为无线电在周围会产生一个磁场，会不断的向外界发送一个不断变化的电场（不断改变电压值）

我们的手机或者其他可以上网的设备可以接收它，当然我目前还没弄明白它是怎么做的?

我们可以把其中一种波形表示为0,另外一种波形表示为1

## 3. 时钟同步与曼彻斯特编码

两台计算机交换数据，需要时钟同步，这个时钟可以理解为我们接收0和1的频率(时间)

发送方和接收方都必须保持在同一个时钟频率上，否则数据将会丢失

**一个时钟的脉冲就是在一个固定的时间内完成一个从0到1的一个变化过程**

发送方发送的时间要和接收方接收的时间要一致

![](./image/网络传输的介质.png)

如何才能保证时钟同步呢？

![](./image/调制信号.png)

- 不归零：高电平代表1，低电平代表0
- 归零：0的话向下跳，跳完归0，1的话向上跳，跳完归0
- 曼彻斯特：0的话从低电平跳向高电平，1的话从高电平跳向低电平
- 差分曼彻斯特：根据前一个电平的高低，0的话跳变，1的话不跳变。
- 调幅：0的话没有幅度，1的话幅度会大
- 调频：0的话频率小，1的话频率大
- 调相：波形不一样

> 注意一下它的时间间隔

## 4. 实际网络中的0和1是如何传递的

![](./image/如何处理0和1.png)

我们先了解下数据传输的一些相关单位

10M/bps:代表的含义是每秒传送10M的数据量(一千万个0和1)

100M/bps:每秒传送100M的数据量

1G/bps:每秒传送1G的数据量(0和1)

20G/bps:每秒传送20G的数据量(0和1)

![](./image/示波器.png)

x轴为时间间隔,这里的单位是100纳秒

y轴为电压,500mv一格，所以电压是在1v和-1v之间来回

图中代表着每一个0和1传递的波形(在100M/bps下)

看它的话就是看在100ns的结束点是从低电压 ---> 高电压 还是 高电压 ----> 低电压

> 我们需要注意的是计算机网络的传递数据是从后面往前传递的，注意我们的网络字节序是不一样的

## 5. 0和1如何变成网络中的帧

数据像流水一样不断的传递 0 和 1 ,虽然可以得到数据，但是我们怎么才能分辨哪些数据是整体

我们需要确定数据的边界

![](./image/以太网帧.png)

以太网帧刚开始的时候会经历一个96bit传输的静默期(保持0电压),表示我还没有开始

然后传递56个0和1交替组成，是以太网帧的7字节同步码

然后是以太网开始帧的定界符 10101011

接下来就可以读取数据了，每8位读取一字节的数据

最后还有个帧的校验（记录帧的大小，防止帧大小变化）

## 6. 网络帧中到底包含什么东西？

![](./image/以太网帧结构.png)

一旦帧开始传输数据，它包括了 6字节的源mac地址,6字节的目标mac地址，2字节的类型,46-1500字节大小的数据

以太网帧 = 6字节源mac地址  +  6字节的目标mac地址  + 2 字节的类型 + 46-1500字节的数据 + 4字节的校验

PPP也是比较常用的帧格式，常用在大型的主干网络中,但是我们在这里就忽略不计了.....

交换机直接和计算机相连，A计算机接到交换机的E0口，B接到交换机的E1口，A和B通信的时候，从E0口直接给E1口，数据只会从A计算机传递到B计算机

交换机的工作原理:交换机属于二层设备,可以识别以太网帧中的MAC地址信息,根据MAC地址转发,并把MAC地址与对应的端口记录在自己内部的地址表中

- 当交换机从某个端口收到一个数据包，它先读取包头中的源MAC地址，这样它就知道源MAC地址的机器是连在哪个端口上的
- 再去读取包头中的目的MAC地址，并在地址表中查找相应的端口
- 如表中有与这目的MAC地址对应的端口，把数据包直接复制到这端口上
- 如表中找不到相应的端口则把数据包广播到所有端口上，当目的机器对源机器回应时，交换机又可以学习一目的MAC地址与哪个端口对应，在下次传送数据时就不再需要对所有端口进行广播了
- 不断的循环这个过程，对于全网的MAC地址信息都可以学习到，二层交换机就是这样建立和维护它自己的地址表

> 交换机一般都有跟路由器的接口，如果找不到局域网内的MAC地址，就直接转发给路由器了,由路由器来解IP包

## 7. 七层协议

其实，我更愿意看成是四层模型 

数据链路层 -----> 网络层 -----> 传输层 ------> 应用层

以太网帧 -------> IP 包 ------> TCP包 ---------> HTTP应用包

以太网协议/PPP协议 ------> ARP/ICMP ------> TCP/UDP -------> HTTP/FTP/SMTP/DNS....



## 8. IP协议

加入IP协议的主要目的我觉得简单的说，或者是通俗点来说的话，就是我们需要路由器将不同的局域网连接起来，所以我们需要一个IP地址

当我们带着自己的IP地址 + 目标IP地址 + 源MAC地址 + 未知的目标MAC地址的时候,我们首先判断目标IP地址和自己的IP地址是否在同一个网段，这里分了两种情况

1. 同一个网段

- 发送ARP协议,询问所有的网络中的主机，谁的IP地址是目标IP地址(目标MAC ff::ff::ff交换机转发)
- 目标IP地址的主机回应ARP响应，告诉源IP地址，我是目标IP地址，并给到自己的MAC地址
- 这一次，它带着自己的IP地址 + 目标IP地址 + 源MAC地址 + 目标MAC地址直接将数据通过交换机传递给目标主机

2. 不同的网段

- 我们事先已经在电脑中设置了我们的网关 -- 也就是我们的路由器的IP地址
- 发送ARP协议，询问所有的网络中的主机，谁的IP地址是网关(目标MAC ff::ff::ff交换机转发)
- 路由器响应ARP,告诉源IP地址，我是目标IP地址，并给到自己的MAC地址
- 我们带着IP地址 + 目标IP地址 + 源MAC地址 + 目标MAC地址 通过交换机发送给路由器



## 9. 一步一步的路由

路由器拿到之后，会根据路由表不断的变化MAC地址将数据传递出去


- 路由首先查看自己的路由表，看看路由表中是否有目标IP对应的出口，如果没有，则丢掉
- 如果有，将数据转发到对应的出口，修改源mac地址是出口，目标MAC地址是下一跳
- 然后找自己的ARP缓存中是否缓存了下一跳的MAC地址，如果有直接发过去，如果没有的话，发送ARP请求下一跳的MAC地址，并记录到ARP缓存表中
- 最后重新封装成帧发送出去

![](./image/路由跳转1.png)

![](./image/路由跳转2.png)


## 10. TCP

数据链路层在交换机 ---> 主机之间传递,交换机只能根据MAC地址进行转发，并不能识别IP包的内容

网络层工作在 路由器 ----> 路由器之间,根据路由表中的IP地址对应的出口进行转发,需要修改目标MAC地址为下一跳的路由MAC地址,它会向所有连接的设备(交换机、主机)发送ARP来确定下一跳的MAC地址

传输层才是真正负责数据传输部分的。。。假设我们什么都不管，网络中一定会出现丢包、延迟发送、堵塞等一系列的问题的

而TCP就是为了解决拥塞控制(滑动窗口、拥塞窗口)、可靠传输(重传)的

- 滑动窗口



- 拥塞窗口



- 重传机制


## 11. Http -> TCP ---> IP ---> 帧

我们在传输数据时，可以只使用（传输层）TCP/IP协议，但是那样的话，如果没有应用层，便无法识别数据内容，如果想要使传输的数据有意义，则必须使用到应用层协议，应用层协议有很多，比如HTTP、FTP、TELNET等，也可以自己定义应用层协议。WEB使用HTTP协议作应用层协议，以封装HTTP文本信息，然后使用TCP/IP做传输层协议将它发到网络上

客户端 ------------SYN--------------->  服务器

客户端 <-----------ACK + SYN ---------------- 服务器

客户端 -------------ACK--------------> 服务器

客户端http请求 index.html ------------request Header---------------> 服务器

客户端http请求 /image/logo.png ------------request Header---------------> 服务器

客户端http请求 /lib/index.js ------------request Header---------------> 服务器

客户端http请求 /css/main.css ------------request Header---------------> 服务器

客户端接收  <------------response Header index.html --------------- 服务器

客户端接收  <------------response Header logo.png --------------- 服务器

客户端接收  <------------response Header index.js --------------- 服务器

客户端接收  <------------response Header main.css --------------- 服务器

客户端 ------------FIN + ACK--------------->  服务器

客户端 <-----------ACK---------------- 服务器

客户端 <-------------FIN + ACK-------------- 服务器

客户端 -------------ACK--------------> 服务器


                                                                                      http:request Header + request content
                                   tcp:port + ack + sequeue + window        















