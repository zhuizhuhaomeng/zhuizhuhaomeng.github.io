---
layout: post
title: "epoll 的一些学习笔记"
description: "epoll 重新学习"
date: 2020-03-03
tags: [epoll]
---

# 边沿触发，分多次系统调用读完所有数据

该实验的目的是展示边沿触发需要一次性处理完所有socket里面的数据。
实际的工程中不一定是一次性处理完，有可能是推后处理：比如为了公平性，一个 socket 最多处理
64k 的数据，如果还有未处理完毕的数据，那么加入待处理队列接着处理下一个socket的数据。

## server 的代码

```C
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define DEFAULT_PORT 8080
#define MAX_CONN     16
#define MAX_EVENTS   32
#define BUF_SIZE     16
#define MAX_LINE     256

void server_run();

int
main(int argc, char *argv[])
{
    server_run();
    return 0;
}

/*
 * register events of fd to epfd
 */
static void
epoll_ctl_add(int epfd, int fd, uint32_t events)
{
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl()\n");
        exit(1);
    }
}

static void
set_sockaddr(struct sockaddr_in *addr)
{
    bzero((char *) addr, sizeof(struct sockaddr_in));
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = INADDR_ANY;
    addr->sin_port = htons(DEFAULT_PORT);
}

static int
setnonblocking(int sockfd)
{
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFL, 0) | O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

void
server_run()
{
    int                rc;
    int                i;
    int                n;
    int                fd;
    int                epfd;
    int                nfds;
    int                listen_sock;
    int                enable = 1;
    int                conn_sock;
    socklen_t          socklen;
    char               buf[BUF_SIZE];
    struct sockaddr_in srv_addr;
    struct sockaddr_in cli_addr;
    struct epoll_event events[MAX_EVENTS];

    listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &enable,
                   sizeof(int)) < 0) {
        printf("failed to set SO_REUSEADDR error: %s\n", strerror(errno));
        exit(1);
    }

    set_sockaddr(&srv_addr);
    rc = bind(listen_sock, (struct sockaddr *) &srv_addr, sizeof(srv_addr));
    if (rc != 0) {
        printf("failed to bind on 0.0.0.:%d, error: %s\n", DEFAULT_PORT,
               strerror(errno));
        exit(1);
    }

    rc = setnonblocking(listen_sock);
    if (rc != 0) {
        printf("failed to set nonblock for socket: %s\n", strerror(errno));
    }

    rc = listen(listen_sock, MAX_CONN);
    if (rc != 0) {
        printf("failed to listen, erorr: %s\n", strerror(errno));
        exit(1);
    }

    printf("listen on port %d\n", DEFAULT_PORT);

    epfd = epoll_create(1);
    if (epfd == -1) {
        printf("failed to create epoll, error: %s\n", strerror(errno));
        exit(1);
    }

    epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT | EPOLLET);

    socklen = sizeof(cli_addr);
    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < nfds; i++) {
            fd = events[i].data.fd;
            if (fd == listen_sock) {
                /* handle new connection */
                conn_sock = accept(listen_sock, (struct sockaddr *) &cli_addr,
                                   &socklen);
                if (conn_sock == -1) {
                    printf("accept failed, %s\n", strerror(errno));
                    continue;
                }

                inet_ntop(AF_INET, (char *) &(cli_addr.sin_addr), buf,
                          sizeof(cli_addr));
                printf("[+] accept client %s:%d\n", buf,
                       ntohs(cli_addr.sin_port));

                setnonblocking(conn_sock);
                epoll_ctl_add(epfd, conn_sock,
                              EPOLLIN | EPOLLET | EPOLLRDHUP | EPOLLHUP);

            } else {
                /* check if the connection is closing */
                if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
                    printf("[+] connection closed\n");
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                    continue;
                }

                if (events[i].events & EPOLLIN) {
                    printf("client socket events: %#x\n", events[i].events);
                    for (;;) {
                        bzero(buf, sizeof(buf));
                        n = read(fd, buf, sizeof(buf));
                        if (n < 0) {
                            if (errno != EAGAIN) {
                                printf("receive len %d, error %s\n", n,
                                       strerror(errno));
                                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                                close(fd);
                            }
                            break;

                        } else {
                            printf("[+] data: %.*s\n", n, buf);
                            write(fd, buf, n);
                        }
                    }
                }
            }
        }
    }
}
```

