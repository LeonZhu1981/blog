title: TCP CLOSE_WAIT 详解
date: 2018-02-28 16:41:48
categories: programming
tags:
- network
- tcp/ip
- tcp protocol
- CLOSE_WAIT
---

# 什么是 CLOSE_WAIT

[主动关闭]的一方发出 FIN 包，[被动关闭]的一方响应 ACK 包，此时，[被动关闭]的一方就进入了 CLOSE_WAIT 状态。如果一切正常，稍后[被动关闭]的一方也会发出 FIN 包，然后迁移到 LAST_ACK 状态。

![tcp-close](https://www.zhuxiaodong.net/static/images/tcp_close.png)

<!--more-->

# CLOSE_WAIT 过多会导致出现的问题和原因

一般的情况下， CLOSE_WAIT 状态应该会停留的时间非常短，如果我们发现了某台服务器上出现了大量的 CLOSE_WAIT 状态的连接， 并且持续的时间比较长，那么一定是一种非正常的情况。

我们可以通过如下的几种可能去进行分析程序：

* 应用程序本身的问题：如果代码层面忘记了或者由于异常情况没有正常执行 socket 连接，那么自然不会发出 FIN 包，导致 CLOSE_WAIT 积累。
* 响应太慢或者超时时间设置的过小：例如客户端（ 这里我们假设客户端是一个 nginx 反向代理 ）和服务器（ tomcat web 服务器 ）端建立了连接之后，客户端发起了一次 rpc 调用，该 rpc 调用在服务器端又会再请求另外一个很慢的第三方 api 请求，由于第三方 api 性能比较差，很长时间都无法返回请求结果，超过了客户端设置的 timeout 时间，此时客户端就直接关闭了连接，而服务器端由于还在等待第三方 api 返回结果，就一直停留在 CLOSE_WAIT 状态。要解决上述问题有三个方案：1. 增加客户端的超时时间。 2. 在业务允许的情况下，将第三方 api 调整为异步调用。 3. 让第三方 api 程序进行优化。

如果你通过「netstat -ant」或者「ss -ant」命令发现了很多 CLOSE_WAIT 连接，请注意结果中的「Recv-Q」和「Local Address」字段，通常「Recv-Q」会不为空，它表示应用还没来得及接收数据，而「Local Address」表示哪个地址和端口有问题，我们可以通过「lsof -i:<PORT>」来确认端口对应运行的是什么程序以及它的进程号是多少。

此外还有一点需要说明：按照前面图例所示，当被动关闭的一方处于 CLOSE_WAIT 状态时，主动关闭的一方处于 FIN_WAIT2 状态。 那么为什么我们总听说 CLOSE_WAIT 状态过多的故障，但是却相对少听说 FIN_WAIT2 状态过多的故障呢？这是因为 Linux 有一个「tcp_fin_timeout」设置，控制了 FIN_WAIT2 的最大生命周期。坏消息是 CLOSE_WAIT 没有类似的设置，如果不重启进程，那么 CLOSE_WAIT 状态很可能会永远持续下去；好消息是如果 socket 开启了 keepalive 机制，那么可以通过相应的设置来清理无效连接，不过 keepalive 是治标不治本的方法，还是应该找到问题的症结才对。
