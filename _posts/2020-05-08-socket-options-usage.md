---
layout: post
title: "Socket 的选项详解"
description: "Socket 的选项多如牛毛，我们应该怎么使用这些选项呢？"
date: 2022-01-05
feature_image: img/road.jpg
tags: [glibc, performance]
---

# 这些socket选项都有什么用

[TOC]

# SOL_SOCKET

## SOL_SOCKET.SO_SNDLOWAT

### 作用

man 7 socket

```txt
       SO_RCVLOWAT and SO_SNDLOWAT
              Specify the minimum number of bytes in the buffer until the
              socket layer will pass the data to the protocol (SO_SNDLOWAT)
              or the user on receiving (SO_RCVLOWAT).  These two values are
              initialized to 1.  SO_SNDLOWAT is not changeable on Linux
              (setsockopt(2) fails with the error ENOPROTOOPT).  SO_RCVLOWAT
              is changeable only since Linux 2.4.	
```



### 应用场景

### 代码举例

``` c
ngx_int_t
set_send_lowat(int fd, size_t lowat)
{
    int  sndlowat;

    sndlowat = (int) lowat;

    if (setsockopt(fd, SOL_SOCKET, SO_SNDLOWAT,
                   (const void *) &sndlowat, sizeof(int))
        == -1)
    {
        perror("setsockopt(SO_SNDLOWAT) failed");
        return -1;
    }

    return 0;
}
```

## SOL_SOCKET.SO_TYPE

### 作用

获取socket的类型

### 应用场景

比如nginx升级程序，通过`NGINX`环境变量将当前监听的fd传递给新的进程，新的进程起来后需要还原fd的类型。

### 代码举例
```c
 #include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

/*
 * build: gcc SOL_SOCKET-SO_TYPE.c
 */

static void
get_socket_type(int fd)
{
    int       type;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    if (getsockopt(fd, SOL_SOCKET, SO_TYPE, (void *)&type, &olen) == -1) {
        printf("failed to get socket type for fd %d, error: %s\n",
               fd, strerror(errno));
    } else {
        switch (type) {
        case SOCK_STREAM:
            typename = "SOCK_STREAM";
            break;

        case SOCK_DGRAM:
            typename = "SOCK_DGRAM";
            break;

        case SOCK_RAW:
            typename = "SOCK_RAW";
            break;

        default:
            snprintf(buf, sizeof(buf), "%d", type);
            break;
        }

        printf("fd %d has type %s\n", fd, typename);
    }
}


int
main(int argc, char **argv)
{
    int       socket_fd;

    socket_fd = socket(AF_INET , SOCK_STREAM , 0);

    get_socket_type(socket_fd);
    get_socket_type(2);

    return 0;
}
```

## SOL_SOCKET.SO_RCVBUF

### 作用
获取/设置socket接收缓冲区的大小
### 应用场景

1. 查询当前的socket缓冲区大小，确认大小配置正确。
2. nginx二进制升级，从旧进程继承的socket需要还原socket缓冲区的大小。
3. 改变socket缓冲区的大小，让socket可以接收更多的数据。

### 代码举例

``` c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

/*
 * build: gcc SOL_SOCKET-SO_RCVBUF.c
 */

static void
get_socket_rcvbuf(int fd)
{
    int       bufsz;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    if (getsockopt(fd, SOL_SOCKET, SO_RCVBUF, (void *)&bufsz, &olen) == -1) {
        printf("failed to get buffer size for fd %d, error: %s\n",
               fd, strerror(errno));
    } else {
        printf("fd %d has default recv buffer size %d\n", fd, bufsz);
    }
}


int
main(int argc, char **argv)
{
    int       socket_fd;

    socket_fd = socket(AF_INET , SOCK_STREAM , 0);

    get_socket_rcvbuf(socket_fd);

    return 0;
}
```

### 执行结果

```shell
$ ./a.out 
fd 3 has default buffer size 87380
```

## SOL_SOCKET.SO_SNDBUF

### 作用

获取/设置socket接收缓冲区的大小

### 应用场景

1. 查询当前的socket缓冲区大小，确认大小配置正确。
2. nginx二进制升级，从旧进程继承的socket需要还原socket缓冲区的大小。
3. 改变socket缓冲区的大小，让socket可以发送更多的数据。

### 代码举例

``` c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

/*
 * build: gcc SOL_SOCKET-SO_SNDBUF.c
 */

static void
get_socket_rcvbuf(int fd)
{
    int       bufsz;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    if (getsockopt(fd, SOL_SOCKET, SO_SNDBUF, (void *)&bufsz, &olen) == -1) {
        printf("failed to get buffer size for fd %d, error: %s\n",
               fd, strerror(errno));
    } else {
        printf("fd %d has default send buffer size %d\n", fd, bufsz);
    }
}


int
main(int argc, char **argv)
{
    int       socket_fd;

    socket_fd = socket(AF_INET , SOCK_STREAM , 0);

    get_socket_rcvbuf(socket_fd);

    return 0;
}
```

### 执行结果

```shell
$ ./a.out 
fd 3 has default send buffer size 16384
```

可见Linux的默认接收缓冲区和发送缓冲区的大小是不同的

