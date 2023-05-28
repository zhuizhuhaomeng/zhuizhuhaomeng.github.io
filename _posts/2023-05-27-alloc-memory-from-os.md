---
layout: post
title: "Get Memory from OS and Put Memory back to OS"
description: ""
date: 2023-05-27
tags: [brk, sbrk, mmap, munmap, madvise]
---

# 概述

一般情况下，C 语言申请和释放内存是使用 malloc, calloc, free 这些接口。
而这些接口是 libc/tcmalloc/jemalloc 等内存分配器提供的接口，而不是操作系统提供的接口。
那么像 libc/tcmalloc/jemalloc 等这些内存分配器的实现是如何向操作系统申请内存的呢？

libc 主要使用的是 brk/sbrk 这个接口，而对于 tcamlloc 和 jemalloc 则主要使用的是 mmap/munmap 的接口。

# brk/sbrk 接口的实验

brk/sbrk 的函数前面如下：

```C
int brk(void *addr);
void *sbrk(intptr_t increment);
```

通过 man 手册我们可以看到这两个接口的描述如下：

```man
brk() and sbrk() change the location of the program break, which defines the end of the process's data seg‐
ment (i.e., the program break is the first location after the  end  of  the  uninitialized  data  segment).
Increasing the program break has the effect of allocating memory to the process; decreasing the break deal‐
locates memory.

brk() sets the end of the data segment to the value specified by addr, when that value is  reasonable,  the
system has enough memory, and the process does not exceed its maximum data size (see setrlimit(2)).

sbrk()  increments  the program's data space by increment bytes.  Calling sbrk() with an increment of 0 can
be used to find the current location of the program break.
```

