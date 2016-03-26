title: 高性能Linux服务器编程-Part 1-TCP/IP协议概述.
date: 2016-03-25 16:53:03
categories: programming
tags: 
- linux 
- tcp/ip
---
## 关于本系列文章
---
我在学习TCP/IP协议以及Linux服务器端编程时, 原本打算学习和参考三本经典书籍:
1. [TCP/IP详解 卷1：协议](http://book.douban.com/subject/1088054/)
![TCP/IP详解 卷1：协议](http://img3.doubanio.com/lpic/s1543906.jpg)
2. [UNIX环境高级编程(第3版)](http://book.douban.com/subject/25900403/)
![UNIX环境高级编程(第3版)](http://img3.douban.com/lpic/s4436543.jpg)
3. [UNIX网络编程 卷1：套接字联网API](http://book.douban.com/subject/4859464/)
![UNIX网络编程 卷1：套接字联网API](http://img3.doubanio.com/lpic/s4437258.jpg)

在经过阅读了之后, 发现这三本书不是那么容易入门, 书也太厚了, 不方便阅读. 偶然的机会, 我在多看上发现了一本还算不错的入门书籍:
**[Linux高性能服务器编程](http://book.douban.com/subject/24722611/)**
![Linux高性能服务器编程](http://img3.doubanio.com/lpic/s27327287.jpg)

这本书有以下几个特点:
* 书比较薄, 并涵盖了我想了解和学习的知识点, 很适合入门.
* 有很多需要实际动手的Demo, 能够帮助读者理解.
* 有电子版, 随时都能阅读: [豆瓣电子版](https://read.douban.com/ebook/15233070/?dcs=subject-einfo&dcm=douban&dct=24722611) [多看电子版](http://www.duokan.com/book/55818) PS: 多看电子版的代码排版有点问题, 希望多看能够早点修复.

这个系列的文章是我阅读[Linux高性能服务器编程](http://book.douban.com/subject/24722611/)的读书笔记, 并对需要深入学习的部分加入了一些自己的理解和认识, 以及额外的参考资料. 如果你认为本系列对你的学习能够有所帮助, 希望能够购买原版图书. 

## RFC文档
---
What is RFC?
> A Request for Comments (RFC) is a formal document from the Internet Engineering Task Force ( IETF ) that is the result of committee drafting and subsequent review by interested parties. Some RFCs are informational in nature.

当你对某个TCP/IP的问题搞不明白时, 最好去查看的RFC文档, 这才是最权威的解释.

* [中文](http://man.chinaunix.net/develop/rfc/default.htm)
* [英文](https://www.ietf.org/rfc.html)
  

## TCP/IP协议族体系结构以及主要协议
---
四层:

* 链路层
* 网络层
* 传输层
* 应用层

**NOTE: 链路层, 网络层, 传输层由于需要稳定而可靠的服务, 都在内核空间当中实现. 只有应用层是交给用户空间实现.**
![tcp/ip协议体系结构图](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/tcp-ip-overview-1.png)

### 数据链路层
---
**定义/职责**: 实现了网卡接口和网络驱动程序, 以处理数据在物理媒介(例如, 以太网和令牌环网络)上的传输. 定义了统一的接口, 屏蔽了物理层传输的细节.

**常用协议**: ARP(Address Resolve Protocol) and RARP(Reverse Address Resolve Protocol), 实现了IP地址到物理地址(通常是mac地址, 以太网, 令牌环, 802.11无线网络都使用mac地址作为物理地址.)之间的转换.
* ARP比较好理解, 也是大多数情况下会使用的协议.
* RARP用于无盘工作站, 由于无存储设备, 因此无法记录IP地址, 但是可以通过mac地址向某个IP地址提供者(例如其它服务器 or 网络管理软件)查询自生的IP地址. 相当于通过一个第三方, 进行mac地址到IP地址的反向查询.

### 网络层
---
**定义/职责**: 实现了数据包的路由和转发. 由于WAN(Wide area network-广域网) 通常使用众多分级的路由器来连接分散的机器或者LAN(Local area netword-局域网), 两台主机一般不会直接连接, 而是需要多个中间节点(路由器)来连接. 

**网络层的任务**:

* 选择数据报要经过哪些中间节点, 以确定两台之间的通信路径.
* 屏蔽了链路层的网络拓扑结构的细节, 让上层(传输层, 应用层)看起来像是直接在通信.

**常用协议**: IP协议 and ICMP协议.

* IP协议: IP协议根据数据包的目的IP地址来决定如何投递它.**如果数据包不能直接发送给目标主机,那么IP协议就为它寻找一个合适的下一跳(next hop)(TTL减1直至被丢弃)路由器,并将数据包交付给该路由器来转发.**多次重复这一过程,数据包最终到达目标主机,或者由于发送失败而被丢弃.
* ICMP(Internet Control Message Protocol)协议: 它是IP协议的重要补充,主要用于**检测网络连接**.

下图为ICMP报文格式:
![ICMP报文格式](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/icmp-datagram.png)

* 8 bit类型字段: 标识报文类型. 
  * 差错报文: 主要用于回应网络错误. 比如目标不可达(值为3), 目标重定向(值为5)
  * 查询报文: 主要用于查询网络信息. 比如ping程序使用ICMP查询目标是否可达(值为8).有的ICMP报文还使用8位代码字段来进一步细分不同的条件.比如重定向报文使用代码值0表示对网络重定向,代码值1表示对主机重定向.ICMP报文使用16位校验和字段对整个报文(包括头部和内容部分)进行循环冗余校验(Cyclic RedundancyCheck,CRC),以检验报文在传输过程中是否损坏. 关于完整的类型编码, 参考: http://www.nthelp.com/icmp.html
* 完整的协议介绍参考RFC 792
* 需要指出的是,**ICMP协议并非严格意义上的网络层协议,因为它使用处于同一层的IP协议提供的服务(一般来说,上层协议使用下层协议提供的服务).**

### 传输层
---
**定义/职责**: 传输层为两台主机上的应用程序提供端到端(end to end)的通信.与网络层使用的逐跳通信方式不同,传输层只关心通信的起始端和目的端,而不在乎数据包的中转过程.
![传输层与网络层的区别](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/diff-network-transmission.png)

**传输层的任务**: 为应用程序封装了一条端到端的逻辑通信链路,它负责数据的收发、链路的超时重连等.

**常用协议**: TCP UDP SCTP
* TCP(Transmission Control Protocol): 面向连接, 可靠, 基于流(stream)的协议.
  * **可靠性:** 使用超时重传、数据确认等方式来确保数据包被正确地发送至目的端.
  * **面向连接:** 使用TCP通信两端之前必须先建立连接,并在内核中为该连接维持一些必要的数据结构,比如连接的状态、读写缓冲区,以及诸多定时器等.当通信结束时,双方必须关闭连接以释放这些内核数据. 连接 and 关闭.
  * **基于流:** 基于流的数据没有边界(长度)限制,它源源不断地从通信的一端流入另一端.发送端可以逐个字节地向数据流中写入数据,接收端也可以逐个字节地将它们读出.

* UDP(User Datagram Protocol): 无连接, 不可靠, 基于数据报的协议.
  * **不靠性:** 无法保证数据从发送端正确地传送到目的端.如果数据在中途丢失,或者目的端通过数据校验发现数据错误而将其丢弃,则UDP协议只是简单地通知应用程序发送失败.
  * **无连接:** 使用TCP通信两端之前必须先建立连接,并在内核中为该连接维持一些必要的数据结构,比如连接的状态、读写缓冲区,以及诸多定时器等.当通信结束时,双方必须关闭连接以释放这些内核数据. 连接 and 关闭.
  * **基于数据报:** 基于数据报的服务,是相对基于流的服务而言的.每个UDP数据报都有一个长度,接收端必须以该长度为最小单位将其所有内容一次性读出,否则数据将被截断.

* **是UDP还是TCP? 到底如何选?**
“When in doubt, use TCP"
参考: https://www.zhihu.com/question/20060141

* SCTP: SCTP 是在 IP 网络上使用的一种可靠的通用传输层协议.尽管 SCTP 协议最初是为发送电话信号而设计的(RFC 2960),但带来了一个意外的收获：它通过借鉴 UDP 的优点解决了 TCP 的某些局限.SCTP 提供的特性使套接字初始化的可用性、可靠性和安全性都得以提高.
参考: https://www.ibm.com/developerworks/cn/linux/l-sctp/

### 应用层
---
**定义/职责**: 负责处理应用程序的逻辑. 应用层在用户空间实现,因为它负责处理众多逻辑,比如文件传输、名称查询和网络管理等. 如果应用层也在内核中实现,则会使内核变得非常庞大.当然,也有少数服务器程序是在内核中实现的,这样代码就无须在用户空间和内核空间来回切换(主要是数据的复制),极大地提高了工作效率.

**常见应用程序/协议:**
* ping
* telnet
* SSH
* DNS
* http

**NOTE:**
* 某些应用程序/协议**会跳过传输层**, 直接和网络层进行通信, 例如: ping程序.
* 应用程序/协议**可以使用任意传输层协议**来实现, 例如TCP, UDP. 还可以**组合一个或多个传输层协议**来实现. (DNS).
* 通过/etc/services来查看有哪些应用层程序/协议-端口的实现.
```
sudo vi /etc/services
```
![/etc/services](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/etc-services.png)

### 封装(**Encapsulation**)
---
**问题:** 上层协议是如何使用下层协议提供的服务?如何将应用层的数据转换为frame并在物理链路层传输?

**定义/职责:** 应用程序数据在发送到物理网络上之前,将沿着协议栈**从上往下依次传递**.**每层协议都将在上层数据的基础上加上自己的头部信息(有时还包括尾部信息)**,以实现该层的功能,这个过程就称为封装.
![封装](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/encapsulation.png)

**封装步骤:**
1. 经过TCP封装后的数据称为TCP报文段(**TCP message segment**),或者简称TCP段.TCP协议为通信双方维持一个连接,并且在内核中存储相关数据.这部分数据中的TCP头部信息和TCP内核缓冲区发送缓冲区或接收缓冲区)数据一起构成了TCP报文段.当发送端应用程序使用send(或者write)函数向一个TCP连接写入数据时,内核中的TCP模块首先把这些数据复制到与该连接对应的TCP内核发送缓冲区中,然后TCP模块调用IP模块提供的服务,传递的参数包括TCP头部信息和TCP发送缓冲区中的数据,即TCP报文段.
![tcp封装过程](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/tcp-ip-encapsulation.png)
2. 如果是UDP协议, 经过UDP封装后的数据称为UDP数据报(**UDP Datagram**).不同的是,**UDP无须为应用层数据保存副本**,因为它提供的服务是不可靠的.**当一个UDP数据报被成功发送之后,UDP内核缓冲区中的该数据报就被丢弃了.**如果应用程序检测到该数据报未能被接收端正确接收,并打算重发这个数据报,则应用程序需要重新从用户空间将该数据报拷贝到UDP内核发送缓冲区中.
3. 经过IP封装后的数据称为IP数据报(**IP Datagram**).IP数据报也包括头部信息和数据部分,其中数据部分就是一个TCP报文段、UDP数据报或者ICMP报文.
4. 经过数据链路层封装的数据称为帧(**frame**).传输媒介不同,帧的类型也不同.比如,以太网上传输的是以太网帧(Ethernet frame),而令牌环网络上传输的则是令牌环帧(token ring frame).
5. 以Ethernet frame为例:
![以太网封装过程](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/ethernet-encapsulation.png)以太网帧使用6字节的目的物理地址和6字节的源物理地址来表示通信的双方.4字节CRC字段对帧的其他部分提供循环冗余校验.帧的最大传输单元(**Max Transmit Unit,MTU**),即帧最多能携带多少上层协议数据(比如IP数据报),通常受到网络类型的限制.**以太网帧的MTU是1500字节.正因为如此,过长的IP数据报可能需要被分片(fragment)传输.**
6. 上述3个步骤, 完成了最终的封装过程. **frame才是最终在物理链路层上传输的二进制字节方式.**

### 分用(**Demultiplexing**)
---
**问题:** 当frame通过物理链路层到达目的主机之后, 如何将frame进行分解并传递给上层直至应用程序？

**定义/职责:** 当帧到达目的主机时,将沿着协议栈**自底向上依次传递**.各层协议依次处理帧中本层负责的头部数据,以获取所需的信息,并最终将处理后的帧交给目标应用程序.这个过程称为分用(**demultiplexing**).分用是依靠头部信息中的类型字段实现的.标准文档RFC 1700定义了所有标识上层协议的类型字段以及每个上层协议对应的数值.
![分用](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/demultiplexing.png)

**分用步骤:**
1. 在物理链路层传递给**网卡驱动程序**, 将frame分解为网络层可以理解的数据格式(满足某种协议规范), 包括: IP, ARP, RARP. 根据frame当中的Type字段来标识出对应的协议类型. 比如: Eternet frame中, 包括2字节类型字段. 具体的值包括:
>* 0x800: IP
>* 0x806: ARP
>* 0x835: RARP
2. 同样, 基于IP协议的ICMP, TCP, UDP, 会根据IP Datagram Header当中的16 bit协议类型字段, 来标识需要提供给传输层的哪一个具体的协议去处理.
3. 最后, 传输层的TCP, UDP会根据TCP Message Segment or UDP Datagram中header部分的16 bit 端口号, (再次参考: /etc/services中的定义)来决定交给哪个应用层协议/程序去进行处理. 比如DNS的53, HTTP的80, FTP的22.

最终上述3个过程完成了封送, 最终将封装前的原始数据送至目标服务:ARP服务、RARP服务、ICMP服务或者应用程序/协议.

