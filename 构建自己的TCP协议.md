# 构建自己的TCP协议

## 可靠的概念以及完成的服务

- 基础的数据传输

TCP能够在其用户之间的每个方向传输连续的字节流,将一些字节数据打包成段,通过互联网传输.

一般情况下,TCP根据自己的情况来决定阻止和转发数据

有时候,用户需要确定他们提交给TCP的所有数据都已传输

为此，定义了推送功能。(设置PSH标记来提示接收方立即处理这些数据,而不必等待更多数据到达)

推送会使TCP迅速转发并将该点之前的数据传递给接收方


- 可靠性

TCP必须从因特网通信系统损坏、丢失、复制或乱序传送的数据中恢复

这是通过给传输的每个字节分配一个序列号,并要求接受的TCP回复一个确认(ACK)来实现的.

如果在规定的时间间隔内没有收到ACK,则重传数据.

在接收方,序列号用来顺序排列可能接收到的乱序的片段,并消除重复的片段.

通过在传输的每个片段上添加一个校验和,在接收方进行检查,并丢弃损坏的片段.

只要TCP各端继续正常运行,网络系统也没有断开,传输错误就不会影响到用户.

- 流量控制

TCP为接收方提供了一种方法来管理发送方发送的数据量,注意这里是接收方可以管理发送方的发送速度

这是通过在每个ACK中返回一个`窗口`来实现的,窗口表示在成功接收的最后一个片段之外的可接受的序列号范围

该窗口表示发送方在接受到进一步确认之前可以传输的字节数量

这里的流量控制实际是为了防止发送方发送过快,接收方无法进行处理设置的....


- 多路复用

在TCP协议中,为了让同一个主机上的多个进程可以同时进行通信,TCP使用了`端口`来区分不同的通信通道,每个进程通过特定的端口进行网络通信

一个套接字由IP地址和端口号组合而成,每个TCP连接都是由一对套接字唯一标识的.

例如192.168.1.1:8080与192.168.1.2:9090这一对套接字就唯一标识了这条TCP连接



- 连接

上述的可靠性和流量控制机制要求TCP初始化和维护每个水流的某些状态信息

这些信息(包括套接字、序列号和窗口大小)的组合称为连接

每个连接都由标识其两端的一对套接字唯一指定

当两个进程想要进行通信的时候,他们必须先建立TCP连接(初始化每一端的状态信息)

当他们的通信完成后,连接被终止或关闭,以释放资源用于其他用途


## 可靠通信

通过TCP连接上发送的数据流在目的地可靠且有序的传送

通过使用序号和确认机制,使得传输变得可靠

从概念上讲,每个字节的数据都分配有一个序列号

TCP段中数据的第一个字节的序号是与该TCP段一起传输的序列号,称为segment sequence number

TCP段还携带一个确认号码，这是期望对方传输的下一个字节数据包的序列号

当TCP传输一个TCP段时,它会将TCP段的一个副本放在重传队列中,并启动一个计时器;当收到该数据的确认时,则将TCP段从重传队列中删除

如果在定时器结束之前没有收到确认,则重传该TCP段

为了管理进入TCP的数据流,采用了流量控制机制

数据接收TCP向发送TCP报告一个窗口

该窗口指定字节的数量,从数据接收的TCP目前准备接收的确认号码开始


## 数据通信

当应用程序调用`SEND`时,数据会被放入TCP的发送缓冲区,发送缓冲区中的数据不会立即发送,而是由TCP协议根据自己的传输策略(如流量控制、拥塞控制)来决定何时发送

应用程序可以通过`PUSH`标志告诉TCP将数据立即发送出去,`PUSH`标志是对TCP的提示,要求TCP不要再等待,必须尽快发送当前所有未发送的数据

TCP协议可以将从应用程序收到的数据进行分片,例如一次`SEND`调用中的数据可能会被拆成多个TCP段发送出去,或者多个`SEND`调用中的数据可以被合并到一个段中

单个TCP段中的数据可以来源于一次完整的`SEND`调用,也可能是一次`SEND`调用的一部分,或者包括多个`SEND`调用的数据

## 头部格式


```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |          Source Port          |       Destination Port        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                        Sequence Number                        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Acknowledgment Number                      |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |  Data |           |U|A|P|R|S|F|                               |
  | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
  |       |           |G|K|H|T|N|N|                               |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |           Checksum            |         Urgent Pointer        |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                    Options                    |    Padding    |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  |                             data                              |
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```

