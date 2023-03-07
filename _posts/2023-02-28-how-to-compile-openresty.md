---
layout: post
title: "OpenResty 是如何打包的"
description: "OpenResty 打包过程"
date: 2023-02-28
tags: [OpenResty, packaging, How-To]
---

很多公司因为各种原因不能直接试用 OpenResty 官方提供的预编译的包。
但是由于对打包过程并不熟悉，因此他们可能不打包，也可能打包不规范。
因此这里介绍以下 OpenResty 官方是如何打包的。


# 下载 openresty 的 tar 包

```shell
cd ~
wget -O openresty-1.19.3.1.tar.gz  https://openresty.org/download/openresty-1.19.3.1.tar.gz
```

这个tar包是通过这个脚本生成的 https://github.com/openresty/openresty/blob/master/util/mirror-tarballs


# 下载打包脚本

```shell
cd ~
git clone  git@github.com:openresty/openresty-packaging.git
```

## 编译 rpm 包

```shell
cd ~
cp openresty-1.19.3.1.tar.gz ~/rpmbuild/SOURCES
cd openresty-packaging
cp rpm/SOURCES/* ~/rpmbuild/SOURCES
rpmbuild -bb rpm/SPECS/openresty.spec
```

# 编译 deb 包

```shell
cd ~
cp openresty-1.19.3.1.tar.gz openresty-packaging/deb
cd openresty-packaging/deb
make openresty-build
```

在编译的过程中可能会遇到一些错误，只要根据错误信息安装相应的软件包即可。

## 如何增加自己的模块

一个很简单的方法就是在修改mirror-tarballs，然后将模块加入openresty-1.19.3.1.tar.gz， 在 openresty.spec 的文件中增加--add-module指定要添加的模块。

如果是 deb 包，那么修改 deb 目录下相应的 文件即可。比如：
openresty-packaging/deb/openresty/debian/rules 这个文件。

## 如何给 spec 文件传递参数

假设 spec 中使用 enable-valgrind 来控制是否启用valgrind，如下所示。

```spec
%if 0%{?enable-valgrind}
 -I/usr/local/valgrind/include
%endif
```

我们可以通过 --define 来传递变量 enable-valgrind 的值

```shell
rpmbuild -bb  --define 'enable-valgrind 1'  openresty-plus-core.spec
```

### 通过 with 传递变量

```spec
# bcond_with
%bcond_with     feature_a
%bcond_without  feature_b


%if %{with feature_a}
    --with-feature_a \
%endif
%if %{with feature_b}
    --with-feature_b \
%endif
```

bcond_with 定义的变量默认是关闭的，bcond_without 定义的变量默认是 打开的。

1. 如果要开启某一个功能，应该是 `rpmbuild --with=feature_a`
1. 如果要关闭一个功能，应该用 `rpmbuild --without=feature_b`