## SOL_SOCKET.SO_REUSEPORT

### 作用

让同一个主机上的多个socket可以绑定到同一个端口，用于改进多核系统上的多线程/多进程服务器的性能。

see https://lwn.net/Articles/542629/

### 应用场景

nginx服务器可以通过开启reuse port选项，让linux内核在内核层面把新建连接分配给不同进程监听的fd。如果不开启reuse port，那么多个进程其实监听的是同一个fd句柄，会出现一个新建连接唤醒所有进程来accept的所谓惊群现象。

### 代码举例

``` c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdlib.h>


/*
 * build: gcc SOL_SOCKET-SO_REUSEPORT.c
 */

static void
get_socket_reuseport(int fd)
{
    int       on;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    if (getsockopt(fd, SOL_SOCKET, SO_REUSEPORT, (void *)&on, &olen) == -1) {
        printf("failed to get socket type for fd %d, error: %s\n",
               fd, strerror(errno));
    } else {
        printf("fd %d has reuseport %s\n", fd, on ? "on" : "off");
    }
}


static void
set_socket_reuseport(int fd)
{
    int       on;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    on = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, (void *)&on, olen) == -1) {
        printf("failed to set reuseport for fd %d, error: %s\n",
               fd, strerror(errno));
    }
}


int
main(int argc, char **argv)
{
    int       socket_fd;
    int       socket_fd2;
    char      cmd[128];
    struct sockaddr_in name;
    socket_fd = socket(AF_INET , SOCK_STREAM , 0);

    printf("before set reuseport\n");
    get_socket_reuseport(socket_fd);

    name.sin_family = AF_INET;
    name.sin_port = htons (8888);
    name.sin_addr.s_addr = htonl (INADDR_ANY);


    set_socket_reuseport(socket_fd);
    printf("after set reuseport\n");
    get_socket_reuseport(socket_fd);

    if (bind(socket_fd, (struct sockaddr *) &name, sizeof(name)) != 0) {
        printf("fd %d failed to bind to port 8888, error:%s\n", 
               socket_fd, strerror(errno));
    } else {
        printf("fd %d bind to 8888 successful\n", socket_fd);
    }
    if (listen(socket_fd, 10) != 0) {
        printf("fd %d failed to listen\n", socket_fd);
    }

    socket_fd2 = socket(AF_INET , SOCK_STREAM , 0);
    set_socket_reuseport(socket_fd2);
    if (bind(socket_fd2, (struct sockaddr *) &name, sizeof(name)) != 0) {
        printf("fd %d failed to bind to port 8888, error:%s\n", 
               socket_fd2, strerror(errno));
    } else {
        printf("fd %d bind to 8888 successful\n", socket_fd2);
    }

    if (listen(socket_fd2, 10) != 0) {
        printf("fd %d failed to listen\n", socket_fd2);
    }

    snprintf(cmd, sizeof(cmd), "sudo lsof -n -p %d  | grep TCP", getpid());
    system(cmd);

    return 0;
}
```



### 执行结果

``` shell
before set reuseport
fd 3 has reuseport off
after set reuseport
fd 3 has reuseport on
fd 3 bind to 8888 successful
fd 4 bind to 8888 successful
a.out   696249  ljl    3u  sock    0,9      0t0   3169549 protocol: TCP
a.out   696249  ljl    4u  sock    0,9      0t0   3169550 protocol: TCP

```



## SOL_SOCKET.SO_REUSEPORT_LB

### 作用

这个接口是FreeBSD 12+系统上才有，linux上没有该接口。功能和SO_REUSEPORT是一致的。

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_REUSEADDR

### 作用

### 应用场景

### 代码举例

``` C
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdlib.h>


/*
 * build: gcc SOL_SOCKET-SO_REUSEADDR.c
 */


static void
set_socket_reuseaddr(int fd)
{
    int       on;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    on = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (void *)&on, olen) == -1) {
        printf("failed to set reuseaddr for fd %d, error: %s\n",
               fd, strerror(errno));
    }
}


int
main(int argc, char **argv)
{
    int       socket_fd;
    int       socket_fd2;
    char      cmd[128];
    struct sockaddr_in name;


    name.sin_family = AF_INET;
    name.sin_port = htons (8888);
    name.sin_addr.s_addr = htonl (INADDR_ANY);

    socket_fd = socket(AF_INET , SOCK_STREAM , 0);
    set_socket_reuseaddr(socket_fd);

    if (bind(socket_fd, (struct sockaddr *) &name, sizeof(name)) != 0) {
        printf("fd %d failed to bind to port 8888, error:%s\n", 
               socket_fd, strerror(errno));
    } else {
        printf("fd %d bind to 8888 successful\n", socket_fd);
    }

    if (listen(socket_fd, 10) != 0) {
        printf("fd %d failed to listen\n", socket_fd);
    }

    socket_fd2 = socket(AF_INET , SOCK_STREAM , 0);
    set_socket_reuseaddr(socket_fd2);
    if (bind(socket_fd2, (struct sockaddr *) &name, sizeof(name)) != 0) {
        printf("fd %d failed to bind to port 8888, error:%s\n", 
               socket_fd2, strerror(errno));
    } else {
        printf("fd %d bind to 8888 successful\n", socket_fd2);
    }
    
    if (listen(socket_fd2, 10) != 0) {
        printf("fd %d failed to listen\n", socket_fd2);
    }

    snprintf(cmd, sizeof(cmd), "sudo netstat -tpln | grep 8888", getpid());
    system(cmd);

    return 0;
}
```



