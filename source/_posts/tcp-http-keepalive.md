title: tcp & http keep-alive 详解
date: 2018-03-22 09:10:44
categories: programming
tags:
- network
- tcp/ip
- http
- keepalive
- Content-Length
- Transfer-Encoding
---

# TCP 协议 keepalive
---

## 概述

TCP keepalive 在很多场景下都不是必须的，但是在某些特定的场景下，这个特性却会是非常有用。我们从 TCP keepalive 这个名字就可以大概理解这个特性的作用： `keep tcp alive` 。我们可以通过检查我们的 socket（TCP sockets） 来判断网络连接是不是正常的（ running or broken ）。

<!--more-->

keepalive 的概念非常的简单：当你建立一个 TCP连 接的时候，变有一组定时器（ timer ）与之绑定（ associate ）在一起。其中的一些定时器就用于处理 keepalive 过程。当 keepalive 定时器到0的时候，我们便会给对端发送一个不包含数据部分的 keepalive 探测包（ probe packet ），然后打开 ACK 标志。我们可以这样做（发送一个不包含数据部分的数据包）是因为 TCP/IP 规格里面有重复确认（ duplicate ACK ）机制，而且由于 TCP 是基于流的协议，所以对端也是没有任何参数的。另一方面，我们会受到对端（对端可以不支持 keepalive 选项，只要是 TCP/IP 就行）的确认消息，该消息也没有数据部分，只有 ACK 。

如果我们收到了 keepalive 探测包的回复消息，那么我们就可以断定连接依然是 OK 的，也不用担心用户层的实现。实际上， TCP 允许我们处理没有数据包的流，而且数据部分长度为0的数据包对于用户程序来说也是没有太多的网络开销带来的影响。如果我们没有收到对端 keepalive 探测包的 ack 消息，我们便可以断定连接已经不可用，进而采取一些措施，比如断开连接。keepalive 除了会额外产生一些网络数据包外（这些包将加大网络流量，对路由器和防火墙造成一定的负担，但由于数据包本身比较小，对网络的影响仍然在可控的范围之内），其它并没有什么太多的影响。

## TCP Keepalive 使用场景

### 检测对端的死连接

这里所谓的对端连接已经挂掉分两种场景：
* 对端还没有来得及通知我们就已经死掉了。比如系统内核突然挂掉，或是进程被直接终止等。
* 对端进程虽然是正常的，但是网络链路却出了故障。这种场景下，如果网络链路不恢复正常的话，对我们来说，对端依旧是挂掉的。在这两种场景下，对端在挂掉之前都是无法通知我们的。这些场景下，一般的 TCP 操作是检测不出来连接状态的。

我们假设一种A和B的连接场景，参考下图：

```
 _____                                                     _____
|     |                                                   |     |
|  A  |                                                   |  B  |
|_____|                                                   |_____|
   ^                                                         ^
   |--->--->--->-------------- SYN -------------->--->--->---|
   |---<---<---<------------ SYN/ACK ------------<---<---<---|
   |--->--->--->-------------- ACK -------------->--->--->---|
   |                                                         |
   |                                       system crash ---> X
   |
   |                                     system restart ---> ^
   |                                                         |
   |--->--->--->-------------- PSH -------------->--->--->---|
   |---<---<---<-------------- RST --------------<---<---<---|
   |                                                         |
```

A 和 B 已经通过三次握手建立了连接，此时我们我们认为连接已经稳定，我们可以在这条链路上面发送数据包了。但此时突然发生了一个意外： B 端机器突然断电了，而 B 还没有来得及通知A连接出问题了。而再看此时的 A 端， A 已经准备好接收 B 端发来的数据，却根本不知道 B 端已经 crash 了。此时，我们再回复 B 端的电源等待系统重启。此时的状态就是 A 和 B 都正常运行，并且 A 知道它和 B 之间有一条已经建立好的连接，但是 B 却不知道。这个时候，如果 A 试图通过这条连接向 B 发送数据，B 将回复一个 RST 数据包（在一个已关闭的 socket 上收到数据时，将发送 RST 数据包，要求对端关闭异常连接且对端不需要回复 ACK ），这样将导致 A 最终关闭这个连接。至此，这个死连接才算清理掉。

keepalive 可以帮助我们判断出对端变得不可达（ unreachable ），并且不会误报。实际上，如果是因为两端的网络导致的问题， keepalive 会过一些时间再重试一下，多次尝试之后才会将这个连接标记为不可用。

### 防止因为网络不活动而断连

