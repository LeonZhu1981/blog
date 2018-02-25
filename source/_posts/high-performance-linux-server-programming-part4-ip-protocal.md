title: 高性能Linux服务器编程-Part 4-IP协议
date: 2016-04-25 13:36:06
categories: programming
tags:
- network
- tcp/ip
- ip protocal
- 高性能Linux服务器编程
---

## 关于本系列文章
---
[开篇里的说明](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part1-tcp-ip-summarize/#关于本系列文章)

## 网络测试环境搭建和说明
---
[参考这里](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part2-arp-protocol/#网络测试环境搭建和说明)

## 概述
---
**定义/职责**: IP协议是上层协议的基础(TCP, UDP), 它为上层协议提供**无状态**, **无连接**, **不可靠**的服务

**无状态**:
定义: 无状态(stateless)是指IP通信双方不同步传输数据的状态信息, 所有数据报的发送, 传输, 接收都是相互独立并且没有上下文关系的.
优点: 简单/高效, 不需要为了保持通信状态而使用额外的内核资源, 也不需要在数据报当中包含额外的状态信息, 从而减少了包的大小. 
缺点: 无状态会导致数据报的乱序和重复发送. 例如第N个IP数据报会比N+1个IP数据报后到达接收端, 或者同一个IP数据报会多次到达同一个接收端. 
  * 传输层的TCP协议是有状态/面向连接的协议, 它能够自己来处理乱序的, 重复的IP数据报, 以确保传输的内容是绝对有序和可靠的.
  * IP数据报头部提供了一个标识字段用以唯一标识一个IP数据报, 但它是被用来处理IP分片和重组的, 而不是用来指示接收顺序的.
常见的无状态网络协议: UDP, Http.


**无连接**:
定义: 无连接(connectionless)是指IP通信双方都不长久地维持对方的任何信息. 基于IP协议的上层协议在每次发送数据时, 都必须明确地给出对方的IP地址.

**不可靠**:
定义: IP协议不能保证IP数据报准确地到达接收端, 它只是承诺尽最大努力(best effort).
导致IP数据报无法到达接收端的原因/场景: 
  * 中转路由器发现IP数据报的TTL为0, 会直接丢弃掉, 并返回一个ICMP的错误消息(超时错误)给发送端.
  * 接收端发现IP数据报的内容不正确(通过CRC校验机制), 会直接丢弃掉, 并返回一个ICMP的错误消息(IP头部参数错误)给发送端.
发送端的IP模块一旦检测到IP数据报发送失败, 就通知上层协议发送失败,而不会试图重传.因此,使用IP服务的上层协议(比如TCP协议)需要自己实现数据确认, 超时重传等机制以达到可靠传输的目的.
<!--more-->
## IPV4的头部结构
---

如下图所示:
![ipv4-header-structure](https://www.zhuxiaodong.net/static/images/ipv4-header-structure.jpg)
1. 通常**IP头长度为20字节**, 除非包含可变长度的选项部分.
2. 4位版本号(version): 指定IP协议的版本.对IPv4来说,其值是4.其他IPv4协议的扩展版本(如SIP协议和PIP协议),则具有不同的版本号, 它们的头部结构也和IPV4的结构不同.
3. 4位头部长度(header length): 标识该IP头部有多少个32bit字(4字节).因为4位最大能表示15,所以IP头部**最长是60字节**.
4. 8位TOS(Type of Service)服务类型: 包含3位的优先权字段(目前已经被忽略), 4位TOS字段和1位保留字段(必须置为0). 其中4位TOS字段有4种类型: **最小延迟**/**最大吞吐量**/**最高可用性**/**最小开销(费用)**. 其中只能有一个值能够被设置为1, 应用程序需要根据实际的场景进行设置. 例如ssh和telnet需要将最小延迟设置为1, 而FTP需要将最大吞吐量设置为1.
5. 16位总长度(total length)是指整个IP数据报的长度,以字节为单位,因此**IP数据报的最大长度为65535字节**.但由于MTU的限制,**长度超过MTU的数据报都将被分片传输,所以实际传输的IP数据报(或分片)的长度都远远没有达到最大值**.接下来的3个字段则描述了如何实现分片.
6. 16位标识(identification)**唯一地标识主机发送的每一个数据报**.其初始值由系统随机生成；每发送一个数据报,其值就加1.该值在数据报分片时被复制到每个分片中,因此**同一个数据报的所有分片都具有相同的标识值**.
7. 3位标志字段的第一位保留.第二位(Don’t Fragment, DF)表示“禁止分片”.如果设置了这个位,IP模块将不对数据报进行分片.**在这种情况下,如果IP数据报长度超过MTU的话,IP模块将丢弃该数据报并返回一个ICMP差错报文**.**第三位(More Fragment,MF)表示“更多分片”.除了数据报的最后一个分片外,其他分片都要把它置1**.
8. 13位分片偏移(fragmentation offset)**是分片相对原始IP数据报开始处(仅指数据部分)的偏移**.实际的偏移值是该值左移3位(乘8)后得到的.由于这个原因,**除了最后一个IP分片外,每个IP分片的数据部分的长度必须是8的整数倍**(这样才能保证后面的IP分片拥有一个合适的偏移值).
9. 8位生存时间(Time To Live,TTL)**是数据报到达目的地之前允许经过的路由器跳数**.TTL值被发送端设置(常见的值是64).数据报在转发过程中每经过一个路由,该值就被路由器减1.**当TTL值减为0时,路由器将丢弃数据报,并向源端发送一个ICMP差错报文**.**TTL值可以防止数据报陷入路由循环**.
10. 8位协议(protocol)用来区分上层协议./etc/protocols文件定义了所有上层协议对应的protocol字段的数值.其中,**ICMP是1,TCP是6,UDP是17**./etc/protocols文件是[RFC 1700](https://www.ietf.org/rfc/rfc1700.txt)的一个子集.
11. 16位头部校验和(header checksum)**由发送端填充,接收端对其使用CRC算法以检验IP数据报头部(注意,仅检验头部)在传输过程中是否损坏**.
12. 32位的源端IP地址和目的端IP地址用来标识数据报的发送端和接收端.一般情况下,这两个地址在整个数据报的传递过程中保持不变,而不论它中间经过多少个中转路由器.
13. IPv4最后一个选项字段(option)是可变长的可选信息.这部分最多包含40字节(60 - 20, 即IP数据报header的最大长度减去通常长度). 其中可选的选项包括:
	* 记录路由(record route),告诉数据报途经的所有路由器都将自己的IP地址填入IP头部的选项部分,这样我们就可以跟踪数据报的传递路径.
	* 时间戳(timestamp),告诉每个路由器都将数据报被转发的时间(或时间与IP地址对)填入IP头部的选项部分,这样就可以测量途经路由之间数据报传输的时间.
	* 松散源路由选择(loose source routing),指定一个路由器IP地址列表,数据报发送过程中必须经过其中所有的路由器.
	* 严格源路由选择(strict source routing),和松散源路由选择类似,不过数据报只能经过被指定的路由器.
	* 选项字段很少被使用,使用松散源路由选择和严格源路由选择选项的例子大概仅有traceroute程序.此外,作为记录路由IP选项的替代品,traceroute程序使用UDP报文和ICMP报文实现了更可靠的记录路由功能,详情请参考文档[RFC 1393](https://www.ietf.org/rfc/rfc1393.txt).

**more information: IP协议的标准文档[RFC 791](https://www.ietf.org/rfc/rfc791.txt)**.

## 实践: 使用tcpdump观察IPV4的头部结构
---

**测试说明**: 执行telnet命令登录本机, 并用tcpdump观察telnet客户端和服务器端的通信过程.

**step 1**: 使用tcpdump抓包
```
sudo tcpdump -ntx -i lo #lo表示抓取本地回路上的数据包.
```

**step 2**: 使用telnet登录本机
```
telnet 127.0.0.1
```

此时tcpdump抓取到的第一个数据包为:
```
IP 127.0.0.1.56616 > 127.0.0.1.telnet: Flags [S], seq 461096030, win 43690, options [mss 65495,sackOK,TS val 178824 ecr 0,nop,wscale 7], length 0
0x0000:  4510 003c 7ed6 4000 4006 bdd3 7f00 0001
0x0010:  7f00 0001 dd28 0017 1b7b c45e 0000 0000
0x0020:  a002 aaaa fe30 0000 0204 ffd7 0402 080a
0x0030:  0002 ba88 0000 0000 0103 0307
```

第一行中包含的信息:
* 由于是本机连本机, 因此发送端和接收端的IP地址都是127.0.0.1.
* 客户端开启了56616端口用于连接服务器端(127.0.0.1.telent, telent使用的端口号是23).
* 其余相关信息中: Flags, seq, win, options都是与TCP协议有关的, 会在后续的章节中介绍.

从第二行开始, 显示IP数据报的二进制内容(以16进制的方式显示), **一共包含了60个字节, 前20字节为IP头部, 后40个字节为IP数据报内容**. 我们重点来分析一下:
注意: 第二行的二进制内容中, 开头的0x0000:表示起始地址, 后面的4510 003c 7ed6 .... .... 表示具体的二进制内容, 其中1个数字表示4 bit. 例如: 4510表示由4 * 4 = 16 bit(2 byte)构成.

| 16进制数 | 10进制数 | IP头部信息 |
| :--------: | :-----: | :----: |
| 0x4 | 4 | IP版本号(4表示IPV4),  |
| 0x5 | 5 | 使用4 bit表示IP头部长度, 5个32位(20字节) |
| 0x10 |  | 使用8 bit表示TOS开启了**最小延迟**服务类型 |
| 0x003c | 60 | 使用16 bit表示IP数据报的总长度: 60字节 |
| 0x7ed6 |  | 使用16 bit表示IP数据报的标识, 由发送端OS随机生成 |
| 0x4 |  | 使用3 bit标记字段, 此处表示"禁止分片"的标识: 4 = 0100, 取前3位, 为010, 第二位的1表示DF, 第三位0表示最后一个分片 |
| 0x000 | 0 | 使用13 bit表示分片偏移 |
| 0x40 | 64 | 使用8 bit表示TTL, TTL=64 |
| 0x06 | 6 | 使用8 bit表示上层协议, 6表示TCP |
| 0xbdd3 |  | 使用16 bit表示IP头部校验和(checksum) |
| 0x7f000001 |  | 使用32 bit表示发送端的IP地址, 127.0.0.1 |
| 0x7f000001 |  | 使用32 bit表示接收端的IP地址, 127.0.0.1 |

更多关于3 bit标记字段的信息, RFC中是这样描述的:
> Flags:  3 bits
> 	Various Control Flags.
> 		Bit 0: reserved, must be zero
>		Bit 1: (DF) 0 = May Fragment,  1 = Don't Fragment.
>		Bit 2: (MF) 0 = Last Fragment, 1 = More Fragments.

>          0   1   2
>        +---+---+---+
>        |   | D | M |
>        | 0 | F | F |
>        +---+---+---+

由上述信息得出的结论是: 
* telnet服务选择使用具有**最小延时**的服务.
* telnet服务默认使用的传输层协议是TCP协议.
* 这个IP数据报没有被分片, 因为它没有携带任何应用程序数据.

更多关于8 bit Type of Service(TOS)字段的信息, RFC中是这样描述的:
> The Type of Service provides an indication of the abstract parameters of the quality of service desired.  These parameters are to be used to guide the selection of the actual service parameters when transmitting a datagram through a particular network.  Several networks offer service precedence, which somehow treats high precedence traffic as more important than other traffic (generally by accepting only traffic above a certain precedence at time of high load).  The major choice is a three way tradeoff between low-delay, high-reliability, and high-throughput.

>         Bits 0-2:  Precedence.
>         Bit    3:  0 = Normal Delay,      1 = Low Delay.
>         Bits   4:  0 = Normal Throughput, 1 = High Throughput.
>         Bits   5:  0 = Normal Relibility, 1 = High Relibility.
>         Bit  6-7:  Reserved for Future Use.
>         
>          0     1     2     3     4     5     6     7
>       +-----+-----+-----+-----+-----+-----+-----+-----+
>       |                 |     |     |     |     |     |
>       |   PRECEDENCE    |  D  |  T  |  R  |  0  |  0  |
>       |                 |     |     |     |     |     |
>       +-----+-----+-----+-----+-----+-----+-----+-----+
>         
>         Precedence
>           111 - Network Control
>           110 - Internetwork Control
>           101 - CRITIC/ECP
>           100 - Flash Override
>           011 - Flash
>           010 - Immediate
>           001 - Priority
>           000 - Routine

> The use of the Delay, Throughput, and Reliability indications may increase the cost (in some sense) of the service.  In many networks better performance for one of these parameters is coupled with worse performance on another.  Except for very unusual cases at most two of these
> three indications should be set. 
> The type of service is used to specify the treatment of the datagram during its transmission through the internet system.  Example mappings of the internet type of service to the actual service provided on networks such as AUTODIN II, ARPANET, SATNET, and PRNET is given in "Service Mappings".

由上述信息得出的结论是:
* 0x10 16进制对应的二进制为: 0001 0000.
* 前三位000表示Routine.
* 第四位1表示: 1 = Low Delay(最小延迟)
* 第四位, 第五位, 第六位只能有1 bit为1.

## IP分片
---
**What?**:
IP数据报的长度超过帧的MTU时,它将被分片传输.分片可能发生在发送端,也可能发生在中转路由器上,而且可能在传输过程中被多次分片,但只有在最终的目标机器上,这些分片才会被内核中的IP模块重新组装.

**How?**:
通过IP头部中的3个重要信息来进行标识:
1. 16位标识(identification)**唯一地标识主机发送的每一个数据报**. **同一个IP数据报的不同分片, 该字段的值相同**.
2. 3位Flag标记字段. 标识是否允许分片? 是否为最后一个分片?
3. 13位分片偏移(fragmentation offset)**是分片相对原始IP数据报开始处(仅指数据部分)的偏移**, 通过该字段就能够获取到该分片的数据内容以及数据长度.

**以ICMP协议为示例, 讨论IP数据报的分片行为**:
1. 以太网帧的MTU是1500字节(可以通过ifconfig命令或者netstat命令查看),因此它携带的IP数据报的数据部分最多是1480字节(IP头部占用20字节).以下是通过**ifconfig**查看MTU值的截图:
![ethernet-mtu](https://www.zhuxiaodong.net/static/images/ethernet-mtu.jpg)
2. 考虑用IP数据报封装一个长度为1481字节的ICMP报文(包括8字节的ICMP头部,所以其数据部分长度为1473字节),则该数据报在使用以太网帧传输时必须被分片,如下图所示.
![icmp-fragment](https://www.zhuxiaodong.net/static/images/icmp-fragment.jpg)
3. 上述图示中, 长度为1501字节的IP数据报被拆分成两个IP分片,第一个IP分片长度为1500字节,第二个IP分片的长度为21字节(1501 - 1 + 20[20字节的header长度] = 21 byte).每个IP分片都包含自己的IP头部(20字节),且第一个IP分片的IP头部设置了MF标志(More Fragment表示还包含更多分片),而第二个IP分片的IP头部则没有设置该标志,因为它已经是最后一个分片了.原始IP数据报中的ICMP头部内容被完整地复制到了第一个IP分片中.**第二个IP分片不包含ICMP头部信息**,因为IP模块重组该ICMP报文的时候只需要一份ICMP头部信息,重复传送这个信息没有任何益处.1473字节的ICMP报文数据的前1472字节被IP模块复制到第一个IP分片中,使其总长度为1500字节,从而满足MTU的要求, 而多出的最后1字节则被复制到第二个IP分片中.
4. ICMP报文的头部长度取决于报文的类型,其变化范围很大.本示例中以8字节为例,因为后面的例子用到了ping程序,而ping程序使用的ICMP回显和应答报文的头部长度是8字节.

**使用tcpdump分析IP数据报分片行为**:
环境说明: 在mac机器上使用ping命令(ping使用ICMP协议), ping linux VM.

**step 1**: 抓取本机以太网发出的icmp数据报.
```
sudo tcpdump -ntv icmp
```

**step 2**: 开启另外一个终端, ping linux VM, 通过**-s**参数指定发送数据包的长度为1473(设置为1473的原因为强制IP数据报分片, 1500[mtu] - 20[IP header length] - 8[icmp header length] = 1472, 再+1 = 1473).
```
ping -s 1473 10.211.5.4 
```

**tcpdump获取到的前2个数据报文为**:
```
IP (tos 0x0, ttl 64, id 28431, offset 0, flags [+], proto ICMP (1), length 1500)
    10.211.55.4 > 10.211.55.2: ICMP echo reply, id 44050, seq 1, length 1480
IP (tos 0x0, ttl 64, id 28431, offset 1480, flags [none], proto ICMP (1), length 21)
    10.211.55.4 > 10.211.55.2: ip-proto-1
```

上述报文的信息显示出:
1. 2个报文的id为: 28431, 表示属于同一个数据报的两个分片.
2. 第一个报文的offset为0, 第二个报文的offset为1480, 表示除去IP数据报头部的20字节之外, IP数据内容的长度为1480.
3. 第一个报文中flags [+]表示More fragment. 第二个数据报flags [none]表示没有设置何标记, 因为已经是最后一个分片.
4. 这两个数据报内容的长度分别为: 1480和21.(length 1480/length 21).

## IP路由
---
**What?**:
IP协议的一个核心任务是数据报的路由,即决定发送数据报到目标机器的路径.

**How?**:
IP模块的基本工作流程, 参考下图:
![ip-workflow](https://www.zhuxiaodong.net/static/images/ip-workflow.jpg)

我们从右向左进行分析:
1. 当IP模块接收到来自数据链路层的IP数据报时,它首先对该数据报的头部做CRC校验,确认无误之后就分析其头部的具体信息.
2. 如果该IP数据报的头部设置了源站选路选项(松散源路由选择或严格源路由选择),则IP模块调用数据报转发子模块来处理该数据报.如果该IP数据报的头部中目标IP地址是本机的某个IP地址,或者是广播地址,即该数据报是发送给本机的,则IP模块就根据数据报头部中的协议字段来决定将它派发给哪个上层应用(分用).如果IP模块发现这个数据报不是发送给本机的,则也调用数据报转发子模块来处理该数据报.
3. 数据报转发子模块将首先检测系统是否允许转发,如果不允许,IP模块就将数据报丢弃.如果允许,数据报转发子模块将对该数据报执行一些操作,然后将它交给IP数据报输出子模块.
4. IP数据报应该发送至哪个下一跳路由(或者目标机器),以及经过哪个网卡来发送,就是IP路由过程,即上图中“计算下一跳路由”子模块.IP模块实现数据报路由的核心数据结构是路由表.这个表按照数据报的目标IP地址分类,同一类型的IP数据报将被发往相同的下一跳路由器(或者目标机器).
5. IP输出队列中存放的是所有等待发送的IP数据报,其中除了需要转发的IP数据报外,还包括封装了本机上层数据(ICMP报文、TCP报文段和UDP数据报)的IP数据报.
6. 上图中的虚线箭头显示了路由表更新的过程.这一过程是指通过路由协议或者route命令调整路由表,使之更适应最新的网络拓扑结构,称为IP路由策略.

**路由机制**:

使用route命令在linux VM上查看route table(路由表):
```
route
```

得到的路由表为:
```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    100    0        0 eth0
10.211.55.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```

* Destination: 表示目标网络或主机.
* Gateway: 网关地址. * 表示目标和本机在同一个网络, 不需要进行路由.
* Genmask: 子网掩码.
* Flags: 路由标志项, 常见的包括: 
  * U (route is up)
  * H (target is a host)
  * G (use gateway)
  * R (reinstate route for dynamic routing)
  * M (modified from routing daemon or redirect)
  * A (installed by addrconf)
  * C (cache entry)
  * ! (reject route)
* Metric: 路由距离, 即到达指定网络需要的中转数.
* Ref: 路由被引用的次数. (linux内核并未使用该字段).
* Use: 该路由项被引用的次数.
* Iface: 该路由所使用的网卡接口.

上述路由表中, 第一项default表示默认路由选项, UG表示该路由项是活动的, 并且使用网关, 其地址是: gateway(10.211.55.1).

路由机制的3个步骤:
1. 查找路由表中和数据报的目标IP地址完全匹配的主机IP地址.如果找到,就使用该路由项,没找到则转步骤2.
2. 查找路由表中和数据报的目标IP地址具有相同网路ID的网络IP地址.如果找到,就使用该路由项, 没找到则转步骤3.
3. 选择默认路由项,这通常意味着数据报的下一跳路由是网关.

**路由表的更新**:
目的: 路由表必须能够更新,以反映网络连接的变化,这样IP模块才能准确、高效地转发数据报.

接下来我们将使用route系列命令, 做一系列测试.
**测试环境说明**:
* mac os x上使用parallels desktop搭建虚拟机.
* mac os x, IP: 10.211.55.2
* linux centos 7 VM, IP: 10.211.55.4
* windows 7 VM, IP: 10.211.55.3

如下的命令都在linux centos 7 VM上执行, 初始不做任何调整时, route -n得到的路由表参考上文中的描述.
**step 1**: 添加mac osx的路由.

```
sudo route add 10.211.55.2 dev eth0
```

此时路由表中添加了一条, Flags H表示该路由项的目标是一台主机.

```
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.211.55.2     0.0.0.0         255.255.255.255 UH    0      0        0 eth0
```
**step 2**: 删除掉10.211.55.0的路由, 表示删除掉linux虚拟机与mac osx host(由于步骤1已经手动添加了路由, 因此mac osx还可以访问)或window VM之间的通信.

```
sudo route del -net 10.211.55.0 netmask 255.255.255.0
```

此时你会发现无法访问到windows VM(ping 10.211.55.3), 但可以访问mac osx(ping 10.211.55.2).

**step 3**: 删除掉默认的路由, 此时将linux VM将无法访问internet(curl www.baidu.com).
```
sudo route del default

curl www.baidu.com
curl: (7) Failed to connect to 180.97.33.107: Network is unreachable
```

此时linux vm的路由表为:
```
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.211.55.2     0.0.0.0         255.255.255.255 UH    0      0        0 eth0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```

NOTE: 
1. **使用route命令只能够临时修改路由表, 重启了之后会丢失修改信息.**
2. 如果需要持久化路由信息, 参考:
[http://www.cyberciti.biz/faq/centos-linux-add-route-command/](http://www.cyberciti.biz/faq/centos-linux-add-route-command/)
[https://linuxconfig.org/how-to-add-new-static-route-on-rhel7-linux](https://linuxconfig.org/how-to-add-new-static-route-on-rhel7-linux)

**more information ref**:
* [鸟哥的私房菜](http://vbird.dic.ksu.edu.tw/linux_server/0230router_1.php)
* [http://www.thegeekstuff.com/2012/04/ip-routing-intro/](http://www.thegeekstuff.com/2012/04/ip-routing-intro/)
* [http://www.thegeekstuff.com/2012/04/route-examples/](http://www.thegeekstuff.com/2012/04/route-examples/)
* [http://www.thegeekstuff.com/2012/05/route-flags/](http://www.thegeekstuff.com/2012/05/route-flags/)

## IP转发
---
通常的情况下, 主机只负责IP数据报的接收和发送, 而专门的IP数据报转发的功能则交给路由器来完成.

如果需要设置主机打开IP转发的功能, 可以参考如下的方式: 
**Linux**:
```
echo 1>/proc/sys/net/ipv4/ip_forward
```

**mac os x**:
```
sudo sysctl -e net.inet.ip.forwarding=1 #setting allow ip forwarding.
sudo sysctl -a net.inet.ip.forwarding #view net.inet.ip.forwarding value.
```

允许IP数据报转发的主机或路由器, 数据报转发子模块将会做如下的操作:
1. 检查数据报头部的TTL值.如果TTL值已经是0,则丢弃该数据报.
2. 查看数据报头部的严格源路由选择选项.如果该选项被设置,则检测数据报的目标IP地址是否是本机的某个IP地址.如果不是,则发送一个ICMP源站选路失败报文给发送端.
3. 如果有必要,则给源端发送一个ICMP重定向报文,以告诉它一个更合理的下一跳路由器.
4. TTL值减1.
5. 处理IP头部选项.
6. 如果有必要,则执行IP分片操作.

## 基于ICMP协议的重定向
---

ICMP重定向的报文格式参考下图:
![icmp-redirect](https://www.zhuxiaodong.net/static/images/icmp-redirect.jpg)
我们曾经在[第一章](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part1-tcp-ip-summarize/#%E5%85%B3%E4%BA%8E%E6%9C%AC%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0)中简要介绍过ICMP协议, 其中8 bit类型字段, 8 bit代码字段, 16 bit校验和为固定字段. 当类型字段值为5时表示重定向.

ICMP重定向报文的数据内容部分, 为接收方提供了2部分信息:
1. 引起重定向的IP数据报(上图中的原始IP数据报)的源端IP地址.
2. 应该使用的路由器的IP地址.

接收主机根据这两个信息就可以断定引起重定向的IP数据报应该使用哪个路由器来转发,并且以此来更新路由表(**通常是更新路由表缓冲,而不是直接更改路由表**).

在linux当中我们可以通过如下命令查看当前主机是否允许发送ICMP重定向报文:
```
cat /proc/sys/net/ipv4/conf/all/send_redirects
```

通过如下命令查看当前主机是否允许接收ICMP重定向报文:
```
cat /proc/sys/net/ipv4/conf/all/accept_redirects
```
一般来说,**主机只能接收ICMP重定向报文,而路由器只能发送ICMP重定向报文**.

## IPV6协议
---
**Why**?:
它解决了IPv4地址不够用的问题,还做了很大的改进.比如,增加了多播和流的功能,为网络上多媒体内容的质量提供精细的控制；引入自动配置功能,使得局域网管理更方便；增加了专门的网络安全功能等.

**IPV6与IPV4**:
IPv6协议并不是IPv4协议的简单扩展,而是完全独立的协议.用以太网帧封装的IPv6数据报和IPv4数据报具有不同的类型值.IPv4数据报的以太网帧封装类型值是0x800,而IPv6数据报的以太网帧封装类型值是0x86dd(见RFC2464)

**IPV6协议的头部结构**:
IPv6头部由**40字节的固定头部**和**可变长的扩展头部**组成.

1. 固定头部结构:
![ipv6-fixed-header](https://www.zhuxiaodong.net/static/images/ipv6-fixed-header.jpg)

* 4位版本号(version)指定IP协议的版本.对IPv6来说,**其值是6**.
* 8位通信类型(traffic class)指示数据流通信类型或优先级, 与IPV4的TOS字段类似.
* 20位流标签(flow label)是IPv6新增加的字段,用于某些对连接的服务质量有特殊要求的通信,比如**音频或视频等实时数据传输**.
* 16位净荷长度(payload length)指的是**IPv6扩展头部和应用程序数据长度之和,不包括固定头部长度**.
* 8位下一个包头(next header)指出紧跟IPv6固定头部后的包头类型,如扩展头(如果有的话)或某个上层协议头(比如TCP,UDP或ICMP).**它类似于IPv4头部中的协议字段,且相同的取值有相同的含义**.
* 8位跳数限制(hop limit)和IPv4中的TTL含义相同.
* IPv6用128位(16字节)来表示IP地址,使得IP地址的总量达到了2的128次方个.所以有人说,“IPv6使得地球上的每粒沙子都有一个IP地址”.
* IPV6采用16进制字符串表示IP地址, 例如: "FE80:0000:0000:0000:1234:5678:0000:0012"
  * 使用":"分隔为8组, 每组包括2个字节.
  * 由于上述方式表述比较麻烦, 通常使用**"零压缩发"**来进行表示, 即: "FE80:1234:5678:0000:0012".
  * 零压缩法对一个IPv6地址只能使用一次, 例如上述示例中, 5678之后的0000就无法再进行省略了, 否则我们就无法计算每个":"之间省略了多少个全零组.

2. 扩展头部结构:
可变长的扩展头部使得IPv6能支持更多的选项,并且很便于将来的扩展需要.它的长度可以是0,表示数据报没使用任何扩展头部.一个数据报可以包含多个扩展头部,每个扩展头部的类型由前一个头部(固定头部或扩展头部)中的下一个报头字段指定.
![ipv6-extend-header](https://www.zhuxiaodong.net/static/images/ipv6-extend-header.jpg)


