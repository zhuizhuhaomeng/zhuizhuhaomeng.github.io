---
layout: post
title: "proc of the linux"
description: "proc of the linux"
date: 2023-04-29
tags: [proc]
---

# 获取进程创建时间

如果我们想要获取进程的创建时间，那么可以通过下面的命令 `stat /proc/pid`

```shell
$ stat /proc/1
  File: /proc/1
  Size: 0         	Blocks: 0          IO Block: 1024   directory
Device: 5h/5d	Inode: 13428       Links: 9
Access: (0555/dr-xr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-04-29 08:20:03.159000000 +0800
Modify: 2023-04-29 08:20:03.159000000 +0800
Change: 2023-04-29 08:20:03.159000000 +0800
 Birth: -
```

# 获取 socket 的创建时间

如果想要获取 socket 的创建时间，那么可以通过 `stat /proc/pid/fd/id` 这样的命令。
比如：

```shell
$ stat /proc/2045/fd/33
  File: /proc/2045/fd/33 -> socket:[15849]
  Size: 64        	Blocks: 0          IO Block: 1024   symbolic link
Device: 5h/5d	Inode: 2163860     Links: 1
Access: (0700/lrwx------)  Uid: (65534/  nobody)   Gid: (65534/  nobody)
Access: 2023-04-29 14:34:43.088949028 +0800
Modify: 2023-04-29 14:34:40.394903918 +0800
Change: 2023-04-29 14:34:40.394903918 +0800
 Birth: -
```

# 获取进程运行时的限制

我们可以通过 ulimit 命令调整限制参数。但是如果我们想要看一个进程的限制参数是否跟我们设置的是一致的，
可以通过 proc 文件系统进行确认。命令为 `cat /proc/pid/limits`.

比如，通过下面的命令，我们看到最多可以打开 1024 个文件，这对于 Nginx 服务来说很可能是不够的。

```shell
$ cat /proc/564760/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             127556               127556               processes
Max open files            1024                 262144               files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       127556               127556               signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```

# 进程的命令行参数

在排查故障的时候，我们要了解进程是怎么启动的，都指定了哪些命令行参数，可以通过
`cat /proc/pid/cmdline | tr '\0' ' '` 查看。

因为命令行参数之间是通过 '\0' 分割的，如果不使用 tr 进行转换，那么就会发现进程命令和参数
黏在一起。

```shell
$ cat /proc/1699/cmdline | tr '\0' ' '
/usr/sbin/sshd -D -oCiphers=aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes256-cbc,aes128-gcm@openssh.com,aes128-ctr,aes128-cbc -oMACs=hmac-sha2-256-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha1,umac-128@openssh.com,hmac-sha2-512 -oGSSAPIKexAlgorithms=gss-curve25519-sha256-,gss-nistp256-sha256-,gss-group14-sha256-,gss-group16-sha512-,gss-gex-sha1-,gss-group14-sha1- -oKexAlgorithms=curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1 -oHostKeyAlgorithms=ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com,ssh-rsa,ssh-rsa-cert-v01@openssh.com -oPubkeyAcceptedKeyTypes=ecdsa-sha2-nistp256,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521,ecdsa-sha2-nistp521-cert-v01@openssh.com,ssh-ed25519,ssh-ed25519-cert-v01@openssh.com,rsa-sha2-256,rsa-sha2-256-cert-v01@openssh.com,rsa-sha2-512,rsa-sha2-512-cert-v01@openssh.com,ssh-rsa,ssh-rsa-cert-v01@openssh.com -oCASignatureAlgorithms=ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,ssh-ed25519,rsa-sha2-256,rsa-sha2-512,ssh-rsa
```

# 进程运行时的环境变量

通过 `cat /proc/pid/environ` 可以查看进程运行时的环境变量。

```shell
$ cat /proc/2952/environ
LANG=en_US.utf8GDM_LANG=en_US.utf8GDM_SEAT_ID=seat0GNOME_SHELL_SESSION_MODE=gdmGIO_USE_VFS=localUSERNAME=gdmXDG_VTNR=1GVFS_DISABLE_FUSE=1RUNNING_UNDER_GDM=trueXDG_SESSION_ID=c1USER=gdmPWD=/var/lib/gdmHOME=/var/lib/gdmXDG_SESSION_TYPE=waylandXDG_DATA_DIRS=/usr/share/gdm/greeter:/usr/local/share/:/usr/share/GDM_VERSION=40.0DCONF_PROFILE=gdmGVFS_REMOTE_VOLUME_MONITOR_IGNORE=1SHELL=/sbin/nologinXDG_SESSION_CLASS=greeterXDG_CURRENT_DESKTOP=GNOME-Greeter:GNOMESHLVL=0XDG_SEAT=seat0LOGNAME=gdmDBUS_SESSION_BUS_ADDRESS=unix:abstract=/tmp/dbus-eEyhT1EAiD,guid=1dd0291459fa6836ce30a888644c62beXDG_RUNTIME_DIR=/run/user/42PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/binGDM_SUPPORTED_SESSION_TYPES=wayland:x11QT_IM_MODULE=ibusXMODIFIERS=@im=ibusGNOME_DESKTOP_SESSION_ID=this-is-deprecatedXDG_MENU_PREFIX=gnome-SESSION_MANAGER=local/unix:@/tmp/.ICE-unix/2154,unix/unix:/tmp/.ICE-unix/2154DISPLAY=:1024XAUTHORITY=/run/user/42/.mutter-Xwaylandauth.W7LT31WAYLAND_DISPLAY=wayland-0DESKTOP_AUTOSTART_ID=109c4695bfed9f438168272761528293600000021540012GIO_LAUNCHED_DESKTOP_FILE=/etc/xdg/autostart/org.gnome.SettingsDaemon.Smartcard.desktopGIO_LAUNCHED_DESKTOP_FILE_PID=2952
```

# cgroup

```shell
cat /proc/2952/cgroup 
12:pids:/user.slice/user-42.slice/session-c1.scope
11:blkio:/system.slice/gdm.service
10:cpu,cpuacct:/
9:devices:/system.slice/gdm.service
8:memory:/user.slice/user-42.slice/session-c1.scope
7:rdma:/
6:freezer:/
5:perf_event:/
4:net_cls,net_prio:/
3:hugetlb:/
2:cpuset:/
1:name=systemd:/user.slice/user-42.slice/session-c1.scope
```

# 进程自动分组

这些信息来自 `man sched`

自 Linux 2.6.38 以来，内核提供了称为自动分组的特性来改进面对多进程、CPU 密集型工作负载 (例如构建 Linux 内核时候的大量并行构建进程，比如 make -j) 时的交互式桌面性能。
Linux 上通过 /proc/sys/kernel/sched_autogroup_enabled 来控制进程自动分组。

man 手册举了这样一个例子：

假设有两个自动组在竞争同一个 CPU（即假设是单 CPU 系统或使用 taskset(1) 将所有进程限制在 SMP 系统的同一个 CPU 上）。
第一组包含 10 个 CPU 绑定的进程，这些进程来自用 make -j10 启动的内核构建。另一组包含一个受 CPU 约束的进程：一个视频播放器。自动分组的效果是，这两个组将各自获得一半的 CPU 周期。也就是说，视频播放器将获得 50% 的 CPU 周期，而不是只有 9% 的周期（这可能会导致视频播放质量下降）。SMP 系统上的情况更复杂，但效果是相同的：调度器将 CPU 周期分配给各任务组，这样一个包含大量 CPU 绑定进程的自动组最终不会以牺牲系统上的其他工作为代价占用 CPU 周期。