Source Port : 源端口号 (2字节)

Destination Port: 目标端口号 (2字节)

Sequence Number: 该段数据中的第一个字节的序列号,如果存在SYN，则序列号是初始序列号ISN,第一个字节数据就是ISN + 1 (4字节)

Acknowledgment Number: 如果有ACK标记,这个字段表示发送者期望收到的下一个序列号的值.一旦建立了连接,一直会发送这个字段 (4字节)

Data Offset: 这个数表示TCP头部的长度有多少个32bit,表示真正数据开始的位置,TCP头部的长度是32bit的整数倍 (4位)

Reserved: 保留将来使用，全部为0 (6位)


Control bits: (6位)

- URG 紧急标记
- ACK 确认标记
- PSH 推送标记
- RST 重置标记
- SYN 同步标记
- FIN 断开连接标记

window: 从确认序号开始,发送方可以接受的字节数 (2字节)

checkNum: 检验和是头部和数据部分所有分割成16bit数的经过二进制反码求和得到的数 (2字节)

Urgent Pointer : 这个字段表示当前紧急指针距离本段中序列号的正偏移，紧急指针指向紧急数据后的字节的序列号,这个字段只应在设置了URG标志的TCP段中使用  (2字节)


Options: 可选字段,用于实现基本TCP头部无法实现的功能. (长度不固定,字节为单位)

```
// 一种情况下是仅仅只有一个字节的选项类型
// 一种情况下是一字节的选项类型,以字节的选项长度和真正的选项数据
// length包括了kind、length以及数据所有的字节数
Kind     Length    Meaning
----     ------    -------
 0         -       End of option list.
 1         -       No-Operation.
100        -       Reserved.
105        4       Buffer Size.
```

```
+--------+
|00000000|
+--------+
Kind=0
```

End of Option List该选项代码表示选项列表的结束


```
+--------+
|00000001|
+--------+
Kind=1
```

No-Operation这个也没有什么实际的意义


```
+--------+--------+---------+--------+
|01000101|00000100|    buffer size   |
+--------+--------+---------+--------+
Kind=105 Length=4
```

Buffer Size如果该选项存在，那么它在发送该段的 TCP 上传达接收缓冲区的大小

Padding: 这个用于确保TCP头部的结束和数据的开始都在4字节的边界上,由0进行填充

## 术语表

发送序列变量

```
SND.UNA - send unacknowledged  等待确认已经发送的
SND.NXT - send next 可靠传输的思路
SND.WND - send window 流量控制
SND.UP  - send urgent pointer 数据优先处理
SND.WL1 - segment sequence number used for last window update
SND.WL2 - segment acknowledgment number used for last window update
ISS     - initial send sequence number 握手

// WL1是对面更新窗口大小的时候,数据段的序号
// WL2是对面更新窗口大小的时候,数据段的确认号
// 记录接收方窗口更新的时候,接收方数据发送了多少数据,并确认了多少数据
// 发送方可根据这个,来调整自己的发送速度
```

接受序列变量

```
RCV.NXT - receive next
RCV.WND - receive window
RCV.UP  - receive urgent pointer
IRS     - initial receive sequence number
```

发送空间

```
    1         2          3          4
----------|----------|----------|----------
SND.UNA    SND.NXT    SND.UNA
                     +SND.WND 表示结束点

1 - old sequence numbers which have been acknowledged
2 - sequence numbers of unacknowledged data
3 - sequence numbers allowed for new data transmission
4 - future sequence numbers which are not yet allowed
```

接收空间

```
  1          2          3
----------|----------|----------
RCV.NXT    RCV.NXT
          +RCV.WND

1 - old sequence numbers which have been acknowledged
2 - sequence numbers allowed for new reception
3 - future sequence numbers which are not yet allowed
```

当前TCP段的变量

```
SEG.SEQ - segment sequence number 数据段的序列号
SEG.ACK - segment acknowledgment number 确认号
SEG.LEN - segment length 数据段的长度
SEG.WND - segment window 数据段的窗口大小
SEG.UP  - segment urgent pointer
SEG.PRC - segment precedence value
```


TCP连接状态

