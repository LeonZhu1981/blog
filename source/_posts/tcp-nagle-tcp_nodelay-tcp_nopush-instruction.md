title: TCP Nagle&CORK 算法详解
date: 2018-03-04 15:32:16
categories: programming
tags:
- network
- tcp/ip
- tcp protocol
- nagle
- tcp_nodelay
- tcp_nopush
- tcp_cork
- sendfile
---

# 什么是 Nagle 算法
---

Nagle 算法由 John Nagle 在1984年提出，这个算法可以减少网络中小的 packet 的数量，从而降低网络的拥塞程度。一个常见的例子就是 Telnet 程序，用户在控制台的每次击键都会发送一个 packet ，这个 packet 通常包含41个字节（ IP 头部 20 字节 + TCP 头部 20 字节 + 1 个字节数据 ），然而只有一个字节是 payload ，其余 40 个字节都是 header ，如果每次击键都发送一个 packet ，那就会造成了巨大的开销。

<!--more-->

为了减小这种开销， Nagle 算法指出，当 TCP 发送了一个小的 segment (小于 MSS )，它必须等到接收了对方的 ACK 之后，才能继续发送另一个小的 segment 。那么在等待的过程中(一个 RTT 时间)， TCP 就能尽量多地将要发送的数据收集在一起，从而减少要发送的 segment 的数量。

Nagle算法的基本定义是任意时刻，最多只能有一个未被确认的小段。 所谓“小段”，指的是小于MSS尺寸的数据块，所谓“未被确认”，是指一个数据块发送出去后，没有收到对方发送的ACK确认该数据已收到。

