---
layout: post
title: "resty.core not found"
description: "resty.core not found"
date: 2024-07-16
tags: [resty.core]
---


在搞 OpenResty 应用的测试有时候突然遇到这样的错误，平常跑得好好的突然不行了，这是为什么呢？
如果不是修改一点就执行测试很难想起来是怎么回事。

```shell
nginx: [alert] failed to load the 'resty.core' module (https://github.com/openresty/lua-resty-core); ensure you are using an OpenResty release from https://openresty.org/en/download.html (reason: module 'resty.core' not found:
	no field package.preload['resty.core']
	no file '/home/code/orinc/lua-nginx-module-plus/t/servroot/html/resty/core.lua'
	no file './resty/core.lua'
	no file './resty/core.lua'
	no file '/opt/luajit21-plus/share/luajit-2.1/resty/core.lua'
	no file '/opt/luajit21-plus/share/luajit-2.1/resty/core.ljbc'
	no file '/usr/local/share/lua/5.1/resty/core.lua'
	no file '/usr/local/share/lua/5.1/resty/core/init.lua'
	no file '/usr/local/share/lua/5.1/resty/core.ljbc'
	no file '/usr/local/share/lua/5.1/resty/core/init.ljbc'
	no file '/opt/luajit21-plus/share/lua/5.1/resty/core.lua'
	no file '/opt/luajit21-plus/share/lua/5.1/resty/core/init.lua'
	no file './resty/core.so'
	no file '/usr/local/lib/lua/5.1/resty/core.so'
	no file '/opt/luajit21-plus/lib/lua/5.1/resty/core.so'
	no file '/usr/local/lib/lua/5.1/loadall in /home/code/orinc/lua-nginx-module-plus/t/servroot/conf/nginx.conf:54
```


今天仔细分析了一下，发现是执行 ngx-build.sh 的时候设置了 NGX_BUILD_NO_DEBUG=1 导致的。
设置该选项后，执行 nginx 的 configure 命令时就不会加上 --with-debug 选项。而在没有配置
debug 模式的时候， lua-nginx-module 就不会将 ../lua-resty-core 添加到 lua_package_path 了。

这个我们可以 从 ngx_http_lua_util.c 文件中找到线索。

```C
#if !defined(LUA_DEFAULT_PATH) && (NGX_DEBUG)
#define LUA_DEFAULT_PATH "../lua-resty-core/lib/?.lua;"                      \
                         "../lua-resty-lrucache/lib/?.lua"
#endif
```
