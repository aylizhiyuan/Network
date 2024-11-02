# 构建自己的TCP协议

## 概念

### 基础的数据传输

TCP能够在其用户之间的每个方向传输连续的字节流,将一些字节数据打包成段,通过互联网传输.

一般情况下,TCP根据自己的情况来决定阻止和转发数据

有时候,用户需要确定他们提交给TCP的所有数据都已传输

为此，定义了推送功能。(设置PSH标记来提示接收方立即处理这些数据,而不必等待更多数据到达)

推送会使TCP迅速转发并将该点之前的数据传递给接收方

```

 A ----------- SEQ-A -------------> B // 发送A的数据
 A <----------- ACK --------------- B // 确认A的数据
 A <------------- SEQ-B ----------- B // 发送B的数据
 A ------------- ACK -------------> B // 确认B的数据

```


### 可靠性

TCP必须从因特网通信系统损坏、丢失、复制或乱序传送的数据中恢复

这是通过给传输的每个字节分配一个序列号,并要求接受的TCP回复一个确认(ACK)来实现的.

如果在规定的时间间隔内没有收到ACK,则重传数据.

在接收方,序列号用来顺序排列可能接收到的乱序的片段,并消除重复的片段.

通过在传输的每个片段上添加一个校验和,在接收方进行检查,并丢弃损坏的片段.

只要TCP各端继续正常运行,网络系统也没有断开,传输错误就不会影响到用户.

### 流量控制

TCP为接收方提供了一种方法来管理发送方发送的数据量,注意这里是接收方可以管理发送方的发送速度

这是通过在每个ACK中返回一个`窗口`来实现的,窗口表示在成功接收的最后一个片段之外的可接受的序列号范围

该窗口表示发送方在接受到进一步确认之前可以传输的字节数量

这里的流量控制实际是为了防止发送方发送过快,接收方无法进行处理设置的....


### 多路复用

在TCP协议中,为了让同一个主机上的多个进程可以同时进行通信,TCP使用了`端口`来区分不同的通信通道,每个进程通过特定的端口进行网络通信

一个套接字由IP地址和端口号组合而成,每个TCP连接都是由一对套接字唯一标识的.

例如192.168.1.1:8080与192.168.1.2:9090这一对套接字就唯一标识了这条TCP连接



### 连接

上述的可靠性和流量控制机制要求TCP初始化和维护每个水流的某些状态信息

这些信息(包括套接字、序列号和窗口大小)的组合称为连接

每个连接都由标识其两端的一对套接字唯一指定

当两个进程想要进行通信的时候,他们必须先建立TCP连接(初始化每一端的状态信息)

当他们的通信完成后,连接被终止或关闭,以释放资源用于其他用途


### 可靠通信

通过TCP连接上发送的数据流在目的地可靠且有序的传送

通过使用序号和确认机制,使得传输变得可靠

从概念上讲,每个字节的数据都分配有一个序列号

TCP段中数据的第一个字节的序号是与该TCP段一起传输的序列号,称为`segment sequence number`

TCP段还携带一个确认号码，这是期望对方传输的下一个字节数据包的序列号

当TCP传输一个TCP段时,它会将TCP段的一个副本放在重传队列中,并启动一个计时器;当收到该数据的确认时,则将TCP段从重传队列中删除

如果在定时器结束之前没有收到确认,则重传该TCP段

为了管理进入TCP的数据流,采用了流量控制机制

数据接收TCP向发送TCP报告一个窗口

该窗口指定字节的数量,从数据接收的TCP目前准备接收的确认号码开始


### 数据通信

当应用程序调用`SEND`时,数据会被放入TCP的发送缓冲区,发送缓冲区中的数据不会立即发送,而是由TCP协议根据自己的传输策略(如流量控制、拥塞控制)来决定何时发送

