---
layout: post
title: "OpenResty 是怎么保证软件质量的"
description: "OpenResty 的软件是如何保障高质量的"
date: 2023-03-05
tags: [OpenResty, Nginx, Valgrind, Address Sanity, Mock, Test, Quality]
---

OpenResty 的软件质量是非常高的，这个得益于 OpenResty 创始人 章亦春 对代码要求非常高有直接关系。
除了个人特质只要，OpenResty 使用了哪些技术来提高软件质量呢？

首先，OpenResty 的每一个仓库都有非常详尽的用例覆盖，修改都会自动跑 CI 进行回归。

其次，它使用了非常多的测试技术来保障软件质量。

下面分享其中使用到的一些测试方法和技术，方便大家使用到自己的项目中，提升自己的软件质量。

# 前提准备

## LuaJIT 编译

LuaJIT 默认情况下使用的是自己的内存分配器，通过 mmap 接口向操作系统申请大块内存再分割成小块使用。
因此在使用默认编译选项编译的情况下，valgrind 检查不到 LuaJIT 内存的操作错误。
要让 LuaJIT 的内存操作行为能够被 Valgrind 检查，需要修改 LuaJIT 的内存分配选项，增加 `-DLUAJIT_USE_SYSMALLOC` 这边编译宏。

下面是一个编译 LuaJIT 的例子

```shell
export JOBS=8
export LUAJIT_PREFIX=/opt/luajit21
export LUAJIT_LIB=$LUAJIT_PREFIX/lib
export LUAJIT_INC=$LUAJIT_PREFIX/include/luajit-2.1
export LUA_INCLUDE_DIR=$LUAJIT_INC

make -j"$JOBS" CCDEBUG=-g Q= PREFIX=$LUAJIT_PREFIX CC=$CC XCFLAGS="-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT -DLUAJIT_USE_VALGRIND -I$VALGRIND_INC -DLUAJIT_USE_SYSMALLOC -DLUA_USE_APICHECK -DLUA_USE_ASSERT -msse4.2" > build.log 2>&1 || (cat build.log && exit 1)

```

## Nginx 的编译

Nginx 的一个特色就是内存泄漏不少那么的容易，它是如何做到的呢？

这个得归功于 Nginx 的 内存池接口，通过将内存申请释放都统一到内存池，减少了内存释放得位置。
在一些情况下，新增的模块设置完全没有内存释放是操作。

内存池向内存分配器申请内存是申请大块内存，Nginx 模块在向内存池申请内存的时候再从大块内存中切割出小内存来使用。
这样，使用内存池以后，申请的内存大小也比较统一，降低了内存碎片的概率。
但是这个也导致一个问题就是 Valgrind 这种内存检查工具无法正常工作。

