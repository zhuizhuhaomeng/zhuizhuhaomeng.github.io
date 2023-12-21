---
layout: post
title: "多线程会产生多少额外的内存开销"
description: "多线程的线程栈的内存开销影响"
date: 2023-12-22
tags: [Memory, Thread, Stack]
---

默认情况下，Linux 系统为线程调用栈分配的内存大小为 8M。

- 我们可以通过 `ulimit -s` 指令调整调用栈大小。
- 我们也可以在编程时显示指定调用栈的大小，比如下面通过 pthread_attr_setstacksize 接口设置栈大小。

```C
#include <pthread.h>
#include <stdio.h>

void* thread_function(void* arg) {
    // Thread code here
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_attr_t attr;
    size_t stacksize = 1024 * 1024;  // Set the stack size to 1MB

    // Initialize thread attributes
    pthread_attr_init(&attr);

    // Set the stack size attribute
    pthread_attr_setstacksize(&attr, stacksize);

    // Create the thread with the specified attributes
    pthread_create(&thread, &attr, thread_function, NULL);

    // Destroy the thread attributes object
    pthread_attr_destroy(&attr);

    // Wait for the thread to finish
    pthread_join(thread, NULL);

    return 0;
}
```

一般情况下，我们只会调整为比 8M 更大，而很少调整为更小。调整为比 8M 更小其实意义不大，因为 Linux 的内存是动态加载的，如果没有使用到是不会实际分配物理内存的。

比如下面的 dockerd 进程的虚拟内存为 3998160KB，而物理内存是 122404KB。

```shell
 ps aux | grep dock  
root        1782  0.1  0.1 3998160 122404 ?      Ssl  08:26   0:03 /usr/bin/dockerd --data-root /home/docker -H fd:// --containerd=/run/containerd/containerd.sock
```

那么我们如何知道一个进程的栈实际使用了多少内存呢？我们不能简单的将线程的数量乘以 8M。

我们可以使用 pmap 命令来分析一下一个进程的栈实际占用了多少物理内存。

我们以上面的 docker 进程为例子来查看一下进程的内存分布。

```shell
$ sudo pmap -xp 1782
1782:   /usr/bin/dockerd --data-root /home/docker -H fd:// --containerd=/run/containerd/containerd.sock
Address           Kbytes     RSS   Dirty Mode  Mapping
000000c000000000   22528   22528   22528 rw---   [ anon ]
000000c001600000   14336    8116    8116 rw---   [ anon ]
000000c002400000   28672       0       0 -----   [ anon ]
0000555555554000   21256   10628       0 r---- /usr/bin/dockerd
0000555556a16000   28552   22356       0 r-x-- /usr/bin/dockerd
00005555585f8000    3928    3712       0 r---- /usr/bin/dockerd
00005555589ce000   27028   19640    5708 r---- /usr/bin/dockerd
000055555a433000     740     736     452 rw--- /usr/bin/dockerd
000055555a4ec000     488     212     212 rw---   [ anon ]
00007fff04000000     132       4       4 rw---   [ anon ]
00007fff04021000   65404       0       0 -----   [ anon ]
00007fff08000000     132       4       4 rw---   [ anon ]
00007fff08021000   65404       0       0 -----   [ anon ]
00007fff0c000000     132       4       4 rw---   [ anon ]
00007fff0c021000   65404       0       0 -----   [ anon ]
00007fff10000000     132       4       4 rw---   [ anon ]
00007fff10021000   65404       0       0 -----   [ anon ]
00007fff14000000     132       4       4 rw---   [ anon ]
00007fff14021000   65404       0       0 -----   [ anon ]
00007fff18000000     132       4       4 rw---   [ anon ]
00007fff18021000   65404       0       0 -----   [ anon ]
00007fff1c000000     132       4       4 rw---   [ anon ]
00007fff1c021000   65404       0       0 -----   [ anon ]
00007fff20000000     132       4       4 rw---   [ anon ]
00007fff20021000   65404       0       0 -----   [ anon ]
00007fff24000000     132       4       4 rw---   [ anon ]
00007fff24021000   65404       0       0 -----   [ anon ]
00007fff28000000     132       4       4 rw---   [ anon ]
00007fff28021000   65404       0       0 -----   [ anon ]
00007fff2c000000     132       4       4 rw---   [ anon ]
00007fff2c021000   65404       0       0 -----   [ anon ]
00007fff34000000     132       4       4 rw---   [ anon ]
00007fff34021000   65404       0       0 -----   [ anon ]
00007fff38000000     132       4       4 rw---   [ anon ]
00007fff38021000   65404       0       0 -----   [ anon ]
00007fff3f7ff000       4       0       0 -----   [ anon ]
00007fff3f800000    8192    2048    2048 rw---   [ anon ]
00007fff40000000     132       4       4 rw---   [ anon ]
00007fff40021000   65404       0       0 -----   [ anon ]
00007fff44600000    8192    5380       0 r--s- /home/docker/buildkit/containerdmeta.db
00007fff44ffa000       4       0       0 -----   [ anon ]
00007fff44ffb000    8192       8       8 rw---   [ anon ]
00007fff457fb000       4       0       0 -----   [ anon ]
00007fff457fc000    8192       8       8 rw---   [ anon ]
00007fff45ffc000       4       0       0 -----   [ anon ]
00007fff45ffd000    8192       8       8 rw---   [ anon ]
00007fff467fd000       4       0       0 -----   [ anon ]
00007fff467fe000    8192       8       8 rw---   [ anon ]
00007fff46ffe000       4       0       0 -----   [ anon ]
00007fff46fff000    8192       8       8 rw---   [ anon ]
00007fff477ff000       4       0       0 -----   [ anon ]
00007fff47800000    8192    2048    2048 rw---   [ anon ]
...
00007ffffffde000     132      12      12 rw---   [ stack ]
ffffffffff600000       4       0       0 --x--   [ anon ]
---------------- ------- ------- ------- 
total kB         3998164  128284   64004
```

