---
layout: post
title: "How to build envoy"
description: "How to build envoy"
date: 2025-03-12
modified: 2025-03-12
tags: [Gateway, Envoy]
---

# download source code

```shell
git clone git@github.com:envoyproxy/envoy.git
cd envoy
git checkout v1.32.3
```

# check bazel version

```shell
$ cat .bazelversion
6.5.0
```

# download bazel version

Find bazel in https://github.com/bazelbuild/bazel/releases/tag/7.5.0.
Change the version to your own.

```shell
wget https://github.com/bazelbuild/bazel/releases/download/6.5.0/bazel-6.5.0-linux-x86_64
chmod a+x bazel-6.5.0-linux-x86_64
mv bazel-6.5.0-linux-x86_64 /usr/bin
```
