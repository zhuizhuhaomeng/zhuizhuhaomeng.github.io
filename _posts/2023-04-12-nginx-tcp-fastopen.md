---
layout: post
title: "给 nginx 开启 TCP Fast Open 的支持"
description: "给 nginx 开启 TCP Fast Open 的支持"
date: 2023-04-05
tags: [TCP]
---

# 什么是 TCP Fast Open

TCP 握手需要一个完整的 RTT（往返时间）。RTT 是指一个数据包从发送方到达接收方再返回所需的时间。
对于 "短暂的 "和 "时间敏感的 "流量来说，一个 RTT 是一个很大的时间，例如网络流量；在浏览器上浏览网页，如访问一个网站。如果传播延迟非常高（例如，地面站和卫星之间的链接）或移动网络非常慢，则性能会进一步下降。一个 RTT 不会（显著）降低 "弹性流量"（如文件传输）的性能，因为相对于整个连接的时间，一个 RTT 的开销相当小。

很多的 CDN 测试指标中有首包到达时间，TCP Fast Open 对于优化首包时间非常有意义。
TFO 是 TCP Fast Open 的缩写。它是一种传输层解决方案，用于避免客户和服务器之间的一个完整 RTT。它避免了重复连接的 TCP-3 方式握手。TFO 是由谷歌的一个团队提出的，并在 RFC 7413 中描述。

在一个正常的 TCP 连接中，一个 RTT 在连接中被浪费了，然后从第三个数据包开始，真正的通信开始了。TFO 说，客户端可以在第一个数据包中发送 GET 请求而不浪费一个 RTT。但这样做有一些条件：

1. 它必须不是一个 "新 "连接

TFO 只对重复连接起作用，因为它需要一个加密的 cookie。当客户端第一次与服务器互动时，它必须执行 3 种方式的握手。在该握手过程中，服务器与客户共享一个加密的 cookie。在接下来的连接中，客户端只需在 SYN 数据包中插入 GET 请求和 cookie。然后，服务器使用 cookie 来验证客户端，接受连接请求，并在 ACK 包中发送数据。

2. 与 SYN 包一起捎带的数据总量必须在规定的限制之内。

指定的限制是什么？对于 IPv4 来说，总共有 1460 个字节可以被捎带进数据包。因此，一个网段的大小成为 1460B。

3. 只有某些类型的 HTTP 请求可以使用 TFO 发送

TFO 支持 GET 请求，不支持 POST 请求，因为 SYN 报文携带应用层数据，如果重传会导致重复的 POST 操作，而 POST 操作一般是有副作用的，会改变数据的状态。
比如：SYN+DATA 由于某种原因被中间的路由器缓存了，但是后续的重传完成的整个 TCP 的握手和应用的数据处理过程并最终关闭 TCP 连接。这个时候被缓存的 SYN+DATA 的数据包再次被发出来，这个时候就会被认为是一个权限的 TCP 连接。因此这种情况下就导致同一个数据被两次提交应用层处理。比如两次扣款，这种对于用户是无法忍受的。

如果是同一个 TCP 连接过程数据包被缓存，在该 TCP 连接未关闭之前又收到该缓存的数据包，则该数据报文会被内核丢弃，因此不会有影响。

TFO 的工作原理：

TCP 客户端和服务器都必须支持 TFO，以便使用它。现在问题来了，客户端和服务器如何让对方知道他们支持 TFO？答案就在 TCP 头中。

TCP 头中的 "选项 "字段被用于 TFO。

TCP 客户端用它来请求一个 TFO Cookie
TCP 服务器用它来发送一个 TFO Cookie
在选项字段中，Kind 的值为 34，表示 cookie 类型。Cookie 的长度为 16 字节。当客户端向服务器发送 SYN 数据包时，它在选项字段中设置 Kind = 34。当服务器看到 Kind 值等于 34 时，它就明白客户端支持 TFO 并要求获得 cookie。服务器生成一个独特的 cookie，并使用客户端的 IP 地址对其进行加密，这样每个客户端都有一个独特的 cookie。在选项文件中，服务器将 cookie 发送给客户端。

因此，使用选项字段，客户和服务器让对方知道他们支持 TFO 并共享 cookie。

TFO 是否支持有条件的 GET 请求？

它应该支持有条件的 GET。GET 请求基本上是只读操作。因此，它们应该被 TFO 所允许。不应该从时间延迟的角度来看待这个问题。有条件的 GET 请求基本上是在本地服务器上的数据过期而无法提供给客户端时从本地服务器发送到中央服务器。在这种情况下，新鲜的数据会从中央服务器获取并缓存到本地服务器，以服务本地客户。

# nginx 配置文件

下面是一个 nginx 支持 TCP Fast Open 的测试代码

```nginx
    server {
        listen 80 reuseport fastopen=100;
        location / {
            content_by_lua_block {
                ngx.say("Hello world")
            }
        }
    }
```