实际上这里使用 dockerd 进程是不太好的，因为 docker 是 Go 编写的程序。
Go 大量使用协程而不是普通的操作系统进程，因此 Go 的栈占用的内存应该包含协程的内存更准确。
不过我们这里只关心操作系统线程的内存了。

那么上面的哪些地址是给栈使用的呢？这个其实是很难准确知道的，除了 [ stack ] 这个明显标记为栈的范围。但是天无绝人之路，我们总是可以通过一定的特征来框定我们感兴趣的目标而又不失准确性。

这个就是根据栈大小为 8M 以及在 8M 之后会有一个 4k 的保护区域为特征来查找栈。比如：

```shell
$ sudo pmap -xp 1782
1782:   /usr/bin/dockerd --data-root /home/docker -H fd:// --containerd=/run/containerd/containerd.sock
Address           Kbytes     RSS   Dirty Mode  Mapping
00007fff457fb000       4       0       0 -----   [ anon ]
00007fff457fc000    8192       8       8 rw---   [ anon ]
```

为什么 8M 之后会有一个 4K 的保护范围呢？我们仔细观察可以知道，这个 4K 的保护范围的属性是 `----`。
也就是进程对这段内存既没有写权限，也没有读权限。同时我们可以看到这段内存是没有加载的 (RSS 为 0)。
如果程序读写内存超过 8M 的范围，到达 4K 的地方，那么操作系统就会发送读写异常的信号给进程。

我们看看这个 Go 进程有几个操作系统线程：

```shell
$ sudo ls -l /proc/1782/task | wc -l
45
```

可以看到有 45 个操作系统线程。我们看看 pmap 的结果有几个 8M的内存段。

```shell
sudo pmap -xp 1782 | grep 8192 | grep '\[ anon \]' | wc -l
39
```

可以看到这里的数量是对不上的，因此我们说 Go 程序不是很好的分析例子。

我们看看 Redis 程序的情况：

