---
layout: post
title: "Glibc Memory Allocation Tune"
description: "How to tune the Glibc Allocation parameters."
date: 2022-01-05
feature_image: img/road.jpg
tags: [glibc, memory, malloc, free, performance]
---

<2.26
export MALLOC_ARENA_MAX=1

>=2.26
GLIBC_TUNABLES=glibc.malloc.mmap_max=1:glibc.malloc.top_pad=1


# 参考资料

https://sourceware.org/glibc/wiki/MallocInternals
https://www.gnu.org/software/libc/manual/html_node/Malloc-Tunable-Parameters.html
https://lpc.events/event/11/contributions/1006/attachments/857/1626/limitations_malloc.pdf
https://devcenter.heroku.com/articles/tuning-glibc-memory-behavior#what-value-to-choose-for-malloc_arena_max
https://devcenter.heroku.com/articles/testing-cedar-14-memory-use