应用程序可以通过`PUSH`标志告诉TCP将数据立即发送出去,`PUSH`标志是对TCP的提示,要求TCP不要再等待,必须尽快发送当前所有未发送的数据

TCP协议可以将从应用程序收到的数据进行分片,例如一次`SEND`调用中的数据可能会被拆成多个TCP段发送出去,或者多个`SEND`调用中的数据可以被合并到一个段中

单个TCP段中的数据可以来源于一次完整的`SEND`调用,也可能是一次`SEND`调用的一部分,或者包括多个`SEND`调用的数据

### 头部格式


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

### 术语表

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


### 序列号

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

### 1.正常三次握手

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

### 2.同时建立连接

```
  TCP A                                                TCP B
  1.  CLOSED                                           CLOSED
  2.  SYN-SENT     --> <SEQ=100><CTL=SYN>              ...
  3.  SYN-RECEIVED <-- <SEQ=300><CTL=SYN>              <-- SYN-SENT
  4.               ... <SEQ=100><CTL=SYN>              --> SYN-RECEIVED // 一般不会发生这个,可忽略
  5.  SYN-RECEIVED --> <SEQ=100><ACK=301><CTL=SYN,ACK> ...
  6.  ESTABLISHED  <-- <SEQ=300><ACK=101><CTL=SYN,ACK> <-- SYN-RECEIVED
  7.               ... <SEQ=101><ACK=301><CTL=ACK>     --> ESTABLISHED
```

大致是这个样子的

```
A --> SYN --> B
A <-- SYN <-- B

A --> SYN +ACK --> B
A <-- SYN +ACK <-- B
A --> ACK --> B
```

### 3.从之前重复SYN中恢复

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

- 如果连接不存在(CLOSED),那么除了收到的是一个重置段之外,对其他任何收到的段都会回复一个重置段,特别的是,通过这种方式拒绝向不存在的连接发送的SYN,如果接收的段有ACK字段,重置从该段的ACK字段中获取其序列号，否则重置的序列号为0，ACK字段设置为接收段的序列号和段长之和

- 如果连接处于任何非同步状态(LISTEN/SYN-SENT/SYN-RECEIVED),并且接收段确认没有发送内容,或者如果接收段具有安全等级或区段与连接请求的层级和区段不完全匹配，则会发送重置

- 如果连接处于同步状态（ESTABLISHED、FIN-WAIT-1、FIN-WAIT-2、CLOSE-WAIT、CLOSING、LAST-ACK、TIME-WAIT），任何无效的段（超出窗口序列号或无效的确认号）只需回复一个空确认段，其中包含当前发送序列号和一个表示下一个预期接收序列号的确认，并且连接保持相同状态


## 关闭连接

CLOSE是一个操作,意思是我没有更多的数据要发送了

主动关闭的用户可以继续接收，知道他被告知另一方也关闭了

因此,一个程序可以多次发送，然后关闭，再继续接收，直到有信号说接收失败，因为对方已经关闭了

TCP将在连接关闭前可靠的发送所有缓冲区的数据，因此没有数据接收的用户只需等到连接被成功关闭，就能知道他的所有数据已经成功发送到目的地的TCP



### 1.用户主动告诉TCP关闭连接,例如调用close()方法

在这种情况下,会生成一个FIN段,并将其加入到发送段队列中

TCP将不再接受用户的发送,并进入到FIN-WAIT-1状态

在这种状态下,允许接收

在FIN之前和包括FIN在内的所有段超时将被重传,直到被确认

当另一个TCP既确认了FIN又发送了自己的FIN的时候,第一个TCP可以对这个FIN进行ACK



### 2.远程TCP通过发送FIN控制标志开始关闭

如果来自网络的未经请求的FIN到达，接收的TCP可以确认该FIN并告诉用户连接正在关闭

用户应该用CLOSE来回应,在此基础上,TCP可以向另一端TCP发送FIN

然后TCP等待,直到它自己的FIN被确认，然后它删除连接

