---
layout: post
title: "Linux coredump 配置"
description: ""
date: 2023-08-18
tags: [coredump]
---

# apport

## 默认的 core_pattern
ubuntu 22 的 core pattern 默认如下：

```shell
$ cat /proc/sys/kernel/core_pattern
|/usr/share/apport/apport -p%p -s%s -c%c -d%d -P%P -u%u -g%g -- %E
```

## 日志文件

在默认配置下，如果进程产生崩溃，那么生成的日志会在 /var/log/apport* 的文件里面。

比如：

```shell
/var/log/apport.log
/var/log/apport.log.*.gz
```

## 内存转储

如果生成了 core 文件，那么 core 文件位于 /var/crash 目录下。
我们可以使用 apport-unpack 工具解压这个文件，然后就可以用查看该文件的内容。
甚至可以用 gdb 调试了。

比如：

```shell
mkdir /root/coredump,
apport-unpack /var/crash/_usr_bin_nginx.0.crash /root/coredump,
```

解压后得到这样的文件

```shell
$ ls /root/coredump
/root/coredump/
/root/coredump/ProcMaps
/root/coredump/CoreDump
/root/coredump/ExecutableTimestamp
/root/coredump/ProblemType
/root/coredump/ProcEnviron
/root/coredump/CrashCounter
/root/coredump/Architecture
/root/coredump/ProcStatus
/root/coredump/Date
/root/coredump/ExecutablePath
/root/coredump/Uname
/root/coredump/UserGroups
/root/coredump/DistroRelease
/root/coredump/ProcCmdline
/root/coredump/ProcCwd
/root/coredump/Signal
```
# apport 服务

如果不想让 apport 管理 crash，那么应该停止该服务

```shell
systemctl stop apport
systemctl disable apport
```

# 修改 core_pattern

## 查看当前的 kernel.core_pattern 参数值：

```shell
sysctl kernel.core_pattern
```

这将显示当前的 kernel.core_pattern 参数值。

## 临时更改 kernel.core_pattern 参数值：

```shell
sudo sysctl -w kernel.core_pattern=<new_pattern>
```

将 <new_pattern> 替换为你想要的新的 core dump 文件路径和命名模板。这种更改只是临时的，重启系统后将恢复为默认值。

## 永久更改 kernel.core_pattern 参数值：

在文件 /etc/sysctl.conf 的末尾添加或修改以下行：

``` shell
echo kernel.core_pattern = <new_pattern> | sudo tee /etc/sysctl.conf
```

```shell
sudo sysctl -p
```

这将使更改生效并永久保存在系统中。

注意，系统可能会有其它的服务修改 kernel.core_pattern, 因此禁用这些服务器。比如 ubuntu 的 apport。

## kernel.core_pattern 占位符说明

在 kernel.core_pattern 参数中，可以使用一些占位符来定义生成 core dump 文件时的文件名模板。这些占位符会被内核替换为相应的值。以下是一些常用的占位符及其意义：

%p：进程 ID（PID）。
%u：实际用户 ID（UID）。
%g：实际组 ID（GID）。
%s：信号编号，导致 core dump 的信号。
%t：时间戳，表示生成 core dump 的时间。
%h：主机名。
%e：可执行文件的名称。
%E：可执行文件的路径。
使用这些占位符可以方便地在生成的 core dump 文件名中包含相关的信息，以便更好地标识和组织这些文件。

例如，如果 kernel.core_pattern 参数设置为 /var/core/core.%e.%p.%t，那么生成的 core dump 文件将会以以下格式命名：

```text
core.<可执行文件名>.<进程 ID>.<时间戳>
```

这样的命名模板可以使生成的 core dump 文件具有清晰的标识，并且可以根据特定的信息进行过滤和处理。

注意：不同的操作系统和内核版本可能支持不同的占位符，具体的支持情况可以参考 Linux 内核文档或相关文档资源。

# systemctl 设置 core 文件大小

如果要临时生效，那么可以使用下面的命令行

```shell
ulimit -c unlimited
```

## 在 systemd 配置文件中修改

### 首先切换编辑器为熟悉的 vim

```shell
export SYSTEMD_EDITOR="/usr/bin/vim"
```

### 打开服务配置文件

```shell
sudo -E systemctl edit <service-name>
```

将<service-name>替换为你要编辑的服务的名称。

在打开的编辑器中，添加以下内容来指定核心文件大小的限制：

```conf
[Service]
LimitCORE=<size>
```

将<size>替换为所需的核心文件大小限制。大小可以使用以下单位表示：K（千字节）、M（兆字节）、G（吉字节）等。

例如，要将 OpenResty 核心文件大小限制为 100 兆字节，可以添加以下行：

```conf
[Service]
LimitCORE=100M
```

保存并关闭文件，可以看到编辑器提示新增内容保存在 /etc/systemd/system/openresty.service.d/override.conf。

重新加载 systemd 配置：

```shell
sudo systemctl daemon-reload
```

重新启动相应的服务以应用新的核心文件大小限制：

```shell
sudo systemctl restart <service-name>
```
将<service-name>替换为你编辑过的服务的名称。

通过上述步骤，你可以在 systemd 服务配置文件中指定特定服务的核心文件大小限制。请注意，该限制可能会受到系统配置和安全策略的限制

# 为什么不生成 coredump

1. /proc/sys/kernel/core_pattern 配置没有修改
1. 进程没有目标目录的写入权限 (master/worker 这种模式要求 master/worker 都有写权限)
1. CORE 大小被限制为 0
