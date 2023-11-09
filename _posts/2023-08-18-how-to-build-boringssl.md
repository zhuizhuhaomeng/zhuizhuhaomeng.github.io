---
layout: post
title: "how to build boringssl for HTTP3"
description: "build boringssl for HTTP3"
date: 2023-09-24
tags: [boringssl, HTTP3]
---

# compile command

```shell
git clone git@github.com:google/boringssl
cd boringssl

mkdir -p build
cd build
cmake -DBUILD_SHARED_LIBS=1 ..
make -j$(nproc)
make install
cd ..
rm -fr install/lib
cd install && ln -s lib64 lib && cd ..
tar -czf boringssl-$(date '+%Y%m%d').tar.gz install
sudo mkdir -p /opt/boringssl && sudo tar -C /opt/boringssl -xf boringssl-$(date '+%Y%m%d').tar.gz --strip-components=1
```

# installation command

```shell
sudo mkdir -p /opt/ssl && sudo tar -C /opt/ssl -xf boringssl-$(date '+%Y%m%d').tar.gz --strip-components=1
```