### 执行结果

```
fd 3 bind to 8888 successful
fd 4 failed to bind to port 8888, error:Address already in use
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN      697459/./a.out  
```

可见在设置了SOCK_REUSEADDR后，同一个进程也不能够同时让两个socket监听在同一个地址端口对上。



## SOL_SOCKET.SO_ACCEPTFILTER

### 作用

FreeBSD系统的选项。

SO_ACCEPTFILTER在套接字上放置一个accept_filter，它将在监听流套接字上过滤传入的连接，通过过滤后再触发accept。 必须在套接字上调用listen才尝试在其上安装过滤器，否则setockopt系统调用将失败。

**未使用该系统验证，不做详细描述。**

### 应用场景

### 代码举例

``` C
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>

/*
 * build: gcc SOL_SOCKET-SO_TYPE.c
 */

static void
get_socket_acceptfilter(int fd)
{
    int type;
    socklen_t olen;
    char buf[32];
    const char *typename;
    struct accept_filter_arg af;

    memset(&af, 0, sizeof(struct accept_filter_arg));
    olen = sizeof(struct accept_filter_arg);

    if (fd, SOL_SOCKET, SO_ACCEPTFILTER, &af, &olen) == -1) {
        printf("fd %d failed to get SO_ACCEPTFILTER, error %s\n",
               fd, strerror(errno));
    }

    if (olen < sizeof(struct accept_filter_arg) || af.af_name[0] == '\0') {
        printf("fd %d unknown SO_ACCEPTFILTER\n", fd);;
    }
}

int main(int argc, char **argv)
{
    int socket_fd;

    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    get_socket_acceptfilter(socket_fd);

    return 0;
}
```



### 执行结果



## SOL_SOCKET.SO_KEEPALIVE

### 作用

如果没有开启keepalive，那么在长期的空闲后，网络上路由器/防火墙等设备会丢弃该TCP的流表项。当再次发送数据时，会直接收到reset或者数据直接丢弃（进入黑洞）。

### 应用场景

TCP长连接的场景下，建议开启keepalive。

### 代码举例

```C

/* --- begin of keepalive test program --- */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void);

int main()
{
   int s;
   int optval;
   socklen_t optlen = sizeof(optval);

   /* Create the socket */
   if((s = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0) {
      perror("socket()");
      exit(EXIT_FAILURE);
   }

   /* Check the status for the keepalive option */
   if(getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE is %s\n", (optval ? "ON" : "OFF"));

   /* Set the option active */
   optval = 1;
   optlen = sizeof(optval);
   if(setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, optlen) < 0) {
      perror("setsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE set on socket\n");

   /* Check the status again */
   if(getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE is %s\n", (optval ? "ON" : "OFF"));

   close(s);

   exit(EXIT_SUCCESS);
}

```

### 执行结果

``` shell
SO_KEEPALIVE is OFF
SO_KEEPALIVE set on socket
SO_KEEPALIVE is ON
```



## SOL_SOCKET.SO_SETFIB

BSD相关的接口，不做描述

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_BINDANY

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果

## SOL_SOCKET.SO_

### 作用

### 应用场景

### 代码举例

### 执行结果



# IPPROTO_IP

## IPPROTO_IP.IP_BIND_ADDRESS_NO_PORT

## IPPROTO_IP.IP_BINDANY

## IPPROTO_IP.IP_TRANSPARENT

## IPPROTO_IP.IP_RECVDSTADDR

### 作用

服务器上存在多个IP地址，UDP应用监听在0.0.0.0:1234 这种只有端口没有IP的套接字上。在收到一个UDP报文时，应用程序不知道报文是发给服务器的哪个IP地址的。因此需要通过设置该选项获取UDP报文的目的IP地址。

### 应用场景

透明代理

### 代码举例

### 执行结果

## IPPROTO_IP.IP_PKTINFO

### 作用

服务器上存在多个IP地址，UDP应用监听在0.0.0.0:1234 这种只有端口没有IP的套接字上。在收到一个UDP报文时，应用程序不知道报文是发给服务器的哪个IP地址的。因此需要通过设置该选项获取UDP报文的目的IP地址。

### 应用场景

场景1：

客户端发送给服务器的IP1，那么应答客户端时需要以IP1作为应答的IP地址。

场景2：

在透明代理中使用该选项，获取真正的目的IP地址。正常情况下，网关上目的IP非网关的数据会在走三层转发直接转发出去。但是如果希望代理一些数据（比如代理外国的IP流量), 那么可以配置透明代理，将外国流量的UDP流转发给本地的应用，本地应用再通过代理等方式将流量转发给原始的服务器。因为使用getsockname获取的是本地应用监听的IP和端口，无法知道原始的服务器的IP和端口。所以本地应用需要知道原始服务器的IP和端口，就需要使用IP_RECVDSTADDR这个选项。