# Linux 系统配置

要临时生效，可以使用下面的命令

```shell
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
# or
sysctl -w net.ipv4.tcp_fastopen=3
```

如果需要持久化，在系统重启后还可以生效，那么配置如下：

```shell
echo "net.ipv4.tcp_fastopen=3" > /etc/sysctl.d/30-tcp_fastopen.conf
sysctl --system
```

# Linux TC 增加延迟仿真公网

因为我们在同一台机器上测试该功能，报文走的是 lo 接口，延迟非常的低。
为了更容易看出效果，我们给 lo 接口增加 10ms 的延迟。

```shell
tc qdisc add dev lo root netem delay 10ms
```

如果上面的命令出现错误，可能是没有安装对应的内核包，可以执行下面的命令
```shell
yum install -y kernel-modules-extra
```

# curl 测试 TCP-Fast-Open

我们先不开启 TCP-Fast-Open 功能进行测试，总的请求时间是 40ms。

```shell
curl -s -o /dev/null -w "Connect: %{time_connect} TTFB: %{time_starttransfer} Total time: %{time_total} \n" http://127.0.0.1
Connect: 0.020433 TTFB: 0.040811 Total time: 0.040942
```

开启 TCP-Fast-Open 进行测试后，可以看到第一次请求时间是 40ms，后续的请求时间是 20ms。

```shell
curl -s -o /dev/null -w "Connect: %{time_connect} TTFB: %{time_starttransfer} Total time: %{time_total} \n" --tcp-fastopen http://127.0.0.1
Connect: 0.020464 TTFB: 0.040959 Total time: 0.041145
curl -s -o /dev/null -w "Connect: %{time_connect} TTFB: %{time_starttransfer} Total time: %{time_total} \n" --tcp-fastopen http://127.0.0.1  
Connect: 0.000189 TTFB: 0.020779 Total time: 0.020923 
```

# tcpdump 观察效果

查看第一条 TCP 连接，可以 SYN 报文看到有 tfo cookiereq 的选项，SYN ACK 有 tfo cookie 的应答选项。

查看第二个 TCP 连接，可以看到客户端在发送 SYN 报文的同时把 HTTP 请求捎带上了。

```shell
$ tcpdump -i lo tcp port 80 -ttt -nnn
 00:00:22.346478 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [S], seq 1089705660, win 43690, options [mss 65495,sackOK,TS val 3213968563 ecr 0,nop,wscale 7,tfo  cookiereq,nop,nop], length 0
 00:00:00.010092 IP 127.0.0.1.80 > 127.0.0.1.53132: Flags [S.], seq 2562728912, ack 1089705661, win 43690, options [mss 65495,sackOK,TS val 877001526 ecr 3213968563,nop,wscale 7,tfo  cookie 6720256534125c83,nop,nop], length 0
 00:00:00.010086 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [.], ack 1, win 342, options [nop,nop,TS val 3213968583 ecr 877001526], length 0
 00:00:00.000076 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [P.], seq 1:74, ack 1, win 342, options [nop,nop,TS val 3213968583 ecr 877001526], length 73: HTTP: GET / HTTP/1.1
 00:00:00.010059 IP 127.0.0.1.80 > 127.0.0.1.53132: Flags [.], ack 74, win 342, options [nop,nop,TS val 877001546 ecr 3213968583], length 0
 00:00:00.000304 IP 127.0.0.1.80 > 127.0.0.1.53132: Flags [P.], seq 1:246, ack 74, win 342, options [nop,nop,TS val 877001546 ecr 3213968583], length 245: HTTP: HTTP/1.1 200 OK
 00:00:00.000130 IP 127.0.0.1.80 > 127.0.0.1.53132: Flags [P.], seq 246:1343, ack 74, win 342, options [nop,nop,TS val 877001546 ecr 3213968583], length 1097: HTTP
 00:00:00.009927 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [.], ack 246, win 350, options [nop,nop,TS val 3213968603 ecr 877001546], length 0
 00:00:00.000119 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [.], ack 1343, win 367, options [nop,nop,TS val 3213968603 ecr 877001546], length 0
 00:00:00.000228 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [F.], seq 74, ack 1343, win 367, options [nop,nop,TS val 3213968604 ecr 877001546], length 0
 00:00:00.010196 IP 127.0.0.1.80 > 127.0.0.1.53132: Flags [F.], seq 1343, ack 75, win 342, options [nop,nop,TS val 877001567 ecr 3213968604], length 0
 00:00:00.010147 IP 127.0.0.1.53132 > 127.0.0.1.80: Flags [.], ack 1344, win 367, options [nop,nop,TS val 3213968624 ecr 877001567], length 0

 00:00:02.313928 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [S], seq 63609622:63609695, win 43690, options [mss 65495,sackOK,TS val 3213970938 ecr 0,nop,wscale 7,tfo  cookie 6720256534125c83,nop,nop], length 73: HTTP: GET / HTTP/1.1
 00:00:00.010126 IP 127.0.0.1.80 > 127.0.0.1.53148: Flags [S.], seq 621921974, ack 63609696, win 43690, options [mss 65495,sackOK,TS val 877003901 ecr 3213970938,nop,wscale 7], length 0
 00:00:00.000233 IP 127.0.0.1.80 > 127.0.0.1.53148: Flags [P.], seq 1:246, ack 1, win 342, options [nop,nop,TS val 877003901 ecr 3213970938], length 245: HTTP: HTTP/1.1 200 OK
 00:00:00.000092 IP 127.0.0.1.80 > 127.0.0.1.53148: Flags [P.], seq 246:1343, ack 1, win 342, options [nop,nop,TS val 877003901 ecr 3213970938], length 1097: HTTP
 00:00:00.009752 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [.], ack 1, win 342, options [nop,nop,TS val 3213970958 ecr 877003901], length 0
 00:00:00.000211 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [.], ack 246, win 350, options [nop,nop,TS val 3213970958 ecr 877003901], length 0
 00:00:00.000083 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [.], ack 1343, win 367, options [nop,nop,TS val 3213970958 ecr 877003901], length 0
 00:00:00.000245 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [F.], seq 1, ack 1343, win 367, options [nop,nop,TS val 3213970959 ecr 877003901], length 0
 00:00:00.010169 IP 127.0.0.1.80 > 127.0.0.1.53148: Flags [F.], seq 1343, ack 2, win 342, options [nop,nop,TS val 877003922 ecr 3213970959], length 0
 00:00:00.010108 IP 127.0.0.1.53148 > 127.0.0.1.80: Flags [.], ack 1344, win 367, options [nop,nop,TS val 3213970979 ecr 877003922], length 0
```