也就是说 brk()和sbrk() 通过改变程序中断(program break)的位置来实现内存的分配和释放。这里所谓的程序中断是指进程的数据段的截止地址。(即，程序中断是未初始化数据段结束后的第一个位置）。增加程序中断的效果是为进程分配内存；减少中断的效果是释放进程分配内存。

brk() 将数据段的末端设置为addr所指定的值， sbrk()将程序的数据空间以增量字节的方式增加。 
在增量为 0 的情况下调用 sbrk() 可以 可以用来查找程序中断的当前位置。

我们通过下面这个 C 程序来模拟向操作系统申请内存。

```C
#include <stdio.h>
#include <unistd.h>


int
main(int argc, char *argv[])
{
    void* c1 = sbrk(0);
    fprintf(stderr, "program break address: %p\n", c1);
    c1 = (void*) ((char*) c1 + 128);
    fprintf(stderr, "c1: %p\n", c1);
    brk(c1);
    void* c2 = sbrk(0);
    fprintf(stderr, "program break address: %p\n", c2);
    return 0;
}
```

编译并执行上面的程序，可以得到类似如下的结果：

```shell
$ gcc -Wall brk.c
$ ./a.out                               
program break address: 0x54f000
c1: 0x54f080
program break address: 0x54f080
```

上面只是观察到 program break address 变化了，那么如何确认是真的向系统申请了内存了呢？
为了观察系统调用，我们可以使用 strace 这个工具。

```shell
strace ./a.out 2>&1 | grep -E "(brk|program break)"
brk(NULL)                               = 0x518000
brk(NULL)                               = 0x518000
write(2, "program break address: 0x518000\n", 32program break address: 0x518000
write(2, "brk c1: 0x518080\n", 17brk c1: 0x518080
brk(0x518080)                           = 0x518080
brk(NULL)                               = 0x518080
write(2, "program break address: 0x518080\n", 32program break address: 0x518080
```

通过 strace 我们观察到，实际上调用操作系统的结果是 brk，并没有 sbrk 这个接口。
而接口的参数符合我们的 C 代码设置的预期，增加了 80 个字节。

而操作系统是按页分配内存的，我们如何观察实际上操作系统分配的多少虚拟内存呢？
这个时候我们可以使用外部程序来查看 /proc/pid/maps 这个接口的数据。为了方便，
我们让程序自己输出自己当前的 maps 文件。当然这个有点 tricky，有可能造成不准确的情况。
因为如果我们输出的接口设计内存分配，那么在调用之前和调用之后可能就是不一样的值了。

我们使用如下的 C 代码进行分析：

```C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void
print_file_map(void)
{
    FILE    *fp;
    char    *line;
    int      len;
    char     line_buf[512];
     
    fp = fopen("/proc/self/maps", "r");
    if(fp == NULL) {
        fprintf(stderr, "failed to open /proc/self/maps\n");
        return;
    }

    while (1) {
        len = sizeof(line_buf);
        line = fgets(line_buf, len, fp);
        if (line == NULL) {
            break;
        }

        fprintf(stderr, "%s", line);
    }

    fprintf(stderr, "\n");

    fclose(fp);
}


int
main(int argc, char *argv[])
{
    void* c1 = sbrk(0);
    fprintf(stderr, "program break address: %p\n", c1);
    print_file_map();

    c1 = (void*) ((char*) c1 + 128);
    fprintf(stderr, "c1: %p\n", c1);
    brk(c1);
    void* c2 = sbrk(0);
    fprintf(stderr, "program break address: %p\n", c2); 
    print_file_map();

    return 0;
}
```

编译上述的代码并执行得到如下的结果:

```shell
$ gcc -Wall a.c
$ ./a.out                               
program break address: 0x1254000
00400000-00401000 r--p 00000000 fd:02 6598                               /home/ljl/a.out
00401000-00402000 r-xp 00001000 fd:02 6598                               /home/ljl/a.out
00402000-00403000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00403000-00404000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00404000-00405000 rw-p 00003000 fd:02 6598                               /home/ljl/a.out
01254000-01275000 rw-p 00000000 00:00 0                                  [heap]
7f94e0c5a000-7f94e0e16000 r-xp 00000000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e0e16000-7f94e1016000 ---p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e1016000-7f94e101a000 r--p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e101a000-7f94e101c000 rw-p 001c0000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e101c000-7f94e1020000 rw-p 00000000 00:00 0 
7f94e1020000-7f94e104d000 r-xp 00000000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f94e123b000-7f94e123d000 rw-p 00000000 00:00 0 
7f94e124d000-7f94e124e000 r--p 0002d000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f94e124e000-7f94e1250000 rw-p 0002e000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7ffd55e7d000-7ffd55e9e000 rw-p 00000000 00:00 0                          [stack]
7ffd55ff6000-7ffd55ffa000 r--p 00000000 00:00 0                          [vvar]
7ffd55ffa000-7ffd55ffc000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

brk c1: 0x1254080
program break address: 0x1254080
00400000-00401000 r--p 00000000 fd:02 6598                               /home/ljl/a.out
00401000-00402000 r-xp 00001000 fd:02 6598                               /home/ljl/a.out
00402000-00403000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00403000-00404000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00404000-00405000 rw-p 00003000 fd:02 6598                               /home/ljl/a.out
01254000-01255000 rw-p 00000000 00:00 0                                  [heap]
7f94e0c5a000-7f94e0e16000 r-xp 00000000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e0e16000-7f94e1016000 ---p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e1016000-7f94e101a000 r--p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e101a000-7f94e101c000 rw-p 001c0000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f94e101c000-7f94e1020000 rw-p 00000000 00:00 0 
7f94e1020000-7f94e104d000 r-xp 00000000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f94e123b000-7f94e123d000 rw-p 00000000 00:00 0 
7f94e124d000-7f94e124e000 r--p 0002d000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f94e124e000-7f94e1250000 rw-p 0002e000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7ffd55e7d000-7ffd55e9e000 rw-p 00000000 00:00 0                          [stack]
7ffd55ff6000-7ffd55ffa000 r--p 00000000 00:00 0                          [vvar]
7ffd55ffa000-7ffd55ffc000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

我们对比两次执行得到的 heap 数据如下:

```text
01254000-01275000 rw-p 00000000 00:00 0                                  [heap]
01254000-01255000 rw-p 00000000 00:00 0                                  [heap]
```

可以看到虚拟内存的变化是 01275000 -> 01255000。可见虚拟内存不但没有变大反而变小。
这个是怎么回事呢？实际上这个就是我们上面说的，brk/sbrk 是根据传入参数来扩大或者收缩内存的结果。
因为传入的虚拟地址比原来的虚拟地址要小，因此导致收缩内存了。

我们调用 brk(0x1254080) 这个地址不是页面的整数倍，因此需要向上取证得到页面的整数倍。
我们可以这么计算最终的地址： 0x1254080 + page_size - 1 & ~(page_size - 1)。
我的内存页面地址大小是 4096，因此计算得到的结果就是：01255000。

可以在Python中这么计算: `hex((0x1254080 + 4095) & ~(4095))`。

我们需要注意的是： brk 方式申请内存有个缺点是申请的内存只有顶部的内存被释放后，内部的内存才能跟着释放。
比如：申请了编号为 1， 2， 3， 4， 5, 6的页面。那么假设我们 2， 3， 4 已经不需要的，但是也 5， 6确是一直需要的，
那么我们没有办法通过 brk 的方式来将页面 2， 3， 4 的物理内存给操作系统。

# mmap/munmap 的结果实验

mmap 的接口申请内存有很多的参数组合，不同的参数组合的意义完全不一样。比如是否是映射到文件，是否跨进程共享等等。
我们这里重点关注的是给 malloc、free 使用的 mmap的调用。因此这些内存是不能跨进程共享，不能映射到文件的。
这种申请得到的内存也就是我们会经常见到的匿名映射页, 在 /proc/pid/maps 的最后字段是空的，不行其它的 so 文件会线上文件名。

```C
#include <stdio.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/mman.h>

void
print_file_map(void)
{
    FILE    *fp;
    char    *line;
    int      len;
    char     line_buf[512];
     
    fp = fopen("/proc/self/maps", "r");
    if(fp == NULL) {
        fprintf(stderr, "failed to open /proc/self/maps\n");
        return;
    }

    while (1) {
        len = sizeof(line_buf);
        line = fgets(line_buf, len, fp);
        if (line == NULL) {
            break;
        }

        fprintf(stderr, "%s", line);
    }
     
    fprintf(stderr, "\n");
    fclose(fp);
}

int
main(int argc, char **argv)
{

    int  N = 5;

    print_file_map();
    int *ptr = mmap(NULL, N * sizeof(int), PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS, 0, 0);

    if (ptr == MAP_FAILED) {
        fprintf(stderr, "Mapping Failed\n");
        return 1;
    }

    fprintf(stderr, "Mapping address: %p\n", ptr);
    print_file_map();

    for (int i = 0; i < N; i++)
        ptr[i] = i * 10;

    int err = munmap(ptr, 10 * sizeof(int));
    if (err != 0) {
        fprintf(stderr, "UnMapping Failed\n");
        return 1;
    }

    print_file_map();

    return 0;
}
```

编译以上程序并执行得到如下的结果:

```shell
00400000-00401000 r--p 00000000 fd:02 6598                               /home/ljl/a.out
00401000-00402000 r-xp 00001000 fd:02 6598                               /home/ljl/a.out
00402000-00403000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00403000-00404000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00404000-00405000 rw-p 00003000 fd:02 6598                               /home/ljl/a.out
017cf000-017f0000 rw-p 00000000 00:00 0                                  [heap]
7f46dc68c000-7f46dc848000 r-xp 00000000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dc848000-7f46dca48000 ---p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca48000-7f46dca4c000 r--p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4c000-7f46dca4e000 rw-p 001c0000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4e000-7f46dca52000 rw-p 00000000 00:00 0 
7f46dca52000-7f46dca7f000 r-xp 00000000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc6d000-7f46dcc6f000 rw-p 00000000 00:00 0 
7f46dcc7f000-7f46dcc80000 r--p 0002d000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc80000-7f46dcc82000 rw-p 0002e000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7ffeb1755000-7ffeb1776000 rw-p 00000000 00:00 0                          [stack]
7ffeb17f9000-7ffeb17fd000 r--p 00000000 00:00 0                          [vvar]
7ffeb17fd000-7ffeb17ff000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

Mapping address: 0x7f46dcc7e000
00400000-00401000 r--p 00000000 fd:02 6598                               /home/ljl/a.out
00401000-00402000 r-xp 00001000 fd:02 6598                               /home/ljl/a.out
00402000-00403000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00403000-00404000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00404000-00405000 rw-p 00003000 fd:02 6598                               /home/ljl/a.out
017cf000-017f0000 rw-p 00000000 00:00 0                                  [heap]
7f46dc68c000-7f46dc848000 r-xp 00000000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dc848000-7f46dca48000 ---p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca48000-7f46dca4c000 r--p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4c000-7f46dca4e000 rw-p 001c0000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4e000-7f46dca52000 rw-p 00000000 00:00 0 
7f46dca52000-7f46dca7f000 r-xp 00000000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc6d000-7f46dcc6f000 rw-p 00000000 00:00 0 
7f46dcc7e000-7f46dcc7f000 rw-p 00000000 00:00 0 
7f46dcc7f000-7f46dcc80000 r--p 0002d000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc80000-7f46dcc82000 rw-p 0002e000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7ffeb1755000-7ffeb1776000 rw-p 00000000 00:00 0                          [stack]
7ffeb17f9000-7ffeb17fd000 r--p 00000000 00:00 0                          [vvar]
7ffeb17fd000-7ffeb17ff000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

00400000-00401000 r--p 00000000 fd:02 6598                               /home/ljl/a.out
00401000-00402000 r-xp 00001000 fd:02 6598                               /home/ljl/a.out
00402000-00403000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00403000-00404000 r--p 00002000 fd:02 6598                               /home/ljl/a.out
00404000-00405000 rw-p 00003000 fd:02 6598                               /home/ljl/a.out
017cf000-017f0000 rw-p 00000000 00:00 0                                  [heap]
7f46dc68c000-7f46dc848000 r-xp 00000000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dc848000-7f46dca48000 ---p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca48000-7f46dca4c000 r--p 001bc000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4c000-7f46dca4e000 rw-p 001c0000 fd:00 67815250                   /usr/lib64/libc-2.28.so
7f46dca4e000-7f46dca52000 rw-p 00000000 00:00 0 
7f46dca52000-7f46dca7f000 r-xp 00000000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc6d000-7f46dcc6f000 rw-p 00000000 00:00 0 
7f46dcc7f000-7f46dcc80000 r--p 0002d000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7f46dcc80000-7f46dcc82000 rw-p 0002e000 fd:00 67789606                   /usr/lib64/ld-2.28.so
7ffeb1755000-7ffeb1776000 rw-p 00000000 00:00 0                          [stack]
7ffeb17f9000-7ffeb17fd000 r--p 00000000 00:00 0                          [vvar]
7ffeb17fd000-7ffeb17ff000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

可以看到，新增的虚拟内存段为 `7f46dcc7e000-7f46dcc7f000 rw-p 00000000 00:00 0`。
虽然我们申请的只有5个整数的大小，但是因为操作系统是按页分配内存的，因此这段内存的大小为一个页面的大小：4k。

# 释放物理内存

上面我们介绍了内存申请的接口，这些接口申请到的其实还是虚拟内存，并不是物理内存。只有当向这些虚拟内存读取
写入的时候操作系统才会真正分配物理内存。而物理内存是非常报告的，因此我们就不能长时间持有物理内存而不使用。

前面提到释放内存可以用 brk /munmap 的接口。这些接口在释放物理内存的时候也会同时释放虚拟内存。
如果上面的接口不能满足我们的要求，那么还可以使用 madvise 接口来释放物理内存。
比如： brk 申请内存的时候不断的把 program break 往高地址推，但是高地址的内存没有释放，而低地址的内存释放了。
这个时候就不能通过 brk 的方式来释放内存，而这时候使用 madvise 结果就是个很好的选项。

madvise 的函数签名如下：

```C
int madvise(void *addr, size_t length, int advice);
```

如果我们要释放内存，那么应该告诉操作系统指定的内存不需要了，advice 参数的值为 MADV_DONTNEED。

摘录一个来自 man7 的例子 https://man7.org/tlpi/code/online/dist/vmem/madvise_dontneed.c

```C
#ifdef __linux__
#define _BSD_SOURCE
#endif
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"

#define MAP_SIZE 4096
#define WRITE_SIZE 10

int
main(int argc, char *argv[])
{
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file\n", argv[0]);

    setbuf(stdout, NULL);

    unlink(argv[1]);
    int fd = open(argv[1], O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd == -1)
        errExit("open");

    for (int j = 0; j < MAP_SIZE; j++)
        write(fd, "a", 1);
    if (fsync(fd) == -1)
        errExit("fsync");
    close(fd);

    fd = open(argv[1], O_RDWR);
    if (fd == -1)
        errExit("open");

    char *addr = mmap(NULL, MAP_SIZE, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED)
        errExit("mmap");

    printf("After mmap:          ");
    write(STDOUT_FILENO, addr, WRITE_SIZE);
    printf("\n");

    /* Copy-on-write semantics mean that the following modification
       will create private copies of the pages for this process */

    for (int j = 0; j < MAP_SIZE; j++)
        addr[j]++;

    printf("After modification:  ");
    write(STDOUT_FILENO, addr, WRITE_SIZE);
    printf("\n");

    /* After the following, the mapping contents revert to the original file
       contents (if MADV_DONTNEED has destructive semantics, as on Linux) */

    if (madvise(addr, MAP_SIZE, MADV_DONTNEED) == -1)
        errExit("madvise");

    printf("After MADV_DONTNEED: ");
    write(STDOUT_FILENO, addr, WRITE_SIZE);
    printf("\n");

    exit(EXIT_SUCCESS);
}
```