如果没有收到ACK,在超时后，连接将被终止,并告诉用户

### 3.两个用户同时关闭

连接两端的用户同时关闭会交换FIN段

当FIN之前的所有段都被处理并确认后，每个TCP可以对它所收到的FIN进行ACK

两者都将在收到这些ACK后，删除连接


***正常四次挥手的情况***

```
TCP A                                                TCP B
  1.  ESTABLISHED                                          ESTABLISHED
  2.  (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  --> CLOSE-WAIT
  3.  FIN-WAIT-2  <-- <SEQ=300><ACK=101><CTL=ACK>      <-- CLOSE-WAIT
  4.                                                       (Close)
      TIME-WAIT   <-- <SEQ=300><ACK=101><CTL=FIN,ACK>  <-- LAST-ACK
  5.  TIME-WAIT   --> <SEQ=101><ACK=301><CTL=ACK>      --> CLOSED
  6.  (2 MSL)
      CLOSED
```

***同时关闭的情况***

```
  TCP A                                                TCP B
  1.  ESTABLISHED                                          ESTABLISHED
  2.  (Close)                                              (Close)
      FIN-WAIT-1  --> <SEQ=100><ACK=300><CTL=FIN,ACK>  ... FIN-WAIT-1
                  <-- <SEQ=300><ACK=100><CTL=FIN,ACK>  <--
                  ... <SEQ=100><ACK=300><CTL=FIN,ACK>  --> // 忽略这个即可
  3.  CLOSING     --> <SEQ=101><ACK=301><CTL=ACK>      ... CLOSING
                  <-- <SEQ=301><ACK=101><CTL=ACK>      <--
                  ... <SEQ=101><ACK=301><CTL=ACK>      --> // 忽略这个即可
  4.  TIME-WAIT                                            TIME-WAIT
      (2 MSL)                                              (2 MSL)
      CLOSED                                               CLOSED
```

## 数据通信

- A发送一个SYN,序号为1000,此时,SND.NXT = 1001(A下一个将要发送的数据段的序列号),SND.UNA = 1000(已经发送但是没有确认)

- B收到了这个SYN,并返回SYN-ACK(同步确认)段,序列号为2000,确认号为1001,表示B期望收到A发送的下一个序列号,此时,RCV.NXT = 1001 

- A收到B的确认后,发送ACK段，确认B的序列号2001，连接建立

假设A要发送一个500字节的数据段，序列号从1001开始

- A将SND.NXT推进到1001 + 500 = 1501,下一个即将发送序列号为1501

- A等待B的确认,并保持SND.UNA = 1001,最早未确认的序列号是1001

- B收到数据后,检查序列号为1001,符合预期的RCV.NXT = 1001,说明数据是顺利到达的

- B处理数据,并将RCV.NXT更新为 1001 + 500 = 1501,表示下一个期望的序列号为1501

- B发送确认段,确认号为1501,告诉A我已经收到了序列号1001到1500之间的数据了

- A收到了B发来的确认段,确认号位1501

- A将SND.UNA更新为1501,表示之前的所有数据都已经成功接受并确认了

- 假设A继续发送序列号为1501的数据段,长度为500字节,但是由于网络原因，B没有收到这个数据段

- A等待了一段时候后未收到确认(超时),会自动重传序列号1501的数据段,超时定时器只会观察SYN.UNA的确认

- 在此期间，A不会停止发送它的数据,依然会继续的发送序号为2001的数据,A启动超时计时器,等待1501的数据确认,如果超时,A则会重传1501的数据段,而不影响后续数据的传输

- B预期接收到的是1501,但是实际接收到的是2001, B将会缓存乱序的数据,并且B会发送一个重复的ACK,确认号位1500,表明自己依然在等待1501的数据段

- A收到了重复的ACK(确认号1500),就知道1501的数据可能丢失了,启动快速重传机制,重新发送1501的数据段

