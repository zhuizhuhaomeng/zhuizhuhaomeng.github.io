---
layout: post
title: "build curl that support http3 from source code"
description: "build curl that support http3 from source code"
date: 2023-08-18
tags: [curl, http3]
---

I build the curl on ubuntu-20.
You need to change the dependences if you want to build on other OS.

# install the dependences first

```shell
 sudo apt-get install libsystemd-dev
 sudo apt-get install cython
 sudo apt-get install cython3
 sudo apt-get install libcunit1-dev
 sudo apt-get install libev-dev
 sudo apt-get install libc-ares-dev
 sudo apt-get install libpython3-dev libpython2-dev
 sudo apt-get install libjansson-dev
```

# build from the source

```shell
#!/bin/bash

git clone --depth 1 -b openssl-3.0.0+quic https://github.com/quictls/openssl
git clone https://github.com/ngtcp2/nghttp3
git clone https://github.com/nghttp2/nghttp2
git clone https://github.com/ngtcp2/ngtcp2
git clone https://github.com/curl/curl

function build_openssl()
{
    cd openssl
    ./config enable-tls1_3 --prefix=/opt/curl-h3/openssl-quic
    make -j$(nproc) -v
    sudo make install
    sudo ln -s /opt/curl-h3/openssl-quic/lib64 /opt/curl-h3/openssl-quic/lib
    cd ..
}

function build_nghttp3()
{
    cd nghttp3
    autoreconf -fi
    ./configure --prefix=/opt/curl-h3/nghttp3 --enable-lib-only
    make -j$(nproc) -v
    sudo make install
    cd ..
}

function build_ngtcp2()
{
    cd ngtcp2
    autoreconf -fi
    ./configure PKG_CONFIG_PATH=/opt/curl-h3/openssl-quic/lib/pkgconfig:/opt/curl-h3/nghttp3/lib/pkgconfig LDFLAGS="-Wl,-rpath,/opt/curl-h3/openssl-quic/lib" --prefix=/opt/curl-h3/ngtcp2 --enable-lib-only
    make -j$(nproc) -v
    sudo make install
    cd ..
}

function build_nghttp2()
{
    cd nghttp2
    autoreconf -ivf
    CPPFLAGS="-I/opt/curl-h3/openssl-quic/include/" CFLAGS="-I/opt/curl-h3/openssl-quic/include/include/" LDFLAGS="-L/opt/curl-h3/openssl-quic/lib/ -lssl -lcrypto -Wl,-rpath,/opt/curl-h3/openssl-quic/lib" ./configure --prefix=/opt/curl-h3/nghttp2
    make -j$(nproc) -v
    sudo make install
    cd ..
}

function build_curl()
{
    cd curl
    autoreconf -fi
    LDFLAGS="-L/opt/curl-h3/libidn2/lib -L/opt/curl-h3/nghttp2/lib -L/opt/curl-h3/lib -L/opt/curl-h3/nghttp3/lib -L/opt/curl-h3/ngtcp2/lib -Wl,-rpath,/opt/curl-h3/libidn2/lib:/opt/curl-h3/lib:/opt/curl-h3/openssl-quic/lib:/opt/curl-h3/nghttp3/lib:/opt/curl-h3/ngtcp2/lib:/opt/curl-h3/nghttp2/lib" ./configure --with-openssl=/opt/curl-h3/openssl-quic --with-nghttp2=/opt/curl-h3/nghttp2 --with-nghttp3=/opt/curl-h3/nghttp3 --with-ngtcp2=/opt/curl-h3/ngtcp2 --with-libidn2 --prefix=/opt/curl-h3 --disable-static --enable-symbol-hiding --enable-ipv6 
    make -j$(nproc) -v
    sudo make install
    cd ..
}

build_openssl
build_nghttp3
build_ngtcp2
build_nghttp2
build_curl
```

# create a tarball

tar -czf curl-h3.tar.gz /opt/curl-h3

# install on the target machine

sudo tar -C / -xf curl-h3.tar.gz