# 应用层代码实现

## 服务端的实现

```C
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/tcp.h>
#include <unistd.h>


int main(void)
{
    int rc;
    int ls_fd, client_sock, client_size;
    struct sockaddr_in server_addr, client_addr;
    char server_message[2000], client_message[2000];
    
    ls_fd = socket(AF_INET, SOCK_STREAM, 0);
    if(ls_fd < 0){
        perror("Error while creating socket\n");
        return -1;
    }

    int qlen = 128;
    setsockopt(ls_fd, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));

    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(2000);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    
    // Bind to the set port and IP:
    rc = bind(ls_fd, (struct sockaddr*) &server_addr, sizeof(server_addr));
    if(rc < 0){
        perror("Couldn't bind to the port\n");
        return -1;
    }
    
    // Listen for clients:
    if(listen(ls_fd, 1) < 0) {
        perror("Error while listening\n");
        return -1;
    }

    printf("\nListening for incoming connections.....\n");
    
    // Accept an incoming connection:
    client_size = sizeof(client_addr);
    client_sock = accept(ls_fd, (struct sockaddr*)&client_addr, &client_size);
    
    if (client_sock < 0){
        printf("Can't accept\n");
        return -1;
    }
    printf("Client connected at IP: %s and port: %i\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
    
    // Receive client's message:
    rc = recv(client_sock, client_message, sizeof(client_message), 0);
    if (rc < 0){
        printf("Couldn't receive\n");
        return -1;
    }

    printf("Msg from client: %.*s\n", rc, client_message);
    
    // Respond to client:
    strcpy(server_message, "This is the server's message.");
    
    if (send(client_sock, server_message, strlen(server_message), 0) < 0){
        printf("Can't send\n");
        return -1;
    }
    
    // Closing the socket:
    close(client_sock);
    close(ls_fd);
    
    return 0;
}
```

## 客户端代码

```shell
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <netinet/tcp.h>

int main(void)
{
    int fd;
    int rc;
    struct sockaddr_in server_addr;
    char server_message[2000], client_message[2000];
    
    fd = socket(AF_INET, SOCK_STREAM, 0);
    
    if (fd < 0){
        printf("Unable to create socket\n");
        return -1;
    }
    
    printf("Socket created successfully\n");
    
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(2000);
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    
    strcpy(client_message, "Hello world");

    rc = sendto(fd, client_message, strlen(client_message), MSG_FASTOPEN, 
		(struct sockaddr *) &server_addr, sizeof(server_addr));
    if (rc < 0) {
        perror("sendto(MSG_FASTOPEN) failed");
        exit(1);
    }

    rc = recv(fd, server_message, sizeof(server_message), 0);
    if (rc < 0){
        printf("Error while receiving server's msg\n");
        return -1;
    }
    
    printf("Server's response: %.*s\n", rc, server_message);
    
    close(fd);
    
    return 0;
}
```

# 参考链接

1. https://lwn.net/Articles/508865/
1. https://www.geeksforgeeks.org/what-is-tcp-fast-open/
