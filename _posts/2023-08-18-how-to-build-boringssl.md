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
cmake ..
make -j$(nproc)
make install
cd ..
tar -czf boringss-$(date '+%Y%m%d').tar.gz install
```

# installation command

```shell
sudo mkdir -p /opt/ssl && sudo tar -C /opt/ssl -xf boringssl-20230902.tar.gz --strip-components=1
```
