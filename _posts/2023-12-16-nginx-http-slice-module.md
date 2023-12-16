---
layout: post
title: "how does nginx http slice module work"
description: "Analyze the work flow of the nginx_http_slice_module"
date: 2023-12-16
tags: [Nginx, OpenResty, slice_range]
---

我想分析一下 nginx_http_slice_module 的工作原理，因为自带的 OpenResty 不含该模块，因此需要自己编译。
在深入代码分析之前，我们需要先了解 HTTP bytes range 的请求

# 了解 HTTP bytes range 的原理

可以在点击 [mozilla rnage request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) 查看详细的信息。
我们这里抽取一个请求响应的例子。

```shell
$ curl -v http://i.imgur.com/z4d4kWk.jpg -i -H "Range: bytes=0-1023"
GET /z4d4kWk.jpg HTTP/1.1
Host: i.imgur.com
Range: bytes=0-1023

HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1023/146515
Content-Length: 1024
```


# 编译 OpenResty

使用下面的命令从 OpenResty 官方站点下载

```shell
wget https://openresty.org/download/openresty-1.21.4.3.tar.gz
```


使用下面的命令编译 OpenResty。注意这里编译选项加上 **-O0** 防止编译器优化尾调用函数导致调用栈的函数不连续。
另外就是还需要加上本次分析的目标模块 **--with-http_slice_module**。

```shell
#!/bin/bash

tar -xf openresty-1.21.4.3.tar.gz
cd openresty-1.21.4.3

orprefix=/usr/local/openresty
zlib_prefix=${orprefix}/zlib
pcre_prefix=${orprefix}/pcre
openssl_prefix=${orprefix}/openssl111

./configure \
    --prefix="${orprefix}" \
    --with-cc='ccache gcc -fdiagnostics-color=always' \
    --with-cc-opt="-DNGX_LUA_ABORT_AT_PANIC -I${zlib_prefix}/include -I${pcre_prefix}/include -O0" \
    --with-ld-opt="-L${zlib_prefix}/lib -L${pcre_prefix}/lib -Wl,-rpath,${zlib_prefix}/lib:${pcre_prefix}/lib" \
    --with-pcre-jit \
    --with-http_slice_module \
    --without-http_rds_json_module \
    --without-http_rds_csv_module \
    --without-lua_rds_parser \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_ssl_preread_module \
    --with-http_v2_module \
    --without-mail_pop3_module \
    --without-mail_imap_module \
    --without-mail_smtp_module \
    --with-http_stub_status_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_auth_request_module \
    --with-http_secure_link_module \
    --with-http_random_index_module \
    --with-http_gzip_static_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-threads \
    --with-compat \
    --with-luajit-xcflags='-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT' \
    -j`nproc`  2>&1 | tee test.log

make -j`nproc` 2>&1 | tee -a test.log


sudo make install
```

# 配置 OpenResty 支持 ngx_http_slice_module

```nginx
user  root;
worker_processes  1;
worker_cpu_affinity auto;
error_log  logs/error.log  info;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - [$request_time] [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log main buffer=64k;

    keepalive_timeout  65;

    keepalive_requests 1000;

    lua_check_client_abort  on;
    lua_package_path   "/usr/local/openresty/site/?.ljbc;;";
    lua_package_cpath  "/usr/local/openresty-yajl/lib/?.so;/usr/local/openresty/site/?.so;;";

    proxy_cache_path cache keys_zone=one:10m;

    server {
        listen       80 reuseport;
        server_name  localhost;
        client_header_buffer_size 1024;

        large_client_header_buffers 4 8k;
        ignore_invalid_headers on;

        location / {
            root   html;
            access_log off;
            index  index.html index.htm;
        }

        location /slice/index.html {
            slice             50k;
            proxy_cache       one;
            proxy_cache_key   $uri$is_args$args$slice_range;
            proxy_set_header  Range $slice_range;
            proxy_cache_valid 200 206 1h;
            proxy_pass http://192.168.111.162/ai.jpg;
        }
    }
}
```

使用下面的命令进行测试:

```shell
# 如果不删除，那么走的是缓存的代码路径
sudo rm -fr /usr/local/openresty/nginx/cache/*
curl http://127.0.0.1:810/slice/index.html
```

# gdb 断点打印堆栈

了解代码路径是学习软件行为的良好方法。我们通过 gdb 的命令自动打印出出重点函数的代码路径。

```gdb
$ pid=$(ps aux | grep nginx | grep worker | awk '{print $2}')
$ gdb -p $pid
break ngx_http_slice_header_filter
  commands
    bt
    c
  end
break ngx_http_slice_body_filter
  commands
    bt
    c
  end
break ngx_http_slice_range_variable
  commands
    bt
    c
  end
break writev
  commands
    bt
    c
  end
```

