---
layout: post
title: "How to install debuginfo"
description: "How to install debuginfo"
date: 2023-04-23
tags: [debuginfo, debuginfo-install, rpm, deb, debuginfod, gdb]
---

# 为什么需要调试信息

对于大部分 Linux 用户来说，安装调试信息包并没有任何意义，因为他们只是使用软件而不会去调试软件故障。

作为开发者来说，在遇到崩溃等软件故障时，安装调试信息包是一个必不可少的步骤。
但是不同的发行版本的操作系统的安装方法又不一样，确实让人头疼。

# Fedora 序列

如果不知道某一个软件对应的软件包的名字，可以通过 `rpm -qf $(which $exe)` 这样的命令来查询软件包的名称。

```shell
rpm -qf $(which nginx)
```

反过来，如果你要查询一个软件包包含了哪些文件，可以使用

```shell
rpm -qpl the-pkg-name.rpm

#或者

rpm -ql the-pkg-name
```

Fedora 安装调试信息报的方式非常的简单，只要使用 DNF 默认已经安装好的 debuginfo-install 插件即可。 
通过运行命令 `sudo dnf debuginfo-install $package` 来安装：

```shell
sudo dnf debuginfo-install openresty
```

如果是安装内核的调试信息包，可以使用命令如下：

```shell
sudo dnf debuginfo-install kernel-$(uname -r)
```

如果想下载而不是要安装，可以使用如下的命令

```shell
dnf download openresty
```

# CentOS 序列

CentOS 的低版本可以通过 yum-utils 软件包中的 debuginfo-install 命令来安装。

```shell
sudo yum install -y yum-utils
sudo debuginfo-install --enablerepo="*" openresty
```

如果不知道某一个软件对应的软件包的名字，可以通过 `rpm -qf $(which $exe)` 这样的命令来查询软件包的名称。

```shell
rpm -qf $(which nginx)
```

CentOS 的高版本也支持了 DNF，跟 Fedora 序列一样安装的方法，但是需要增加 --enablerepo 选项。 --enablerepo 可以指定具体的仓库，但是如果你不清楚仓库的情况下，用 "*" 指代所有的仓库，这时候会更新所有仓库元数据，执行会慢很多。

```shell
sudo dnf --enablerepo="*" debuginfo-install openresty
```

如果是安装内核的调试信息包，可以使用命令如下：

```shell
sudo dnf debuginfo-install kernel-$(uname -r)
```

如果想下载而不是要安装，可以使用如下的命令

```shell
yum download openresty
```

如果想要下载的源码，可以使用

```shell
yumdownloader --source coreutils
```

# Rocky 序列

Rocky 序列相当于 CentOS 的高版本。Rocky8 对应 CentOS8，Rocky 9 对应 CentOS 9.

# Debian 序列

Debian 序列需要手动添加调试符号的仓库，命令如下：

```shell
sudo tee /etc/apt/sources.list.d/debug.list << EOF
deb http://deb.debian.org/debian-debug/ $(lsb_release -cs)-debug main
deb http://deb.debian.org/debian-debug/ $(lsb_release -cs)-proposed-updates-debug main
EOF
sudo apt update
```

然后我们可以通过 `sudo apt-get install $dbg-pkg` 的方式来安装：

```shell
sudo apt-get install openresty-dbg
```

有的软件后缀可能是 dbgsym，因此使用如下命令：

```shell
sudo apt-get install openresty-dbgsym
```

也可以使用 debian-goodies 软件包中的 find-dbgsym-packages 命令来找到正确的调试信息的包名：

```shell
sudo apt install debian-goodies
find-dbgsym-packages $(which nginx)
```

如果要安装当前运行内核的调试信息，可以使用：

```shell
sudo apt install linux-image-$(uname -r)-dbg
```

如果不知道某一个软件对应的软件包的名字，可以通过 `dpkg -S $(which $exe)` 这样的命令来查询软件包的名称。

```shell
dpkg -S $(which nginx)
```

# Ubuntu

Ubuntu 系统上，你需要安装调试符号的签名密钥并手动添加调试符号的仓库。

```shell
sudo apt update
sudo apt install ubuntu-dbgsym-keyring
sudo tee /etc/apt/sources.list.d/debug.list << EOF
deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse
EOF
sudo apt update
```

如果不知道某一个软件对应的软件包的名字，可以通过 `dpkg -S $(which $exe)` 这样的命令来查询软件包的名称。

```shell
dpkg -S $(which nginx)
```

如果想知道一个软件包包含哪些文件，可以使用命令

```shell
dpkg -c the-package-name.deb

# 或者

dpkg -L the-package-name
```

和 Debian 类似，有些调试包以 -dbg 为后缀，有些以 -dbgsym 作为后缀。
有些包甚至更奇怪，比如 perl-base 的调试信息在 perl-debug 中。

```shell
sudo apt install python3-dbg
sudo apt install coreutils-dbgsym
```

也可以使用 debian-goodies 软件包中的 find-dbgsym-packages 命令来找到正确的调试信息的包名：

```shell
sudo apt install debian-goodies
find-dbgsym-packages $(which nginx)
```

如果要安装当前运行内核的调试信息，可以使用：

```shell
sudo apt install linux-image-$(uname -r)-dbgsym
```

如果想下载而不是要安装，可以使用如下的命令

```shell
apt-get download python3.9-minimal
```

更多信息查看官方文档 [调试信息包](https://wiki.ubuntu.com/Debug%20Symbol%20Packages)

## 内核软件包

apt-get -y install linux-image
apt-get -y install linux-headers-$(uname -r)
apt-get -y install linux-image-$(uname -r)-dbgsym
apt-get -y install linux-libc-dev

# 使用 debuginfod 自动下载调试信息

因为下载 debuginfo 包是很麻烦，而且不熟悉的人还经常下载错了版本，因此衍生了 debuginfod 的服务。
特别是如果你想在宿主机上调试 docker 容器进程的时候，那么更应该使用 debuginfod 的服务。不过有个缺点是不是所有的发行版本都支持 debuginfod，有些发现版本虽然支持了，但是老版本的系统是不支持的。比如 CentOS 7 的 GDB 默认就不支持。

我们只要先设置环境变量 `export DEBUGINFOD_URLS="xxx"`，然后执行 gdb，readelf 等命令的时候就会自动下载相关的调试信息了。

1. debian 的链接为：https://debuginfod.debian.net/
1. Ubuntu 的链接为：https://debuginfod.ubuntu.com/
1. CentOS 的链接为：https://debuginfod.centos.org/
1. Fedora 的链接为：https://debuginfod.fedoraproject.org/
1. Arch Linux 的链接为：https://debuginfod.archlinux.org/

举例如下：

```shell
$ export DEBUGINFOD_URLS="https://debuginfod.debian.net"
$ gdb -q /usr/bin/cat
Reading symbols from /usr/bin/cat...
Downloading separate debug info for /usr/bin/cat...
Reading symbols from /home/ljl/.cache/debuginfod_client/e6afa43e1e280bd06c018f541c7ae46a2ebda83c/debuginfo...
Downloading separate debug info for /home/ljl/.cache/debuginfod_client/e6afa43e1e280bd06c018f541c7ae46a2ebda83c/debuginfo...
(gdb)
```

## 参考连接

1. https://wiki.debian.org/Debuginfod
1. https://ubuntu.com/server/docs/service-debuginfod
1. https://fedoraproject.org/wiki/Debuginfod
1. https://wiki.archlinux.org/title/Debuginfod
1. https://debuginfod.centos.org/
