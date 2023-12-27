---
layout: post
title: "使用 jemalloc 检测内存问题"
description: "内存改写，重复释放，访问已释放内存"
date: 2023-04-06
tags: [Jemalloc, Memory]
---

Jemalloc 虽然也可以用来检测内存重复释放，访问后还使用的问题，不够如果 AddressSanitizer
和 Valgrind 可以使用的情况下，还是应该优先使用他们两个。

# Jemalloc 检测内存问题

1. 首先我们要确保编译的 jemalloc 是带有 --enable-debug 选项的。
2. 其次，运行的时候应该加上 MALLOC_CONF=tcache:false 选项，这样才能够尽快的检测到错误。
3. 否则这个内存缓存在线程缓存，还是处于已经分配的状态。
再次，我们可以让分配的内存填充随机的数据，这样更容易暴露问题，可以加上运行选项 MALLOC_CONF=junk:true 达到该目的。

也就是说，我们应该用 MALLOC_CONF=tcache:false,junk=true 这样的选项来运行。

举例如下

```shell
$ export MALLOC_CONF=tcache:false,junk:true
$ export LD_PRELOAD=/usr/local/lib/libjemalloc.so
$ ls .
```

## glibc 的类似选项

glibc 也支持运行时的检查和填充随机数，举例如下。

```shell
export MALLOC_CHECK_=3
export MALLOC_PERTURB_=99
ls /home/
```

详情参考 [glibc 手册](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html)


# 参考链接

https://github.com/jemalloc/jemalloc/wiki/Use-Case%3A-Find-a-memory-corruption-bug