## 代码路径

这个是执行上面的 gdb 得到的代码路径

```js
# 构建 cache_key 
Breakpoint 4, ngx_http_slice_range_variable (r=0x6f3470, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x6f3470, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x6f3470, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd1c0) at src/http/ngx_http_script.c:944
#3  0x00000000004a239d in ngx_http_complex_value (r=0x6f3470, val=0x708ff8, value=0x6f43d8) at src/http/ngx_http_script.c:83
#4  0x00000000004fba00 in ngx_http_proxy_create_key (r=0x6f3470) at src/http/modules/ngx_http_proxy_module.c:1161
#5  0x00000000004a6e39 in ngx_http_upstream_cache (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:861
#6  0x00000000004a6237 in ngx_http_upstream_init_request (r=0x6f3470) at src/http/ngx_http_upstream.c:581
#7  0x00000000004a61c8 in ngx_http_upstream_init (r=0x6f3470) at src/http/ngx_http_upstream.c:554
#8  0x000000000049a7ac in ngx_http_read_client_request_body (r=0x6f3470, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:84
#9  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x6f3470) at src/http/modules/ngx_http_proxy_module.c:1023
#10 0x0000000000481d56 in ngx_http_core_content_phase (r=0x6f3470, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#11 0x00000000004810ed in ngx_http_core_run_phases (r=0x6f3470) at src/http/ngx_http_core_module.c:885
#12 0x000000000048105a in ngx_http_handler (r=0x6f3470) at src/http/ngx_http_core_module.c:868
#13 0x00000000004905eb in ngx_http_process_request (r=0x6f3470) at src/http/ngx_http_request.c:2127
#14 0x000000000048eeee in ngx_http_process_request_headers (rev=0x7bfae0) at src/http/ngx_http_request.c:1529
#15 0x000000000048e457 in ngx_http_process_request_line (rev=0x7bfae0) at src/http/ngx_http_request.c:1196
#16 0x000000000048cef3 in ngx_http_wait_request_handler (rev=0x7bfae0) at src/http/ngx_http_request.c:523
#17 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=456, flags=1) at src/event/modules/ngx_epoll_module.c:901
#18 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#19 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 添加请求头
Breakpoint 4, ngx_http_slice_range_variable (r=0x6f3470, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x6f3470, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x6f3470, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd160) at src/http/ngx_http_script.c:944
#3  0x00000000004fc32c in ngx_http_proxy_create_request (r=0x6f3470) at src/http/modules/ngx_http_proxy_module.c:1357
#4  0x00000000004a6524 in ngx_http_upstream_init_request (r=0x6f3470) at src/http/ngx_http_upstream.c:641
#5  0x00000000004a61c8 in ngx_http_upstream_init (r=0x6f3470) at src/http/ngx_http_upstream.c:554
#6  0x000000000049a7ac in ngx_http_read_client_request_body (r=0x6f3470, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:84
#7  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x6f3470) at src/http/modules/ngx_http_proxy_module.c:1023
#8  0x0000000000481d56 in ngx_http_core_content_phase (r=0x6f3470, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#9  0x00000000004810ed in ngx_http_core_run_phases (r=0x6f3470) at src/http/ngx_http_core_module.c:885
#10 0x000000000048105a in ngx_http_handler (r=0x6f3470) at src/http/ngx_http_core_module.c:868
#11 0x00000000004905eb in ngx_http_process_request (r=0x6f3470) at src/http/ngx_http_request.c:2127
#12 0x000000000048eeee in ngx_http_process_request_headers (rev=0x7bfae0) at src/http/ngx_http_request.c:1529
#13 0x000000000048e457 in ngx_http_process_request_line (rev=0x7bfae0) at src/http/ngx_http_request.c:1196
#14 0x000000000048cef3 in ngx_http_wait_request_handler (rev=0x7bfae0) at src/http/ngx_http_request.c:523
#15 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=456, flags=1) at src/event/modules/ngx_epoll_module.c:901
#16 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#17 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 向上游发送请求
Breakpoint 9, __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f6d8, vec=0x7fffffffd510) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f6d8, in=0x7b5930, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x000000000042da74 in ngx_chain_writer (data=0x7b4ef0, in=0x0) at src/core/ngx_output_chain.c:774
#4  0x000000000042bf7f in ngx_output_chain (ctx=0x7b4e70, in=0x7b58b0) at src/core/ngx_output_chain.c:70
#5  0x00000000004a98e9 in ngx_http_upstream_send_request_body (r=0x6f3470, u=0x7b4dc0, do_write=1) at src/http/ngx_http_upstream.c:2227
#6  0x00000000004a948a in ngx_http_upstream_send_request (r=0x6f3470, u=0x7b4dc0, do_write=1) at src/http/ngx_http_upstream.c:2113
#7  0x00000000004a9cb8 in ngx_http_upstream_send_request_handler (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:2356
#8  0x00000000004a7c65 in ngx_http_upstream_handler (ev=0x7d7b50) at src/http/ngx_http_upstream.c:1303
#9  0x000000000046a339 in ngx_epoll_process_events (cycle=0x6e4760, timer=439, flags=1) at src/event/modules/ngx_epoll_module.c:930
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

#上游返回的响应头过滤，注意这里的 r 值是客户端发起的主请求
Breakpoint 1, ngx_http_slice_header_filter (r=0x6f3470) at src/http/modules/ngx_http_slice_filter_module.c:111
111	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_header_filter (r=0x6f3470) at src/http/modules/ngx_http_slice_filter_module.c:111
#1  0x000000000048309e in ngx_http_send_header (r=0x6f3470) at src/http/ngx_http_core_module.c:1859
#2  0x00000000004ab30a in ngx_http_upstream_send_response (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:3034
#3  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:2535
#4  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#5  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=439, flags=1) at src/event/modules/ngx_epoll_module.c:901
#6  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#7  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 过滤请求体
Breakpoint 3, ngx_http_slice_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x7b5fa8) at src/http/ngx_http_core_module.c:1874
#2  0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x6f3470, chain=0x7b5fa8) at src/http/ngx_http_upstream.c:4027
#3  0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7b52b0) at src/event/ngx_event_pipe.c:551
#4  0x000000000045ee42 in ngx_event_pipe (p=0x7b52b0, do_write=1) at src/event/ngx_event_pipe.c:33
#5  0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:4123
#6  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#7  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#8  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#9  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 将响应返回给客户端, 这里的 iovcnt 是 2
Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=2) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=2) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffcf20) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7b5930, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x7b68a0) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x6f3470, in=0x7b68a0) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x6f3470, in=0x7b68a0) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x6f3470, in=0x7b68a0) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x7b67c0, in=0x7b5fa8) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x6f3470, in=0x7b5fa8) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d672a in ngx_http_slice_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_slice_filter_module.c:240
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x7b5fa8) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x6f3470, chain=0x7b5fa8) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7b52b0) at src/event/ngx_event_pipe.c:551
#22 0x000000000045ee42 in ngx_event_pipe (p=0x7b52b0, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#25 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#26 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#27 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 将响应返回给客户端, 这里的 iovcnt 是 1
Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	in ../sysdeps/unix/sysv/linux/writev.c
#0  __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffcf20) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7b68b0, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x7b68a0) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x6f3470, in=0x7b68a0) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x6f3470, in=0x7b68a0) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x6f3470, in=0x7b68a0) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x6f3470, in=0x7b68a0) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x7b67c0, in=0x7b5fa8) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x6f3470, in=0x7b5fa8) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d672a in ngx_http_slice_body_filter (r=0x6f3470, in=0x7b5fa8) at src/http/modules/ngx_http_slice_filter_module.c:240
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x7b5fa8) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x6f3470, chain=0x7b5fa8) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7b52b0) at src/event/ngx_event_pipe.c:551
#22 0x000000000045ee42 in ngx_event_pipe (p=0x7b52b0, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#25 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#26 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#27 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 过滤响应头，这里是 ngx_http_upstream_finalize_request 调用 ngx_http_send_special
Breakpoint 3, ngx_http_slice_body_filter (r=0x6f3470, in=0x7fffffffd630) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x6f3470, in=0x7fffffffd630) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x7fffffffd630) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000492e38 in ngx_http_send_special (r=0x6f3470, flags=1) at src/http/ngx_http_request.c:3606
#3  0x00000000004aead3 in ngx_http_upstream_finalize_request (r=0x6f3470, u=0x7b4dc0, rc=0) at src/http/ngx_http_upstream.c:4654
#4  0x00000000004add16 in ngx_http_upstream_process_request (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:4217
#5  0x00000000004ada13 in ngx_http_upstream_process_upstream (r=0x6f3470, u=0x7b4dc0) at src/http/ngx_http_upstream.c:4129
#6  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#7  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#8  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#9  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 可以看到这是 ngx_http_run_posted_requests 发起的子请求
Breakpoint 4, ngx_http_slice_range_variable (r=0x7f9910, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x7f9910, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x7f9910, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd310) at src/http/ngx_http_script.c:944
#3  0x00000000004a239d in ngx_http_complex_value (r=0x7f9910, val=0x708ff8, value=0x7fa7f8) at src/http/ngx_http_script.c:83
#4  0x00000000004fba00 in ngx_http_proxy_create_key (r=0x7f9910) at src/http/modules/ngx_http_proxy_module.c:1161
#5  0x00000000004a6e39 in ngx_http_upstream_cache (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:861
#6  0x00000000004a6237 in ngx_http_upstream_init_request (r=0x7f9910) at src/http/ngx_http_upstream.c:581
#7  0x00000000004a61c8 in ngx_http_upstream_init (r=0x7f9910) at src/http/ngx_http_upstream.c:554
#8  0x000000000049a6ed in ngx_http_read_client_request_body (r=0x7f9910, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:47
#9  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x7f9910) at src/http/modules/ngx_http_proxy_module.c:1023
#10 0x0000000000481d56 in ngx_http_core_content_phase (r=0x7f9910, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#11 0x00000000004810ed in ngx_http_core_run_phases (r=0x7f9910) at src/http/ngx_http_core_module.c:885
#12 0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#13 0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#14 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#15 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#16 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 这个是子请求发起上游请求
Breakpoint 4, ngx_http_slice_range_variable (r=0x7f9910, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x7f9910, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x7f9910, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd2b0) at src/http/ngx_http_script.c:944
#3  0x00000000004fc32c in ngx_http_proxy_create_request (r=0x7f9910) at src/http/modules/ngx_http_proxy_module.c:1357
#4  0x00000000004a6524 in ngx_http_upstream_init_request (r=0x7f9910) at src/http/ngx_http_upstream.c:641
#5  0x00000000004a61c8 in ngx_http_upstream_init (r=0x7f9910) at src/http/ngx_http_upstream.c:554
#6  0x000000000049a6ed in ngx_http_read_client_request_body (r=0x7f9910, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:47
#7  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x7f9910) at src/http/modules/ngx_http_proxy_module.c:1023
#8  0x0000000000481d56 in ngx_http_core_content_phase (r=0x7f9910, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#9  0x00000000004810ed in ngx_http_core_run_phases (r=0x7f9910) at src/http/ngx_http_core_module.c:885
#10 0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#11 0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#12 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=432, flags=1) at src/event/modules/ngx_epoll_module.c:901
#13 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#14 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 子请求向上游发送请求
Breakpoint 9, __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f6d8, vec=0x7fffffffd510) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f6d8, in=0x7fb158, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x000000000042da74 in ngx_chain_writer (data=0x7faa50, in=0x0) at src/core/ngx_output_chain.c:774
#4  0x000000000042bf7f in ngx_output_chain (ctx=0x7fa9d0, in=0x7b5930) at src/core/ngx_output_chain.c:70
#5  0x00000000004a98e9 in ngx_http_upstream_send_request_body (r=0x7f9910, u=0x7fa920, do_write=1) at src/http/ngx_http_upstream.c:2227
#6  0x00000000004a948a in ngx_http_upstream_send_request (r=0x7f9910, u=0x7fa920, do_write=1) at src/http/ngx_http_upstream.c:2113
#7  0x00000000004a9cb8 in ngx_http_upstream_send_request_handler (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:2356
#8  0x00000000004a7c65 in ngx_http_upstream_handler (ev=0x7d7b50) at src/http/ngx_http_upstream.c:1303
#9  0x000000000046a339 in ngx_epoll_process_events (cycle=0x6e4760, timer=427, flags=1) at src/event/modules/ngx_epoll_module.c:930
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 子请求响应头的过滤
Breakpoint 1, ngx_http_slice_header_filter (r=0x7f9910) at src/http/modules/ngx_http_slice_filter_module.c:111
111	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_header_filter (r=0x7f9910) at src/http/modules/ngx_http_slice_filter_module.c:111
#1  0x000000000048309e in ngx_http_send_header (r=0x7f9910) at src/http/ngx_http_core_module.c:1859
#2  0x00000000004ab30a in ngx_http_upstream_send_response (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:3034
#3  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:2535
#4  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#5  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=383, flags=1) at src/event/modules/ngx_epoll_module.c:901
#6  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#7  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 子请求响应体过滤
Breakpoint 3, ngx_http_slice_body_filter (r=0x7f9910, in=0x7ff9a0) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x7f9910, in=0x7ff9a0) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7ff9a0) at src/http/ngx_http_core_module.c:1874
#2  0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7f9910, chain=0x7ff9a0) at src/http/ngx_http_upstream.c:4027
#3  0x000000000045ffde in ngx_event_pipe_write_to_downstream (p=0x7fa6c8) at src/event/ngx_event_pipe.c:694
#4  0x000000000045ee42 in ngx_event_pipe (p=0x7fa6c8, do_write=1) at src/event/ngx_event_pipe.c:33
#5  0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4123
#6  0x00000000004ac139 in ngx_http_upstream_send_response (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:3364
#7  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:2535
#8  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#9  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=383, flags=1) at src/event/modules/ngx_epoll_module.c:901
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 子请求发送数据
Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffca80, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=15, iov=0x7fffffffca80, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffce80) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7b68b0, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x7ffb30) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x7f9910, in=0x7ffb30) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x7ffa40, in=0x7ff9a0) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x7f9910, in=0x7ff9a0) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x7f9910, in=0x7ff9a0) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d669e in ngx_http_slice_body_filter (r=0x7f9910, in=0x7ff9a0) at src/http/modules/ngx_http_slice_filter_module.c:228
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7ff9a0) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7f9910, chain=0x7ff9a0) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045ffde in ngx_event_pipe_write_to_downstream (p=0x7fa6c8) at src/event/ngx_event_pipe.c:694
#22 0x000000000045ee42 in ngx_event_pipe (p=0x7fa6c8, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004ac139 in ngx_http_upstream_send_response (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:3364
#25 0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:2535
#26 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#27 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=383, flags=1) at src/event/modules/ngx_epoll_module.c:901
#28 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#29 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

#子请求过滤响应头
Breakpoint 3, ngx_http_slice_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7ffa30) at src/http/ngx_http_core_module.c:1874
#2  0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7f9910, chain=0x7ffa30) at src/http/ngx_http_upstream.c:4027
#3  0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7fa6c8) at src/event/ngx_event_pipe.c:551
#4  0x000000000045ee42 in ngx_event_pipe (p=0x7fa6c8, do_write=1) at src/event/ngx_event_pipe.c:33
#5  0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4123
#6  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#7  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#8  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#9  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

#子请求向下游发送数据
Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffcf20) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7fff50, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x7ffb30) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x7f9910, in=0x7ffb30) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x7ffa40, in=0x7ffa30) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x7f9910, in=0x7ffa30) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d669e in ngx_http_slice_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_slice_filter_module.c:228
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7ffa30) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7f9910, chain=0x7ffa30) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7fa6c8) at src/event/ngx_event_pipe.c:551
#22 0x000000000045ee42 in ngx_event_pipe (p=0x7fa6c8, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#25 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#26 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#27 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

#子请求想下游发送数据
Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	in ../sysdeps/unix/sysv/linux/writev.c
#0  __GI___writev (fd=15, iov=0x7fffffffcb20, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffcf20) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7fff50, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x7ffb30) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x7ffb30) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x7f9910, in=0x7ffb30) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x7f9910, in=0x7ffb30) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x7f9910, in=0x7ffb30) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x7ffa40, in=0x7ffa30) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x7f9910, in=0x7ffa30) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d669e in ngx_http_slice_body_filter (r=0x7f9910, in=0x7ffa30) at src/http/modules/ngx_http_slice_filter_module.c:228
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7ffa30) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7f9910, chain=0x7ffa30) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x7fa6c8) at src/event/ngx_event_pipe.c:551
#22 0x000000000045ee42 in ngx_event_pipe (p=0x7fa6c8, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#25 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#26 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#27 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

# 子请求响应头过滤，send_special
Breakpoint 3, ngx_http_slice_body_filter (r=0x7f9910, in=0x7fffffffd630) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x7f9910, in=0x7fffffffd630) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x7f9910, in=0x7fffffffd630) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000492e38 in ngx_http_send_special (r=0x7f9910, flags=1) at src/http/ngx_http_request.c:3606
#3  0x00000000004aead3 in ngx_http_upstream_finalize_request (r=0x7f9910, u=0x7fa920, rc=0) at src/http/ngx_http_upstream.c:4654
#4  0x00000000004add16 in ngx_http_upstream_process_request (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4217
#5  0x00000000004ada13 in ngx_http_upstream_process_upstream (r=0x7f9910, u=0x7fa920) at src/http/ngx_http_upstream.c:4129
#6  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#7  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#8  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#9  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 3, ngx_http_slice_body_filter (r=0x6f3470, in=0x0) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x6f3470, in=0x0) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x0) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000491bdf in ngx_http_writer (r=0x6f3470) at src/http/ngx_http_request.c:2880
#3  0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#4  0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#5  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#6  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#7  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 4, ngx_http_slice_range_variable (r=0x7fffc0, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x7fffc0, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x7fffc0, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd310) at src/http/ngx_http_script.c:944
#3  0x00000000004a239d in ngx_http_complex_value (r=0x7fffc0, val=0x708ff8, value=0x80e638) at src/http/ngx_http_script.c:83
#4  0x00000000004fba00 in ngx_http_proxy_create_key (r=0x7fffc0) at src/http/modules/ngx_http_proxy_module.c:1161
#5  0x00000000004a6e39 in ngx_http_upstream_cache (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:861
#6  0x00000000004a6237 in ngx_http_upstream_init_request (r=0x7fffc0) at src/http/ngx_http_upstream.c:581
#7  0x00000000004a61c8 in ngx_http_upstream_init (r=0x7fffc0) at src/http/ngx_http_upstream.c:554
#8  0x000000000049a6ed in ngx_http_read_client_request_body (r=0x7fffc0, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:47
#9  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x7fffc0) at src/http/modules/ngx_http_proxy_module.c:1023
#10 0x0000000000481d56 in ngx_http_core_content_phase (r=0x7fffc0, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#11 0x00000000004810ed in ngx_http_core_run_phases (r=0x7fffc0) at src/http/ngx_http_core_module.c:885
#12 0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#13 0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#14 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#15 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#16 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 4, ngx_http_slice_range_variable (r=0x7fffc0, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
402	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_range_variable (r=0x7fffc0, v=0x6f4238, data=0) at src/http/modules/ngx_http_slice_filter_module.c:402
#1  0x000000000049d79a in ngx_http_get_indexed_variable (r=0x7fffc0, index=12) at src/http/ngx_http_variables.c:637
#2  0x00000000004a3d24 in ngx_http_script_copy_var_len_code (e=0x7fffffffd2b0) at src/http/ngx_http_script.c:944
#3  0x00000000004fc32c in ngx_http_proxy_create_request (r=0x7fffc0) at src/http/modules/ngx_http_proxy_module.c:1357
#4  0x00000000004a6524 in ngx_http_upstream_init_request (r=0x7fffc0) at src/http/ngx_http_upstream.c:641
#5  0x00000000004a61c8 in ngx_http_upstream_init (r=0x7fffc0) at src/http/ngx_http_upstream.c:554
#6  0x000000000049a6ed in ngx_http_read_client_request_body (r=0x7fffc0, post_handler=0x4a60ad <ngx_http_upstream_init>) at src/http/ngx_http_request_body.c:47
#7  0x00000000004fb4cf in ngx_http_proxy_handler (r=0x7fffc0) at src/http/modules/ngx_http_proxy_module.c:1023
#8  0x0000000000481d56 in ngx_http_core_content_phase (r=0x7fffc0, ph=0x79ccc8) at src/http/ngx_http_core_module.c:1271
#9  0x00000000004810ed in ngx_http_core_run_phases (r=0x7fffc0) at src/http/ngx_http_core_module.c:885
#10 0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#11 0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#12 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=378, flags=1) at src/event/modules/ngx_epoll_module.c:901
#13 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#14 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 9, __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=17, iov=0x7fffffffd110, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f6d8, vec=0x7fffffffd510) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f6d8, in=0x80e8b8, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x000000000042da74 in ngx_chain_writer (data=0x80df70, in=0x0) at src/core/ngx_output_chain.c:774
#4  0x000000000042bf7f in ngx_output_chain (ctx=0x80def0, in=0x800940) at src/core/ngx_output_chain.c:70
#5  0x00000000004a98e9 in ngx_http_upstream_send_request_body (r=0x7fffc0, u=0x80de40, do_write=1) at src/http/ngx_http_upstream.c:2227
#6  0x00000000004a948a in ngx_http_upstream_send_request (r=0x7fffc0, u=0x80de40, do_write=1) at src/http/ngx_http_upstream.c:2113
#7  0x00000000004a9cb8 in ngx_http_upstream_send_request_handler (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:2356
#8  0x00000000004a7c65 in ngx_http_upstream_handler (ev=0x7d7b50) at src/http/ngx_http_upstream.c:1303
#9  0x000000000046a339 in ngx_epoll_process_events (cycle=0x6e4760, timer=359, flags=1) at src/event/modules/ngx_epoll_module.c:930
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 1, ngx_http_slice_header_filter (r=0x7fffc0) at src/http/modules/ngx_http_slice_filter_module.c:111
111	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_header_filter (r=0x7fffc0) at src/http/modules/ngx_http_slice_filter_module.c:111
#1  0x000000000048309e in ngx_http_send_header (r=0x7fffc0) at src/http/ngx_http_core_module.c:1859
#2  0x00000000004ab30a in ngx_http_upstream_send_response (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:3034
#3  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:2535
#4  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#5  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#6  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#7  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 3, ngx_http_slice_body_filter (r=0x7fffc0, in=0x810100) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x7fffc0, in=0x810100) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x7fffc0, in=0x810100) at src/http/ngx_http_core_module.c:1874
#2  0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7fffc0, chain=0x810100) at src/http/ngx_http_upstream.c:4027
#3  0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x80e280) at src/event/ngx_event_pipe.c:551
#4  0x000000000045ee42 in ngx_event_pipe (p=0x80e280, do_write=1) at src/event/ngx_event_pipe.c:33
#5  0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:4123
#6  0x00000000004ac139 in ngx_http_upstream_send_response (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:3364
#7  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:2535
#8  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#9  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 9, __GI___writev (fd=15, iov=0x7fffffffca80, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
25	../sysdeps/unix/sysv/linux/writev.c: No such file or directory.
#0  __GI___writev (fd=15, iov=0x7fffffffca80, iovcnt=1) at ../sysdeps/unix/sysv/linux/writev.c:25
#1  0x0000000000463289 in ngx_writev (c=0x7ffff765f5e0, vec=0x7fffffffce80) at src/os/unix/ngx_writev_chain.c:189
#2  0x000000000046a77a in ngx_linux_sendfile_chain (c=0x7ffff765f5e0, in=0x7fff50, limit=2097152) at src/os/unix/ngx_linux_sendfile_chain.c:191
#3  0x00000000004bcd6d in ngx_http_write_filter (r=0x6f3470, in=0x810290) at src/http/ngx_http_write_filter_module.c:299
#4  0x00000000004be5f9 in ngx_http_chunked_body_filter (r=0x6f3470, in=0x810290) at src/http/modules/ngx_http_chunked_filter_module.c:115
#5  0x00000000004c43c3 in ngx_http_gzip_body_filter (r=0x6f3470, in=0x810290) at src/http/modules/ngx_http_gzip_filter_module.c:310
#6  0x00000000004c5dfd in ngx_http_postpone_filter (r=0x7fffc0, in=0x810290) at src/http/ngx_http_postpone_filter_module.c:91
#7  0x00000000004c6859 in ngx_http_ssi_body_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_ssi_filter_module.c:440
#8  0x00000000004cca18 in ngx_http_charset_body_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_charset_filter_module.c:557
#9  0x00000000004cf711 in ngx_http_sub_body_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_sub_filter_module.c:299
#10 0x00000000004d14f4 in ngx_http_addition_body_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_addition_filter_module.c:149
#11 0x00000000004d1aef in ngx_http_gunzip_body_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_gunzip_filter_module.c:185
#12 0x00000000004d42e1 in ngx_http_trailers_filter (r=0x7fffc0, in=0x810290) at src/http/modules/ngx_http_headers_filter_module.c:264
#13 0x000000000058b8c6 in ngx_http_lua_capture_body_filter (r=0x7fffc0, in=0x810290) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_capturefilter.c:133
#14 0x00000000005a21be in ngx_http_lua_body_filter (r=0x7fffc0, in=0x810290) at ../ngx_lua-0.10.26rc1/src/ngx_http_lua_bodyfilterby.c:249
#15 0x000000000042c643 in ngx_output_chain (ctx=0x8101a0, in=0x810100) at src/core/ngx_output_chain.c:230
#16 0x00000000004d573e in ngx_http_copy_filter (r=0x7fffc0, in=0x810100) at src/http/ngx_http_copy_filter_module.c:145
#17 0x00000000004c336b in ngx_http_range_body_filter (r=0x7fffc0, in=0x810100) at src/http/modules/ngx_http_range_filter_module.c:651
#18 0x00000000004d669e in ngx_http_slice_body_filter (r=0x7fffc0, in=0x810100) at src/http/modules/ngx_http_slice_filter_module.c:228
#19 0x00000000004830d3 in ngx_http_output_filter (r=0x7fffc0, in=0x810100) at src/http/ngx_http_core_module.c:1874
#20 0x00000000004ad767 in ngx_http_upstream_output_filter (data=0x7fffc0, chain=0x810100) at src/http/ngx_http_upstream.c:4027
#21 0x000000000045fbe1 in ngx_event_pipe_write_to_downstream (p=0x80e280) at src/event/ngx_event_pipe.c:551
#22 0x000000000045ee42 in ngx_event_pipe (p=0x80e280, do_write=1) at src/event/ngx_event_pipe.c:33
#23 0x00000000004ad9de in ngx_http_upstream_process_upstream (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:4123
#24 0x00000000004ac139 in ngx_http_upstream_send_response (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:3364
#25 0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:2535
#26 0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#27 0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#28 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#29 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 3, ngx_http_slice_body_filter (r=0x7fffc0, in=0x7fffffffd590) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x7fffc0, in=0x7fffffffd590) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x7fffc0, in=0x7fffffffd590) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000492e38 in ngx_http_send_special (r=0x7fffc0, flags=1) at src/http/ngx_http_request.c:3606
#3  0x00000000004aead3 in ngx_http_upstream_finalize_request (r=0x7fffc0, u=0x80de40, rc=0) at src/http/ngx_http_upstream.c:4654
#4  0x00000000004add16 in ngx_http_upstream_process_request (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:4217
#5  0x00000000004ada13 in ngx_http_upstream_process_upstream (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:4129
#6  0x00000000004ac139 in ngx_http_upstream_send_response (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:3364
#7  0x00000000004aa235 in ngx_http_upstream_process_header (r=0x7fffc0, u=0x80de40) at src/http/ngx_http_upstream.c:2535
#8  0x00000000004a7c7e in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1306
#9  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#10 0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#11 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 3, ngx_http_slice_body_filter (r=0x6f3470, in=0x0) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x6f3470, in=0x0) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x0) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000491bdf in ngx_http_writer (r=0x6f3470) at src/http/ngx_http_request.c:2880
#3  0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#4  0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#5  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#6  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#7  0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793

Breakpoint 3, ngx_http_slice_body_filter (r=0x6f3470, in=0x7fffffffd610) at src/http/modules/ngx_http_slice_filter_module.c:225
225	    ctx = ngx_http_get_module_ctx(r, ngx_http_slice_filter_module);
#0  ngx_http_slice_body_filter (r=0x6f3470, in=0x7fffffffd610) at src/http/modules/ngx_http_slice_filter_module.c:225
#1  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x7fffffffd610) at src/http/ngx_http_core_module.c:1874
#2  0x0000000000492e38 in ngx_http_send_special (r=0x6f3470, flags=1) at src/http/ngx_http_request.c:3606
#3  0x00000000004d6813 in ngx_http_slice_body_filter (r=0x6f3470, in=0x0) at src/http/modules/ngx_http_slice_filter_module.c:258
#4  0x00000000004830d3 in ngx_http_output_filter (r=0x6f3470, in=0x0) at src/http/ngx_http_core_module.c:1874
#5  0x0000000000491bdf in ngx_http_writer (r=0x6f3470) at src/http/ngx_http_request.c:2880
#6  0x0000000000490ec9 in ngx_http_run_posted_requests (c=0x7ffff765f5e0) at src/http/ngx_http_request.c:2457
#7  0x00000000004a7c8a in ngx_http_upstream_handler (ev=0x7bfb40) at src/http/ngx_http_upstream.c:1309
#8  0x000000000046a24b in ngx_epoll_process_events (cycle=0x6e4760, timer=318, flags=1) at src/event/modules/ngx_epoll_module.c:901
#9  0x0000000000459ef2 in ngx_process_events_and_timers (cycle=0x6e4760) at src/event/ngx_event.c:258
#10 0x0000000000467cf1 in ngx_worker_process_cycle (cycle=0x6e4760, data=0x0) at src/os/unix/ngx_process_cycle.c:793
```