- B收到了1501的数据段后,将其与之前缓存的数据段进行拼接,更更新自己的RCV.NXT,B在没有收到1501之前,是不会更新自己的RVC.NXT的,而是将数据缓存起来

所以,假如是A的定时触发了(未正常收到1501的ACK),那么A则会重传1501号数据段,B在收到后则会跟缓存的乱序数据一起确认,那么还有一种情况下是在定时未触发的时候,B发来了重复ACK了,那么A则会快速重传1501,而不用等待定时触发,这时候B的处理跟上面的情况一模一样的...


TCP的接受数据函数(receive)和发送数据函数(send),建立连接(三次握手)和断开连接(四次挥手),窗口的大小判断来调整发送速率,重传功能(乱序处理)以及处理接受到数据后的ACK(确认数据包)


TCP 使用以下条件来决定何时在收到的数据包上发送 ACK

1. 如果在延迟(200ms)计时器过期之前收到了第二个数据包,则发送 ACK
2. 如果在收到第二个数据包且延迟计时器过期之前,有于 ACK 相同方向发送的数据,则会将该 ACK 与数据段合并立即发送
3. 当延迟计时器过期的时候,发送 ACK

这基本就是一个基础的TCP应该具备的功能



### 1.三次握手的序号变化规则

A发起连接:

- 发送SYN, 自己的序号SEQ = X  (当前数据X)
- A.SND.UNA = X
- A.SND.NXT = X + 1

B响应连接:

- 收到SYN
- 发送SYN + ACK, 自己的序号SEQ = Y, ACK = X + 1  (ACK是期待下一个数据)
- B.SND.UNA = Y
- B.SND.NXT = Y + 1

A 确认连接:

- 收到SYN + ACK,A.RCV.NXT = Y + 1,A期待接收到B发送的下一个字节
- 更新A.SND.UNA = X + 1(已经被确认了)
- 发送ACK,SEQ = X + 1, ACK = Y + 1
- A.SND.NXT = X  + 1(还未发送任何数据)

B最终确认:

- 收到ACK,B.RCV.NXT = X + 1
- 更新B.SND.UNA = Y + 1


### 2.数据发送的时候的变化


客户端A的初始化序列号为X,三次握手后

- A.SND.UNA = X + 1 (表示A已经发送的SYN被确认)
- A.SND.NXT = X + 1 (表示下一个要发送的序列号)
- A.SEQ = X + 1 (当前的SND.UNA)

服务器B的初始化序列号为Y,在三次握手完成后

- B.SND.UNA = Y + 1 (表示B已发送的SYN被确认)
- B.SND.NXT = Y + 1 (表示下一个将要发送的序列号)
- B.SEG = Y + 1 (当前的SND.UNA)

数据发送的过程:

1. A发送数据包(数据长度为L)

- 报文内容
  - SEG.SEQ = X + 1 (当前的SND.NXT)
  - SEG.ACK = Y + 1 (确认B的SYN)
- A的状态变化 -- 主要是更新NXT
  - A.SND.UNA = X + 1 (已发送并确认的数据依然是X + 1)
  - A.SND.NXT = X + 1 + L (下一个要发送的序列号,因为发送的数据是L字节)
- B接收到数据包 -- 主要是更新RCV.NXT
  - B更新B.RCV.NXT = X + 1 + L (期望接收到的下一个序列号是A数据包结束后的序列号)
- B发送ACK -- 更新ACK
 - 报文内容
    - SEG.SEQ = Y + 1 (当前的SND.NXT)
    - SEG.ACK = X + 1 + L (确认A发送的数据包) 
- B的状态变化(只是确认,不需要更新)
  - B.SND.UNA = Y + 1 (确认发送的数据)
  - B.SND.NXT = Y + 1 (下一个将要发送的序列号)     
- A收到ACK后 (更新UNA + NXT)
  - A.SND.UNA = X + 1 + L
  - A.SND.NXT = X + 1 + L


