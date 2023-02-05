---
layout: post
title: "排查 PostgreSQL 慢查询发现 mlocate-updatedb 导致的 IO 瓶颈"
description: "mlocate-updatedb 占用了大部分磁盘 IO"
date: 2023-01-21
feature_image: img/Hard-Disk.png
tags: [locate, disk, mlocate, mlocate-updatedb.service, mlocate-updatedb.timer]
---

# 背景

客户其中一套系统的 PostgreSQL 出现很多的慢查询，发现 PostgreSQL 所在机器的 IO 利用率很高。
因为该机器有好多块硬盘，而且硬盘的型号以及相应的服务都不一样，因此我们只能一块块硬盘来分析。

# 分析过程

我们使用 iotop 工具来分析磁盘 IO 的占用问题。 比如

```shell
Total DISK READ :      66.61 K/s | Total DISK WRITE :       2.45 M/s
Actual DISK READ:      66.61 K/s | Actual DISK WRITE:    1630.07 K/s
    PID  PRIO  USER     DISK READ DISK WRITE>    COMMAND
   9472 be/4 pg 0.00 B/s 1191.21 K/s postgres: walwriter
 478127 be/4 pg 0.00 B/s  438.87 K/s postgres: walwriter
1929495 be/4 nobody 0.00 B/s   70.53 K/s nginx: worker process
```

在使用 iotop 分析问题的过程当中我们发现有一个 updatedb 的进程占用了其中一块磁盘的大多数 IO。
通过下面的命令我们很快确认该命令是 mlocate软件包提供的。

```shell
rpm -qf `which updatedb`
```

通过 [mlocate 的 man 手册](https://www.unix.com/man-page/linux/1/mlocate/) 我们可以知道 mlocate 提供了 locate 命令用来查找机器上的文件。比如我想查找所有文件名为 nginx 的文件，那么可以通过如下命令来查找。

```shell
locate -b -r "^nginx$"
```

要提供文件查找服务，那么必现要有一个后台的索引服务，否则实时遍历所有的磁盘文件，那么性能肯定惨不忍睹。
我们看看 mlocate 都提供了哪些服务呢？通过以下命令可以知道软件包提供您了两个服务。

```shell
$ rpm -ql mlocate | grep systemd
/usr/lib/systemd/system/mlocate-updatedb.service
/usr/lib/systemd/system/mlocate-updatedb.timer
```

我们进一步看看这个服务是怎么调度的。


```shell
cat /usr/lib/systemd/system/mlocate-updatedb.service
[Unit]
Description=Update a database for mlocate

[Service]
ExecStart=/usr/libexec/mlocate-run-updatedb
Nice=19
IOSchedulingClass=2
IOSchedulingPriority=7

PrivateTmp=true
PrivateDevices=true
PrivateNetwork=true
ProtectSystem=true

$ cat /usr/libexec/mlocate-run-updatedb
#!/bin/sh

nodevs=$(< /proc/filesystems awk '$1 == "nodev" && $2 != "rootfs" && $2 != "zfs" { print $2 }')
/usr/bin/updatedb -f "$nodevs"
```

通过执行上述命令， 我们知道 mlocate-updatedb.service 的服务会调用 updatedb 命令。
updatedb 命令是一个不是后台常驻进程，因此执行结束进程就退出了。

我们再看看 mlocate-updatedb.timer 是怎么工作的。


```shell
cat /usr/lib/systemd/system/mlocate-updatedb.timer
[Unit]
Description=Updates mlocate database every day

[Timer]
OnCalendar=daily
AccuracySec=24h
Persistent=true

[Install]
WantedBy=timers.target
```

通过上述命令，我们知道这个服务是24小时执行一次的, 由timers 这个服务来调度。


# 解决方案

因为我们并不需要使用 locate 命令来定位文件，因此没有必要开启该服务。

```shell
systemctl stop mlocate-update.service
systemctk stop mlocate-update.timer
systemctl disable mlocate-update.service
systemctl disable mlocate-update.timer
```

# 遗留问题

为什么该要默认开启该服务呢？为什么会默认安装 mlocate 这个软件包呢？这个等待进一步挖掘。
