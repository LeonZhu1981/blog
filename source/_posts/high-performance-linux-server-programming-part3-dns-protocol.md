title: 高性能Linux服务器编程-Part 3-DNS协议
date: 2016-03-27 23:09:06
categories: programming
tags:
- network
- tcp/ip
- dns
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
**定义/职责**: 将域名转换为IP地址.

**浏览器中输入一个域名之后**:
1. 获取输入URL的域名地址, 例如：www.google.com.
2. 向DNS服务器发送DNS查询请求, DNS服务器在网络连接的属性里面配置, 或者使用自动获取的.
3. 接收DNS服务器返回的响应报文, 并解析获得IP地址.

## DNS查询和应答报文详解
---
![dns数据报格式](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dns-datagram.png)

**DNS header**:
* **16 bit标识字段**: 用于标识**一组**DNS查询和应答, 以区分哪一个DNS应答是哪一个DNS查询的回应.
* **16 bit标记字段**: 用于协商具体的通信方式和反馈通信状态, 具体参考下图:
![dns标识字段](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dns-identity.png)
  * QR: 0表示查询, 1标识应答.
  * opcode: 定义查询和应答的类型, 0--标准查询, 1--反向查询(通过IP获得主机名), 2--请求服务器状态.
  * AA: 占1位, 授权应答(Authoritative Answer) – 这个比特位在应答的时候才有意义, 指出给出应答的服务器是查询域名的授权解析服务器.
  * TC: 截断标志, 仅当DNS报文使用UDP服务时使用. 因为UDP数据报有长度限制, 所以过长的DNS报文将被截断. 1表示DNS报文超过512字节，并被截断.
  * RC: 递归查询标志. 1表示执行递归查询, 即如果目标DNS服务器无法解析某个主机名, 则它将向其他DNS服务器继续查询, 如此递归, 直到获得结果并把该结果返回给客户端. 0表示执行迭代查询, 即如果目标DNS服务器无法解析某个主机名, 则它将自己知道的其他DNS服务器的IP地址返回给客户端, 以供客户端参考.
  * RA: 允许递归标志. 仅由应答报文使用, 1表示DNS服务器支持递归查询.
  * zero: 未使用, 必须都社会为0.
  * rcode: 4位返回码, 表示应答的状态. 常用值有0--无错误, 3--域名不存在.

* 16 bit查询问题个数--QDCOUNT: 指明报文请求段中的问题记录数.
* 16 bit回答问题个数--ANCOUNT: 指明报文回答段中的回答记录数.
* 16 bit授权问题个数--NSCOUNT: 指明报文授权段中的授权记录数.
* 16 bit额外资源问题个数--ARCOUNT: 指明报文附加段中的附加记录数.

对于上述4个字段, 查询报文和回答报文的值分别如下:
| 请求类型  | QDCOUNT | ANCOUNT | NSCOUNT | ARCOUNT |
| --------   | :-----: | :----: | :----: | :----: |
| 查询报文     | 1 | 0 | 0 | 0 |
| 回答报文     | 0 | 1 | 1 or 0 | 1 or 0 |

<!--more-->

**DNS body**:

* **查询问题字段的格式**:
![查询问题字段的格式](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dns-q-datagram.png)
  * 查询名以一定的格式封装了要查询的主机域名.
  * 16位查询类型: A(A记录)--值为1, 表示类型为IP地址; CNAME--值为5, 表示类型为别名; PTR--值为12, 表示反向查询.
  * 16位查询类: 通常值为1, 表示获取IP地址.

* 应答字段/授权字段/额外信息字段都使用资源记录(Resource Record, RR)格式, RR格式如下图:
![dns资源记录格式](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dns-rr-datagram.png)
  * 32位域名是该记录中与资源对应的名字, 其格式和查询问题中的查询名字段相同.
  * 16位类型和16位类字段的含义也与DNS查询问题的对应字段相同.
  * 32位生存时间表示该查询记录结果可被本地客户端程序缓存多长时间, 单位是秒. (下图是DNSPod中的ttl设置)
  ![dnspod](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dnspod-ttl.png)
  * 16位资源数据长度字段和资源数据字段的内容取决于类型字段. 对A记录而言, 资源数据是32位的IPv4地址, 资源数据长度则为4(以字节为单位). 对于CNAME, 或者MX类型而言, 资源数据应该是对应的值. (这就是为什么资源数据的长度是可变的原因.)

* Related RFC: RFC 1035, RFC 1886.

## 访问DNS服务
---
我们需要通过DNS服务器去查询对应域名的IP地址, 那么DNS服务器是如何设置的了?

