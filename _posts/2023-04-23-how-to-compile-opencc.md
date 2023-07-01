---
layout: post
title: "How to compile opencc"
description: "How to compile opencc"
date: 2023-04-23
tags: [opencc, rpm, deb, spec]
---
# 序列文章

[怎么编译 OpenResty](./2023-02-28-how-to-compile-openresty.md)
[怎么编译 OpenCC](./2023-04-23-how-to-compile-opencc.md)

# 更新 cmake

如果机器上的cmake比较老，需要更新cmake，那么最快的方式就是使用已经编译好的包。

```shell
wget https://github.com/Kitware/CMake/releases/download/v3.26.3/cmake-3.26.3-linux-x86_64.sh
bash cmake-3.26.3-linux-x86_64.sh --prefix=/usr --skip-license
```

# opencc 找不到 libopencc 库

使用下面的命令编译 opencc，安装后 opencc 无法正常使用，报错 libopencc.so 找不到。

编译命令如下，

```shell
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr ../
make -j8
sudo make install
```

报错信息如下

```shell
$ opencc
opencc: error while loading shared libraries: libopencc.so.1.1: cannot open shared object file: No such file or directory
```

不知道为什么 opencc 不去搜索默认的 /usr/local/lib/ 目录, 使用 strace 命令跟踪结果如下。
从这个输出可以看到 opencc 会查找 /lib64/libopencc.so.1.1 的文件。这里正确的解法是修改 CMakeLists.txt。
但是对于 CMake 不熟悉，不知道怎么修改，因此就修改编译选项来得容易多了。

```shell
$ strace opencc
execve("/usr/bin/opencc", ["opencc"], 0x7ffec9bf4c40 /* 52 vars */) = 0
brk(NULL)                               = 0xd2c000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffe7c2b9f70) = -1 EINVAL (Invalid argument)
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=63295, ...}) = 0
mmap(NULL, 63295, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f5002971000
close(3)                                = 0
openat(AT_FDCWD, "/lib64/glibc-hwcaps/x86-64-v3/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/glibc-hwcaps/x86-64-v3", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/glibc-hwcaps/x86-64-v2/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/glibc-hwcaps/x86-64-v2", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/tls/haswell/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/tls/haswell/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/tls/haswell/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/tls/haswell", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/tls/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/tls/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/tls/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/tls", {st_mode=S_IFDIR|0555, st_size=6, ...}) = 0
openat(AT_FDCWD, "/lib64/haswell/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/haswell/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/haswell/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/haswell", 0x7ffe7c2b9170)  = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64/x86_64", 0x7ffe7c2b9170)   = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/lib64", {st_mode=S_IFDIR|0555, st_size=98304, ...}) = 0
openat(AT_FDCWD, "/usr/lib64/glibc-hwcaps/x86-64-v3/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/glibc-hwcaps/x86-64-v3", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/glibc-hwcaps/x86-64-v2/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/glibc-hwcaps/x86-64-v2", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/tls/haswell/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/tls/haswell/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/tls/haswell/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/tls/haswell", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/tls/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/tls/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/tls/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/tls", {st_mode=S_IFDIR|0555, st_size=6, ...}) = 0
openat(AT_FDCWD, "/usr/lib64/haswell/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/haswell/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/haswell/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/haswell", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/x86_64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64/x86_64", 0x7ffe7c2b9170) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib64/libopencc.so.1.1", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
stat("/usr/lib64", {st_mode=S_IFDIR|0555, st_size=98304, ...}) = 0
writev(2, [{iov_base="opencc", iov_len=6}, {iov_base=": ", iov_len=2}, {iov_base="error while loading shared libra"..., iov_len=36}, {iov_base=": ", iov_len=2}, {iov_base="libopencc.so.1.1", iov_len=16}, {iov_base=": ", iov_len=2}, {iov_base="cannot open shared object file", iov_len=30}, {iov_base=": ", iov_len=2}, {iov_base="No such file or directory", iov_len=25}, {iov_base="\n", iov_len=1}], 10opencc: error while loading shared libraries: libopencc.so.1.1: cannot open shared object file: No such file or directory
) = 122
exit_group(127)                         = ?
+++ exited with 127 +++
```

# 编译成功的命令

```shell
mkdir build
cd build
cmake -DCMAKE_INSTALL_PREFIX:PATH=/usr -DLIB_SUFFIX=64 ../
make -j8
sudo make install
```


# 打包问题如果解决

很多软件可以配置的参数很多，而有些软件的配置脚本并没有适配各种发行版本的操作系统，
经常导致在特定的系统上打包失败。而打包这种非高频操作经常是查不到资料，而各种 CMake 脚本的调整选项，
configure 的选项等要修复门槛比较高。

因此遇到打包问题，我们最好参考各个发行版本打包的脚本，基本上他们可以打包成功，那么把相关的参数，补丁拷贝过来就可以了。

比如：

RPM 打包参考 Fedora 官方的打包脚本，链接为 [https://src.fedoraproject.org](https://src.fedoraproject.org/projects/rpms/%2A)

RPM 打包也可以参考 CentOS 官方的打包脚本，链接为 [https://git.centos.org/projects/rpms/%2A](https://git.centos.org/projects/rpms/%2A)

Deb 打包参考 debian 官方的打包脚本，链接为 [https://salsa.debian.org/public](https://salsa.debian.org/public)