```
                              +---------+ ---------\      active OPEN
                              |  CLOSED |            \    -----------
                              +---------+<---------\   \   create TCB
                                |     ^              \   \  snd SYN
                   passive OPEN |     |   CLOSE        \   \
                   ------------ |     | ----------       \   \
                    create TCB  |     | delete TCB         \   \
                                V     |                      \   \
                              +---------+            CLOSE    |    \
                              |  LISTEN |          ---------- |     |
                              +---------+          delete TCB |     |
                   rcv SYN      |     |     SEND              |     |
                  -----------   |     |    -------            |     V
 +---------+      snd SYN,ACK  /       \   snd SYN          +---------+
 |         |<-----------------           ------------------>|         |
 |   SYN   |                    rcv SYN                     |   SYN   |
 |   RCVD  |<-----------------------------------------------|   SENT  |
 |         |                    snd ACK                     |         |
 |         |------------------           -------------------|         |
 +---------+   rcv ACK of SYN  \       /  rcv SYN,ACK       +---------+
   |           --------------   |     |   -----------
   |                  x         |     |     snd ACK
   |                            V     V
   |  CLOSE                   +---------+
   | -------                  |  ESTAB  |
   | snd FIN                  +---------+
   |                   CLOSE    |     |    rcv FIN
   V                  -------   |     |    -------
 +---------+          snd FIN  /       \   snd ACK          +---------+
 |  FIN    |<-----------------           ------------------>|  CLOSE  |
 | WAIT-1  |------------------                              |   WAIT  |
 +---------+          rcv FIN  \                            +---------+
   | rcv ACK of FIN   -------   |                            CLOSE  |
   | --------------   snd ACK   |                           ------- |
   V        x                   V                           snd FIN V
 +---------+                  +---------+                   +---------+
 |FINWAIT-2|                  | CLOSING |                   | LAST-ACK|
 +---------+                  +---------+                   +---------+
   |                rcv ACK of FIN |                 rcv ACK of FIN |
   |  rcv FIN       -------------- |    Timeout=2MSL -------------- |
   |  -------              x       V    ------------        x       V
    \ snd ACK                 +---------+delete TCB         +---------+
     ------------------------>|TIME WAIT|------------------>| CLOSED  |
                              +---------+                   +---------+
```


## 序列号

TCP设计中的一个基本的概念就是通过TCP连接发送的每个字节的数据都有一个序列号

由于每个字节都是有顺序的,所以每个字节都可以被确认

TCP所采用的确认机制是累积性的,因此序列号为X的确认表示已经收到了之前但不包括X的所有字节

这种机制使得在存在重传的情况下可以直接进行重复检测

假设发送方发送了1-100,接收方可能受到了1-90,并发送ack确认后为91。

一个新的可接受的确认要满足

```
// 在确认当前的数据之前,必须先确认之前的所有的已经得到了确认,不然无法进行确认
// NXT代表着即将要发送的数据,还没有发送
SND.UNA < SEG.ACK(可接受的ACK) =< SND.NXT

```

对于每个连接,都有一个发送序列号和接收序列号

初始发送序列号(ISS)由发送方的TCP选择,初始接受序列号(IRS)在连接建立过程中得到

如果要建立或初始化的连接,两个TCP必须同步对方的初始化序列号

这是通过交换建立连接的信息来完成的,这些信息带有一个称为`SYN`的控制位和初始序列号

1) A –> B 同步自己的序列号 X
2) A <– B 确认你的序列号是 X
3) A <– B 同步自己的序列号 Y
4) A –> B 确认你的序列号是 Y

第二步和第三步可以结合在一个消息中,三次握手


## 建立连接

三次握手是用于建立连接的过程

这个过程通常由一个TCP发起,由另一个TCP响应

如果两个TCP同时发起连接,该过程也应正常工作

当同时尝试建立连接的时候,TCP在发送`SYN`后,收到没有携带确认的`SYN`段

当然,当接收者受到一个旧的重复的`SYN`段时,有可能会认为是同时建立连接

```
      TCP A                                                 TCP B
  1.  CLOSED                                                LISTEN
  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>                --> SYN-RECEIVED
  3.  ESTABLISHED <-- <SEQ=300><ACK=101><CTL=SYN,ACK>   <-- SYN-RECEIVED
  4.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK>       --> ESTABLISHED
  5.  ESTABLISHED --> <SEQ=101><ACK=301><CTL=ACK><DATA> --> ESTABLISHED
```
TCP A 开始发送一个`SYN`段,表明它将使用从序号100开始的序列号

