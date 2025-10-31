---
layout: post
title: "Inline SSL"
description: "Inline SSL"
date: 2025-09-15
tags: [Cross compile]
---

Go 软件的一个好处就是没有依赖那么多的系统库，因此不会因为系统稍微变化就导致编译不过或者运行不起来的问题。

有时候我们为了减少 C 软件程序跟系统库的冲突，选择将一些库以静态编译的方式内联编译到目标软件。

将 OpenSSL 编译进目标软件就是一个很常见的例子。

```shell
wget https://github.com/openssl/openssl/releases/download/openssl-3.5.2/openssl-3.5.2.tar.gz
tar -xf openssl-3.5.2.tar.gz
cd openssl-3.5.2
./config -fPIC no-shared no-docs no-unit-test --libdir=lib \
    --prefix=$PWD/../install \
make -j$(nproc)
make install
cd ..

CFLAGS="-g -O2 -DHIREDIS_USE_FREE_LISTS -I$PWD/install/include"  LIBS="-ldl -pthread" \
LDFLAGS="-L$PWD/install/lib/ -Wl,--whole-archive $PWD/install/lib/libssl.a $PWD/install/lib/libcrypto.a -Wl,--no-whole-archive" \
    make -j`nproc` PREFIX=${prefix} \
        OPENSSL_PREFIX=$PWD/install \
        V=1
```