### 总结:

- 发送者发送数据之前,SND.UNA = SND.NXT = SEQ

- 发送者更新SND.NXT = SEQ + L

- 接收端更新RCV.NXT,并且回复ACK = SEQ + L

- 发送端接收到ACK后,更新SND.UNA = SND.NXT = ACK

- 重复以上

> 接收端回复ACK以及更新NXT就行了,它不需要去关心什么,发送端在ACK后更新自己的SND.UNA = SND.NXT = SEQ 就完成了数据的向前移动了....


SND.UNA < SEG.ACK(可接受的ACK) =< SND.NXT 

发送端已发送但是未确认 < 接收方已经确认的发送端数据 <= 发送端下一个要发送的字节


## 代码理解


### 1.初始化脚本


```shell

$ sudo setcap cap_net_admin=eip $CARGO_TARGET_DIR/release/trust // 为我们的编译后的程序设置网络管理权限

$ $CARGO_TAEGET_DIR/release/trust // 运行我们的应用程序
```

运行我们的`程序`,即可创建一个网卡,通过`ip addr`可以看到我们创建了一个名字为`tun0`的虚拟网卡设备,但是我们还未给它分配地址


```shell
$ sudo ip addr set 192.168.0.1/24 dev tun0 // 为我们的tun0设置设置IP地址

$ ip link set up dev tun0 // 激活我们的tun0设备
```

我们的重启脚本

```shell
#!/bin/bash 
cargo -b --release
# 当我们编译失败的时候直接退出
ext=$?
echo "$ext"
if [[ $ext -ne 0]]; then
    exit $ext
fi    
sudo setcap cap_net_admin=eip $CARGO_TARGET_DIR/release/trust
$CARGO_TARGET_DIR/release/trust & pid=$!
sudo ip addr add 192.168.0.1/24 dev tun0
# 当我们杀掉当前的进程的时候,它也会杀掉底层的程序
sudo ip link set dev tun0

# kill pid当你按下ctrl + c的时候,触发INT信号,或者是通过kill命令,触发TERM信号,杀掉该进程

# 区别于脚本结束,这里是将该进程清理掉哦
trap "kill $pid" INT TERM
# 进程结束后,则脚本结束
wait $pid
```


### 2.服务器端 -- 通用:拆包&获取到TCP数据包

```rust
//  ------ main.rs  Loop 循环接受任意的数据包
// 使用connections.get(&quad)来获取connection的值
// 使用connections.get_mut(&quad)来修改connection的值
let mut connections:HashMap<Quad,tcp::Connection> = Default::default();
let mut nic = tun_tap::Iface::new("tun0",tun_tap::Mode::Tun)?;
let mut buf = [0u8;1500];

loop {
  //  IPHeader + TCPHeader + Data
  match etherparse::TcpHeaderSlice::from_slice(&buf[4+p.slice().len()..nbytes]){
    Ok(tcph) => {
      // 数据部分的起始位置
      let datai = 4 + iph.slice().len() + tcph.slice().len();
      // tcph就是我们的TCPHeader
      connections.entry(Quad {
        src:(src,tcph.source_port()),
        dst:(dst,tcph.destination_port_port())
      }).or_default().on_packet(&mut nic,iph,tcph,&buf[datai..nbytes])
    }
  }
}

// ----- tcp.rs
enum State {
  Closed,
  Listen,
  SynRcvd,
  Estab,
}
// 结构体可以有字段,也可以有方法
pub struct Connection {
  state: State
}
// 为我们的状态设置默认值
impl Default for Connection {
  fn default() -> Self {
    Connection {
      state: State::Listen
    }
  }
}
// 我们拆包完后的逻辑判断在这里开始
// state结构体中的方法
impl State {
  pub fn on_packet<'a>(
    &mut self,
    &mut nic,
    iph:etherparse::Ipv4HeaderSlice<'a>,tcph:etherparse::TcpHeaderSlice<'a>,
    data:&'a[u8], // 数据部分
    ) {
      // 待补充....
  }
}
```