第三行,TCP B 发送了一个SYN,并确认了它从TCP A收到的SYN(SYN + ACK)

第四行,TCP A发送了一个包含ACK的空段回应TCP B中的SYN

第五行,TCP A开始发送数据了....

同时建立连接稍微复杂一点

```
  TCP A                                                TCP B
  1.  CLOSED                                           CLOSED
  2.  SYN-SENT     --> <SEQ=100><CTL=SYN>              ...
  3.  SYN-RECEIVED <-- <SEQ=300><CTL=SYN>              <-- SYN-SENT
  4.               ... <SEQ=100><CTL=SYN>              --> SYN-RECEIVED
  5.  SYN-RECEIVED --> <SEQ=100><ACK=301><CTL=SYN,ACK> ...
  6.  ESTABLISHED  <-- <SEQ=300><ACK=101><CTL=SYN,ACK> <-- SYN-RECEIVED
  7.               ... <SEQ=101><ACK=301><CTL=ACK>     --> ESTABLISHED
```

大致是这个样子的

```
A --> SYN --> B
A <-- SYN <-- B
A --> SYN --> B

A --> SYN +ACK --> B
A <-- SYN +ACK <-- B
A --> ACK --> B
```

从之前重复SYN中恢复

```
 TCP A                                                TCP B
  1.  CLOSED                                               LISTEN
  2.  SYN-SENT    --> <SEQ=100><CTL=SYN>               ...
  3.  (duplicate) ... <SEQ=90><CTL=SYN>                --> SYN-RECEIVED
  4.  SYN-SENT    <-- <SEQ=300><ACK=91><CTL=SYN,ACK>   <-- SYN-RECEIVED
  5.  SYN-SENT    --> <SEQ=91><CTL=RST>                --> LISTEN
  6.              ... <SEQ=100><CTL=SYN>               --> SYN-RECEIVED
  7.  SYN-SENT    <-- <SEQ=400><ACK=101><CTL=SYN,ACK>  <-- SYN-RECEIVED
  8.  ESTABLISHED --> <SEQ=101><ACK=401><CTL=ACK>      --> ESTABLISHED
```

第三行中,一个之前重复的SYN 达到了 TCP B

TCP B无法判断这是之前的SYN,所以它正常响应

TCP A检测到ACK的字段不正确,然后返回了一个RST(重置),同时选择SEQ字段以使该TCP段可信

TCP B则收到RST之后,返回到Listen状态


如果其中一个TCP在另一个不知道的情况下关闭或终止了连接,或者连接的两端由于崩溃导致内存丢失而变得不同步，则已建立的连接称为”半开放“

如果尝试向任一方向发送数据，这种连接将自动重置

```
 TCP A                                           TCP B
  1.  (CRASH)                               (send 300,receive 100)
  2.  CLOSED                                           ESTABLISHED
  3.  SYN-SENT --> <SEQ=400><CTL=SYN>              --> (??)
  4.  (!!)     <-- <SEQ=300><ACK=100><CTL=ACK>     <-- ESTABLISHED
  5.  SYN-SENT --> <SEQ=100><CTL=RST>              --> (Abort!!)
  6.  SYN-SENT                                         CLOSED
  7.  SYN-SENT --> <SEQ=400><CTL=SYN>              -->
```

TCP A崩溃后,用户试图重新打开连接

在此期间，TCP B认为连接是打开的

在第三行,当SYN到达的时候，TCP B处于同步状态,而接收端在接收窗口之外,返回一个确认,ACK=100,表示它期望收到的下一个序列号

TCP A看到这个TCP段没有确认它所发送的任何东西,由于不同步，发送了一个重置(RST),因为它检测到一个半开放的连接

在第五行的时候，TCP B终止了

TCP A 会继续尝试建立连接,开始三次握手了...


```
  TCP A                                         TCP B
  1.  LISTEN                                        LISTEN
  2.       ... <SEQ=Z><CTL=SYN>                -->  SYN-RECEIVED
  3.  (??) <-- <SEQ=X><ACK=Z+1><CTL=SYN,ACK>   <--  SYN-RECEIVED
  4.       --> <SEQ=Z+1><CTL=RST>              -->  (return to LISTEN!)
  5.  LISTEN                                        LISTEN
```

上述也是导致RST的一种情况,在这里就不多解释了

三种情况导致发送RST








## 关闭连接




## 数据通信







































- 连接


- 优先级和安全性
