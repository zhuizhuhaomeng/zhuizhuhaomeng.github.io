---
layout: post
title: "如何运行 systemtap 的测试用例"
description: ""
date: 2023-06-03
tags: [systemtap, test]
---

参与一个软件的开发，提交 PR 就需要能够通过相关的用例集合。因此跑用例是一个基础的步骤。
这里主要记录如何跑 systemtap 的测试用例。

# 安装依赖

测试的时候要用到 runtest 这个组件，需要安装 dejagnu。
编译的时候依赖 elfutils-devel, 因此需要安装改组件。
其它的 python3 的系统应该默认就存在了。

```shell
sudo yum install -y dejagnu
sudo yum install -y elfutils-devel

# or apt

sudo apt-get install dejagnu
sudo apt-get install libdw-dev
sudo apt-get install lib-dev
sudo apt-get install libdebuginfod-dev
sudo apt-get install elfutils
```

# 编译

```shell
./configure \
        --disable-docs --disable-publican \
        --with-python3 \
        --without-nss \
        --without-openssl \
        --without-avahi \
        --without-bpf \
        --without-python2-probes \
        --without-python3-probes \
        --disable-refdocs \
	CC="ccache gcc -fdiagnostics-color=always" \
	CXX="ccache g++ -fdiagnostics-color=always" \

make -j10
sudo make install
```

# 测试

以下的命令用例跑单个用例文件。

```shell
make installcheck RUNTESTFLAGS=tls.exp
```
