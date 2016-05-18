title: 高性能Linux服务器编程-Part 5-TCP协议-卷1
date: 2016-04-27 20:56:26
categories: programming
tags:
- network
- tcp/ip
- tcp protocal
- 高性能Linux服务器编程
---

## 关于本系列文章
---
[开篇里的说明](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part1-tcp-ip-summarize/#关于本系列文章)

## 网络测试环境搭建和说明
---
[参考这里](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part2-arp-protocol/#网络测试环境搭建和说明)

## TCP协议的特点
---

**面向连接**: 
使用TCP协议通信的双方必须先建立连接, 然后才能开始数据的读写.双方都必须为该连接分配必要的内核资源, 以管理连接的状态和连接上数据的传输.TCP连接是全双工的, 即双方的数据读写可以通过一个连接进行.完成数据交换之后, 通信双方都必须断开连接以释放系统资源.

正因为其面向链接,  并且连接是1对1的,  因此TCP协议不适用于广播和多播(其特点是,  目的地对应多个IP地址)的应用. 而无链接协议UDP则非常适合于多播和广播.

**基于字节流**: 
基于字节流的服务(TCP)与基于数据报的服务(UDP)的区别是,  从实际编码的角度上来对比,  体现为通信双方是否必须执行相同次数的读、写操作(当然, 这只是表现形式).

当发送端应用程序连续执行多次写操作时, TCP模块先将这些数据放入TCP发送缓冲区中.当TCP模块真正开始发送数据时, 发送缓冲区中这些等待发送的数据可能被封装成一个或多个TCP报文段发出.因此, TCP模块发送出的TCP报文段的个数和应用程序执行的写操作次数之间没有固定的数量关系.

当接收端收到一个或多个TCP报文段后, TCP模块将它们携带的应用程序数据按照TCP报文段的序号(见后文)依次放入TCP接收缓冲区中, 并通知应用程序读取数据.接收端应用程序可以一次性将TCP接收缓冲区中的数据全部读出, 也可以分多次读取, 这取决于用户指定的应用程序读缓冲区的大小.因此, 应用程序执行的读操作次数和TCP模块接收到的TCP报文段个数之间也没有固定的数量关系.

综上所述, 发送端执行的写操作次数和接收端执行的读操作次数之间没有任何数量关系, 这就是字节流的概念：应用程序对数据的发送和接收是没有边界限制的.UDP则不然.发送端应用程序每执行一次写操作, UDP模块就将其封装成一个UDP数据报并发送之.接收端必须及时针对每一个UDP数据报执行读操作(通过recvfrom系统调用), 否则就会丢包(这经常发生在较慢的服务器上).并且, 如果用户没有指定足够的应用程序缓冲区来读取UDP数据, 则UDP数据将被截断.

下面的2个图示说明了基于数据流的TCP协议与基于数据报的UDP协议的具体区别:
![tcp-baseon-stream](http://static.zhuxiaodong.net/blog/static/images/tcp-baseon-stream.jpg)
![udp-baseon-datagram](http://static.zhuxiaodong.net/blog/static/images/udp-baseon-datagram.jpg)

**传输的可靠性**:
1. TCP协议采用发送应答机制, 即发送端发送的每个TCP报文段都必须得到接收方的应答, 才认为这个TCP报文段传输成功.
2. TCP协议采用超时重传机制, 发送端在发送出一个TCP报文段之后启动定时器, 如果在定时时间内未收到应答, 它将重发该报文段.
3. 因为TCP报文段最终是以IP数据报发送的, 而IP数据报到达接收端可能乱序、重复, 所以TCP协议还会对接收到的TCP报文段重排、整理, 再交付给应用层.

与TCP协议对比,  UDP协议则和IP协议一样, 提供不可靠服务.它们都需要上层协议来处理数据确认和超时重传.
<!--more-->
## TCP头部结构
---
![tcp-header](http://static.zhuxiaodong.net/blog/static/images/tcp-header.jpg)

* 16位端口号(port number): 告知主机该报文段是来自哪里(源端口)以及传给哪个上层协议或应用程序(目的端口)的.进行TCP通信时, 客户端通常使用系统自动选择的临时端口号, 而服务器则使用知名服务端口号(参考/etc/services中的定义).

* 32位序号(sequence number): 一次TCP通信(从TCP连接建立到断开)过程中某一个传输方向上的字节流的每个字节的编号.假设主机A和主机B进行TCP通信, A发送给B的第一个TCP报文段中, **序号值被系统初始化为某个随机值ISN(Ini-tial Sequence Number, 初始序号值)**.那么在该传输方向上(从A到B), 后续的TCP报文段中序号值将被系统设置成ISN加上该报文段所携带数据的第一个字节在整个字节流中的偏移.例如, 某个TCP报文段传送的数据是字节流中的第1025～2048字节, 那么该报文段的序号值就是ISN+1025.另外一个传输方向(从B到A)的TCP报文段的序号值也具有相同的含义.

* 32位确认号(acknowledgement number): **用作对另一方发送来的TCP报文段的响应.其值是收到的TCP报文段的序号值加1**.假设主机A和主机B进行TCP通信, 那么A发送出的TCP报文段不仅携带自己的序号, 而且包含对B发送来的TCP报文段的确认号.反之, B发送出的TCP报文段也同时携带自己的序号和对A发送来的报文段的确认号.

* 4位头部长度(header length): 标识该TCP头部有多少个32bit字(4字节).因为4位最大能表示15, 所以**TCP头部最长是60字节**.

* 6位标记字段:
  * URG标志: 表示紧急指针(urgent pointer)是否有效.
  * ACK标志: 表示确认号是否有效,  包含ack标志的TCP报文段被称为确认报文段.
  * PSH标志: 提示接收端应用程序应该立即从TCP接收缓冲区中读走数据, 为接收后续数据腾出空间(如果应用程序不将接收到的数据读走, 它们就会一直停留在TCP接收缓冲区中).
  * RST标记: 表示要求对方重新建立连接.我们称携带RST标志的TCP报文段为复位报文段.(TCP connection pool当中的链接实例,  就不应该出现该标记字段?)
  * SYN标记: 表示请求建立一个连接.我们称携带SYN标志的TCP报文段为同步报文段.
  * FIN标记: 表示通知对方本端要关闭连接了.我们称携带FIN标志的TCP报文段为结束报文段.

* 16位窗口大小(window size): 窗口大小TCP进行流量控制的手段,  这里是指接收窗口(Receiver Window,  简称: RWND),  它用来告诉对方,  自己的TCP接收缓冲端还能够容纳多少数据,  这样对方就能够控制发送的速度.

* 16位校验和(TCP checksum): 由发送端填充, 接收端对TCP报文段执行CRC算法以检验TCP报文段在传输过程中是否损坏.注意, 这个校验不仅包括TCP头部, 也包括数据部分.这也是TCP可靠传输的一个重要保障.

* 16位紧急指针(urgent pointer)：是一个正的偏移量.它和序号字段的值相加表示最后一个紧急数据的下一字节的序号.确切地说, 这个字段是紧急指针相对当前序号的偏移, 不妨称之为紧急偏移.TCP的紧急指针是发送端向接收端发送紧急数据的方法.

## TCP头部的选项字段
---

TCP头部的最后一个字段是可选并且是可变长的选项字段,  它最多包含40字节(60[TCP Header max size] - 20[TCP header fixed part size]),  其结构请参考下图:
![tcp-header-options](http://static.zhuxiaodong.net/blog/static/images/tcp-header-options.jpg)

分为了3部分:
* kind(1 byte): 标识选项的类型,  某些tcp选项只包含了kind字段,  而不包括length和info字段.
* length(1 byte): 标识了该选项字段的长度(包括了kind + length所占用的2 byte).
* info: 标识了具体内容.

常见的TCP选项有7种: 
![tcp-header-options-type](http://static.zhuxiaodong.net/blog/static/images/tcp-header-options-type.jpg)

1. kind=0: 表示结束选项.
2. kind=1: 是空操作(nop)选项, 没有特殊含义, 一般用于将TCP选项的总长度填充为4字节的整数倍.
3. kind=2: 是最大报文段长度选项.TCP连接初始化时, 通信双方使用该选项来协商最大报文段长度(Max Segment Size, MSS).**TCP模块通常将MSS设置为(MTU - 40)字节(减掉的这40字节包括20字节的TCP头部和20字节的IP头部)**.这样携带TCP报文段的IP数据报的长度就不会超过MTU(假设TCP头部和IP头部都不包含选项字段, 并且这也是一般情况), 从而避免本机发生IP分片.对以太网而言, MSS值是1460(1500-40)字节.
4. kind=3: 是窗口扩大因子选项.TCP连接初始化时, 通信双方使用该选项来协商接收通告窗口的扩大因子.在TCP的头部中, 接收通告窗口大小是用16位表示的, 最大为65535字节, 但实际上TCP模块允许的接收通告窗口大小远不止这个数(为了提高TCP通信的吞吐量).窗口扩大因子解决了这个问题.假设TCP头部中的接收通告窗口大小是N, 窗口扩大因子(移位数)是M, 那么TCP报文段的实际接收通告窗口大小是N乘2M, 或者说N左移M位.注意, M的取值范围是0～14.**我们可以通过修改/proc/sys/net/ipv4/tcp_window_scaling内核变量来启用或关闭窗口扩大因子选项**.和MSS选项一样, 窗口扩大因子选项只能出现在同步报文段中, 否则将被忽略.但同步报文段本身不执行窗口扩大操作, 即同步报文段头部的接收通告窗口大小就是该TCP报文段的实际接收通告窗口大小.当连接建立好之后, 每个数据传输方向的窗口扩大因子就固定不变了.关于窗口扩大因子选项的细节, 可参考标准文档[RFC 1323](https://www.ietf.org/rfc/rfc1323.txt).
5. kind=4: 是选择性确认(Selective Acknowledgment, SACK)选项.TCP通信时, 如果某个TCP报文段丢失, 则TCP模块会重传最后被确认的TCP报文段后续的所有报文段, 这样原先已经正确传输的TCP报文段也可能重复发送, 从而降低了TCP性能.SACK技术正是为改善这种情况而产生的, 它使TCP模块只重新发送丢失的TCP报文段, 不用发送所有未被确认的TCP报文段.选择性确认选项用在连接初始化时, 表示是否支持SACK技术.我们可以通过修改/proc/sys/net/ipv4/tcp_sack内核变量来启用或关闭选择性确认选项.
6. kind=5: 是SACK实际工作的选项.该选项的参数告诉发送方本端已经收到并缓存的不连续的数据块, 从而让发送端可以据此检查并重发丢失的数据块.每个块边沿(edge of block)参数包含一个4字节的序号.其中块左边沿表示不连续块的第一个数据的序号, 而块右边沿则表示不连续块的最后一个数据的序号的下一个序号.这样一对参数(块左边沿和块右边沿)之间的数据是没有收到的.因为一个块信息占用8字节, 所以TCP头部选项中实际上最多可以包含4个这样的不连续数据块(考虑选项类型和长度占用的2字节).
7. kind=8: 是时间戳选项.该选项提供了较为准确的计算通信双方之间的回路时间(Round Trip Time, RTT)的方法, 从而为TCP流量控制提供重要信息.我们可以通过修改/proc/sys/net/ipv4/tcp_timestamps内核变量来启用或关闭时间戳选项.

## 使用tcpdump观察TCP头部信息
---

我们使用[之前](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part4-ip-protocal/#实践-使用tcpdump观察ipv4的头部结构)抓取的报文信息进行TCP部分的分析:

```
IP 127.0.0.1.56616 > 127.0.0.1.telnet: Flags [S],  seq 461096030,  win 43690,  options [mss 65495, sackOK, TS val 178824 ecr 0, nop, wscale 7],  length 0
0x0000:  4510 003c 7ed6 4000 4006 bdd3 7f00 0001
0x0010:  7f00 0001 dd28 0017 1b7b c45e 0000 0000
0x0020:  a002 aaaa fe30 0000 0204 ffd7 0402 080a
0x0030:  0002 ba88 0000 0000 0103 0307
```

* Flags [S]表示这是一个SYN同步报文字段,  如果该TCP报文还包含其它标记,  也会显示在这个方括号中.
* seq 461097030表示序号值.
* win 是RWND(接收窗口)的值. **因为这是一个同步报文段, 所以win值反映的是实际的接收通告窗口大小**.
* options是TCP选项, 包括:
	* mss是发送端(客户端)通告的最大报文段长度.通过ifconfig命令查看回路接口的MTU为65535字节, 因此可以预想到TCP报文段的MSS为65495(65535-40)字节.
	* sackOK表示发送端支持并同意使用SACK选项.
	* TS val是发送端的时间戳.
	* ecr是时间戳回显应答.因为这是一次TCP通信的第一个TCP报文段, 所以它针对对方的时间戳的应答为0(尚未收到对方的时间戳).
	* nop是一个空操作选项.
	* wscale指出发送端使用的窗口扩大因子为7.

接下来分析,  二进制代码对应的TCP头部的细节,  我们从第21个字节处开始分析(前20个字节为IP头部的内容.)

| 16进制数 | 10进制数 | TCP头部信息 |
| :--------: | :-----: | :----: |
| 0xdd28 | 56616 | 16bit源端口号  |
| 0x0017 | 23 | 16bit目的端口号(telnet服务的端口号) |
| 0x1b7bc45e | 461096030 | 32bit序号:  (与上面的seq 461096030一致)|
| 0x00000000 | 0 | 32bit确认号: 由于是通信的第一个报文,  因此值为0 |
| 0xa | 10 | 4bit头部长度: 10 * (32bit)4 byte = 40 |
| 0x002 |  | 6bit的标记字段: 这里表示SYN |
| 0xaaaa | 43690 | 16bit接收窗口(RWND)大小:43690 |
| 0xfe30 | 65072 | 16bit头部校验和 |
| 0x0000 |  | 16bit紧急指针,  由于在上面的6bit标记字段中并没有设置URG的FLag,  因此这里紧急指针没有意义 |
| 0x0204 |  | 16 bit(1 byte kind,  1 byte length)表示选项为: MSS(kind = 2)选项的kind值和length值 |
| 0xffd7 | 65495 | 16bit表示MSS的值 |
| 0x0402 |  | 16bit(1 byte kind,  1 byte length)表示选项为: SACK(kind = 4)和length值 |
| 0x080a |  | 16bit(1 byte kind,  1 byte length)表示选项为: Timestamps(kind = 8)和length值 |
| 0x0002ba88 | 178824 | 32bit Timestamps的值为:178824 |
| 0x00000000 | 0 | 32bit 回显应答时间戳 |
| 0x01 |  | 8bit 空操作选项. |
| 0x0303 |  | 16bit(1 byte kind,  1 byte length)表示选项为: 窗口扩大因子(kind = 3)和length值 |
| 0x07 | 7 | 窗口扩大因子值为7 |

**上文中0x002为什么表示标识字段(Control bits)的值为SYN?**:
0x002当中前6 bit为保留字段,  后6 bit才是标记字段. 因此转换为2进制的值为: 000010,  第二位为1,  因此表示SYN. 下图为wireshark抓取的示例:
![tcp-header-flags-sync](http://static.zhuxiaodong.net/blog/static/images/tcp-header-flags-sync.jpg)

完整的定义参考[RFC 7125](https://tools.ietf.org/html/rfc7125):
```
       MSb                                                         LSb
        0   1   2   3   4   5   6   7   8   9  10  11  12  13  14  15
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
      |               |           | N | C | E | U | A | P | R | S | F |
      |     Zero      |   Future  | S | W | C | R | C | S | S | Y | I |
      | (Data Offset) |    Use    |   | R | E | G | K | H | T | N | N |
      +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

      bit    flag
      value  name  description
      ------+-----+-------------------------------------
      0x8000       Zero (see tcpHeaderLength)
      0x4000       Zero (see tcpHeaderLength)
      0x2000       Zero (see tcpHeaderLength)
      0x1000       Zero (see tcpHeaderLength)
      0x0800       Future Use
      0x0400       Future Use
      0x0200       Future Use
      0x0100   NS  ECN Nonce Sum
      0x0080  CWR  Congestion Window Reduced
      0x0040  ECE  ECN Echo
      0x0020  URG  Urgent Pointer field significant
      0x0010  ACK  Acknowledgment field significant
      0x0008  PSH  Push Function
      0x0004  RST  Reset the connection
      0x0002  SYN  Synchronize sequence numbers
      0x0001  FIN  No more data from sender
```

## 使用tcpdump观察TCP的三次握手和四次挥手的过程
---

```
sudo tcpdump -nt '(src 192.168.1.122 and dst 10.24.241.125) or (src 10.24.241.125 and dst 192.168.1.122)'

telnet 10.24.241.125 8000 #在mac的机器上telnet 远程服务器的http服务.
```

抓取到的http包为:
```
IP 192.168.1.122.64543 > 192.168.1.123.8000: Flags [S],  seq 768366895,  win 65535,  options [mss 1460, nop, wscale 5, nop, nop, TS val 1148856935 ecr 0, sackOK, eol],  length 0
IP 192.168.1.123.8000 > 192.168.1.122.64543: Flags [S.],  seq 2363927244,  ack 768366896,  win 14480,  options [mss 1440, sackOK, TS val 519920553 ecr 1148856935, nop, wscale 7],  length 0
IP 192.168.1.122.64543 > 192.168.1.123.8000: Flags [.],  ack 1,  win 4105,  options [nop, nop, TS val 1148856999 ecr 519920553],  length 0
IP 192.168.1.122.64543 > 192.168.1.123.8000: Flags [F.],  seq 1,  ack 1,  win 4105,  options [nop, nop, TS val 1148874006 ecr 519920553],  length 0
IP 192.168.1.123.8000 > 192.168.1.122.64543: Flags [.],  ack 2,  win 114,  options [nop, nop, TS val 519937652 ecr 1148874006],  length 0
IP 192.168.1.123.8000 > 192.168.1.122.64543: Flags [F.],  seq 1,  ack 2,  win 114,  options [nop, nop, TS val 519937652 ecr 1148874006],  length 0
IP 192.168.1.122.64543 > 192.168.1.123.8000: Flags [.],  ack 2,  win 4105,  options [nop, nop, TS val 1148874084 ecr 519937652],  length 0
```

相关的客户端与服务器端的通信时序图为:
```
192.168.1.122                            192.168.1.123
      +                                          +
      |            seq 768366895 SYN             |
      +------------------------------------------>
      |                                          |
      |                                          |
      |            seq 2363927244 SYN            |
      <------------------------------------------+
      |            ack 768366896                 |
      |                                          |
      |            ack 2363927245                |
      +------------------------------------------>
      |                                          |
      |                                          |
      |      seq 768366896 ack 2363927245 FIN    |
      +------------------------------------------>
      |                                          |
      |                                          |
      |            ack 768366897                 |
      <------------------------------------------+
      |                                          |
      |                                          |
      |      seq 2363927245 ack 768366897 FIN    |
      <------------------------------------------+
      |                                          |
      |                                          |
      |            ack 768366897                 |
      +------------------------------------------>
      |                                          |
```

我们在telnet 10.24.241.125 8000,  完成了下面的3个动作: 
1. 第一个数据报由192.168.1.122发送给192.168.1.123的连接请求,  由于包含SYN标志,  表示是一个同步标志数据报,  该同步报文段包含ISN为788366895的序号.
2. 192.168.1.123在接收到第一个数据报之后,  返回给192.168.1.122的数据报包括了: 
  * ISN为2363927244的同步标记数据报.
  * 对第一个数据报ISN: 788366895的ACK: 788366895 + 1 = 768366896
3. 192.168.1.122最后在对第二个数据报进行确认,  发送第三个数据报给192.168.1.123的ack: 2363927244 + 1 = 2363927245

NOTE: 从第三个报文段开始,  seq和ack的值都显示的相对于ISN(初始值)的偏移,  例如上面报文中的seq 1,  ack 1,  seq 2等,  **我们可以在使用tcpdump时添加-S参数来显示绝对值.**

上述3个数据报完成了TCP连接的建立,  称之为TCP建立连接的**3次握手**.

当telnet中quit时,  有4个报文显示了TCP关闭连接的过程,  与**3次握手相对应**,  被称之为**4次挥手**.
```
telnet> quit
```

1. 第一个数据报由192.168.1.122发送给192.168.1.123的关闭连接请求,  包含了seq,  ack和FIN标记.
2. 第二个数据报由192.168.1.123发送给192.168.1.122的ack. seq + 1 = 768366897
3. 第三个数据报由192.168.1.123发送给192.168.1.122,  包含了seq,  ack和FIN标记,  要求关闭连接.
4. 第四个数据报由192.168.1.122发送给192.168.1.123的ack.

上述过程中,  由于192.168.1.122是主动发起关闭连接的一方,  因此被称之为**主动关闭**,  相对而言,  192.168.1.123则被称之为**被动**关闭.

通常的情况下,  TCP连接由客户端发起,  并通过**三次握手**建立连接; 而TCP连接关闭的过程可能是由客户端发起,  执行主动关闭,  例如上述的例子. 也有可能是由服务端执行主动关闭,  例如服务器端进程退出了之后强制关闭连接.

## TCP半关闭状态(TCP Half-Close)
---
TCP连接是全双工的, 允许两个方向的数据传输被独立关闭.通信的一端可以发送结束报文段给对方, 告诉它本端已经完成了数据的发送, 但允许继续接收来自对方的数据, 直到对方也发送结束报文段以关闭连接.TCP连接的这种状态称为半关闭(half close)状态,  参考下图:
![tcp-half-close](http://static.zhuxiaodong.net/blog/static/images/tcp-half-close.jpg)

服务器和客户端应用程序判断对方是否已经关闭连接的方法是：read系统调用返回0(收到结束报文段),  此外,  socket网络编程接口中通过shutdown函数提供了对半关闭状态的支持.

一般而言,  我们很少会出现半关闭状态的应用程序.

ref: [http://superuser.com/questions/298919/what-is-tcp-half-open-connection-and-tcp-half-closed-connection](http://superuser.com/questions/298919/what-is-tcp-half-open-connection-and-tcp-half-closed-connection)

## TCP连接超时(TCP connection timeout)
---
如果客户端访问一个距离它很远的服务器, 或者由于网络繁忙, 导致服务器对于客户端发送出的同步报文段没有应答, 此时客户端程序将产生什么样的行为呢？显然, 对于提供可靠服务的TCP来说, 它必然是先进行重连(可能执行多次), 如果重连仍然无效, 则通知应用程序连接超时.

我们通过如下的方式,  来模拟一个网络非常繁忙的环境,  在Linux VM: 10.211.55.4上执行:
```
sudo iptables -F
sudo iptables -I INPUT -p tcp --syn -i eth0 -j DROP
```

然后我们在mac os x上执行如下的命令,  来抓取相关的tcp数据报.

```
sudo tcpdump -n -i eth0 port 23 #只抓取telnet的请求.
```

可以看到从连接到超时花费了75s的时间.
```
date;telnet 10.211.55.4 80;date #使用date能够显示出telnet从连接到超时所使用的时间

###Output
Wed May 18 10:35:00 CST 2016
Trying 10.211.55.4...
telnet: connect to address 10.211.55.4: Operation timed out
telnet: Unable to connect to remote host
Wed May 18 10:36:15 CST 2016
```

抓取到的数据报为:

```
1.21：23：35.612136 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
2.21：23：36.613146 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
3.21：23：38.617279 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
4.21：23：42.625140 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
5.21：23：50.641344 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
6.21：24：06.673331 IP 10.211.55.2.39385＞10.211.55.4.telnet：Flags[S], seq 1355982096, length 0
```

从上述报文分析,  第一个报文之后的seq的序号都是一致的,  因此从2 ~ 6都是retry的报文连接. 每个报文之间的时间间隔为: 1s,  2s,  4s,  8s,  16s,  32s. 一共执行了5次重连操作.

linux内核对tcp执行多少次重连可以通过如下的内核变量确定:

```
cat /proc/sys/net/ipv4/tcp_syn_retries
```