### 3.服务器端 -- 情况1:客户端发来SYN包,我们发送SYN + ACK来确认客户端的数据包

上述步骤的情况是我们已经拿到了客户端的TCP数据包,然后我们要根据实际的情况来对不同类型的包进行响应了

这里,我们只需要在保持`listen`状态的情况下,接收到`SYN`的包,我们发送`SYN + ACK`并将状态转化为`SYN-REVD`状态,在此状态下,我们接收`ACK`并最终将状态更新为`ESTAB`的连接状态即可

```rust
// 状态
pub enum State {
  CLosed, // 关闭的状态
  Listen, // 正常情况下监听状态
  SynRcvd, // 接收到对方的SYN的状态
  Estab,  // 成功连接后的状态
}

pub struct Connection {
  state: State
}

// 发送空间
struct SendSequenceSpace {

}
struct RecvSequenceSpace {
  
}

impl Default for Connection {
  fn default() -> Self {
    Connection {
      state: State::Listen
    }
  }
}

impl State {
    pub fn on_packet<'a>(
    &mut self,
    nic: &mut tun_tap:Iface,
    iph: etherparse::Ipv4HeaderSlice<'a>,
    tcph: etherparse::TcpHeaderSlice<'a>,
    data: &'a[u8],
  ) -> io:Result<usize> {
    let mut buf = [0u8;1500];
    // 它的取值就是个状态
    match *self {
      State::Closed => {
        return Ok(0);  // 如果它处于关闭状态,我们什么也不用做的
      }
      State::Listen => {
        // 这个状态下,我们唯一收到的数据包就是SYN
        if !tcph.syn() {
          // 处理错误
          return Ok(0);
        }
        // 在此位置,我们需要完成我们的连接
      }
    }
  }
}
```

### 4.服务器端 -- 通用:发送数据之前检查发送数据的合法性

```rust
//  SND.UNA < SEG.ACK =< SND.NXT
//  发送空间中未确定的数据 < 已经被确认的数据 =< 发送空间中即将要发送的数据
//  确保我们发送空间的数据是正确的

fn is_between_wrapped(start usize,x usize,end usize) -> bool {
  use std::cmp::(Ord,Ordering);
  match start.cmp(x) {
    // 如果相等的话,直接返回
    Ordering::Equal => return false; 
    // S < X
    // ---- S --- X -----
    Ordering::Less => {
      // 简单的理解的话就是满足这个条件的基本都是不在两者之间的....
      // ----- S --- E --- X ----
      // ---- S+E(两者重合) ---- x ----- 
      // --- S ----- X+E(两者重合) ----- 
      if end >= start && end <= x {
        return false; // 不满足条件
      }

    }
    // S > X 
    // ---- X ----- S -----
    // 下面的条件是无法满足
    //  ---- X ---- S ----- E ----
    //  --- E ---- X ---- S ----
    // ----- X+E ----- S -----
    // ---- X ------ S+E -----
    Ordering::Greater => {
      if end < start && end > x {

      } else {
        return false
      }
    }
  }
  true
}

let ackn = tcph.acknowledgment_number();
if(!is_between_wrappend(self.send.una,ackn,self.send.nxt.wrapping_add(1))) {
  return Ok(());
}

```

### 5.服务器端 -- 通用:检查接受数据的正确性


RCV.NXT =< SEG.SEQ  < RCV.NXT + RCV.WND
RCV.NXT =< SEG.SEQ + SEG.LEN - 1 < RCV.NXT + RCV.WND