通常的情况下, 我们是通过DHCP协议来获取到默认的DNS Server IP的. [参考这里](https://www.zhihu.com/question/21436768)

我们可以如下的命令来获取dns server ip address.
```
cat /etc/resolv.conf
```
![dns-resolv](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dns-resolv1.png)

当然, 我们可以手动来设置一些公共dns服务器, 例如google的8.8.8.8和114dns 114.114.114.114. 这里我们使用**阿里**([看上去很专业](http://www.alidns.com/))和**dnspod**([貌似被腾讯收购了?](https://www.dnspod.cn/Products/Public.DNS))的public DNS+ server来做下实验.

```
sudo vi /etc/resolv.conf
```
>nameserver 223.5.5.5
>nameserver 119.29.29.29
>nameserver 223.6.6.6
>nameserver 182.254.116.116

设置好了之后, 我们可以通过dig or host or nslookup这3个shell来查询一个域名. 这三者有什么区别了?
> * Dig: Linux utility, needs compiled to run in windows. Provides more detail, more advanced domain info. The more information the better.
> * Host: Unix based. Replacement for nslookup with more options and detail. 
> * nslookup: Windows and Unix based. No longer supported in Unix-based systems. Provides basic info. for name queries. 
>
> **Dig is highly recommended given its options and detailed information if you require such info. For simple queries, nslookup or host will provide you the info. you need.**

ref: [here](https://www.quora.com/Bash-shell/What-are-the-differences-between-host-dig-and-nslookup-and-when-should-I-use-each)

具体命令为:
```
dig www.baidu.cn
```
![dns-baidu](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dig-baidu.jpg)
上述图例中比较重要的部分:
>;; QUESTION SECTION:
>;www.baidu.com.			IN	A
可以看到查询域名为: www.baidu.com, 类型是A记录.

>;; ANSWER SECTION:
>www.baidu.com.		127	IN	CNAME	www.a.shifen.com.
>www.a.shifen.com.	127	IN	A	180.97.33.107
>www.a.shifen.com.	127	IN	A	180.97.33.108
可以看到首先查询到[nolink]www.baidu.com是www.a.shifen.com("**十分**"是什么?)的别名. 
后返回了两个IP address: 180.97.33.107, 180.97.33.108

**程序员必须要会用google** **程序员必须要会用google** **程序员必须要会用google**

**Q1**: 为什么有2个IP?
google search "one domain name with multiple ip addresses". 
Wow, 原来是为了让DNS server做load balancing, 使用了round robin DNS.
> This is round robin DNS. This is a quite simple solution for **load balancing**. Usually DNS servers rotate/shuffle the DNS records for each incoming DNS request. Unfortunately it's not a **real solution for fail-over**. If one of the servers fail, some visitors will still be directed to this failed server.

> Normally you would not uses hosts to do this, but your DNS. Most DNS will provide what's called a "Round Robin" if you assign multiple A records to the one name in the zone.
>
> What it would do then, is the first request comes through would receive 192.168.244.128, the next would receive 192.168.226.129, so on and so forth. **However, by design, your local machine will cache its DNS resolution, and will usually use the same IP address over and over, until it expires (Time To Live, TTL).**

ref: 
[here 1](http://stackoverflow.com/questions/10257969/is-it-possible-that-one-domain-name-has-multiple-corresponding-ip-addresses)
[here 2](http://serverfault.com/questions/69836/point-multiple-ip-addresses-to-a-single-host-name)

注意第二个在serverfault.com的回答中有一句提到了:
> However, by design, your local machine will cache its DNS resolution, and will usually use the same IP address over and over, until it expires (Time To Live, TTL).

**Q2**: 那么缓存的时间到底是多久了? 继续google之. oh, 原来这里的ANSWER SECTION里面的246就是TTL.
![dns-ttl1](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dig-ttl1.jpg)

dig命令是显示当前的ttl时间, 每隔1秒钟会减1, 因此每次执行dig命令时, 显示是当前的缓存时间, 下图是在上一次执行dig之后大概66秒(**246 - 180 = 66**)之后的图例: 
![dns-ttl2](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dig-ttl2.jpg)

**Q3**: 有没有办法trace到绝对的ttl时间了? 答案是+trace命令参数: 
```
dig +trace +nocmd +noall +answer +ttlid www.baidu.com
```
![dns-ttl3](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dig-ttl3.jpg)

**Q4**: 如果我换成DNSPod的DNS服务器, www.baidu.com的ttl时间是一致的么?
![dns-ttl4](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dig-ttl4.jpg)
wow, 原来是一样的, 都是1200, 参考我们在[这里](http://7i7i6p.com1.z0.glb.clouddn.com/blog/static/images/dnspod-ttl.png)提到的是一致的.

**Q5**: 假设域名对应的IP变化了, 我们想要ttl失效, 该怎么办?
* mac os x: [more information](https://support.apple.com/en-us/HT202516)
```
sudo killall -HUP mDNSResponder # os x 10.10.4 or later.
```
* windows:
```
ipconfig /flushdns
```
* linux: [more information](http://www.cyberciti.biz/faq/rhel-debian-ubuntu-flush-clear-dns-cache/)
```
sudo /etc/init.d/nscd restart
or
sudo service nscd restart
or
sudo service nscd reload
```

当然你也可以通过host or nslookup来查询.
```
host -a www.baidu.com
nslookup www.baidu.com
```

ref:
[10 Linux DIG Command Examples for DNS Lookup](http://www.thegeekstuff.com/2012/02/dig-command-examples/)
[HowTo: Find Out DNS Server IP Address Used By My Router?](http://www.cyberciti.biz/faq/how-to-find-out-dns-for-router/)
[How can I see Time-To-Live (TTL) for a DNS record?](http://serverfault.com/questions/179630/how-can-i-see-time-to-live-ttl-for-a-dns-record)
[Dig Command Find Out TTL (Time to Live) Value For DNS Records](http://www.cyberciti.biz/faq/howto-use-dig-to-find-dns-time-to-live-ttl-values/)

## 使用tcpdump观察DNS协议的通信过程
---
使用tcpdump抓取dns查询的数据报:
```
sudo tcpdump -i en0 -nt port domain
```
显示的结果为:
> IP 192.168.11.2.55411 > 223.5.5.5.53: 57184+ A? www.baidu.com. (31)
> IP 223.5.5.5.53 > 192.168.11.2.55411: 57184 3/0/0 CNAME www.a.shifen.com., A 180.97.33.107, A 180.97.33.108 (90)

* 第一行: 由local(192.168.11.2:55411端口)向我们配置的阿里public DNS server(223.5.5.5:53)发起DNS QUERY请求, 57184是DNS查询报文的值, "+"表示递归查询, "A?"表示使用A类型的查询方式, 31表示查询报文的长度(byte).
* 第二行: 3/0/0表示3个回答记录, 0个授权资源记录, 0个额外信息记录. 其余部分与我们在前面介绍的部分相同.
