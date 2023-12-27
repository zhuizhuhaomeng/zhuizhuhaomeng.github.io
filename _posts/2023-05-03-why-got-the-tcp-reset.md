---
layout: post
title: "Why got the TCP reset pakcet"
description: "Why got the TCP reset Packet"
date: 2023-05-03
tags: [TCP]
---

我们在排查网络问题的时候有时候会发现由于收到 TCP reset 报文导致连接中断或者连接不上的问题。
那么到底有哪些情况会导致收到 TCP reset 报文呢？下面列举一些常见的例子，如果有更多的情况，欢迎补充。

[TOC]

# 目的端口没有打开

目的端口没有打开这种情况是比较常见的。
比如：
1. 机器重启了，但是服务没有设置开启启动。
1. 程序重启了，端口被关闭又打开的瞬间接收到的 TCP 连接

这种情况很容易的验证，比如 `telnet 127.0.0.1 4444`。

```shell
$ telnet 192.168.0.203 5000
Trying 192.168.0.203...
telnet: connect to address 192.168.0.203: Connection refused
```

通过 tcpdump 可以看到发送了 reset 报文，并且 reset 报文的 seq 为 0。

```shell
sudo tcpdump -nnn -i eth0 tcp port 5000 && host 192.168.0.203
[sudo] password for ljl: 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:48:07.432563 IP 192.168.0.181.51796 > 192.168.0.203.5000: Flags [S], seq 3663045003, win 64240, options [mss 1460,sackOK,TS val 3514322324 ecr 0,nop,wscale 7], length 0
23:48:07.432851 IP 192.168.0.203.5000 > 192.168.0.181.51796: Flags [R.], seq 0, ack 3663045004, win 0, length 0
```

如果想观察 tcp 报文，可以在另一个窗口执行 `tcpdump -i lo tcp port 4444`。
我们可以看到，这种情况下，Reset 报文的 seq 是 0。

# 防火墙发送发送 reset 报文

比如在机器 1 上执行下面的命令。

```shell
$ sudo iptables -I INPUT -p tcp -s 192.168.0.181 -j REJECT --reject-with tcp-reset
```

在机器 2 上执行如下的 telnet 命令，可以看到 telnet 被 reset 了。

```shell
$ telnet 192.168.0.203 80
Trying 192.168.0.203...
telnet: connect to address 192.168.0.203: Connection refused
```

一般来说，不会单独添加这样的 iptables 语句，而是类似下面这样的语句。
这样配置的问题是，如果 TCP 的状态是 INVALID，那么就会发送 Reset 报文出去。
比如一条 TCP 连接已经关闭，但是一些报文由于被中间路由器缓存等原因导致在连接关闭后收到，那么这个时候的流表状态就是 INVALID。

https://stackoverflow.com/questions/251243/what-causes-a-tcp-ip-reset-rst-flag-to-be-sent

```shell
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT 
iptables -A FORWARD -p tcp -j REJECT --reject-with tcp-reset 
```

因此，这种情况应该增加一个 Drop 的语句。

```shell
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT 
iptables -A FORWARD -m state --state INVALID -j DROP 
iptables -A FORWARD -p tcp -j REJECT --reject-with tcp-reset 
```

上面的是防火墙部署在服务端机器上的一个例子。
而防火墙可能位于任何位置，防火墙会向服务端也会向客户端发送 reset 报文。
最著名的这类防火墙属于中国的国家防火墙。防火墙发送 reset 报文的规则也很复杂。
可能根据流量大小触发，也可能根据连接的目的 IP 触发，可能是根据前面流量检测构造的动态黑名单。


# socket linger timeout 设置为 0

在关闭一个套接字之前，SO_LINGER 选项被设置为超时值为 0。当套接字被关闭时，TCP RST 被发送到客户端，并且这个套接字占用的所有内存被释放。这有助于避免让一个已经关闭的套接字的缓冲区长时间处于 FIN_WAIT1 状态。

Nginx 服务器通过配置 `reset_timedout_connection on` 来达到上述功能。可以通过下面的配置来快速模拟。

```nginx
reset_timedout_connection on;
client_header_timeout 5s;
```

我们在另一个终端通过执行 telnet 命令并输入 GET 语句来模拟发送不完整请求头的情况。

```shell
$ telnet 127.0.0.1 80
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
GET / HTTP/1.1
Connection closed by foreign host.
```

在另一个终端上，我们通过 tcpdump 可以看到发送了 reset 报文。这次 seq 不再是 0，而是 1。这里的 1 是 tcpdump 计算的相对序列号。

