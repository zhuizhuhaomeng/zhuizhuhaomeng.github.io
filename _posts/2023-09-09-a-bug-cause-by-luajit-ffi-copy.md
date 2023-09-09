---
layout: post
title: "一个 LuaJIT 的 ffi.copy 内存越界导致进程崩溃的问题"
description: "ffi.copy 拷贝字符串内存写越界"
date: 2023-09-09
tags: [LuaJIT, ffi.copy]
---

# 问题代码

我们 OpenResty 的 privilege 进程经常崩溃，最终发现是类似于下面这样的代码出现的问题导致的。

```lua
local src = "a string value"
local len = #src
local dst = ffi.new(len)
ffi.copy(dst, src)
```

# 导致改写的原因

如果你没有办法一眼看出来是怎么回事，那么是很正常的事情。

我们看看 `ffi.copy` 的 [接口说明](https://luajit.org/ext_ffi_api.html) 。

```text
ffi.copy(dst, src, len)
ffi.copy(dst, str)
Copies the data pointed to by src to dst. dst is converted to a "void *" and src is converted to a "const void *".

In the first syntax, len gives the number of bytes to copy. Caveat: if src is a Lua string, then len must not exceed #src+1.

In the second syntax, the source of the copy must be a Lua string. All bytes of the string plus a zero-terminator are copied to dst (i.e. #src+1 bytes).

Performance notice: ffi.copy() may be used as a faster (inlinable) replacement for the C library functions memcpy(), strcpy() and strncpy().
```

这里说得很清楚，对于字符串，使用 `ffi.copy(dst, str)` 的形式，将会拷贝字符串末尾的 '\0'。 然而，我们分配内存的时候只分配了字符串的长度而没看考虑结尾的 '\0'。

大部分情况下并不会导致内存写越界导致崩溃的问题，因为内存分配器分配内存一般都是 8 的整数倍，至少是8字节对齐的方式。比如申请分配 13 字节的内存，实际分配了 16 字节的内存。
因此，即使写越界了，很大概率也不会真的影响其它被分配的内存数据。当然，对于计算机来说还是大概率的，因为这里只有 7/8 的概率写超过1字节没有影响。

关于内存分配的块大小，可以查看[这里](https://github.com/bminor/glibc/blob/master/malloc/malloc.c#L1535)。

这种问题的困难之处在于发现崩溃的时候离实际内存改写的地方已经很远了，因此，这种内存改写的问题是非常难于排查的。
那么如果解决这种问题呢？

# 解决问题

## 第一时间发现问题

解决这种困难的问题也跟小学生解决应用题一样需要一步步来。我们首先需要让进程崩溃在实际发生问题的地方。
为了达到这个目标，我们需要使用内存检测的工具，这些工具可以是 valgrind，jemalloc，address sanity。
比如，我们用 address sanity 进行内存检测得到如下的结果。

```
=================================================================
==nginx==3848863==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x622000002c9b at pc 0x7fb63b754a2d bp 0x7ffd1034d9d0 sp 0x7ffd1034d178
WRITE of size 4996 at 0x622000002c9b thread T0
    #0 0x7fb63b754a2c  (/lib64/libasan.so.5+0x40a2c)
    #1 0x7fb63ac4eaff in lj_cf_ffi_copy /usr/src/debug/openresty-1.19.9.1.48/build/LuaJIT-2.1-20230724/src/lib_ffi.c:710
    #2 0x7fb63ab47655  (/opt/openresty-asan/luajit/lib/libluajit-5.1.so.2+0x16655)
    #3 0x7fb63ab793da in lua_resume /usr/src/debug/openresty-1.19.9.1.48/build/LuaJIT-2.1-20230724/src/lj_api.c:1272
    #4 0x565432b65b61 in ngx_http_lua_run_thread ../ngx_lua-0.10.21.11/src/ngx_http_lua_util.c:1185
    #5 0x565432bb8905 in ngx_http_lua_timer_handler ../ngx_lua-0.10.21.11/src/ngx_http_lua_timer.c:660
    #6 0x565432911118 in ngx_event_expire_timers src/event/ngx_event_timer.c:94
    #7 0x5654329107cb in ngx_process_events_and_timers src/event/ngx_event.c:275
    #8 0x56543292b2ec in ngx_privileged_agent_process_cycle src/os/unix/ngx_process_cycle.c:1327
    #9 0x565432926ff2 in ngx_spawn_process src/os/unix/ngx_process.c:207
    #10 0x56543292e600 in ngx_reap_children src/os/unix/ngx_process_cycle.c:698
    #11 0x56543292e600 in ngx_master_process_cycle src/os/unix/ngx_process_cycle.c:188
    #12 0x5654328a3d7b in main src/core/nginx.c:406
    #13 0x7fb639431cf2 in __libc_start_main (/lib64/libc.so.6+0x3acf2)
    #14 0x5654328a7c5d in _start (/opt/openresty-asan/nginx/sbin/nginx+0xe5c5d)

0x622000002c9b is located 0 bytes to the right of 5019-byte region [0x622000001900,0x622000002c9b)
allocated by thread T0 here:
    #0 0x7fb63b803ff8 in __interceptor_realloc (/lib64/libasan.so.5+0xefff8)
    #1 0x7fb63ac35c17 in mem_alloc /usr/src/debug/openresty-1.19.9.1.48/build/LuaJIT-2.1-20230724/src/lib_aux.c:348

SUMMARY: AddressSanitizer: heap-buffer-overflow (/lib64/libasan.so.5+0x40a2c) 
Shadow bytes around the buggy address:
  0x0c447fff8540: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c447fff8550: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c447fff8560: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c447fff8570: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c447fff8580: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x0c447fff8590: 00 00 00[03]fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c447fff85a0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c447fff85b0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c447fff85c0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c447fff85d0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c447fff85e0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==nginx==3848863==ABORTING
```

上面虽然得到了第一现场，但是我们还是没有很好的办法定位问题，因为不知道具体哪个 Lua 源文件的代码触发的问题。
比较土的笨办法也有，可以通过搜索 ffi.copy 去分析所有的相关代码。但是如果事先不知道 `ffi.copy` 拷贝字符串的那个深坑，看代码哪里都没有问题。

## 通过 coredump 文件定位 Lua 代码行

得到调用栈有了极大的帮助，但是像这种需要 Lua 调用栈来进一步分析问题的情况我们需要有 coredump 才能够进一步分析。
在生成 coredump 文件后，我们就可以借助 [OpenResty XRay](https://xray.openresty.com) 来进一步分析问题了。

比如本次 coredump， OpenResty XRay 给出了这样的 Lua 调用栈。

```shell
__index
[builtin#ffi.copy]
__index
@buildroot/usr/local/ai-agent/lua/resty/gpt4-engine.lua:309
__index
@buildroot/usr/local/ai-agent/lua/resty/ai/agent/route.lua:110
__index
@buildroot/usr/local/ai-agent/lua/resty/ai/agent/route.lua:243
@buildroot/usr/local/ai-agent/lua/resty/ai/agent/rpc.lua:794
```

通过这个调用栈，我们就可以明确是哪行代码出了问题。看不懂是怎么回事的时候，把 Lua 代码调用栈和 C 代码调用栈结合起来，也可以发现是字符串拷贝的问题。

lj_cf_ffi_copy 这个函数非常简短，我们可以看到对于字符串，会拷贝末尾的 '\0'。

```C
LJLIB_CF(ffi_copy)	LJLIB_REC(.)
{
  void *dp = ffi_checkptr(L, 1, CTID_P_VOID);
  void *sp = ffi_checkptr(L, 2, CTID_P_CVOID);
  TValue *o = L->base+1;
  CTSize len;
  if (tvisstr(o) && o+1 >= L->top)
    len = strV(o)->len+1;  /* Copy Lua string including trailing '\0'. */
  else
    len = (CTSize)ffi_checkint(L, 3);
  memcpy(dp, sp, len);
  return 0;
}
```

# 打个广告

解决 Bug 就像升级打怪一样，一路上要遇到各种坎坷。[OpenResty XRay](https://xray.openresty.com.cn) 就是专门为解决难题而生，不仅仅是工具更有专家指导。