在该程序中，我们将 read 的缓冲设置为16个字节，只要客户端发送的数据超过16字节，那么就需要多次调用 read。

## client 的代码

```python3
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serverAddr = ('127.0.0.1',8080)
client.connect(serverAddr)

sendData='01234567890123456789'
print("send: %s" % sendData)
client.send(sendData.encode())

client.settimeout(5.0)
print('receive:', end = '', flush=True)


while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nexception: socket timeout')
        break
client.close()
print('close socket!')
```

## 执行测试

我们在一个终端编译并启动该程序,等待客户端连接。

```shell
$ gcc -g -O2 -Wall -o server server.c
$ ./server
listen on port 8080
[+] accept client 127.0.0.1:49986
client socket events: 0x1
[+] data: 0123456789012345
[+] data: 6789
[+] connection closed
```

可以看到接收了两次的数据：一次 16 字节，一次4字节。

我们在另一个终端启动客户端程序

```shell
$ python3 client.py
send: 01234567890123456789
receive:01234567890123456789
exception: socket timeout
close socket!
```

# 边沿触发，读取部分数据

该实验的目的是验证边沿触发的情况下，再次接收到客户端数据是否会再次被通知。

结论：只要客户端有发送数据过来，那么服务端就会被通知。

## server 的代码

```C
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define DEFAULT_PORT 8080
#define MAX_CONN     16
#define MAX_EVENTS   32
#define BUF_SIZE     16
#define MAX_LINE     256

void server_run();

int
main(int argc, char *argv[])
{
    server_run();
    return 0;
}

/*
 * register events of fd to epfd
 */
static void
epoll_ctl_add(int epfd, int fd, uint32_t events)
{
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl()\n");
        exit(1);
    }
}

static void
set_sockaddr(struct sockaddr_in *addr)
{
    bzero((char *) addr, sizeof(struct sockaddr_in));
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = INADDR_ANY;
    addr->sin_port = htons(DEFAULT_PORT);
}

static int
setnonblocking(int sockfd)
{
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFL, 0) | O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

void
server_run()
{
    int                rc;
    int                i;
    int                n;
    int                fd;
    int                epfd;
    int                nfds;
    int                listen_sock;
    int                enable = 1;
    int                conn_sock;
    socklen_t          socklen;
    char               buf[BUF_SIZE];
    struct sockaddr_in srv_addr;
    struct sockaddr_in cli_addr;
    struct epoll_event events[MAX_EVENTS];

    listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &enable,
                   sizeof(int)) < 0) {
        printf("failed to set SO_REUSEADDR error: %s\n", strerror(errno));
        exit(1);
    }

    set_sockaddr(&srv_addr);
    rc = bind(listen_sock, (struct sockaddr *) &srv_addr, sizeof(srv_addr));
    if (rc != 0) {
        printf("failed to bind on 0.0.0.:%d, error: %s\n", DEFAULT_PORT,
               strerror(errno));
        exit(1);
    }

    rc = setnonblocking(listen_sock);
    if (rc != 0) {
        printf("failed to set nonblock for socket: %s\n", strerror(errno));
    }

    rc = listen(listen_sock, MAX_CONN);
    if (rc != 0) {
        printf("failed to listen, erorr: %s\n", strerror(errno));
        exit(1);
    }

    printf("listen on port %d\n", DEFAULT_PORT);

    epfd = epoll_create(1);
    if (epfd == -1) {
        printf("failed to create epoll, error: %s\n", strerror(errno));
        exit(1);
    }

    epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT | EPOLLET);

    socklen = sizeof(cli_addr);
    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < nfds; i++) {
            fd = events[i].data.fd;
            if (fd == listen_sock) {
                /* handle new connection */
                conn_sock = accept(listen_sock, (struct sockaddr *) &cli_addr,
                                   &socklen);
                if (conn_sock == -1) {
                    printf("accept failed, %s\n", strerror(errno));
                    continue;
                }

                inet_ntop(AF_INET, (char *) &(cli_addr.sin_addr), buf,
                          sizeof(cli_addr));
                printf("[+] accept client %s:%d\n", buf,
                       ntohs(cli_addr.sin_port));

                setnonblocking(conn_sock);
                epoll_ctl_add(epfd, conn_sock,
                              EPOLLIN | EPOLLET | EPOLLRDHUP | EPOLLHUP);

            } else {
                /* check if the connection is closing */
                if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
                    printf("[+] connection closed\n");
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                    continue;
                }

                if (events[i].events & EPOLLIN) {
                    printf("client socket events: %#x\n", events[i].events);
                    bzero(buf, sizeof(buf));
                    n = read(fd, buf, sizeof(buf));
                    if (n < 0) {
                        if (errno != EAGAIN) {
                            printf("receive len %d, error %s\n", n,
                                   strerror(errno));
                            epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                            close(fd);
                        }
                        break;

                    } else {
                        printf("[+] data: %.*s\n", n, buf);
                        write(fd, buf, n);
                    }
                }
            }
        }
    }
}
```