### 调用栈解说

从上面的代码路径可以看到 `ngx_http_slice_range_variable` 都会被连续调用两次，对应的是这段配置。

```nginx
            proxy_cache_key   $uri$is_args$args$slice_range;
            proxy_set_header  Range $slice_range;
```

整体的流程是

1. 客户端发送请求
2. OpenResty 网关代理到上游服务器，带上了 `bytes=0-1023` 这样的请求头
3. 上游服务器返回请求的内容
4. 过滤上游服务器响应头，解析 `Content-Range: bytes 0-1023/146515` 这样的响应头, 设置下一个要发起的请求的开始值 ctx->start
5. 过滤上游服务器的响应体: 创建子请求，将 Content-Range 的范围修改为 `ctx->start, ctx->start + (off_t) slcf->size - 1`
    - 这里跳过了子请求：`if (ctx == NULL || r != r->main) { return ngx_http_next_body_filter(r, in); }`
    - 如果子请求还没有完成，那么不再创建子请求：`if (ctx->sr && !ctx->sr->done) { return rc; }`
    - 如果请求结束，发送LAST 结束请求： `if (ctx->start >= ctx->end) ngx_http_set_ctx(r, NULL, _http_slice_filter_module); ngx_http_send_special(r, NGX_HTTP_LAST); return rc; }`
6. 跳到步骤4，直到获取完整的文件
