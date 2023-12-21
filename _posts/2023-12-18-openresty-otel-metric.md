---
layout: post
title: "为 OpenResty 编译 OpenTelemetry"
description: "为 OpenResty 编译 OpenTelemetry"
date: 2023-12-18
tags: [OpenResty, Nginx, OpenTelemetry]
---

使用 OTEL 官方的 nginx otel 模块，编译非常的麻烦，2023-05 以前一直编译不通过，不知道现在是否改善了。

目前 Nginx 官方的搞了 otel 模块，性能好而且编译更容易了。
编译步骤如下：


1. 安装依赖

```shell
sudo apt-get install libc-ares2 libc-ares-dev
sudo apt-get install libre2-dev
sudo apt-get install openresty-openssl111 openresty-openssl111-dev
sudo apt-get install openresty-pcre openresty-pcre2-dev
sudo apt-get install openresty-zlib openresty-zlib-dev
```

2. 下载源码

```shell
git clone git@github.com:nginxinc/nginx-otel.git
cd nginx-otel
```

3. 打上补丁

```patch
diff --git a/CMakeLists.txt b/CMakeLists.txt
index 96ac4cd..ae3828b 100644
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -124,6 +124,7 @@ target_include_directories(ngx_otel_module PRIVATE
     ${NGX_OTEL_NGINX_DIR}/src/http/modules
     ${NGX_OTEL_NGINX_DIR}/src/http/v2
     ${NGX_OTEL_NGINX_DIR}/src/http/v3
+    /usr/local/openresty/pcre/include
     ${PROTO_OUT_DIR})

 target_link_libraries(ngx_otel_module
```

4. 执行编译

```shell
mkdir build
cd build
cmake -DNGX_OTEL_NGINX_BUILD_DIR=/var/code/openresty-1.25.3.1rc1/build/nginx-1.25.3/objs -DOPENSSL_ROOT_DIR=/usr/local/openresty/openssl111 -DZLIB_ROOT=/usr/local/openresty/zlib ..
make -j10
```