在该程序中，我们将 read 的缓冲设置为16个字节，只要客户端发送的数据超过16字节，那么就需要多次调用 read。

## client 的代码

```python3
#!/usr/bin/python3

import socket


client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serverAddr = ('127.0.0.1',8080)
client.connect(serverAddr)

sendData='01234567890123456789'
print("send: %s" % sendData)
client.send(sendData.encode())

client.settimeout(5.0)
print('receive: ', end = '', flush=True)


while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nwait timeout')
        break

client.send(sendData.encode())

print('receive: ', end = '', flush=True)
while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nwait timeout')
        break

client.close()
print('close socket!')
```

## 执行测试

我们在一个终端编译并启动该程序,等待客户端连接。

```shell
$ gcc -g -O2 -Wall -o server server.c
$ ./server
[+] accept client 127.0.0.1:36074
client socket events: 0x1
[+] data: 0123456789012345
client socket events: 0x1
[+] data: 6789012345678901
[+] connection closed
```

可以看到在客户端第二次发送后，服务端才执行第二次数据接收。

我们在另一个终端启动客户端程序

```shell
$ python3 client.py
send: 01234567890123456789
receive: 0123456789012345
wait timeout
receive: 6789012345678901
wait timeout
close socket!
```

# EPOLL 水平触发

## 服务端 C 代码

