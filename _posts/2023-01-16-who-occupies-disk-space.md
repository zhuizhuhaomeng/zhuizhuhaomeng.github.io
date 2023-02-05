---
layout: post
title: "谁占用了我的磁盘空间"
description: "告警系统报告磁盘空间不足，但是登陆机器找不到哪些文件占用的"
date: 2023-01-16
feature_image: img/Hard-Disk.png
tags: [lsof, disk, find, df, mount]
---

# 背景

有一台线上机器报警说磁盘空间不足，磁盘利用率接近95%。
同事上登陆机器查看并清理了数据库的日志文件，但是磁盘空间利用率并没有明显变化。
我接手分析该问题，刚开始也是按照常规的思路分析，发现有一些备份文件占用了绝大多数的磁盘空间，
因此给出了将备份数据转移到其它机器的建议。 但是同时我还发现 `du -hs` 统计的磁盘空间和实际的
磁盘空间利用率差别比较大，因此就不得不深入分析到底谁还占用了磁盘空间。

# 常规的排查步骤

## 确认对应的磁盘

首先，通过 `df -hl` 确认是哪个磁盘的利用率比较高。
比如，下面的命令可以看到根分区对应的磁盘 CPU 利用率是比较高的, 对应的磁盘利用率达到 69%,
该磁盘有 70G的空间，使用了 48G, 剩下 23G。（这个不是问题的机器）

```shell
$ df -l
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs              16G     0   16G   0% /dev
tmpfs                 16G     0   16G   0% /dev/shm
tmpfs                 16G   50M   16G   1% /run
tmpfs                 16G     0   16G   0% /sys/fs/cgroup
/dev/mapper/rl-root   70G   48G   23G  69% /
/dev/vda1           1014M  360M  655M  36% /boot
/dev/mapper/rl-home  126G   61G   66G  49% /home
tmpfs                3.2G   12K  3.2G   1% /run/user/42
tmpfs                3.2G  4.0K  3.2G   1% /run/user/1000
```

## 统计目录占用的磁盘空间

如果磁盘分区加载到特定的目录，那么一般就可以从该特定目录开始逐层递进的统计。
如果是根分区，那么需要注意排除相关的非磁盘目录以及其它的磁盘的目录。
以上面的目录为例，/home 是另外的一个磁盘，因此有必要将该目录排除。
我们使用下面的命令来分析目录的空间占用。注意用root权限，否则有些目录没有权限会统计不到。

```shell
$ cd /
$ find . -maxdepth 1 -type d -not -path "." -not -path "./home" -not -path "./lost+found" -not -path "./dev" -not -path "./proc"  | xargs -n1 du -hs | sort -hk1
0	./cores
0	./media
0	./mnt
0	./srv
0	./sys
12M	./tmp
33M	./etc
50M	./run
99M	./opt
320M	./boot
1.2G	./root
19G	./usr
28G	./var
```

我们可以看到磁盘总的空间使用大概是 (1.2 + 19 + 28 + 0.32 + 0.09 + 0.05)G = 48.6G.
这个数值和 df -hl 显示的 48G的磁盘使用是相符合的。

另外，我们可以看到目录占用最大的是 /var。因此，我们可以进一步的分析 /var 下的子目录哪个占用最大。
我们接着使用下面这样的命令来分析 /var 下的子目录的空间占用大小。

``` shell
find . -maxdepth 1 -type d -not -path "." | xargs -n1 du -hs | sort -hk1
0	./account
0	./adm
0	./crash
0	./empty
0	./ftp
0	./games
0	./opt
12K	./spool
70M	./log
86M	./tmp
236M	./cache
4.9G	./lib
23G	./code
```

上面的分析过程就是逐层递进的过程。当然在分析目录的同时，也应该使用 `ls -hl` 看看，免得
有些特别大的文件不在子目录底下而错过了。

因为线上机器的磁盘大小是100G，磁盘利用率是95%，因此应该是 95G 的磁盘被使用了。但是 `du -hs` 统计发现
磁盘对应的加载目录只有 76G 的空间, 跟 95G 的磁盘空间差的有点远。

#  分析丢失的磁盘空间

如果查询 2~3G 甚至是 5G 的空间，那么我也不会深究的。但是相差太大，我觉得肯定是哪里有问题。
如果你压力很大，需要赶快给个结论，那么你可能会胡思乱想，甚至认为（向上报告）可能是统计工具错了。

但是我们按照正常的思维来考虑，应该先假设工具给出的数据都是准确的，毕竟这些都是经过千锤百炼的工具。
这时候我们就需要分析为什么两个工具给出的磁盘空间统计差别那么大。`df` 这个工具是直接根据磁盘的分配情况
展示数据的，因此就是磁盘空间被使用了那么多，没有商量的余地。而 `du` 这个工具则根据是统计目录使用的磁盘
空间大小，也就是说只能通过目录能够查找到的文件才能被 `du` 统计到。