```Shell
$ ps aux | grep redis
redis       1322  0.1  0.0 276136 14500 ?        Ssl  08:26   0:03 /usr/bin/redis-server 127.0.0.1:6379
ljl        20460  0.0  0.0 221800  2332 pts/0    S+   09:17   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox redis
$ sudo pmap -xp 1322 | grep 8192 | grep '\[ anon \]'
00007fffe75fd000    8192      16      16 rw---   [ anon ]
00007fffe7dfe000    8192      12      12 rw---   [ anon ]
00007fffe85ff000    8192      12      12 rw---   [ anon ]
00007fffe8e00000    8192      12      12 rw---   [ anon ]
00007ffff6c00000    8192    5860    5860 rw---   [ anon ]
$ sudo ls -l /proc/1322/task        
total 0
dr-xr-xr-x 7 redis redis 0 Dec 21 09:18 1322
dr-xr-xr-x 7 redis redis 0 Dec 21 09:18 1458
dr-xr-xr-x 7 redis redis 0 Dec 21 09:18 1459
dr-xr-xr-x 7 redis redis 0 Dec 21 09:18 1460
dr-xr-xr-x 7 redis redis 0 Dec 21 09:18 1462
$ sudo pmap -xp 1322 | grep -B1 8192 
00007fffe75fc000       4       0       0 -----   [ anon ]
00007fffe75fd000    8192      16      16 rw---   [ anon ]
00007fffe7dfd000       4       0       0 -----   [ anon ]
00007fffe7dfe000    8192      12      12 rw---   [ anon ]
00007fffe85fe000       4       0       0 -----   [ anon ]
00007fffe85ff000    8192      12      12 rw---   [ anon ]
00007fffe8dff000       4       0       0 -----   [ anon ]
00007fffe8e00000    8192      12      12 rw---   [ anon ]
00007fffe9600000  218304     400       0 r---- /usr/lib/locale/locale-archive
00007ffff6c00000    8192    5860    5860 rw---   [ anon ]
```

我们看到这个进程总共有 5 个线程，其中一个是主线程。有 5 个 8192 的地址范围。其中 00007ffff6c00000 前面的地址并不是 4K大小的保护空间。因此这里可以把这个前面没有 4k 保护地址空间的 8M 地址范围排除。

那么如何才能准确的知道线程的地址呢，答案就是 gdb 和 pmap 结合。
下面的 gdb 命令 给出的 `Thread 0x7ffff7c38000` 这样的地址在哪个地址范围，那个地址范围对应的肯定是线程栈。并且这个范围不受调用栈大小的影响，我们不必关系调用栈是否 8M 了。

```shell
$ sudo gdb -batch -p 1322 -ex "info threads" -ex "quit"
[New LWP 1458]
[New LWP 1459]
[New LWP 1460]
[New LWP 1462]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
0x00007ffff754e84e in epoll_wait (epfd=5, events=0x7ffff7162680, maxevents=10128, timeout=100) at ../sysdeps/unix/sysv/linux/epoll_wait.c:30
30	../sysdeps/unix/sysv/linux/epoll_wait.c: No such file or directory.
  Id   Target Id                                          Frame 
* 1    Thread 0x7ffff7c38000 (LWP 1322) "redis-server"    0x00007ffff754e84e in epoll_wait (epfd=5, events=0x7ffff7162680, maxevents=10128, timeout=100) at ../sysdeps/unix/sysv/linux/epoll_wait.c:30
  2    Thread 0x7fffe95ff640 (LWP 1458) "bio_close_file"  __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555557cb408) at futex-internal.c:57
  3    Thread 0x7fffe8dfe640 (LWP 1459) "bio_aof_fsync"   __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555557cb438) at futex-internal.c:57
  4    Thread 0x7fffe85fd640 (LWP 1460) "bio_lazy_free"   __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555557cb468) at futex-internal.c:57
  5    Thread 0x7fffe7dfc640 (LWP 1462) "jemalloc_bg_thd" __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x7ffff72073b4) at futex-internal.c:57
A debugging session is active.

	Inferior 1 [process 1322] will be detached.

Quit anyway? (y or n) [answered Y; input not from terminal]
[Inferior 1 (process 1322) detached]
```

只要找到了所有的地址范围，把对应的 RSS 加起来就是所有线程栈消耗的物理内存了。