其定义可以参考 linux 内核源码的[这里](https://elixir.bootlin.com/linux/v3.4.113/source/net/ipv4/tcp_output.c#L1393)

```
/* Return 0, if packet can be sent now without violation Nagle's rules:
 * 1. It is full sized.
 * 2. Or it contains FIN. (already checked by caller)
 * 3. Or TCP_CORK is not set, and TCP_NODELAY is set.
 * 4. Or TCP_CORK is not set, and all sent packets are ACKed.
 *    With Minshall's modification: all sent small packets are ACKed.
 */
static inline int tcp_nagle_check(const struct tcp_sock *tp,
				  const struct sk_buff *skb,
				  unsigned mss_now, int nonagle)
{
	return skb->len < mss_now &&
		((nonagle & TCP_NAGLE_CORK) ||
		 (!nonagle && tp->packets_out && tcp_minshall_check(tp)));
}
```


# Nagle 算法的开启会带来什么问题
---

Nagle 算法是在上个世纪 80 年代提出的，当时的网络带宽非常有限，而目前带宽和机器都大大提升，由小包造成网络拥堵的可能性非常低。因此，我们可以通过如下的方式来设置关闭 Nagle 算法，以避免 200 ms 的延迟：

设置 socket 的 TCP_NODELAY 选项

```
int opt_val = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, (void *)&opt_val, sizeof(opt_val));
```

可以通过 TCP_NODELAY 的值查看 Nagle 算法的设置状态：

```
int opt_val;
socklen_t opt_len;
opt_len = sizeof(opt_val);
getsockopt(sock, IPPROTO_TCP, TCP_NODELAY, (void*)&opt_val, &opt_len);
```

如果已禁用 Nagle 算法，opt_val 变量的值为1。


# Nginx 当中设置 TCP_NODELAY
---

Nginx 当中可以通过设置 nginx.conf 来禁用 Nagle 算法的开启。

```
http {
    tcp_nodelay        on;
}
```

# 什么是 CORK 算法
---

CORK 算法的设计目的与 Nagle 算法非常类似，也有人把 Cork 算法称呼为 super-Nagle 。 所谓的 CORK 就是塞子的意思，形象地理解就是用 CORK 将连接塞住，使得数据先不发出去，等到拔去塞子后再发出去。设置该选项后，内核会尽力把小数据包拼接成一个大的数据包（一个 MSS ）再发送出去，当然若一定时间后（一般为 200 ms，该值尚待确认），内核仍然没有组合成一个 MTU 时也必须发送现有的数据。

那么什么时候会打开这个塞子了？让 TCP 能够继续发送这个 segment ：

* 程序取消设置 TCP_CORK 选项。
* 内核 TCP 发送缓冲中的数据大于一个 MSS 的大小。
* 当塞子堵住第一个字节时，超过了 200 ms。
* socket 被关闭了。

CORK 与 Nagle 的区别是，Nagle 算法是尽量避免大量的小包发送，而 CORK 算法是期望完全避免发送小包（无论是大量还是少量的小包）。

可以通过设置 TCP_CORK 选项来开启 CORK 算法：

```
int state = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &state, sizeof(state));
```

# Nginx 当中设置开启 CORK 算法
---

Nginx 当中可以通过设置 nginx.conf 来启用 CORK 算法的开启。

```
http {
    tcp_nopush        on;
}
```

# sendfile 系统调用
---

linux 下 I/O 读写基本会使用到 read 和 write 系统调用，其工作流程如下:

```
read(file,tmp_buf, len);

write(socket,tmp_buf, len);

硬盘 >> kernel buffer >> user buffer >> kernel socket buffer >> 协议栈
```

上述过程大概产生了 4 次上下文切换。

sendfile(2) 系统调用允许在传输数据时直接使用内核空间，从而提升传输效率：

* 由于 sendfile(2) 是系统调用，直接在内核空间中执行，避免了上下文切换带来的开销。
* sendfile(2) 能够将读和写操作合并成一个。
* sendfile(2) 由于直接使用 DMA 机制直接将磁盘当中的数据映射到了内核空间中，避免了用户态和内核态之间的数据拷贝的过程（ zero copy 零拷贝 ）。

其工作过程大致如下：

```
硬盘 >> kernel buffer (快速拷贝到kernelsocket buffer) >>协议栈
```

因此 sendfile(2) 调用非常适合网络文件传输或下载的高性能场景，例如 http 静态资源服务。

# Nginx 设置
---

在 nginx 下，如果想要打开 CORK 算法（即设置 tcp_nopush 开启），必须将 sendfile 设置为开启。 此外，这里看似 tcp_nopush 和 tcp_nodelay 都开启的情况下，两者显得会有矛盾（ tcp_nopush 是期望数据包积累到一定程度之后才进行发送，tcp_nodelay 是期望数据尽快地发送 ），但实际上它们是可以一起使用的，最终的效果是先填满包，再尽快地发送。

```
http {
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
}
```

参看内核的[代码](https://elixir.bootlin.com/linux/latest/source/net/ipv4/tcp.c#L2697)里的注释：

```
case TCP_CORK:
		/* When set indicates to always queue non-full frames.
		 * Later the user clears this option and we transmit
		 * any pending partial frames in the queue.  This is
		 * meant to be used alongside sendfile() to get properly
		 * filled frames when the user (for example) must write
		 * out headers with a write() call first and then use
		 * sendfile to send out the data parts.
		 *
		 * TCP_CORK can be set together with TCP_NODELAY and it is
		 * stronger than TCP_NODELAY.
		 */
		if (val) {
			tp->nonagle |= TCP_NAGLE_CORK;
		} else {
			tp->nonagle &= ~TCP_NAGLE_CORK;
			if (tp->nonagle&TCP_NAGLE_OFF)
				tp->nonagle |= TCP_NAGLE_PUSH;
			tcp_push_pending_frames(sk);
		}
		break;
```

# 参考资料
---
https://thoughts.t37.net/nginx-optimization-understanding-sendfile-tcp-nodelay-and-tcp-nopush-c55cdd276765
http://senlinzhan.github.io/2017/02/10/Linux%E7%9A%84TCP-CORK/