```shell
08:37:08.186174 IP 127.0.0.1.48672 > 127.0.0.1.80: Flags [S], seq 3753585494, win 43690, options [mss 65495,sackOK,TS val 3691146521 ecr 0,nop,wscale 7], length 0
08:37:08.186229 IP 127.0.0.1.80 > 127.0.0.1.48672: Flags [S.], seq 4147526100, ack 3753585495, win 43690, options [mss 65495,sackOK,TS val 3691146521 ecr 3691146521,nop,wscale 7], length 0
08:37:08.186276 IP 127.0.0.1.48672 > 127.0.0.1.80: Flags [.], ack 1, win 342, options [nop,nop,TS val 3691146521 ecr 3691146521], length 0
08:37:09.414037 IP 127.0.0.1.48672 > 127.0.0.1.80: Flags [P.], seq 1:17, ack 1, win 342, options [nop,nop,TS val 3691147749 ecr 3691146521], length 16: HTTP: GET / HTTP/1.1
08:37:09.414085 IP 127.0.0.1.80 > 127.0.0.1.48672: Flags [.], ack 17, win 342, options [nop,nop,TS val 3691147749 ecr 3691147749], length 0
08:37:13.190400 IP 127.0.0.1.80 > 127.0.0.1.48672: Flags [R.], seq 1, ack 17, win 342, options [nop,nop,TS val 3691151525 ecr 3691147749], length 0
```

# SYN 匹配了已经存在的 5 元组

这个情况比较难以通过简单的命令行模拟。
具体的场景是已经有一个 TCP 流已经在收发数据，这个时候又受到另一个建立链接的 SYN 的请求。

# 全连接队列满了

如果全连接队列满了，并且 设置了 tcp_abort_on_overflow，那么服务端就会发送 reset 报文。

```shell
sysctl -w net.ipv4.tcp_abort_on_overflow=1
```

我通过下面的代码片段启动一个只监听不接收连接的服务端。
但是创建了多个连接后，服务端还是不会发送 reset。

经过实验，在 CentOS7 的 3.10 内核上是可以复现，但是在 5.10 的内核上无法复现。
应该是新版本的内核已经把该功能废除了，但是仍然保留了这个设置项。

```python
import socket
import time

def server_program():
    # get the hostname
    #host = socket.gethostname()
    host = "0.0.0.0"
    port = 5000  # initiate port no above 1024

    server_socket = socket.socket()  # get instance
    # look closely. The bind() function takes tuple as argument
    server_socket.bind((host, port))  # bind host address and port together

    # configure how many client the server can listen simultaneously
    server_socket.listen(2)
    time.sleep(1000)

if __name__ == '__main__':
    server_program()
```

# 半开放连接

如果 TCP 连接的一端打开，而另一端在其他端不知情的情况下关闭了它，则称为半打开的连接。
这有很多原因：一种可能是一方崩溃，另一种可能是机器突然断电。连接将保持半开状态，直到没有数据传输。

因此，假设您有 Client1 和 Server1，并且有一个 TCP 连接处于已建立状态。
此时，Client1 没有请求任何数据，处于空闲状态。突然，Server1 崩溃，或者在物理层出现问题，导致网络接口崩溃。
这将冲掉携带连接信息的 Server1 传输控制块 (TCB)。现在，Client1 不知道这一点，并向 Server1 发送一些请求。
Serve1 收到此请求，但是由于没有先前的 TCB 信息，如旧序列和确认号码，Serve1 将拒绝此请求并向客户端发送 Reset。
这导致客户端发起一个新的连接。

# 服务器重启时关闭未连接队列中的连接时发送 reset

当服务器关闭监听的 socket 的时候，还未被应用程序处理的全连接/半连接队列里的连接会被直接关闭，并向客户端发送 reset 报文。

我们以这个代码作为服务端的代码，在一台机器上执行  python3 ./server.py。

```python
import socket
import time

def server_program():
    # get the hostname
    #host = socket.gethostname()
    host = "0.0.0.0"
    port = 5000  # initiate port no above 1024

    server_socket = socket.socket()  # get instance
    # look closely. The bind() function takes tuple as argument
    server_socket.bind((host, port))  # bind host address and port together

    # configure how many client the server can listen simultaneously
    server_socket.listen(2)
    time.sleep(1000)

if __name__ == '__main__':
    server_program()
```

在另一台机器上的终端上执行 tcpdump。