```C
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define DEFAULT_PORT 8080
#define MAX_CONN     16
#define MAX_EVENTS   32
#define BUF_SIZE     16
#define MAX_LINE     256

void server_run();

int
main(int argc, char *argv[])
{
    server_run();
    return 0;
}

/*
 * register events of fd to epfd
 */
static void
epoll_ctl_add(int epfd, int fd, uint32_t events)
{
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl()\n");
        exit(1);
    }
}

static void
set_sockaddr(struct sockaddr_in *addr)
{
    bzero((char *) addr, sizeof(struct sockaddr_in));
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = INADDR_ANY;
    addr->sin_port = htons(DEFAULT_PORT);
}

static int
setnonblocking(int sockfd)
{
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFL, 0) | O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

void
server_run()
{
    int                rc;
    int                i;
    int                n;
    int                epfd;
    int                nfds;
    int                listen_sock;
    int                fd;
    int                enable = 1;
    socklen_t          socklen;
    char               buf[BUF_SIZE];
    struct sockaddr_in srv_addr;
    struct sockaddr_in cli_addr;
    struct epoll_event events[MAX_EVENTS];

    listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &enable,
                   sizeof(int)) < 0) {
        printf("failed to set SO_REUSEADDR error: %s\n", strerror(errno));
        exit(1);
    }

    set_sockaddr(&srv_addr);
    rc = bind(listen_sock, (struct sockaddr *) &srv_addr, sizeof(srv_addr));
    if (rc != 0) {
        printf("failed to bind on 0.0.0.:%d, error: %s\n", DEFAULT_PORT,
               strerror(errno));
        exit(1);
    }

    rc = setnonblocking(listen_sock);
    if (rc != 0) {
        printf("failed to set nonblock for socket: %s\n", strerror(errno));
    }

    rc = listen(listen_sock, MAX_CONN);
    if (rc != 0) {
        printf("failed to listen, erorr: %s\n", strerror(errno));
        exit(1);
    }

    printf("listen on port %d\n", DEFAULT_PORT);

    epfd = epoll_create(1);
    if (epfd == -1) {
        printf("failed to create epoll, error: %s\n", strerror(errno));
        exit(1);
    }

    epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT);

    socklen = sizeof(cli_addr);
    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_sock) {
                /* handle new connection */
                fd = accept(listen_sock, (struct sockaddr *) &cli_addr,
                            &socklen);
                if (fd == -1) {
                    printf("accept failed, %s\n", strerror(errno));
                    continue;
                }

                inet_ntop(AF_INET, (char *) &(cli_addr.sin_addr), buf,
                          sizeof(cli_addr));
                printf("[+] accept client %s:%d\n", buf,
                       ntohs(cli_addr.sin_port));

                setnonblocking(fd);
                epoll_ctl_add(epfd, fd, EPOLLIN | EPOLLRDHUP | EPOLLHUP);
            } else if (events[i].events & EPOLLIN) {
                fd = events[i].data.fd;
                printf("client socket events: %#x\n", events[i].events);
                bzero(buf, sizeof(buf));
                n = read(fd, buf, sizeof(buf));
                if (n < 0 && errno == EAGAIN) {
                    printf("receive len %d, error %d\n", n, errno);
                } else {
                    printf("[+] data: %.*s\n", n, buf);
                    write(fd, buf, n);
                }
            } else {
                printf("[+] unexpected\n");
            }

            /* check if the connection is closing */
            if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
                printf("[+] connection closed\n");
                epoll_ctl(epfd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                close(events[i].data.fd);
                continue;
            }
        }
    }
}
```

## 客户端 Python 代码

```python
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serverAddr = ('127.0.0.1',8080)
client.connect(serverAddr)

sendData='01234567890123456789'
print("send: %s" % sendData)
client.send(sendData.encode())

client.settimeout(5.0)
print('receive:', end = '', flush=True)


while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nexception: socket timeout')
        break
client.close()
print('close socket!')
```

## 执行测试

终端一执行以下命令

```shell
$ gcc -g -O2 -Wall -o server server.c
$ ./server
listen on port 8080
[+] accept client 127.0.0.1:50396
client socket events: 0x1
[+] data: 0123456789012345
client socket events: 0x1
[+] data: 6789
client socket events: 0x2001
[+] data: 
[+] connection closed
```

可以看到操作系统通知了应用层两次，接收的数据分别是：0123456789012345 和 6789。

# EPOLLONESHOT 测试

在这个实验中，我们给 fd 设置了 EPOLLONESHOT 的属性，并且没有一次性读完所有的数据。