keepalive 的另外一个目标就是防止因为网络不活动而断开网络连接。当我们在 NAT 代理或者使用防火墙的时候，经常会出现这种问题。这是由 NAT 代理和防火墙内部的实现导致的：NAT 代理和防火墙一般会记录所有通过他们的连接。但由于机器的物理资源限制，它们只能在内存中保存有限数量的连接。最常见的策略就是保持最新的连接，丢弃掉老的或者不活动的连接。

我们再来看以下的实际场景：

```
 _____           _____                                     _____
|     |         |     |                                   |     |
|  A  |         | NAT |                                   |  B  |
|_____|         |_____|                                   |_____|
  ^               ^                                         ^
  |--->--->--->---|----------- SYN ------------->--->--->---|
  |---<---<---<---|--------- SYN/ACK -----------<---<---<---|
  |--->--->--->---|----------- ACK ------------->--->--->---|
  |               |                                         |
  |               | <--- connection deleted from table      |
  |               |                                         |
  |--->- PSH ->---| <--- invalid connection                 |
  |               |                                         |
```

A 和 B 已经通过三次握手建立了稳定的连接。但是在较长时间间隔之后，A 和 B 会实际向对方发送数据。此时， A 和 B 的连接是有效的（建立连接后，如果不断开，则该连接一直保持），但是代理或者防火墙却并不知道（该连接已经在他们的内存中被新的连接淘汰掉了）。当发出数据后，代理将不能正确处理我们的数据，最终导致连接断开。

## Linux 下相关的内核参数

**tcp_keepalive_time**

表示TCP链接在多少秒之后没有数据报文传输时启动探测报文（发送空的报文，**单位秒**。

```
sysctl -n net.ipv4.tcp_keepalive_time
1200
```

**tcp_keepalive_intvl**

表示前一个探测报文和后一个探测报文之间的时间间隔，**单位秒**。

```
sysctl -n net.ipv4.tcp_keepalive_intvl
75
```

**tcp_keepalive_probes**

表示探测的次数。

```
sysctl -n net.ipv4.tcp_keepalive_probes
9
```

上述3个参数，结合起来解释有如下的关系：
当网络两端建立了 TCP 连接之后，闲置 idle （双方没有任何数据流发送往来）了 tcp_keepalive_time 秒 后，服务器内核就会尝试向客户端发送侦测包，来判断 TCP 连接状况(有可能客户端崩溃、强制关闭了应用、主机不可达等等)。如果没有收到对方的 ack ，则会在 tcp_keepalive_intvl 后再次尝试发送侦测包，直到收到对对方的 ack ，如果一直没有收到对方的 ack ，一共会尝试 tcp_keepalive_probes 次，每次的间隔时间在这里分别是15s, 30s, 45s, 60s, 75s。如果尝试 tcp_keepalive_probes ，依然没有收到对方的 ack 包，则会丢弃该 TCP 连接。

此外，Linux socket api 当中也提供了对应的参数，分别和上述的参数一一对应，调用 setsockopt 函数指定下面的不同的宏：

```
#include <sys/socket.h>

int setsockopt(int socket, int level, int option_name,
      const void *option_value, socklen_t option_len);
```

```
TCP_KEEPCNT: 在当前 socket 中覆盖系统内核的 tcp_keepalive_probes 值

TCP_KEEPIDLE: 在当前 socket 中覆盖系统内核的 tcp_keepalive_time 值

TCP_KEEPINTVL: 在当前 socket 中覆盖系统的 tcp_keepalive_intvl 值
```

## Nginx 当中的 TCP keepalive 配置

Nginx 涉及到 TCP 层面的 keepalive 只有一个： so_keepalive 。它属于 listen 指令的配置参数，具体配置如下：

```
so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]

this parameter (1.1.11) configures the “TCP keepalive” behavior for the listening socket. If this parameter is omitted then the operating system’s settings will be in effect for the socket. If it is set to the value “on”, the SO_KEEPALIVE option is turned on for the socket. If it is set to the value “off”, the SO_KEEPALIVE option is turned off for the socket. Some operating systems support setting of TCP keepalive parameters on a per-socket basis using the TCP_KEEPIDLE, TCP_KEEPINTVL, and TCP_KEEPCNT socket options. On such systems (currently, Linux 2.4+, NetBSD 5+, and FreeBSD 9.0-STABLE), they can be configured using the keepidle, keepintvl, and keepcnt parameters. One or two parameters may be omitted, in which case the system default setting for the corresponding socket option will be in effect. For example,
```

下面是一个配置实例：

