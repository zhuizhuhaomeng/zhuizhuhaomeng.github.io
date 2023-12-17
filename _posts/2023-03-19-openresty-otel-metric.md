---
layout: post
title: "为 OpenResty 编译 OpenTelemetry"
description: "为 OpenResty 编译 OpenTelemetry"
date: 2023-03-19
tags: [OpenResty, Nginx, OpenTelemetry]
---

使用 OTEL 官方的 nginx otel 模块，编译非常的麻烦，2023-05 以前一直编译不通过，不知道现在是否改善了。

目前 Nginx 官方的搞了 otel 模块，性能好而且编译更容易了。
编译使用如下的脚本：


```shell
git clone git@github.com:nginxinc/nginx-otel.git
cd nginx-otel
mkdir build
cd build
cmake -DNGX_OTEL_NGINX_BUILD_DIR=/var/code/openresty-1.25.3.1rc1/build/nginx-1.25.3/objs -DOPENSSL_ROOT_DIR=/usr/local/openresty/openssl111 -DZLIB_ROOT=/usr/local/openresty/zlib ..
make -j10
```