## 服务端 C 代码
```C
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define DEFAULT_PORT 8080
#define MAX_CONN     16
#define MAX_EVENTS   32
#define BUF_SIZE     16
#define MAX_LINE     256

void server_run();

int
main(int argc, char *argv[])
{
    server_run();
    return 0;
}

/*
 * register events of fd to epfd
 */
static void
epoll_ctl_add(int epfd, int fd, uint32_t events)
{
    struct epoll_event ev;
    ev.events = events;
    ev.data.fd = fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev) == -1) {
        perror("epoll_ctl()\n");
        exit(1);
    }
}

static void
set_sockaddr(struct sockaddr_in *addr)
{
    bzero((char *) addr, sizeof(struct sockaddr_in));
    addr->sin_family = AF_INET;
    addr->sin_addr.s_addr = INADDR_ANY;
    addr->sin_port = htons(DEFAULT_PORT);
}

static int
setnonblocking(int sockfd)
{
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFL, 0) | O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

void
server_run()
{
    int                rc;
    int                i;
    int                n;
    int                epfd;
    int                fd;
    int                nfds;
    int                listen_sock;
    int                conn_sock;
    int                enable = 1;
    socklen_t          socklen;
    char               buf[BUF_SIZE];
    struct sockaddr_in srv_addr;
    struct sockaddr_in cli_addr;
    struct epoll_event events[MAX_EVENTS];

    listen_sock = socket(AF_INET, SOCK_STREAM, 0);
    if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &enable,
                   sizeof(int)) < 0) {
        printf("failed to set SO_REUSEADDR error: %s\n", strerror(errno));
        exit(1);
    }

    set_sockaddr(&srv_addr);
    rc = bind(listen_sock, (struct sockaddr *) &srv_addr, sizeof(srv_addr));
    if (rc != 0) {
        printf("failed to bind on 0.0.0.:%d, error: %s\n", DEFAULT_PORT,
               strerror(errno));
        exit(1);
    }

    rc = setnonblocking(listen_sock);
    if (rc != 0) {
        printf("failed to set nonblock for socket: %s\n", strerror(errno));
    }

    rc = listen(listen_sock, MAX_CONN);
    if (rc != 0) {
        printf("failed to listen, erorr: %s\n", strerror(errno));
        exit(1);
    }

    printf("listen on port %d\n", DEFAULT_PORT);

    epfd = epoll_create(1);
    if (epfd == -1) {
        printf("failed to create epoll, error: %s\n", strerror(errno));
        exit(1);
    }

    epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT | EPOLLET);

    socklen = sizeof(cli_addr);
    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < nfds; i++) {
            fd = events[i].data.fd;
            if (fd == listen_sock) {
                /* handle new connection */
                conn_sock = accept(listen_sock, (struct sockaddr *) &cli_addr,
                                   &socklen);
                if (conn_sock == -1) {
                    printf("accept failed, %s\n", strerror(errno));
                    continue;
                }

                inet_ntop(AF_INET, (char *) &(cli_addr.sin_addr), buf,
                          sizeof(cli_addr));
                printf("[+] accept client %s:%d\n", buf,
                       ntohs(cli_addr.sin_port));

                setnonblocking(conn_sock);
                epoll_ctl_add(epfd, conn_sock,
                              EPOLLIN | EPOLLET | EPOLLRDHUP | EPOLLHUP | EPOLLONESHOT);
            } else if (events[i].events & EPOLLIN) {
                printf("client socket events: %#x\n", events[i].events);
                bzero(buf, sizeof(buf));
                n = read(fd, buf, sizeof(buf));
                if (n < 0) {
                    if (errno != EAGAIN) {
                        printf("receive len %d, error %d\n", n, errno);
                    }
                } else {
                    printf("[+] data: %s\n", buf);
                    write(events[i].data.fd, buf, strlen(buf));
                }
            } else {
                printf("[+] unexpected\n");
            }

            /* check if the connection is closing */
            if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
                printf("[+] connection closed\n");
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
                continue;
            }
        }
    }
}
```

## 客户端 python 代码

```python3
import socket

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serverAddr = ('127.0.0.1',8080)
client.connect(serverAddr)

sendData='01234567890123456789'
print("send: %s" % sendData)
client.send(sendData.encode())

client.settimeout(1.0)

print('receive:', end = '', flush=True)
while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nexception: socket timeout')
        print("send: end")
        client.send("end".encode())
        break

print('receive:', end = '', flush=True)
while True:
    try:
        recvData = client.recv(1024)
        if not recvData:
            break
        print('%s' % recvData.decode("utf-8"), end = '', flush=True)
    except socket.timeout:
        print('\nexception: socket timeout')
        break

client.close()
print('close socket!')
```

## 执行测试

终端一执行以下命令