为了让 Valgrind 能够正常工作，那么就需要破坏 Nginx 内存池的这个优秀特性，让每次申请内存都是向内存分配器申请。
为此， OpenResty 增加了一个 [no-pool.patch](https://github.com/openresty/openresty/blob/master/patches/nginx-1.23.0-no_pool.patch) 的补丁。OpenResty 的 模块使用 [ngx-build](https://github.com/openresty/openresty-devel-utils/blob/master/ngx-build#L262) 编译默认都是会打上 no-pool 的补丁。

一个 ngx-build 编译的例子是在 [lua-nginx-module 的 build.sh](https://github.com/openresty/lua-nginx-module/blob/master/util/build.sh#L27)。摘抄部分出来

```shell
time ngx-build $force $version \
            --with-threads \
            --with-pcre-jit \
            --with-ipv6 \
            --with-cc-opt="-DNGX_LUA_USE_ASSERT -I$PCRE_INC -I$OPENSSL_INC" \
            --with-http_v2_module \
            --with-http_realip_module \
            --with-http_ssl_module \
            --add-module=$root/../ndk-nginx-module \
            --add-module=$root/../set-misc-nginx-module \
            --with-ld-opt="-L$PCRE_LIB -L$OPENSSL_LIB -Wl,-rpath,$PCRE_LIB:$LIBDRIZZLE_LIB:$OPENSSL_LIB" \
            --without-mail_pop3_module \
            --without-mail_imap_module \
            --with-http_image_filter_module \
            --without-mail_smtp_module \
            --with-stream \
            --with-stream_ssl_module \
            --without-http_upstream_ip_hash_module \
            --without-http_memcached_module \
            --without-http_auth_basic_module \
            --without-http_userid_module \
            --with-http_auth_request_module \
                --add-module=$root/../echo-nginx-module \
                --add-module=$root/../memc-nginx-module \
                --add-module=$root/../srcache-nginx-module \
                --add-module=$root \
                --add-module=$root/../lua-upstream-nginx-module \
              --add-module=$root/../headers-more-nginx-module \
                --add-module=$root/../drizzle-nginx-module \
                --add-module=$root/../rds-json-nginx-module \
                --add-module=$root/../coolkit-nginx-module \
                --add-module=$root/../redis2-nginx-module \
                --add-module=$root/../stream-lua-nginx-module \
                --add-module=$root/t/data/fake-module \
                $add_fake_shm_module \
                --add-module=$root/t/data/fake-delayed-load-module \
                --with-http_gunzip_module \
                --with-http_dav_module \
          --with-select_module \
          --with-poll_module \
                $opts \
                --with-debug
```

# 测试 Nginx

在完成编译的准备工作以后，如何执行测试用例才能用上 Valgrind 呢？
很简单，只要加上 TEST_NGINX_USE_VALGRIND=1 就可以了。

比如：

```shell
export TEST_NGINX_USE_VALGRIND=1
export TEST_NGINX_SLEEP=0.5

prove -I./ -Itest-nginx/lib -Itest-nginx/inc -r t/
```

**注意：一定要加上 export，否则测试框架 test-ngixn 看不到该环境变量**

## 处理不完整的调用栈

有时候发现 valgrind 给出的调用栈也是不完整的。比如下面这个例子：

```shell
{
   <insert_a_suppression_name_here>
   Memcheck:Leak
   match-leak-kinds: definite
   fun:realloc
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   obj:*
   fun:lj_vm_ffi_call
   fun:lj_ccall_func
   fun:lj_cf_ffi_meta___call
   fun:lj_BC_FUNCC
   fun:ngx_http_lua_run_thread
   fun:ngx_http_lua_content_by_chunk
   fun:ngx_http_lua_content_handler
   fun:ngx_http_core_content_phase
   fun:ngx_http_core_run_phases
   fun:ngx_http_process_request
   fun:ngx_http_process_request_headers
   fun:ngx_http_process_request_line
   fun:ngx_epoll_process_events
   fun:ngx_process_events_and_timers
   fun:ngx_single_process_cycle
   fun:main
}
```

遇到这样的情况该如何是好呢？

1. 首先检查是不是所有的调试符号的安装包都已经安装了。（rpm 对应 debuginfo， deb 对应 dbgsym）
1. 这个也可能是 LuaJIT 通过动态加载 so 导致的问题，可以通过 LD_PRELAOD 将这个 so 预加载来解决该问题。

比如：

```shell
export TEST_NGINX_USE_VALGRIND=1
export TEST_NGINX_SLEEP=0.5
export LD_PRELOAD=/usr/local/openresty/lib/cjson.so

prove -I./ -Itest-nginx/lib -Itest-nginx/inc -r t/
```

## Valgrind 的误报处理

Valgrind 可能会有一些误报或者是没有必要处理的地方。
比如内存泄漏，但是只会初始化的时候泄漏一次处理起来代价很大，这种情况就不是特别有必要处理。
另外有一些数据没有初始化的问题也没有关系，比如结构体中部分成员没有初始化，但是这个是预期的行为。

为了抑制这类不必要的报错，需要将这些报错信息添加到 [valgrind.supress](https://github.com/openresty/lua-nginx-module/blob/master/valgrind.suppress)

比如这是一个结构体成员没有全部赋值的情况。

```
{
   <insert_a_suppression_name_here>
   Memcheck:Param
   sendmsg(msg.msg_iov[0])
   fun:__sendmsg_nocancel
   fun:ngx_write_channel
   fun:ngx_pass_open_channel
   fun:ngx_start_worker_processes
}
```

# LuaJIT GC 内存回收的测试

Lua 是带 GC 功能的语言，不需要程序员自己处理内存释放的问题。内存释放是在 GC 进行垃圾回收的时候
确认内存对象已经没有人使用以后才会释放。一般情况下 Lua 自身管理的内存比较少出现问题，但是由于
Lua 和 C 交换直接的内存生命期的管理上存在问题可能就导致内存改写，释放后访问等各种经典内存问题。

为了让案发现场（软件崩溃）和导致问题的代码更加接近，应该让 GC的执行更加的频繁。
为此有一个专门的测试模式是 `每语句 GC` -- 也就是说每执行一条 Lua 语句就进行一次完整的 GC 回收操作。

```shell
export TEST_NGINX_INIT_BY_LUA="debug.sethook(function () collectgarbage() end, 'l') jit.off() package.path = '/usr/share/lua/5.1/?.lua;$PWD/../lua-resty-core/lib/?.lua;$PWD/../lua-resty-lrucache/lib/?.lua;' .. (package.path or '') require 'resty.core' require('resty.core.base').set_string_buf_size(1) require('resty.core.regex').set_buf_grow_ratio(1)"

export TEST_NGINX_USE_VALGRIND=1

prove -I./ -Itest-nginx/lib -Itest-nginx/inc -r t/
```

这里主要是向 [debug.sethook](https://www.lua.org/pil/23.2.html) 这个钩子注册一个函数。

有四种事件可以触发钩子：
1. 调用事件在 Lua 每次调用函数时发生；
1. 返回事件在函数每次返回时发生；
1. 行事件在Lua开始执行新的一行代码时发生；
1. 计数事件在一定数量的指令后发生。

Lua调用钩子时只有一个参数，一个描述产生调用的事件的字符串。"调用"、"返回"、"行 "或 "计数"。此外，对于行事件，它还传递第二个参数，即新的行号。我们可以随时使用 debug.getinfo 来获取钩子内部的更多信息。

# mock 测试

协议解析涉及从网络读取数据包以及发送数据包到网卡。由于数据分片，MTU 大小不同，每次读取的网络数据包的大小也是不确定的。因此，一个健壮的代码要能够解析各种情况下的数据。发送数据可能由于网络拥塞等各种情况，也会导致发送缓冲区被塞满，因此无法全部发送。

由于测试都是通过环回接口进行的，而环回接口的 MTU 很大，延时很小并且不存在丢包的情况，因此很难模拟复杂的网络情况。为此 OpenResty 开发了一个 [mockeagain](https://github.com/openresty/mockeagain) 的模块来模拟复杂的网络条件。

这个模块就是通过 `LD_PRELOAD` 劫持 "poll", "close", "send" 和 "writev" 这些系统调用，替换成模块内部的接口来实现恶劣网络缓解的模拟。

使用方法 [README.md](https://github.com/openresty/mockeagain) 有详细的介绍。主要是两点，一个是修改 Nginx 的配置文件，增加 `env LD_PRELOAD;`, 因为 Nginx 为了安全性，默认是会删除所有的环境变量。另一个是在启动 Nginx 前添加 LD_PRELOAD， 比如： `MOCKEAGAIN=w LD_PRELOAD=/path/to/mockeagain.so`。

我们测试一般都是结合 test-nginx 中进行的，因此多数情况下使用下面的命令。

```shell
export MOCKEAGAIN=rw
export LD_PRELOAD=/path/to/mockeagain.so
export TEST_NGINX_EVENT_TYPE=poll
export TEST_NGINX_POSTPONE_OUTPUT=1

prove -I./ -Itest-nginx/lib -Itest-nginx/inc -r t/
```

# 内存泄漏模式

Valgrind 虽然可以比较准确的检测内存泄漏，但是有些内存泄漏用 Valgrind 也检测不出来。 比如 Nginx 内存池中的内存泄漏在内存池释放的时候就会被正常释放了。如果一个内存池存生命周期很长，并且不停的从内存池申请内存，那么这个内存池就会消耗很多的内存，而这些内存无法被复用。

test-nginx 中有专门的一个内存泄漏模式 TEST_NGINX_CHECK_LEAK。

一般情况下这样执行内存泄漏模式。

```shell
export TEST_NGINX_CHECK_LEAK=1
export TEST_NGINX_CHECK_LEAK_COUNT=1000

prove -I./ -Itest-nginx/lib -Itest-nginx/inc -r t/
```

# test-nginx 其它选项

test-nginx 有非常多的参数可以调整，都是以 `TEST_NGINX_` 开头。
不同测测试组合模式能够从不同方面检测 OpenResty / Nginx 的软件质量。

OpenResty 每次发版都执行很多不同的模式来保障发版软件的质量，详见： [qa.openresty.org](http://qa.openresty.org/)

这是一个列表：

1. TEST_NGINX_BENCHMARK: 使用 ab/weighttp 命令执行性能测试, 可以比较不同版本之间的性能差异
1. TEST_NGINX_BENCHMARK_WARMUP: 性能测试前请求个数，消除冷启动的异常数据
1. TEST_NGINX_BINARY: 指定 nginx 这个可执行文件的路径
1. TEST_NGINX_EVENT_TYPE: Nginx 多种事件模型, select, poll, epoll, kqueue 等
1. TEST_NGINX_FAST_SHUTDOWN: 默认情况下是发送 QUIT 信号退出，设置后发送 TERM 信号让 Nginx 退出
1. TEST_NGINX_FORCE_RESTART_ON_TEST: 强制 Nginx 用例之间 重启 Nginx 进程
1. TEST_NGINX_INIT_BY_LUA: 在每语句 GC 模式中已经介绍
1. TEST_NGINX_LOAD_MODULES: 加载 Nginx 的动态模块
1. TEST_NGINX_LOG_LEVEL: 指定 Nginx 的日志级别。测试不停日志级别下功是否正常。
1. TEST_NGINX_MASTER_PROCESS: Nginx 配置文件的 `master_process on|off;` 选项控制 
1. TEST_NGINX_NO_CLEAN: 测试用例执行完毕不要杀死 Nginx 进程，一般结合 --- ONLY 使用来排查问题
1. TEST_NGINX_NO_SHUFFLE: 顺序执行测试用例
1. TEST_NGINX_POSTPONE_OUTPUT: Nginx 配置加上 `postpone_output 1;` 这样的配置
1. TEST_NGINX_RANDOMIZE: 并发测试的时候需要加上该选项
1. TEST_NGINX_REUSE_PORT: 给Listen 加上 `reusport` 的选项
1. TEST_NGINX_SLEEP: 一般 Valgrind 模式使用，防止 Nginx进程还没有准备好，连接失败打印大量的错误信息
1. TEST_NGINX_TIMEOUT: 等待进程退出的最长事件。如果测试 Valgrind，那么这个事件需要调整大一点，否则不少用例无法通过
1. TEST_NGINX_USE_HUP: 多个测试用例之间使用 HUP Reload 模式来加载新的配置
1. TEST_NGINX_VERBOSE: 一般是在调试 test-nginx 本身使用

# Address Sanity

Address Sanity 也是检测内存操作的强有力的工具，因此 OpenResty 官方发布的版本也有一个专门的 asan 的版本供大家在遇到问题时候使用该版本进行问题分析。

如果大家自己编译 asan 版本，记得所有依赖的组件都得重新编译。
比如：lua-cjson，luajit，openssl，pcre等。

编译的时候注意需要给 CFLAGS 增加 `-fno-omit-frame-pointer -g` 选项，否则无法得到完整的调用栈，在排查内存问题时候就搞不清楚是什么样的调用顺序。

另外，前面提到的 no-pool 补丁以及 Luajit 的 SYSMALLOC 选项一样需要开启。

比如：

```shell
export CC="gcc -fsanitize=address"
export NGX_BUILD_CC=$CC
export NGX_BUILD_CC_OPTS="-fno-omit-frame-pointer -O1 -g"
export ASAN_OPTIONS=detect_leaks=0
export LD_PRELOAD=/lib64/libasan.so.6:$PWD/mockeagain/mockeagain.so

....

prove -I. -I../test-nginx/inc -I../test-nginx/lib t/
```

这里面要注意 LD_PRELOAD 的顺序，把 libasan.so 防止最前面。

另外，`nginx -s` 发送信号存在内存泄漏，也没有必要解决。所以应该这样发送信号。

```shell
ASAN_OPTIONS=detect_leaks=0 ./sbin/nginx -t
```

如果是线上跑服务的 nginx 后台进程，那么需要让asan日志写入指定文件，可以这么执行。

```shell
mkdir -p /var/asan; chmod a+w /var/asan
ASAN_OPTIONS=log_path=/var/asan/asan,log_exe_name=true ./sbin/nginx
```
