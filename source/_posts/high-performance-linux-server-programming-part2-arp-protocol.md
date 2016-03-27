title: 高性能Linux服务器编程-Part 2-ARP协议
date: 2016-03-27 00:48:10
categories: programming
tags:
- network
- tcp/ip
- 高性能Linux服务器编程
---

## 关于本系列文章
---
[**传送门**](http://www.zhuxiaodong.net/2016/high-performance-linux-server-programming-part1-tcp-ip-summarize/#关于本系列文章)

## 网络测试环境搭建和说明
* mac os x上使用parallels desktop搭建虚拟机.
* mac os x, IP: 10.211.55.2
* linux centos 7 VM, IP: 10.211.55.4
* windows 7 VM, IP: 10.211.55.3

## 概述
**定义/职责**: ARP协议能实现**任意网络层地址到任意物理地址的转换**. 这里重点讨论的是: **将网络层的IP Datagram Header中的IP Address, 解析为物理链路层frame Header中的mac address.**

**原理**: 主机向自己所在的网络**广播**一个ARP请求，该请求包含目标机器的网络地址。此网络上的**其他机器都将收到这个请求**，但只有被请求的**目标机器会回应一个ARP应答**，其中包含自己的物理地址.

## 报文详解
---
![ARP报文格式](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/arp-datagram.png)

* 硬件类型字段定义物理地址的类型，它的值为1表示MAC地址。
* 协议类型字段表示要映射的协议地址类型，它的值为0x800，表示IP地址。
* 硬件地址长度字段和协议地址长度字段，顾名思义，其单位是字节。对MAC地址来说，其长度为6；对IP（v4）地址来说，其长度为4。
* 操作字段指出4种操作类型：ARP请求（值为1）、ARP应答（值为2）、RARP请求（值为3）和RARP应答（值为4）。
* 最后4个字段指定通信双方的以太网地址和IP地址。发送端填充除目的端以太网地址外的其他3个字段，以构建ARP请求并发送之。接收端发现该请求的目的端IP地址是自己，就把自己的以太网地址填进去，然后交换两个目的端地址和两个发送端地址，以构建ARP应答并返回之（当然，如前所述，操作字段需要设置为2）。
<!--more-->
## ARP缓存的查看和修改
---
**缓存的目的/作用**: ARP维护一个高速缓存，其中包含经常访问（比如网关地址）或最近访问的机器的IP地址到物理地址的映射。这样就避免了重复的ARP请求，提高了发送数据包的速度。

**linux下查看ARP缓存**:
```
sudo arp -a
```
![linux下查看ARP缓存](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/arp-a.png)
* 上述该命令是在我的linux VM上执行.
* 第一行是我的mac os x的IP(10.211.55.2)地址与mac address(00:1c:42:00:00:08), 可以看到, 这个映射关系是被缓存了的.
* 第二行是我的windows 7机器的IP(10.211.55.3)地址与mac address(incomplete), 可以看到, 这个映射关系目前还没有被缓存.
* 第三行是网关IP与mac address的映射.

**linux下修改/删除ARP缓存**:
```
sudo arp -d 10.211.55.3 #删除
sudo arp -s 10.211.55.3 00:1c:42:d3:07:ec  #手动添加或修改
```

使用-s手动添加的ARP cache, 会标识为PERM.
![linux下修改/删除ARP缓存](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/arp-s.png)

更多请参考: [http://www.kwx.gd/CentOSApp/centos-arp-bind.html](http://www.kwx.gd/CentOSApp/centos-arp-bind.html)

## 使用tcpdump观察ARP通信过程
---
**准备工作**:
1. Linux centos机器上安装echo service.
首先需要安装xinetd:  sudo yum install xinetd. [ref](http://www.linuxfromscratch.org/blfs/view/svn/server/xinetd.html)
配置开启echo service.
```
sudo vi /etc/xinetd.d/echo-stream
```
2. 修改disable = no.
  ![echo-stream-conf](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/echo-stream-conf.png)
3. 启动或者重启xinetd daemon程序
```
sudo systemctl start xinetd
sudo systemctl restart xinetd
```
4. echo service使用tcp协议, 并占用端口号:7 (查看/etc/services) 如果开启了防火墙, 需要打开port 7的访问:
```
sudo firewall-cmd --zone=public --add-port=7/tcp --permanent
sudo firewall-cmd --reload
```
5. 测试: mac os x机器上telnet, 输入什么, 就会显示什么.
![echo-service-test](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/echo-service-test.png)

**实际抓包测试**:
1. 先删除在mac os x机器上删除对centos 7的arp缓存记录.
```
sudo arp -d 10.211.55.4
```

2. 验证, 可以发现没有10.211.55.4的ARP缓存记录.
```
sudo arp -a
```
PS: mac os x与linux的呈现方式还不一样, linux下是incomplete, mac下是根本都不显示出来.
![arp-a-2](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/arp-a-2.png)

3. mac下开启另外一个终端, 用tcpdump抓包.
```
sudo tcpdump -ent '(dst 10.211.55.4 and src 10.211.55.2) or (dst 10.211.55.2 and src 10.211.55.4)'
```
NOTE:
-e 选项是只获取头部信息.
dst和src 指定抓取的目标IP和源IP.

4. telnet echo service
```
telnet 10.211.55.4 7
```
此时抓到的和ARP协议相关的数据包如下:
> 00:1c:42:00:00:08 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.211.55.4 tell 10.211.55.2, length 28
> 00:1c:42:ad:60:ec > 00:1c:42:00:00:08, ethertype ARP (0x0806), length 42: Reply 10.211.55.4 is-at 00:1c:42:ad:60:ec, length 28

**解读**:
1. 第一个数据包当中的00:1c:42:00:00:08是IP(10.211.55.2)的mac address, ff:ff:ff:ff:ff:ff是以太网的广播地址.  0x806表示类型为ARP, length 42表示整个长度为42 byte(实际长度为46 byte, tcpdump没有统计最后长度为4 byte的CRC校验字段). 其中28 byte为数据部分的长度. Request who-has 10.211.55.4 tell 10.211.55.2 表示是一个ARP request.

2. 第二个数据包当中的00:1c:42:ad:60:ec是IP(10.211.55.4)的目标机器mac address, Reply 10.211.55.4 is-at 00:1c:42:ad:60:ec表示是一个ARP reply.

**完整的通信过程**:
![ARP协议的通信过程](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/arp-c.png)



