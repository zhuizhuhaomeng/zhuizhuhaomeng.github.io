---
layout: post
title: "为什么 CURL 没有收到响应"
description: "在异步框架中通过系统调用同步调用 CURL 的诡异现象"
date: 2021-06-25
tags: [OpenResty, Curl, Test]
---

# 问题的由来

在写测试用例的时候，需要指定源 IP 地址发起连接，这时候想到用 curl 的 --interface 来指定源 IP 地址。

以下是 OpenResty 中的 Lua 通过 os.execute 代码执行 curl 的命令

``` lua
         content_by_lua_block {
             os.execute("curl --interface 127.0.0.100 http://127.0.0.1:1985")
         }
```

但是测试用例出现了超时的失败了。通过抓包发现连接正常建立，发出去的数据也是符合预期，就是服务端不响应。
为什么服务端会出现不响应的问题呢？

# 问题排查

这个测试其实是在 OpenResty 中调用 curl 请求 OpenResty 自身。

服务器不响应，最直接的方式应该是查看服务器当前在做什么事情，为什么会出现不响应的情况。
但是当前没有直接这么排查，执行了一些其它的命令也非常的有趣。我们来看看这些命令的结果。

使用 netstat 查看发现连接是正常建立，就是不结束。第三行的 78 表示 socket 里面有 78 个字节，但是并没有被应用层接收处理。

``` shell
[ljl@localhost lua-conf-nginx-module]$ sudo netstat -tplna | grep 1985
tcp        1      0 0.0.0.0:1985            0.0.0.0:*               LISTEN      4167/nginx
tcp       78      0 127.0.0.1:1985          127.0.0.100:50969       ESTABLISHED -
tcp        0      0 127.0.0.100:50969       127.0.0.1:1985          ESTABLISHED 4168/curl
```

使用 ss 查看，那么发现 1985 端口也被 curl 监听，跟上面的只属于 nginx 在监听是不一样的。


``` shell
[ljl@localhost lua-conf-nginx-module]$ ss -tplna | grep 1985
LISTEN   1   128   0.0.0.0:1985       0.0.0.0:*   users:(("curl",pid=4168,fd=7),("nginx",pid=4167,fd=7))
ESTAB    78   0    127.0.0.1:1985     127.0.0.100:50969
ESTAB    0    0    127.0.0.100:50969  127.0.0.1:1985    users:(("curl",pid=4168,fd=6))
```

使用 lsof 查看 curl 进程，可以看到 curl 监听了好几个端口，同时有一个 ESTABLISHED 的链接。

``` shell
[ljl@localhost lua-conf-nginx-module]$ lsof -P -n -p 4168 | grep TCP
lsof: WARNING: can't stat() tracefs file system /sys/kernel/debug/tracing
      Output information may be incomplete.
curl    4168  ljl    6u     IPv4  88560      0t0        TCP 127.0.0.100:50969->127.0.0.1:1985 (ESTABLISHED)
curl    4168  ljl    7u     IPv4  85961      0t0        TCP *:1985 (LISTEN)
curl    4168  ljl    8u     IPv4  85962      0t0        TCP 127.0.0.1:64321 (LISTEN)
curl    4168  ljl    9u     IPv4  85963      0t0        TCP *:1984 (LISTEN)
```

使用 lsof 查看 nginx 进程

``` shell
[ljl@localhost lua-conf-nginx-module]$ lsof -P -n -p 4167 | grep TCP
lsof: WARNING: can't stat() tracefs file system /sys/kernel/debug/tracing
      Output information may be incomplete.
nginx   4167  ljl    7u     IPv4  85961      0t0        TCP *:1985 (LISTEN)
nginx   4167  ljl    8u     IPv4  85962      0t0        TCP 127.0.0.1:64321 (LISTEN)
nginx   4167  ljl    9u     IPv4  85963      0t0        TCP *:1984 (LISTEN)
nginx   4167  ljl   12u     IPv4  84930      0t0        TCP 127.0.0.1:1984->127.0.0.1:33950 (CLOSE_WAIT)
```

问题 1：为什么 curl 会监听原本属于 nginx 的端口呢？

这里为什么 curl 会监听 1985 端口呢？其实这个是因为 curl 是 OpenResty 进程通过 os.execute 执行导致的。
也是就是说 curl 进程是 OpenResty/Nginx 进程 fork 出来的，所以会继承 OpenResty/Nginx 的 fd。
因为 OpenResty/Nginx 没有设置 listen fd 的 FD_CLOEXEC/O_CLOEXEC 的属性，所以这些 listen fd 就最终出现在 curl 的 fd 中了。