```rust
// 序号
let seqn = tcph.sequence_number();
// 数据包长度
let mut slen = data.len() as u32;
if tcph.fin() {
  slen += 1;
}
if tcph.syn() {
  slen += 1;
}
// 数据的范围
let wend = self.recv.nxt.wrapping_add(self.recv.wnd as u32);
if slen == 0 {
  if self.recv.wnd == 0 {
    if seqn != self.recv.nxt {
      return Ok(());
    } else if !is_between_wrappend(self.recv.nxt.wrapping_sub(1),seqn,wend) {
      return OK(());
    }
  }
} else {
  if self.recv.wnd == 0 {
    return Ok(());
  } else if !is_between_wrappend(self.recv.nxt.wrapping_sub(1),seqn.wend) && !is_between_wrappend(self.recv.nxt.wrapping_sub(1),seqn.wrapping_add(slen-1),wend) {
    return OK(());
  }
}
// 如果我们通过了这个检查,那么就意味着我们接收了至少一个字节了,所以我们需要更新我们的接收序列
// 这里其实就是我们作为接收端唯一要做的事儿就是更新自己的接收的下一个数据序号
self.recv.nxt = seqn.wrapping_add(slen);

// 确保我们接收到的数据
```

检查逻辑:

- 如果数据长度`slen`为0
  - 如果接收窗口`recv.wnd`为0
    - 检查序号是否等于`recv.nxt`,如果不等则返回OK(表示不处理)
    - 如果序号不在`recv.nxt -1,wend`范围内,也返回OK
- 如果数据长度`slen`大于0
  - 如果接收窗口`recv.wnd`为0,直接返回OK
  - 检查序号是否在`recv.nxt - 1,wend`和`recv.nxt-1,seqn + slen -1`范围内,如果都不在,则返回OK

返回`OK`意味着结束


### 6.服务器端 -- 情况3:发送重置信号

SND.UNA < SEG.ACK =< SND.NXT,通常发生这种情况表明客户端已经失去了同步或者发生了其他的错误,在这种情况下发送`RST`可以通知对方连接出现了问题

`RST`意味着强制关闭一个连接


```rust
// 发送数据包的通用逻辑
fn write(&mut self,nic:&mut tun_tap::Iface,payload:&[u8]) -> io::Result<(usize)> {
  let mut buf = [0u8;1500];
  // 1. 发送端取出自己的序号 SND.UNA = SND.NXT = SEQ 
  self.tcp.sequence_number = self.send.nxt;
  // 2. 发送端从接收队列中获取自己的ACK
  self.tcp.acknowledgment_number = self.recv.nxt;
  let size = std::cmp::min(buf.len(),self.tcp.header_len() as usize + self.ip.header_len() as usize + payload.len());
  self.ip.set_payload_len(size);
  // --------------
  use std::io::Write;
  let mut unwritten = &mut buf[...];
  self.ip.write(&mut unwritten);
  self.tcp.write(&mut unwritten);
  let payload_bytes = unwritten.write(payload)?;
  let unwritten = unwritten.len();
  // 3. 发送端更新自己的NXT
  self.send.nxt = self.send.nxt.wrapping_add(payload_bytes as u32);
  if self.tcp.syn {
    self.send.nxt = self.send.nxt.wrapping_add(1);
    self.tcp.syn = false;
  }
  if self.tcp.fin {
    self.send.nxt = self.send.nxt.wrapping_add(1);
    self.tcp.fin = false;
  }
  nic.send(&buf[...buf.len() - unwritten])?;
}


// 发送重置包的逻辑
fn send_rst(&mut self,nic:&mut tun_tap::Iface) -> io::Result<()> {
  self.tcp.rst = true;
  self.tcp.sequence_number = 0;
  self.tcp.acknowledgment_number = 0;
  self.write(nic,&[])?;
  Ok(());
}

if !is_between_wrappend(self.send.una,ackn,self.send.nxt.wrapping_add(1)) {
  if !self.state.is_synchronized() {
    // 发送RST
    self.send_rst(nic);
  }
  return Ok(());
}
```