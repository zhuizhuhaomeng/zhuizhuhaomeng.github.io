---
layout: post
title: "OpenResty 是如何打包的"
description: "OpenResty 打包过程"
date: 2023-02-28
tags: [OpenResty, packaging, How-To]
---
# 序列文章

[怎么编译 OpenResty](./2023-02-28-how-to-compile-openresty.md)
[怎么编译 OpenCC](./2023-04-23-how-to-compile-opencc.md)

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


# 打包问题如果解决

很多软件可以配置的参数很多，而有些软件的配置脚本并没有适配各种发行版本的操作系统，
经常导致在特定的系统上打包失败。而打包这种非高频操作经常是查不到资料，而各种 CMake 脚本的调整选项，
configure 的选项等要修复门槛比较高。

因此遇到打包问题，我们最好参考各个发行版本打包的脚本，基本上他们可以打包成功，那么把相关的参数，补丁拷贝过来就可以了。

比如：

RPM 打包参考 Fedora 官方的打包脚本，链接为 [https://src.fedoraproject.org](https://src.fedoraproject.org/projects/rpms/%2A)

RPM 打包也可以参考 CentOS 官方的打包脚本，链接为 [https://git.centos.org/projects/rpms/%2A](https://git.centos.org/projects/rpms/%2A)

Deb 打包参考 debian 官方的打包脚本，链接为 [https://salsa.debian.org/public](https://salsa.debian.org/public)

## spec 文件指定 strip

```text
/bin/strip: Unable to recognise the format of the input file
`/usr/local/xdp-tools/lib/bpf/xsk_def_xdp_prog.o'
```

编译的时候遇到了这样的错误, 这个错误是因为这个 .o 文件是 llvm 编译的，而 strip 却
试用了 /usr/bin/strip 这个工具，因此要换成 llvm-stip. 只要再 spec 文件中配置如下语句即可。

```shell
%define __strip %{llvm_prefix}/bin/llvm-strip
```

# mariner 系统上编译的注意事项

微软的 mariner 系统上编译会出现各种奇怪的问题。

1. rpm-build 太新导致缺少 debugedit 和 find-debuginfo.sh
1. binutils 需要单独安装, 而通常情况下 gcc 是依赖 binutils的。我们可以通过 `rpm -qR gcc | grep binutils` 确认这点
1. glibc-headers 和 kernel-headers 也需要单独安装，而其它系统 glibc 是依赖 kernel-headers
1. dnf dnf-plugins-core 也需要单独安装，系统默认不带这些命令

话不多说，具体需要执行的命令如下：

```shell
yum install -y binutils glibc-headers kernel-headers dnf dnf-plugins-core time

ln -s /usr/bin/find-debuginfo /usr/lib/rpm/find-debuginfo.sh
ln -s /usr/bin/debugedit /usr/lib/rpm/debugedit
```