问题 2：为什么连接建立成功了，但是却没有被应用层处理？

内核应该是会通知 nginx 和 curl，而 curl 显然不会去 accept，因此就是 nginx 不处理导致的。为什么 nginx 不去 accept 呢？那就看看 nginx 在干嘛吧！

通过 gdb 回溯堆栈问题就一目了然。因为 nginx 以同步阻塞的方式等待 os.execute 执行 curl 命令的结束；而 curl 正在等待 nginx 给应答。这就造成了死循环。

```
(gdb) bt
#0  0x00007fe6a9102cdb in __GI___waitpid (pid=5389, stat_loc=stat_loc@entry=0x7fffd73fde98, options=options@entry=0) at ../sysdeps/unix/sysv/linux/waitpid.c:30
#1  0x00007fe6a907e92f in do_system (line=<optimized out>) at ../sysdeps/posix/system.c:149
#2  0x00007fe6a907ed2e in __libc_system (line=<optimized out>) at ../sysdeps/posix/system.c:185
#3  0x00007fe6aa873aca in lj_cf_os_execute (L=0x7fe6ab016780) at lib_os.c:52
#4  0x00007fe6aa804a7d in lj_BC_FUNCC () from /usr/local/lib/libluajit-5.1.so.2
#5  0x0000562a32e7aafd in ngx_http_lua_run_thread (L=L@entry=0x7fe6ab048380, r=r@entry=0x562a344abb40, ctx=ctx@entry=0x562a344ac530, nrets=nrets@entry=0) at /home/ljl/code/orinc/lua-nginx-module-plus/src/ngx_http_lua_util.c:1167
#6  0x0000562a32e7ef54 in ngx_http_lua_content_by_chunk (L=L@entry=0x7fe6ab048380, r=r@entry=0x562a344abb40) at /home/ljl/code/orinc/lua-nginx-module-plus/src/ngx_http_lua_contentby.c:124
#7  0x0000562a32e7f112 in ngx_http_lua_content_handler_inline (r=0x562a344abb40) at /home/ljl/code/orinc/lua-nginx-module-plus/src/ngx_http_lua_contentby.c:312
#8  0x0000562a32e7e87e in ngx_http_lua_content_handler (r=0x562a344abb40) at /home/ljl/code/orinc/lua-nginx-module-plus/src/ngx_http_lua_contentby.c:222
#9  0x0000562a32da75b6 in ngx_http_core_content_phase (r=0x562a344abb40, ph=<optimized out>) at src/http/ngx_http_core_module.c:1257
#10 0x0000562a32da18b1 in ngx_http_core_run_phases (r=r@entry=0x562a344abb40) at src/http/ngx_http_core_module.c:878
#11 0x0000562a32da1956 in ngx_http_handler (r=r@entry=0x562a344abb40) at src/http/ngx_http_core_module.c:861
#12 0x0000562a32dadc4f in ngx_http_process_request (r=r@entry=0x562a344abb40) at src/http/ngx_http_request.c:2103
#13 0x0000562a32dae2a8 in ngx_http_process_request_headers (rev=rev@entry=0x562a344aa2d0) at src/http/ngx_http_request.c:1505
#14 0x0000562a32dae666 in ngx_http_process_request_line (rev=rev@entry=0x562a344aa2d0) at src/http/ngx_http_request.c:1176
#15 0x0000562a32dae81f in ngx_http_wait_request_handler (rev=0x562a344aa2d0) at src/http/ngx_http_request.c:522
#16 0x0000562a32d906eb in ngx_epoll_process_events (cycle=<optimized out>, timer=<optimized out>, flags=<optimized out>) at src/event/modules/ngx_epoll_module.c:968
#17 0x0000562a32d839e5 in ngx_process_events_and_timers (cycle=cycle@entry=0x562a3448cde0) at src/event/ngx_event.c:262
#18 0x0000562a32d8f3e3 in ngx_single_process_cycle (cycle=0x562a3448cde0) at src/os/unix/ngx_process_cycle.c:335
#19 0x0000562a32d6177e in main (argc=<optimized out>, argv=<optimized out>) at src/core/nginx.c:425
(gdb) quit
```
