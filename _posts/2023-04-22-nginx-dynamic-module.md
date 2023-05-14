---
layout: post
title: "How to compile nginx dynamic module"
description: "How to compile nginx dynamic module"
date: 2023-04-22
tags: [nginx, dynamic module]
---

社区有人反馈说把 lua-nginx-module 模块编译成动态模块后，在加载模块的时候失败了。报错信息是： `unknown directive "lua_ssl_trusted_certificate" in /etc/nginx/conf.d/crowdsec_nginx.conf`。
具体的问题可以参考 [lua-nginx-module/issues/2180](https://github.com/openresty/lua-nginx-module/issues/2180)。

为什么会造成编译的模块没有办法使用呢? 归根结底还是编译参数搞错了。
如果是编译 Nginx 的同时编译动态模块，那么是不会出现这样的错误的。因为编译 Nginx 的时候就知道自己是否需要支持 HTTPS 了。

但是很多时候, Nginx 和 Nginx 动态模块分开编译的。比如使用了 官方的 OpenResty 二进程程序，现在想增加编译一个第三方模块，那边就没有必要完整编译整个 OpenResty。

编译动态模块的时候我们最好是参考 [Nginx 官方的文档](https://www.nginx.com/blog/compiling-dynamic-modules-nginx-plus/)。

官方这里的例子是：

```shell
$ cd nginx-1.11.5/
$ ./configure --with-compat --add-dynamic-module=../nginx-hello-world-module
$ make modules
```

这里重点的参数是 `--with-compat` 这个值。通过搜索代码我们可以看到 只要定义了 NGX_COMPAT 和 NGX_HTTP_SSL 是或的关系，只要定义了其中之一就可以了。

比如下面的例子：

```C
#if (NGX_HTTP_SSL || NGX_COMPAT)
    ngx_str_t                        *ssl_servername;
#if (NGX_PCRE)
    ngx_http_regex_t                 *ssl_servername_regex;
#endif
#endif
```

但是，官方的编译参数是其实也不完美，我们怎么传递编译参数更好呢？
这个就要知道原来编译的 Nginx 使用的是什么样的参数。

要知道原来的编译参数是啥，就得利用 `Nginx -V` 这个命令了。

比如:

```shell
$ /usr/local/openresty/bin/openresty -V
nginx version: openresty/1.19.9.1
built by gcc 8.5.0 20210514 (Red Hat 8.5.0-16) (GCC) 
built with OpenSSL 1.1.1s  1 Nov 2022
TLS SNI support enabled
configure arguments: --prefix=/usr/local/openresty/nginx --with-cc-opt='-O2 -DNGX_LUA_ABORT_AT_PANIC -I/usr/local/openresty/zlib/include -I/usr/local/openresty/pcre/include -I/usr/local/openresty/openssl111/include' --add-module=../ngx_devel_kit-0.3.1 --add-module=../echo-nginx-module-0.62 --add-module=../xss-nginx-module-0.06 --add-module=../ngx_coolkit-0.2 --add-module=../set-misc-nginx-module-0.32 --add-module=../form-input-nginx-module-0.12 --add-module=../encrypted-session-nginx-module-0.08 --add-module=../srcache-nginx-module-0.32 --add-module=../ngx_lua-0.10.20 --add-module=../ngx_lua_upstream-0.07 --add-module=../headers-more-nginx-module-0.33 --add-module=../array-var-nginx-module-0.05 --add-module=../memc-nginx-module-0.19 --add-module=../redis2-nginx-module-0.15 --add-module=../redis-nginx-module-0.3.7 --add-module=../ngx_stream_lua-0.0.10 --with-ld-opt='-Wl,-rpath,/usr/local/openresty/luajit/lib -L/usr/local/openresty/zlib/lib -L/usr/local/openresty/pcre/lib -L/usr/local/openresty/openssl111/lib -Wl,-rpath,/usr/local/openresty/zlib/lib:/usr/local/openresty/pcre/lib:/usr/local/openresty/openssl111/lib' --with-cc='ccache gcc -fdiagnostics-color=always' --with-pcre-jit --with-stream --with-stream_ssl_module --with-stream_ssl_preread_module --with-http_v2_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --with-http_stub_status_module --with-http_realip_module --with-http_addition_module --with-http_auth_request_module --with-http_secure_link_module --with-http_random_index_module --with-http_gzip_static_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-threads --with-compat --with-stream --with-http_ssl_module
```

我们可以把上面的命令输出的配置参数作为编译动态模块的配置参数，这样就可以保证编译的动态模块是完全兼容的。

如果只是指定 `--with-compat` 这个参数，但是没有指定 `--with-http_ssl_module` 这个参数还是会有问题。因为 lua-nginx-module 并没有像 nginx 的核心代码一样存在 `#if (NGX_HTTP_SSL || NGX_COMPAT)` 这样的兼容处理。