```shell
$ gcc -g -O2 -Wall -o server server.c
$ ./server
listen on port 8080
[+] accept client 127.0.0.1:52198
client socket events: 0x1
[+] data: 0123456789012345
```

终端二执行客户端程序

```python3
$ python3 client.py
send: 01234567890123456789
receive:0123456789012345 
exception: socket timeout
send: end
receive:
exception: socket timeout
close socket!
```

我们看到，在客户端关闭 socket 之后，服务端没有任何响应。
这是因为我们设置了 EPOLLONESHOT， 而客户端第一次发送数据后，
服务端读取数据之后没有再将 fd 添加到 epoll中，所以后续数据到来也不会通知该 fd 了。

如果此时用 tcpdmp 抓包，可以看到客户端发送了 fin，而服务端只是内核发送了ack，
因为服务端没有关闭socket，因此也就没有发送fin。

```tcpdump
10:56:12.827350 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [S], seq 4067671835, win 43690, options [mss 65495,sackOK,TS val 1833339252 ecr 0,nop,wscale 7], length 0
10:56:12.827388 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [S.], seq 496646672, ack 4067671836, win 43690, options [mss 65495,sackOK,TS val 1833339252 ecr 1833339252,nop,wscale 7], length 0
10:56:12.827414 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [.], ack 1, win 342, options [nop,nop,TS val 1833339252 ecr 1833339252], length 0
10:56:12.827520 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [P.], seq 1:21, ack 1, win 342, options [nop,nop,TS val 1833339252 ecr 1833339252], length 20: HTTP
10:56:12.827534 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [.], ack 21, win 342, options [nop,nop,TS val 1833339252 ecr 1833339252], length 0
10:56:12.827705 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [P.], seq 1:18, ack 21, win 342, options [nop,nop,TS val 1833339253 ecr 1833339252], length 17: HTTP
10:56:12.827740 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [.], ack 18, win 342, options [nop,nop,TS val 1833339253 ecr 1833339253], length 0
10:56:13.829021 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [P.], seq 21:24, ack 18, win 342, options [nop,nop,TS val 1833340254 ecr 1833339253], length 3: HTTP
10:56:13.869670 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [.], ack 24, win 342, options [nop,nop,TS val 1833340295 ecr 1833340254], length 0
10:56:14.830280 IP 127.0.0.1.52198 > 127.0.0.1.8080: Flags [F.], seq 24, ack 18, win 342, options [nop,nop,TS val 1833341255 ecr 1833340295], length 0
10:56:14.870640 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [.], ack 25, win 342, options [nop,nop,TS val 1833341296 ecr 1833341255], length 0
```

注意：tcpdump 的程序先不要停止。


此时，用 netstat 查看，socket 处于 CLOSE_WAIT 状态

```
sudo netstat -tplna | grep 8080 
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      64830/./server      
tcp        8      0 127.0.0.1:8080          127.0.0.1:52198         CLOSE_WAIT  64830/./server 
```

这时候，我们按 CTRL+C 停止进程，在 tcpdump 所在的终端看到内核发送了 reset 的报文而不是 fin 报文，这是因为 socket 里面还有未接收的数据。
上面的 netstat 也可以看到 socket 里面还有 8 个字节未被应用层接收处理。

```tcpdump
11:01:49.978155 IP 127.0.0.1.8080 > 127.0.0.1.52198: Flags [R.], seq 18, ack 25, win 342, options [nop,nop,TS val 1833676403 ecr 1833341255], length 0
```

# 总结

1. socket 有未读取的数据，关闭 socket会发送 reset
1. 边沿触发情况下，如果没有一次性处理完所有 socket 中的数据，那么操作系统不会再次通知进程。但是如果再次收到客户端是数据，那么操作系统是会再次通知进程的。
1. 我们如果看到大量 close-wait 的socket，那么就是应用层代码没有调用 close(fd) 导致的。
1. 设置 EPOLLONESHOT 的 fd 在被触发后需要重新添加该 fd 到 epoll 中，否则有事件也不会被通知了。
