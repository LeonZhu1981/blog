title: How can I keepalive my SSH sessions and avoid freezing(hang)?
date: 2016-03-26 22:41:24
categories: programming
tags: 
- linux
- shell
- ssh
---

## What and Why?
---
我们经常在ssh连接到VPS上之后, 由于某种原因长时间没有进行任何SSH操作之后, 你会发现当前的ssh终端hang在那里, 没有任何反应, 提示如下:
```
packet_write_wait: Connection to xx.xx.xx.xx: Broken pipe
```
我猜这个应该是由于ssh协议的某种保护机制, 当client长时间没有发送任何请求时, 强行断开连接, 以避免连接数过多的时候占用更多服务器端资源. 这个保护机制对于VPS来说(只有个人使用, 一般只会开几个ssh终端进行操作), 显然用户体验不是太好.

## 修改SSH Clinet配置
---
我们可以设置每间隔多少秒, 让ssh client发出[NULL packet](https://tools.ietf.org/html/rfc6592)数据包给服务器端, 以保持连接会话.

```
sudo vi /etc/ssh/ssh_config #全局

or

sudo vi ~/.ssh/config #本用户
```

添加以下的内容:
>Host *
>ServerAliveInterval 100

如果只是临时使用, 不想修改配置文件, 可以使用以下方式:
```
ssh -o ServerAliveInterval=100 user@example.com
```

## 修改SSH Server配置
---

```
sudo vi /etc/ssh/sshd_config
```

添加以下的内容:
>ClientAliveInterval 60
>TCPKeepAlive yes
>ClientAliveCountMax 100

**ClientAliveInterval:** 服务器端每间隔60秒发出**NULL packet**至客户端, 以保持连接会话.
**TCPKeepAlive:** 防止被防火墙drop掉idle连接.
**ClientAliveCountMax:** 允许最大的客户端连接数量.

设置好了之后, 需要restart sshd service.
```
sudo systemctl restart sshd
```

## Ref
---
[https://docs.oseems.com/general/application/ssh/disable-timeout](https://docs.oseems.com/general/application/ssh/disable-timeout)
[http://unix.stackexchange.com/questions/200239/how-can-i-keep-my-ssh-sessions-from-freezing](http://unix.stackexchange.com/questions/200239/how-can-i-keep-my-ssh-sessions-from-freezing)