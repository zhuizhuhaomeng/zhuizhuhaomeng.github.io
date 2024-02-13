---
layout: post
title: "Add Bridge Interface with nmcli"
description: "Add Bridge Interface with nmcli"
date: 2024-02-13
tags: [nmcli,bridge]
---

使用 qemu 创建的虚拟机，宿主机无法和虚拟机互通。

经过分析发现如果将虚拟机和宿主机的网卡都添加到网桥，那么就可以实现虚拟机和宿主机的互通了。
那么如何在将宿主机添加网桥呢？
可以使用下面的命令，这里假设物理网卡的名称是 `enp4s0`。

这里同时给网桥配置了静态 IP 地址。

```shell
dev=enp4s0
sudo nmcli connection add ifname br0 type bridge con-name br0
sudo nmcli connection add type bridge-slave ifname $dev master br0
sudo nmcli connection show
sudo nmcli connection modify eth0 ipv4.address 192.168.0.203/24 ipv4.gateway 192.168.0.1 ipv4.dns 192.168.0.1,4.4.4.4
sudo nmcli connection down $dev
sudo nmcli connection up $dev
sudo nmcli connection up br0
```

这个只在 Ubuntu 22.04 上配置过，其它系统可能会有不一样的命令参数或者不支持 nmcli。