```shell
$ sudo tcpdump -nnn -i eth0 host 192.168.0.203
23:06:26.339684 IP 192.168.0.181.36052 > 192.168.0.203.5000: Flags [S], seq 2223270099, win 64240, options [mss 1460,sackOK,TS val 3511821231 ecr 0,nop,wscale 7], length 0
23:06:26.340033 IP 192.168.0.203.5000 > 192.168.0.181.36052: Flags [S.], seq 2331257957, ack 2223270100, win 28960, options [mss 1460,sackOK,TS val 1130625489 ecr 3511821231,nop,wscale 7], length 0
23:06:26.340095 IP 192.168.0.181.36052 > 192.168.0.203.5000: Flags [.], ack 1, win 502, options [nop,nop,TS val 3511821232 ecr 1130625489], length 0
23:06:27.650889 IP 192.168.0.181.36062 > 192.168.0.203.5000: Flags [S], seq 3794930265, win 64240, options [mss 1460,sackOK,TS val 3511822542 ecr 0,nop,wscale 7], length 0
23:06:27.651217 IP 192.168.0.203.5000 > 192.168.0.181.36062: Flags [S.], seq 1863976093, ack 3794930266, win 28960, options [mss 1460,sackOK,TS val 1130626800 ecr 3511822542,nop,wscale 7], length 0
23:06:27.651276 IP 192.168.0.181.36062 > 192.168.0.203.5000: Flags [.], ack 1, win 502, options [nop,nop,TS val 3511822543 ecr 1130626800], length 0
23:06:28.634666 IP 192.168.0.181.42022 > 192.168.0.203.5000: Flags [S], seq 2489131190, win 64240, options [mss 1460,sackOK,TS val 3511823526 ecr 0,nop,wscale 7], length 0
23:06:28.634959 IP 192.168.0.203.5000 > 192.168.0.181.42022: Flags [S.], seq 2010628400, ack 2489131191, win 28960, options [mss 1460,sackOK,TS val 1130627784 ecr 3511823526,nop,wscale 7], length 0
23:06:28.635011 IP 192.168.0.181.42022 > 192.168.0.203.5000: Flags [.], ack 1, win 502, options [nop,nop,TS val 3511823527 ecr 1130627784], length 0
23:06:29.475319 IP 192.168.0.181.42024 > 192.168.0.203.5000: Flags [S], seq 2478944339, win 64240, options [mss 1460,sackOK,TS val 3511824367 ecr 0,nop,wscale 7], length 0
23:06:30.485809 IP 192.168.0.181.42034 > 192.168.0.203.5000: Flags [S], seq 1505942970, win 64240, options [mss 1460,sackOK,TS val 3511825377 ecr 0,nop,wscale 7], length 0
23:06:30.529739 IP 192.168.0.181.42024 > 192.168.0.203.5000: Flags [S], seq 2478944339, win 64240, options [mss 1460,sackOK,TS val 3511825421 ecr 0,nop,wscale 7], length 0
23:06:31.489797 IP 192.168.0.181.42034 > 192.168.0.203.5000: Flags [S], seq 1505942970, win 64240, options [mss 1460,sackOK,TS val 3511826381 ecr 0,nop,wscale 7], length 0
23:06:32.577800 IP 192.168.0.181.42024 > 192.168.0.203.5000: Flags [S], seq 2478944339, win 64240, options [mss 1460,sackOK,TS val 3511827469 ecr 0,nop,wscale 7], length 0
23:06:33.537793 IP 192.168.0.181.42034 > 192.168.0.203.5000: Flags [S], seq 1505942970, win 64240, options [mss 1460,sackOK,TS val 3511828429 ecr 0,nop,wscale 7], length 0
23:06:33.954768 IP 192.168.0.203.5000 > 192.168.0.181.36052: Flags [R.], seq 1, ack 1, win 227, options [nop,nop,TS val 1130633104 ecr 3511821232], length 0
23:06:33.954785 IP 192.168.0.203.5000 > 192.168.0.181.36062: Flags [R.], seq 1, ack 1, win 227, options [nop,nop,TS val 1130633104 ecr 3511822543], length 0
23:06:33.954803 IP 192.168.0.203.5000 > 192.168.0.181.42022: Flags [R.], seq 1, ack 1, win 227, options [nop,nop,TS val 1130633104 ecr 3511823527], length 0
23:06:36.609803 IP 192.168.0.181.42024 > 192.168.0.203.5000: Flags [S], seq 2478944339, win 64240, options [mss 1460,sackOK,TS val 3511831501 ecr 0,nop,wscale 7], length 0
23:06:36.610134 IP 192.168.0.203.5000 > 192.168.0.181.42024: Flags [R.], seq 0, ack 2478944340, win 0, length 0
```

在执行 tcpdump 的机器上执行 telnet 程序