因此，按照正常的思维我们就要问：为什么磁盘空间被使用了，但是通过目录却统计不到呢？这时候有经验的工程师
就会想到文件被删除了，但是空间却没有归还给文件系统。

我们进一步的提问：什么情况下文件删除了却不把空间规划给操作系统呢？有了正确的问题，你通过 google 很快就可以找打答案了。
当然，如果直接提问： 为什么 `du -hs` 和 `df -l` 统计的磁盘空间差别很大也是可以很快找到答案的，只不过缺少了自己能够推理的过程。

比如我搜索到了这个：https://access.redhat.com/solutions/2316#:~:text=On%20Linux%20or%20Unix%20systems,to%20occupy%20space%20on%20disk 以及
 https://serverfault.com/questions/232525/df-in-linux-not-showing-correct-free-space-after-file-removal 都给出了很好的解答。


我喜欢用这个命令, 加上 -P 选项可以加速 lsof的过程。这样在输出socket的时候就不需要将端口转换为名字了。

```
lsof -P | grep -i deleted
```

# lsof 是去哪里把删除的文件找回来的

linux 的 proc 文件系统下有每一个进程的信息。比如我们可以通过下面这个例子查看

```shell
ls -l /proc/2322384/fd
total 0
lrwx------ 1 nobody nobody 64 Jan 16 17:20 0 -> /dev/null
lrwx------ 1 nobody nobody 64 Jan 16 17:20 1 -> /dev/null
lrwx------ 1 nobody nobody 64 Jan 16 17:20 10 -> 'socket:[4368249]'
lrwx------ 1 nobody nobody 64 Jan 16 17:20 11 -> 'anon_inode:[eventpoll]'
lrwx------ 1 nobody nobody 64 Jan 16 17:20 12 -> 'anon_inode:[eventfd]'
lrwx------ 1 nobody nobody 64 Jan 16 17:20 13 -> 'anon_inode:[signalfd]'
l-wx------ 1 nobody nobody 64 Jan 16 17:20 2 -> '/usr/local/openresty/nginx/logs/error.log.bak (deleted)'
l-wx------ 1 nobody nobody 64 Jan 16 17:20 4 -> '/usr/local/openresty/nginx/logs/error.log.bak (deleted)'
l-wx------ 1 nobody nobody 64 Jan 16 17:20 5 -> /usr/local/openresty/nginx/logs/access.log
lrwx------ 1 nobody nobody 64 Jan 16 17:20 6 -> 'socket:[4373797]'
lrwx------ 1 nobody nobody 64 Jan 16 17:20 7 -> 'socket:[4373798]'
lrwx------ 1 nobody nobody 64 Jan 16 17:20 8 -> 'socket:[4376712]'
lr-x------ 1 nobody nobody 64 Jan 16 17:20 9 -> /var/lib/sss/mc/initgroups
```

lsof是如何得到删除文件的大小呢？其实也很简单，就是通过 proc 文件系统。使用 `ls -lL` 或者 `stat -L` 即可。

```shell
$ ls -l /proc/2322384/fd/2
l-wx------ 1 nobody nobody 64 Jan 16 17:20 /proc/2322384/fd/2 -> '/usr/local/openresty/nginx/logs/error.log.bak (deleted)'
$ ls -lL /proc/2322384/fd/2
-rw-r--r--. 0 root root 43313576 Jan 15 16:14 /proc/2322384/fd/2
$ stat -L /proc/2322384/fd/2
  File: /proc/2322384/fd/2
  Size: 43313576  	Blocks: 84600      IO Block: 4096   regular file
Device: fd00h/64768d	Inode: 136744064   Links: 0
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2023-01-16 18:24:50.566075570 +0800
Modify: 2023-01-15 16:14:59.437061693 +0800
Change: 2023-01-16 17:22:12.600638923 +0800
 Birth: 2022-11-16 14:21:39.730742473 +0800
```

# 如何恢复被删除的文件

Linux上没有像 windows 上一样的回收站，如果文件已经被删除那么真是一件糟糕的事情。
但是有时候我们也能够快速恢复被删除的文件，比如上面案例中的文件虽然被删除了，但是其实
文件却还被 /proc/pid/fd/ 所引用。

这个时候我们可以通过 cat /proc/pid/fd/xxxx > /path-to-recovery/file 这样的方式将
文件读取出来达到恢复文件的目的。比如:

```shell
cat /proc/1786/fd/6 > /home/ljl/error.log
```