```
so_keepalive=30m::10

will set the idle timeout (TCP_KEEPIDLE) to 30 minutes, leave the probe interval (TCP_KEEPINTVL) at its system default, and set the probes count (TCP_KEEPCNT) to 10 probes.
```

> Nginx 的实现代码，可以参考： http://hg.nginx.org/nginx/file/tip


# HTTP Keep-Alive
---

## 概述

HTTP keep-alive 在我们平时的工作中可能知道的相对要多一些，但是到底它和 TCP 的 keepalive 有什么区别了？让我们来一起学习一下。

## 短连接 & 长连接 & 并行连接

* 短连接：每次请求一个资源就建立连接，请求完成后连接立马关闭。每次请求都经过“创建tcp连接->请求资源->响应资源->释放连接”这样的过程。

* 长连接（ persistent connection ）：只建立一次连接，多次资源请求都复用该连接，完成后关闭。例如，有一个 http 请求一个页面上的十张图，只需要建立一次tcp连接，然后依次请求十张图，等待资源响应，释放连接。

* 并行连接（ multiple connections ）：并发的短连接。

下面的图比较了并行连接和长连接的区别：

![http-connection-comparation](https://www.zhuxiaodong.net/static/images/http-connection-comparation.png)

## Keep-Alive

HTTP 协议通过如下的规范来将短连接转变为长连接：

* client 在 Request http header 中增加 `Connection: Keep-Alive` 。 **在 HTTP/1.0 协议中，需要显式的在 request http header 中增加 Connection: Keep-Alive ，而在 HTTP/2.0 中则默认就是开启的。**

* server 如果能够识别 `Connection: Keep-Alive` 字段，就会 response 的 http header 中返回 `Connection: Keep-Alive` ，告诉客户端，服务器端能支持 keep-alive 服务，并且服务器端暂时不会关闭 socket 连接。

* 如果需要关闭连接时，会在 HTTP header 中指定 `Connection: Close` 。

## Nginx 相关的配置

* keepalive_timeout

```
Syntax: keepalive_timeout timeout [header_timeout];
Default:    keepalive_timeout 75s;
Context:    http, server, location

The first parameter sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections. The optional second parameter sets a value in the “Keep-Alive: timeout=time” response header field. Two parameters may differ.
```

* keepalive_requests

```
Syntax: keepalive_requests number;
Default:    keepalive_requests 100;
Context:    http, server, location

Sets the maximum number of requests that can be served through one keep-alive connection. After the maximum number of requests are made, the connection is closed.
```

## HTTP Keep-Alive 启用后带来的问题

启用 Keep-Alive ，可以避开缓慢的三次握手，还可以避免遇上 TCP 慢启动的拥塞适应阶段，能够提升性能。但是却会引入额外的问题。

让我们使用 node.js 编写一个 demo 来进行一下验证。

```
require('net').createServer(function(sock) {
    sock.on('data', function(data) {
        sock.write('HTTP/1.1 200 OK\r\n');
        sock.write('Connection: keep-alive\r\n');
        sock.write('\r\n');
        sock.write('hello world!');
        sock.destroy();
    });
}).listen(9090, '127.0.0.1');
```

我们使用 node.js 的 net 包去构建了一个最简单的 http 服务器，此时我们通过浏览器访问 `http://localhost:9090` ，发现能够得到正确的输出。另外我们也可以验证出，HTTP/1.1 协议中，默认的 http request 请求是带有 `Connection: keep-alive` http header 的。

![http-connection-01](https://www.zhuxiaodong.net/static/images/http-connection-01.png)

当我们去掉上面的代码中 `sock.destory();` 时，以模拟一个 HTTP 的持久连接，这时我们会发现，浏览器一直处于 Pending 状态，无法正确的返回结果。

这是因为，对于非持久连接（短连接），浏览器可以通过连接是否关闭来界定请求或响应实体的边界；而对于持久连接，这种方法显然不奏效。因为尽管服务器端已经发送完所有的数据，但浏览器并不知道这一点，它无法得知这个打开的连接上是否还会有新数据进来，只能一直处于等待状态。

### Content-Length

要解决上述的问题，我们需要有一种协商机制，用于服务器端告知客户端，已经完成了数据的传输。比如我们可以通过返回 http body 的长度来让客户端判断，是否数据已经传输完成了。

HTTP 协议中的 Response Header ： `Content-Length` 用于标识 Body 的实际长度。

让我们来改造之前的 node.js 代码：

```
require('net').createServer(function(sock) {
    sock.on('data', function(data) {
        sock.write('HTTP/1.1 200 OK\r\n');
        sock.write('Connection: keep-alive\r\n');
        sock.write('Content-Length: 12\r\n');
        sock.write('\r\n');
        sock.write('hello world!');
    });
}).listen(9090, '127.0.0.1');
```

我们增加了 `Content-Length` 的 response header ，并将其值赋值为 "hello world!" 的字符串长度（12），此时浏览器是能够正常 work 的，因为浏览器可以通过 Content-Length 的长度信息，判断出响应实体已结束。

![http-connection-02](https://www.zhuxiaodong.net/static/images/http-connection-02.png)

那么如果我们将长度计算错误了会出现什么问题了？让我们来验证一下。

#### Content-Length 比实际的长度小

```
require('net').createServer(function(sock) {
    sock.on('data', function(data) {
        sock.write('HTTP/1.1 200 OK\r\n');
        sock.write('Connection: keep-alive\r\n');
        sock.write('Content-Length: 10\r\n');
        sock.write('\r\n');
        sock.write('hello world!');
    });
}).listen(9090, '127.0.0.1');
```

![http-connection-03](https://www.zhuxiaodong.net/static/images/http-connection-03.png)

可以看到将 `Content-Length` 修改为 10 之后，response body 的内容被截断了2个长度的内容，即："hello worl" 。

#### Content-Length 比实际的长度大

```
require('net').createServer(function(sock) {
    sock.on('data', function(data) {
        sock.write('HTTP/1.1 200 OK\r\n');
        sock.write('Connection: keep-alive\r\n');
        sock.write('Content-Length: 13\r\n');
        sock.write('\r\n');
        sock.write('hello world!');
    });
}).listen(9090, '127.0.0.1');
```

`Content-Length` 修改为 13 之后，会发现浏览器又一直处于 Pending 状态，原因是客户端认为 Body 的长度小于 Content-Length ，内容还没有发送完，因此傻傻再等着服务器端发送剩余的一个字节内容。

#### Content-Length 的其它问题

由于 Content-Length 字段必须真实反映 HTTP Body 的实际长度，但某些场景下，实际长度无法进行计算，例如 HTTP Body 在服务器端动态的生成。这时候要想准确获取长度，只能在内核中开启一个足够大的 buffer ，等内容全部生成好再计算。但这样做一方面需要更大的内存开销，另一方面也会让客户端等更久。

我们在做 WEB 性能优化时，有一个重要的指标叫 TTFB（Time To First Byte），它代表的是从客户端发出请求到收到响应的第一个字节所花费的时间。比如 chrome 浏览器的 Network 面板都可以看到每一个 HTTP 请求的 TTFB，越短的 TTFB 意味着用户可以越早看到页面内容，体验越好。

![chrome-ttfb](https://www.zhuxiaodong.net/static/images/chrome-ttfb.png)

服务端如果为了计算响应实体长度而缓存所有内容，就会与更短的 TTFB 时间背道而驰。此外，HTTP Body 一定要在 Header 之后，顺序不能颠倒，为此我们需要一个新的机制：不依赖头部的长度信息，也能知道 HTTP Body 的边界。

### Transfer-Encoding: chunked

为了解决上述 Content-Length 的相关问题，HTTP 协议定义了一个新的 HTTP Header ： `Transfer-Encoding` ，其中 `chunked` 表示：分块编码。

分块编码的规则是，报文中的实体需要改为用一系列分块来传输。每个分块包含十六进制的长度值和数据，长度值独占一行，长度不包括它结尾的 CRLF（\r\n），也不包括分块数据结尾的 CRLF 。最后一个分块长度值必须为 0，对应的分块数据没有内容，表示实体结束。

按照这个规则，我们改造了之前的代码：

```
require('net').createServer(function(sock) {
    sock.on('data', function(data) {
        sock.write('HTTP/1.1 200 OK\r\n');
        sock.write('Transfer-Encoding: chunked\r\n');
        sock.write('\r\n');

        // b 是十六进制，十进制为11，恰好为"01234567890"的长度。
        sock.write('b\r\n');
        sock.write('01234567890\r\n');

        // 同上的规则
        sock.write('5\r\n');
        sock.write('12345\r\n');

        // 最后一个 0 长度的分块，表示数据已经被传输完
        sock.write('0\r\n');
        sock.write('\r\n');
    });
}).listen(9090, '127.0.0.1');
```

最后我们可以看到，浏览器能够正常的显示结果。

![http-connection-chunked](https://www.zhuxiaodong.net/static/images/http-connection-chunked.png)