```shell
[ljl@centos7 ~]$ telnet 192.168.0.203 5000 &
[7] 6061
[ljl@centos7 ~]$ Trying 192.168.0.203...
Connected to 192.168.0.203.
Escape character is '^]'.
telnet 192.168.0.203 5000 &
[8] 6064

[7]+  Stopped                 telnet 192.168.0.203 5000
[ljl@centos7 ~]$ Trying 192.168.0.203...
Connected to 192.168.0.203.
Escape character is '^]'.
telnet 192.168.0.203 5000 &
[9] 6066

[8]+  Stopped                 telnet 192.168.0.203 5000
[ljl@centos7 ~]$ Trying 192.168.0.203...
Connected to 192.168.0.203.
Escape character is '^]'.
telnet 192.168.0.203 5000 &
[10] 6067

[9]+  Stopped                 telnet 192.168.0.203 5000
[ljl@centos7 ~]$ Trying 192.168.0.203...
telnet 192.168.0.203 5000 &
[11] 6069
[ljl@centos7 ~]$ Trying 192.168.0.203...

[ljl@centos7 ~]$ telnet: connect to address 192.168.0.203: Connection refused
telnet: connect to address 192.168.0.203: Connection refused
```

# Time-wait 暗杀 (Time-Wait Assassination)

让我们来理解时间等待暗杀对 TCP reset 是怎么回事。

1. 客户端发送 SEQ = 1000 ACK = 5000 的 TCP FIN，转到 FIN- wait1。
1. 现在服务器为这个 FIN 发送 ACK, SEQ = 5000, ACK = 1001。
1. 下一个服务器发送他的 FIN 与 SEQ = 5000 ACK = 1001
1. 客户端收到 FIN，发送 SEQ = 1001 ACK = 5001 的 ACK，进入 Time-Wait 状态。
1. 服务器接收到这个 FIN 并进入 CLOSED 状态。
1. 稍后，在这个 time - wait 状态的某个时间，客户端接收到一些带有旧 SEQ 和 ACK 号的延迟到达的段。
1. 像往常一样，客户端发送 ACK 给这个带有当前序列的后期段，并确认 SEQ = 1001 ACK = 5001。
1. 当服务器接收到这个 ACK 时，它的内存中没有关于这个连接的信息，因为它已经在步骤 5 中关闭了这个连接。这导致服务器向客户端发送 TCP RESET。

# 如果 socket 中还存在未处理的数据

如果 socket 已经被 accept，但是里面的数据并没有被应用层处理，那么关闭 socket 的时候就会向对端发送一个 reset 报文。

执行下面的代码，在另一个终端用 telnet 连接后输入一些数据，然后关闭服务端程序。

```python
import socket
import time

def server_program():
    # get the hostname
    #host = socket.gethostname()
    host = "0.0.0.0"
    port = 5000  # initiate port no above 1024

    server_socket = socket.socket()  # get instance
    # look closely. The bind() function takes tuple as argument
    server_socket.bind((host, port))  # bind host address and port together

    # configure how many client the server can listen simultaneously
    server_socket.listen(2)
    new_socket = server_socket.accept()
    time.sleep(1000)

if __name__ == '__main__':
    server_program()
```

通过 tcpdump 我们可以看到，服务端向客户端发送了一个 reset 报文

```shell
[ljl@centos7 ~]$ sudo tcpdump -nnn -i eth0 tcp port 5000 && host 192.168.0.203
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:26:41.194281 IP 192.168.0.181.52412 > 192.168.0.203.5000: Flags [S], seq 1142462528, win 64240, options [mss 1460,sackOK,TS val 3513036086 ecr 0,nop,wscale 7], length 0
23:26:41.194622 IP 192.168.0.203.5000 > 192.168.0.181.52412: Flags [S.], seq 2190404931, ack 1142462529, win 28960, options [mss 1460,sackOK,TS val 1131840346 ecr 3513036086,nop,wscale 7], length 0
23:26:41.194696 IP 192.168.0.181.52412 > 192.168.0.203.5000: Flags [.], ack 1, win 502, options [nop,nop,TS val 3513036086 ecr 1131840346], length 0
23:26:43.286533 IP 192.168.0.181.52412 > 192.168.0.203.5000: Flags [P.], seq 1:5, ack 1, win 502, options [nop,nop,TS val 3513038178 ecr 1131840346], length 4
23:26:43.286842 IP 192.168.0.203.5000 > 192.168.0.181.52412: Flags [.], ack 5, win 227, options [nop,nop,TS val 1131842438 ecr 3513038178], length 0
23:26:46.314911 IP 192.168.0.203.5000 > 192.168.0.181.52412: Flags [R.], seq 1, ack 5, win 227, options [nop,nop,TS val 1131845466 ecr 3513038178], length 0
^C
```

# NAT 流表项被删除的情况

如果 TCP 连接涉及 NAT，那么 reset 报文也可能是 NAT 模块发送的。
比如 NAT conntrack 的过期时间为 5 分钟，在超过 5 分钟没有报文的情况下，conntrack flow 被删除。
然后，这时候 NAT 收到了已经删除了的 TCP 连接的报文，这时候就会发送 reset 报文出去。


# 参考文档

https://iponwire.com/tcp-reset-rst-reasons/
https://stackoverflow.com/questions/251243/what-causes-a-tcp-ip-reset-rst-flag-to-be-sent
