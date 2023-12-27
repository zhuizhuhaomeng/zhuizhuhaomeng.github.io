---
layout: post
title: "查看添加到 epoll 的文件描述符对应的事件"
description: "check the epoll fd events"
date: 2023-01-28
feature_image: img/epoll-event-loop.png
tags: [epoll, debug]
---

# 找到要分析的进程

比如我们要分析 nginx worker 进程的 epoll 都添加了哪些句柄，他们的 events 是如何。
我们用下列命令找到要分析的 nginx 进程。

```shell
$ ps aux | grep "nginx: worker"
nobody    606535  0.0  0.0  78076  8488 ?        S    17:22   0:11 nginx: worker process
```

# 找到 epoll 的文件句柄

```shell
$ sudo ls -l /proc/606535/fd | awk '/eventpoll/'
lrwx------ 1 nobody nobody 64 Jan 28 21:43 12 -> anon_inode:[eventpoll]
```

通过上述命令，我们知道 epoll 的文件句柄是 12.

# 查看添加到 epoll 的文件句柄和对应的监听事件

```shell
sudo cat /proc/606535/fdinfo/12          
pos:	0
flags:	02
mnt_id:	15
tfd:        6 events:     2019 data:     7fd0008c6010  pos:0 ino:b59a sdev:9
tfd:        7 events:     2019 data:     7fd0008c6100  pos:0 ino:b59b sdev:9
tfd:       10 events:     2019 data:     7fd0008c62e0  pos:0 ino:30a651 sdev:9
tfd:       14 events: 80002019 data:     7fd0008c61f0  pos:0 ino:2a5a sdev:e
tfd:       13 events: 80000019 data:           613780  pos:0 ino:2a5a sdev:e
```

我们看到，有些 events 是 2019，而有些是 80000019，有些是 80002019。他们分别是什么意思呢？
我们通过 sys/epoll.h 这个文件得到 events 的定义如下：

```c
enum EPOLL_EVENTS
  {
    EPOLLIN        = 0x001,
    EPOLLPRI       = 0x002,
    EPOLLOUT       = 0x004,
    EPOLLRDNORM    = 0x040,
    EPOLLRDBAND    = 0x080,
    EPOLLWRNORM    = 0x100,
    EPOLLWRBAND    = 0x200,
    EPOLLMSG       = 0x400,
    EPOLLERR       = 0x008,
    EPOLLHUP       = 0x010,
    EPOLLRDHUP     = 0x2000,
    EPOLLEXCLUSIVE = 1u << 28,
    EPOLLWAKEUP    = 1u << 29,
    EPOLLONESHOT   = 1u << 30,
    EPOLLET        = 1u << 31
};
```

因此，80002019 就是 EPOLLET | EPOLLRDHUP | EPOLLHUP |EPOLLRDBAND | EPOLLIN

# 将 fd 和 socket 对应起来

这个时候我们可以通过 lsof 将 fd 和 socket 的 IP 地址对应起来。

```shell
$ sudo lsof -P -p 606535 | egrep -v "(REG|DIR|CHR)" 
COMMAND    PID   USER   FD      TYPE             DEVICE   SIZE/OFF      NODE NAME
nginx   606535 nobody    6u     IPv4              46490        0t0       TCP *:80 (LISTEN)
nginx   606535 nobody    7u     IPv4              46491        0t0       TCP *:443 (LISTEN)
nginx   606535 nobody   10u     unix 0xffff9fcd9e25c800        0t0   3188305 type=STREAM
nginx   606535 nobody   12u  a_inode               0,14          0     10842 [eventpoll]
nginx   606535 nobody   13u  a_inode               0,14          0     10842 [eventfd]
nginx   606535 nobody   14u  a_inode               0,14          0     10842 [signalfd]
```

有时候 socket 会快速的并打开关闭，因此 losf 命令 和 上面 cat /proc/pid/fdinfo/epollfd-num 的内容不会完全对应，因此分析问题的时候要注意这种情况。
