---
layout: post
title: "How to dump a process memory"
description: "Dig into the process memory"
date: 2024-05-25
tags: [Linux, Process Memory]
---

By the time we realize the problem, it is already happening. 
If we want to know what caused the problem, we usually need to understand how it happened. 
It's kind of like the chicken or the egg came first question.

Linux process memory leaks are such a situation.
We want to solve memory leaks we need to know how memory leaks.
But the memory leak has already happened and we don't know which code path caused it.

How can we better locate such memory leaks?

One way to do this is to reverse-analyze which module is using the data by looking at what data is loaded in memory.

How do I get memory data for a Linux process?
The Linux proc system provides an interface, /proc/pid/mem.

We can use the tool dd to read the memory.

Let's get the map range first.

```shell
$ cat /proc/1753/maps
7fffc3580000-7fffc3590000 r-xp 00000000 00:00 0 
7fffed7f0000-7fffed7f1000 ---p 00000000 00:00 0 
7fffed7f1000-7fffedff1000 rw-p 00000000 00:00 0 
7fffedff1000-7fffedff2000 ---p 00000000 00:00 0 
7fffedff2000-7fffee7f2000 rw-p 00000000 00:00 0 
7fffee7f2000-7fffee7f3000 ---p 00000000 00:00 0 
7fffee7f3000-7fffeeff3000 rw-p 00000000 00:00 0 
7fffeeff3000-7fffeeff4000 ---p 00000000 00:00 0 
7ffff7fc7000-7ffff7fc9000 r--p 00000000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7fc9000-7ffff7fef000 r-xp 00002000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
7ffff7fef000-7ffff7ffa000 r--p 00028000 fd:00 67442267                   /usr/lib64/ld-linux-x86-64.so.2
...
```

And then dump the memory using the `dd` tool.

```shell
dd if/proc/pid/mem bs=4096 iflag=skip_bytes,count_bytes skip=$((0x7fffc3580000)) count=$((0x7fffc3590000 - 0x7fffc3580000)) of=7fffc3580000.bin
```

Typically, we analyze more than one memory range, so we use the following script to dump all memory ranges.
But since we don't care about the memory fragment of the library, we filter out the library.

```shell
procdump() 
( 
    cat /proc/$1/maps | grep "rw-p" | grep -v so | awk '{print $1}' | ( IFS="-"
    while read a b; do
        dd if=/proc/$1/mem bs=$( getconf PAGESIZE ) iflag=skip_bytes,count_bytes \
           skip=$(( 0x$a )) count=$(( 0x$b - 0x$a )) of="$1_mem_$a.bin"
    done )
)
procdump pid_of_process
```

After dumping the memory, we can use the `hexdump` to view the data.

```shell
hexdump -C the-data.bin > the-data.hex
```

You may also use the `strings` if there are special string in the memory.

```shell
strings the-data.bin
```

Good luck!