### 代码举例

### 执行结果

## IPPROTO_IP.IP_TRANSPARENT

### 作用

透明代理获取原始发目的IP地址。

比如客户端192.168.1.100 访问1.2.3.4，网关192.168.1.1上做透明代理，将访问1.2.0.0/16的数据转发给网关上的192.168.1.1:1234端口。网关上的应用在收到数据后需要知道原始的目的IP地址才能够做进一步的处理。

### 应用场景

在透明代理中使用该选项，获取真正的目的IP地址。正常情况下，网关上目的IP非网关的数据会在走三层转发直接转发出去。但是如果希望代理一些数据（比如代理外国的IP流量), 那么可以配置透明代理，将外国流量的UDP流转发给本地的应用，本地应用再通过代理等方式将流量转发给原始的服务器。因为使用getsockname获取的是本地应用监听的IP和端口，无法知道原始的服务器的IP和端口。所以本地应用需要知道原始服务器的IP和端口，就需要使用IP_RECVDSTADDR这个选项。

### 代码举例

### 执行结果

# IPPROTO_IPV6

## IPPROTOV6_IP.IPV6_BINDANY

## IPPROTOV6_IP.IPV6_TRANSPARENT

## IPPROTO_IPV6.IPV6_RECVPKTINFO

参考 [IPPROTO_IP.IP_PKTINFO](##IPPROTO_IP.IP_PKTINFO)



# IPPROTO_TCP

## IPPROTO_TCP.TCP_FASTOPEN

### 作用

tcp 建立连接需要三次握手，这样需要两个RTT（round trip time）的时间。该选项的目的就是为了减少TCP的握手时间。

它的工作原理是使用TFO cookie（TCP选项），TFO cookie是一种存储在客户端的加密cookie。在与服务器初始连接时由服务器发送cookies给客户端，当客户端以后重新连接时，会将初始SYN、数据、TFO cookie一起发送给服务器，其中TFO cookie用于验证自己的身份。如果成功的话，服务器甚至可以在收到三方握手的最后一个ACK包之前就开始向客户端发送数据，这样就跳过了一个往返延迟，降低了数据传输开始的延迟。

https://blog.csdn.net/for_tech/article/details/54237556

### 应用场景

在高延迟的网络中使用特别有优势，特别是跨国网络，但是中国的GFW可能会拦截这样的TCP连接。

### 代码举例

#### 服务端代码

```c

#include <unistd.h>
#include <string.h>
 
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

int main(int argc, char **argv)
{
  int serverSock;
  struct sockaddr_in addr;
 
  int clientSock;
  struct sockaddr_in clientAddr;
  int addrLen;
 
  char buf[1024];
  int read;
 
  serverSock = socket(AF_INET, SOCK_STREAM, 0);
 
  if(serverSock == -1)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 1;
  }
 
#ifdef DEFER_ACCEPT
  int soValue = 1;
  if(setsockopt(serverSock, IPPROTO_TCP, TCP_DEFER_ACCEPT, &soValue, sizeof(soValue))<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 10;
  }
#endif
 
  memset(&addr, 0, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(7890);
  addr.sin_addr.s_addr=inet_addr("127.0.0.1");
 
  if(bind(serverSock, (struct sockaddr*)&addr, sizeof(addr))<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 2;
  }
 
#ifdef FAST_OPEN
  int qlen=5;
  setsockopt(serverSock, SOL_TCP, TCP_FASTOPEN, &qlen, sizeof(qlen));
#endif
 
  if(listen(serverSock,511)<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 3;
  }
 
  while(1)
  {
    addrLen = sizeof(clientAddr);
    clientSock = accept(serverSock, (struct sockaddr*)&clientAddr, &addrLen);
    if(clientSock<0)
    {
      write(STDERR_FILENO, "failed!\n",8);
      return 4;
    }
 
    read = recv(clientSock, buf, 1024, 0);
    write(STDOUT_FILENO, buf, read);
    close(clientSock);
  }
 
  return 0;
}
```



#### 客户端代码

```c

#include <unistd.h>
#include <string.h>
 
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
 
#include <netinet/tcp.h>
#include <arpa/inet.h>
 
#ifndef MSG_FASTOPEN
#define MSG_FASTOPEN   0x20000000
#endif
 
int main(int argc, char **argv)
{
  int clientSock;
  struct sockaddr_in addr;
 
  clientSock = socket(AF_INET, SOCK_STREAM, 0);
 
  if(clientSock == -1)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 1;
  }
 
#ifdef DEFER_ACCEPT
  int soValue = 1;
  if(setsockopt(clientSock, IPPROTO_TCP, TCP_DEFER_ACCEPT, &soValue, sizeof(soValue))<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 10;
  }
#endif
 
  memset(&addr, 0, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(7890);
  addr.sin_addr.s_addr=inet_addr("127.0.0.1");
 
#ifdef FAST_OPEN
 
int ret = sendto(clientSock, "Hello\n", 6, MSG_FASTOPEN,
                   (struct sockaddr *)&addr, sizeof(addr));
 
if(ret<0)
{
    write(STDERR_FILENO, "failed!\n",8);
    return 11;
}
 
#else
  if(connect(clientSock, (struct sockaddr*)&addr, sizeof(addr))<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 2;
  }
 
  if(send(clientSock, "Hello\n", 6, 0)<0)
  {
    write(STDERR_FILENO, "failed!\n",8);
    return 3;
  }
#endif
 
  close(clientSock);
 
  return 0;
}
```

### 系统开启 TCP_FASTOPEN

0 值表明它被禁用了；位 0 对应客户端操作，而位 1 对应服务器操作。把`tcp_fastopen`设为 3 可将两者同时启用。

``` shell
cat /proc/sys/net/ipv4/tcp_fastopen
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```



### 执行结果

#### 执行脚本

```shell
gcc -DFAST_OPEN  IPPROTO_TCP.TCP_FASTOPEN.server.c -o server
gcc -DFAST_OPEN  IPPROTO_TCP.TCP_FASTOPEN.client.c -o client
sudo tcpdump -i lo -n tcp and host 127.0.0.1 and port 7890 &
./server &
sleep 1
./client
sleep 1
echo "=========="

./client
sleep 1
echo "=========="

killall -9 tcpdump
killall -9 client
killall -9 server
```



#### 脚本输出

``` shell
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
Hello
21:49:57.076100 IP 127.0.0.1.40326 > 127.0.0.1.7890: Flags [S], seq 2244958086, win 43690, options [mss 65495,sackOK,TS val 776392891 ecr 0,nop,wscale 7,tfo  cookiereq,nop,nop], length 0
21:49:57.076113 IP 127.0.0.1.7890 > 127.0.0.1.40326: Flags [S.], seq 3697330112, ack 2244958087, win 43690, options [mss 65495,sackOK,TS val 776392891 ecr 776392891,nop,wscale 7,tfo  cookie 80418d24d3dd8b4a,nop,nop], length 0
21:49:57.076123 IP 127.0.0.1.40326 > 127.0.0.1.7890: Flags [P.], seq 1:7, ack 1, win 342, options [nop,nop,TS val 776392891 ecr 776392891], length 6
21:49:57.076136 IP 127.0.0.1.7890 > 127.0.0.1.40326: Flags [.], ack 7, win 342, options [nop,nop,TS val 776392891 ecr 776392891], length 0
21:49:57.076146 IP 127.0.0.1.40326 > 127.0.0.1.7890: Flags [F.], seq 7, ack 1, win 342, options [nop,nop,TS val 776392892 ecr 776392891], length 0
21:49:57.076236 IP 127.0.0.1.7890 > 127.0.0.1.40326: Flags [F.], seq 1, ack 8, win 342, options [nop,nop,TS val 776392892 ecr 776392892], length 0
21:49:57.076247 IP 127.0.0.1.40326 > 127.0.0.1.7890: Flags [.], ack 2, win 342, options [nop,nop,TS val 776392892 ecr 776392892], length 0
==========
21:49:58.079086 IP 127.0.0.1.40328 > 127.0.0.1.7890: Flags [S], seq 3420618283:3420618289, win 43690, options [mss 65495,sackOK,TS val 776393894 ecr 0,nop,wscale 7,tfo  cookie 80418d24d3dd8b4a,nop,nop], length 6
Hello
21:49:58.079105 IP 127.0.0.1.7890 > 127.0.0.1.40328: Flags [S.], seq 3569490838, ack 3420618290, win 43690, options [mss 65495,sackOK,TS val 776393894 ecr 776393894,nop,wscale 7], length 0
21:49:58.079112 IP 127.0.0.1.40328 > 127.0.0.1.7890: Flags [.], ack 1, win 342, options [nop,nop,TS val 776393894 ecr 776393894], length 0
21:49:58.079122 IP 127.0.0.1.40328 > 127.0.0.1.7890: Flags [F.], seq 1, ack 1, win 342, options [nop,nop,TS val 776393894 ecr 776393894], length 0
21:49:58.079179 IP 127.0.0.1.7890 > 127.0.0.1.40328: Flags [.], ack 2, win 342, options [nop,nop,TS val 776393895 ecr 776393894], length 0
21:49:58.079204 IP 127.0.0.1.7890 > 127.0.0.1.40328: Flags [F.], seq 1, ack 2, win 342, options [nop,nop,TS val 776393895 ecr 776393894], length 0
21:49:58.079211 IP 127.0.0.1.40328 > 127.0.0.1.7890: Flags [.], ack 2, win 342, options [nop,nop,TS val 776393895 ecr 776393895], length 0
==========

```



可以看到，第一次TCP连接，第一个SYN报文不携带数据，第二次连接SYN报文旧携带了数据。



## IPPORTO_TCP.TCP_DEFER_ACCEPT

### 作用

设置TCP_DEFER_ACCEPT后，服务端将在收到客户端的数据后才会触发用户空间的服务器进行accept。

### 应用场景

优点：

1. 通过设置该选项，可以一定程度的抵挡TCP的攻击，降低CPU的利用率。
2. 没有被攻击的情况下，打开该选项也可以提升服务器的性能。

缺点:

服务器被攻击时，固定是指定时间才超时，应用层无法介入去及时关闭连接。



### 代码举例

```C
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h> /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>

/*
 * build: gcc IPPORTO_TCP.TCP_DEFER_ACCEPT
 */

static void
get_socket_deferaccept(int fd)
{
    socklen_t olen;
    int timeout;

    timeout = 0;
    olen = sizeof(int);

    if (getsockopt(fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &timeout, &olen) == -1)
    {
        printf("fd %d failed to get TCP_DEFER_ACCEPT, error %s\n", strerror(errno));
    }
    printf("timeout is %d\n", timeout);
}

static void
set_socket_deferaccept(int fd)
{
    socklen_t olen;
    int timeout;

    timeout = 5;
    olen = sizeof(int);

    if (setsockopt(fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &timeout, olen) == -1)
    {
        printf("fd %d failed to get TCP_DEFER_ACCEPT, error %s\n", strerror(errno));
    }
    printf("timeout is %d\n", timeout);
}

int main(int argc, char **argv)
{
    int socket_fd;

    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    get_socket_deferaccept(socket_fd);
    set_socket_deferaccept(socket_fd);
    get_socket_deferaccept(socket_fd);

    return 0;
}
```



### 执行结果



``` shell
timeout is 0
timeout is 5
timeout is 7
```

## IPPROTO_TCP.TCP_KEEPIDLE

### 作用

如果没有开启keepalive，那么在长期的空闲后，网络上路由器/防火墙等设备会丢弃该TCP的流表项。当再次发送数据时，会直接收到reset或者数据直接丢弃（进入黑洞）。

### 应用场景

TCP长连接的场景下，建议开启keepalive。

部分运营商的最长空闲时间设置是5分钟深圳更低，而linux系统默认是7200，因此最好需要调整该时间小于5分钟。

### 代码举例

```C

/* --- begin of keepalive test program --- */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

int main(void);

int main()
{
   int s;
   int optval;
   socklen_t optlen = sizeof(optval);

   /* Create the socket */
   if((s = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0) {
      perror("socket()");
      exit(EXIT_FAILURE);
   }

   /* Check the status for the keepalive option */
   if(getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE is %s\n", (optval ? "ON" : "OFF"));

   /* Set the option active */
   optval = 1;
   optlen = sizeof(optval);
   if(setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, optlen) < 0) {
      perror("setsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE set on socket\n");

   /* Check the status again */
   if(getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &optval, &optlen) < 0) {
      perror("getsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }
   printf("SO_KEEPALIVE is %s\n", (optval ? "ON" : "OFF"));


   optval = 300;
   if(setsockopt(s, IPPROTO_TCP, TCP_KEEPINTVL, &optval, optlen) < 0) {
      perror("setsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }

   optval = 5;
   if(setsockopt(s, IPPROTO_TCP, TCP_KEEPCNT, &optval, optlen) < 0) {
      perror("setsockopt()");
      close(s);
      exit(EXIT_FAILURE);
   }

   close(s);

   exit(EXIT_SUCCESS);
}
```

### 执行结果

### 配置接口

```
  # cat /proc/sys/net/ipv4/tcp_keepalive_time
  7200

  # cat /proc/sys/net/ipv4/tcp_keepalive_intvl
  75

  # cat /proc/sys/net/ipv4/tcp_keepalive_probes
  9
```



## IPPROTO_TCP.TCP_KEEPINTVL

参考  [TCP_KEEPIDLE](##IPPROTO_TCP.TCP_KEEPIDLE)

## IPPROTO_TCP.TCP_NODELAY

### 作用

有数据时立即发送，而不要使用Nagle算法进行缓存。

Nagle's algorithm is for reducing more number of small network packets in wire. The algorithm is: if data is smaller than a limit (usually MSS), wait until receiving ACK for previously sent packets and in the mean time accumulate data from user. Then send the accumulated data.

```text
if [ data > MSS ]
    send(data)
else
    wait until ACK for previously sent data and accumulate data in send buffer (data)
    And after receiving the ACK send(data)
```

### 应用场景

telnet/ssh等场景下需要禁用Nagle算法，提升交互的及时性。

### 代码举例

```C
/* --- begin of keepalive test program --- */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>


int main(int argc, char **argv)
{
    int s;
    int optval;
    socklen_t optlen = sizeof(optval);
    struct sockaddr_in addr;

    /* Create the socket */
    if ((s = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
    {
        perror("socket()");
        exit(EXIT_FAILURE);
    }

    /* Check the status for the keepalive option */
    if (getsockopt(s, IPPROTO_TCP, TCP_NODELAY, &optval, &optlen) < 0)
    {
        perror("getsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }
    printf("TCP_NODELAY is %s\n", (optval ? "ON" : "OFF"));

#ifdef DISABLE_TCP_NODELAY
    optval = 0;
#else
    optval = 1;
#endif

    /* Set the option active */
    optlen = sizeof(optval);
    if (setsockopt(s, IPPROTO_TCP, TCP_NODELAY, &optval, optlen) < 0)
    {
        perror("setsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }
    printf("TCP_NODELAY set on socket\n");

    /* Check the status again */
    if (getsockopt(s, IPPROTO_TCP, TCP_NODELAY, &optval, &optlen) < 0)
    {
        perror("getsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }
    printf("TCP_NODELAY is %s\n", (optval ? "ON" : "OFF"));

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(80);
    addr.sin_addr.s_addr = inet_addr("14.215.177.38");

    if (connect(s, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
        perror("connect()");
    }

    write(s, "GET", 5);
    write(s, " /HelloWorld", 5);

    sleep(1);
    close(s);

    exit(EXIT_SUCCESS);
}
```

### 执行结果

#### 开启TCP_NODELAY

```
$ gcc IPPROTO_TCP.TCP_NODELAY.c
$ ./a.out 
TCP_NODELAY is OFF
TCP_NODELAY set on socket
TCP_NODELAY is ON
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [S], seq 2882926778, win 29200, options [mss 1460,sackOK,TS val 3432240100 ecr 0,nop,wscale 7], length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [S.], seq 165056001, ack 2882926779, win 65535, options [mss 1460], length 0
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [.], ack 1, win 29200, length 0
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [P.], seq 1:6, ack 1, win 29200, length 5: HTTP
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [P.], seq 6:11, ack 1, win 29200, length 5: HTTP
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [.], ack 6, win 65535, length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [.], ack 11, win 65535, length 0
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [F.], seq 11, ack 1, win 29200, length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [.], ack 12, win 65535, length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [P.], seq 1:29, ack 12, win 65535, length 28: HTTP: HTTP/1.1 400 Bad Request
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [R], seq 2882926790, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [F.], seq 29, ack 12, win 65535, length 0
IP 10.0.2.15.39416 > 14.215.177.38.http: Flags [R], seq 2882926790, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39416: Flags [R.], seq 4129911295, ack 12, win 0, length 0

```

#### 关闭TCP_NODELAY

```C
$ gcc -DDISABLE_TCP_NODELAY IPPROTO_TCP.TCP_NODELAY.c
$ ./a.out 
TCP_NODELAY is OFF
TCP_NODELAY set on socket
TCP_NODELAY is OFF
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [S], seq 1539607424, win 29200, options [mss 1460,sackOK,TS val 3432226442 ecr 0,nop,wscale 7], length 0
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [S.], seq 163456001, ack 1539607425, win 65535, options [mss 1460], length 0
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [.], ack 1, win 29200, length 0
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [P.], seq 1:6, ack 1, win 29200, length 5: HTTP
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [.], ack 6, win 65535, length 0
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [P.], seq 6:11, ack 1, win 29200, length 5: HTTP
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [.], ack 11, win 65535, length 0
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [F.], seq 11, ack 1, win 29200, length 0
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [.], ack 12, win 65535, length 0
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [P.], seq 1:29, ack 12, win 65535, length 28: HTTP: HTTP/1.1 400 Bad Request
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [R], seq 1539607436, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [F.], seq 29, ack 12, win 65535, length 0
IP 10.0.2.15.39412 > 14.215.177.38.http: Flags [R], seq 1539607436, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39412: Flags [R.], seq 4131511295, ack 12, win 0, length 0
```

## IPPROTO_TCP.TCP_CORK

### 作用

有数据时不直接发送，而是累积到一定大小再发送。

### 应用场景

应用层连续调用write，每次write的都是小的数据。希望能够聚合多次的write，这时候就可以开启TCP_NODELAY，同时设置TCP_CORK，再多次write结束后再关闭TCP_CORK让内核发送数据。

### 代码举例

```C

/* --- begin of keepalive test program --- */

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>


int main(int argc, char **argv)
{
    int s;
    int optval;
    socklen_t optlen = sizeof(optval);
    struct sockaddr_in addr;

    /* Create the socket */
    if ((s = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
    {
        perror("socket()");
        exit(EXIT_FAILURE);
    }

    optval = 1;
    /* Set the option active */
    optlen = sizeof(optval);
    if (setsockopt(s, IPPROTO_TCP, TCP_NODELAY, &optval, optlen) < 0) {
        perror("setsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }

    if (setsockopt(s, IPPROTO_TCP, TCP_CORK, &optval, optlen) < 0) {
        perror("setsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(80);
    addr.sin_addr.s_addr = inet_addr("14.215.177.38");

    if (connect(s, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
        perror("connect()");
    }

    write(s, "GET", 5);
    write(s, " /HelloWorld", 5);

    optval = 0;
    if (setsockopt(s, IPPROTO_TCP, TCP_CORK, &optval, optlen) < 0) {
        perror("setsockopt()");
        close(s);
        exit(EXIT_FAILURE);
    }

    sleep(1);
    close(s);

    exit(EXIT_SUCCESS);
}
```



### 执行结果 

```C
[ljl@localhost sock_options]$ ./a.out 
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [S], seq 665374761, win 29200, options [mss 1460,sackOK,TS val 3432577802 ecr 0,nop,wscale 7], length 0
IP 14.215.177.38.http > 10.0.2.15.39436: Flags [S.], seq 200960001, ack 665374762, win 65535, options [mss 1460], length 0
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [.], ack 1, win 29200, length 0
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [P.], seq 1:11, ack 1, win 29200, length 10: HTTP
IP 14.215.177.38.http > 10.0.2.15.39436: Flags [.], ack 11, win 65535, length 0
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [F.], seq 11, ack 1, win 29200, length 0
IP 14.215.177.38.http > 10.0.2.15.39436: Flags [.], ack 12, win 65535, length 0
[ljl@localhost sock_options]$ IP 14.215.177.38.http > 10.0.2.15.39436: Flags [P.], seq 1:29, ack 12, win 65535, length 28: HTTP: HTTP/1.1 400 Bad Request
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [R], seq 665374773, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39436: Flags [F.], seq 29, ack 12, win 65535, length 0
IP 10.0.2.15.39436 > 14.215.177.38.http: Flags [R], seq 665374773, win 0, length 0
IP 14.215.177.38.http > 10.0.2.15.39436: Flags [R.], seq 4094007295, ack 12, win 0, length 0
```

可以看到11个字节是在同一个报文`seq 1:11`发送的







## 客户端是否可以使用同一个端口连接到不同的服务端

可以看到，设置了SO_REUSEADDR就可以使用同一个客户端连接到不同的服务端端口。

如果取消了SO_REUSEADDR则bind失败。

``` shell
gcc -DUSE_REUSEADDR ./client.c
nc -k -l 12345 &
nc -k -l 12346 &
./a.out  8888 12345 &
./a.out  8888 12346 &
```



可以看到，不设置SO_REUSEADDR，但是绑定到本地不同的客户端IP依然不能够连接到服务端的不同端口。

``` shell
gcc  ./client.c
nc -k -l 12345 &
nc -k -l 12346 &
./a.out  8888 12345 127.0.0.2 &
./a.out  8888 12346 127.0.0.3 &
```



//测试代码 test.c

``` C 
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>
#include <errno.h>

static void
set_socket_reuseaddr(int fd)
{
    int       on;
    socklen_t olen;
    char      buf[32];
    const char *typename;

    olen = sizeof(int);
    on = 1;
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (void *)&on, olen) == -1) {
        printf("failed to set reuseaddr for fd %d, error: %s\n",
               fd, strerror(errno));
    }
}

int main(int argc, char **argv)
{
    int s;
    int cport;
    int sport;
    struct sockaddr_in local_sock;
    struct sockaddr_in addr;

    if (argc < 3) {
        printf("must specify local-port remote-port\n");
        exit(1);
    }

    cport = atoi(argv[1]);

    if (cport <= 0 || cport > 65536) {
        printf("invalid local local-port num %s\n", argv[1]);
        exit(1);
    }

    sport = atoi(argv[2]);
    if (sport <= 0 || sport > 65536) {
        printf("invalid local remote-port num %s\n", argv[2]);
        exit(1);
    }
 
    /* Create the socket */
    if ((s = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
    {
        perror("socket()");
        exit(EXIT_FAILURE);
    }

    set_socket_reuseaddr(s);

    memset(&local_sock, 0, sizeof(local_sock));
    local_sock.sin_family = AF_INET;
    local_sock.sin_port = htons(cport);
    local_sock.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (bind(s, (struct sockaddr *) &local_sock, sizeof(local_sock)) != 0) {
        perror("failed to bind");
        exit(1);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(sport);
    addr.sin_addr.s_addr = inet_addr("127.0.0.1");

    if (connect(s, (struct sockaddr *)&addr, sizeof(addr)) != 0) {
        perror("failed to connect");
        exit(1);
    }

    printf("connect ok\n");

    sleep(10);
    close(s);

    exit(EXIT_SUCCESS);
}
```



## 客户端使用不同的IP连接到不同的服务端

### 测试代码

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdlib.h>
#include <arpa/inet.h>


int
main(int argc, char **argv)
{
    int       socket_fd;
    int       socket_fd2;
    char      cmd[128];
    struct sockaddr_in saddr;
    struct sockaddr_in daddr;
    char      *src;
    char      *dest;

    if (argc != 3) {
        printf("usage: exe src dst\n");
        exit(1);
    }

    src = argv[1];
    dest = argv[2];

    saddr.sin_family = AF_INET;
    saddr.sin_port = htons (8888);
    inet_pton(AF_INET, src, (void *)&saddr.sin_addr);

    daddr.sin_family = AF_INET;
    daddr.sin_port = htons(443);
    inet_pton(AF_INET, dest, (void *)&daddr.sin_addr);

    socket_fd = socket(AF_INET , SOCK_STREAM , 0);

    if (bind(socket_fd, (struct sockaddr *) &saddr, sizeof(saddr)) != 0) {
        printf("fd %d failed to bind to port 8888, error:%s\n",
               socket_fd, strerror(errno));
    } else {
        printf("fd %d bind to 8888 successful\n", socket_fd);
    }

    if (connect(socket_fd, (struct sockaddr *)&daddr, sizeof(daddr)) != 0) {
        printf("fd %d connect to 8888 failed\n", socket_fd);
    }

    printf("connect successful\n");

    sleep(1000);
    return 0;
}
```

### 测试方法

```shell
./a.out 10.0.2.154 14.215.177.38
./a.out 10.0.2.155 14.215.177.38
```

