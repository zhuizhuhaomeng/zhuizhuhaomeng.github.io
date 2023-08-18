---
layout: post
title: "如何更改启动的内核"
description: "如何更改启动的内核"
date: 2023-07-27
tags: [boot, grubby]
---


# 查看当前安装的内核

```shell
$ sudo grubby --info=ALL                         
index=0
kernel="/boot/vmlinuz-4.18.0-477.10.1.el8_8.x86_64+debug"
args="ro crashkernel=auto resume=/dev/mapper/rl-swap rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet console=ttyS0,115200n8 console=tty1"
root="/dev/mapper/rl-root"
initrd="/boot/initramfs-4.18.0-477.10.1.el8_8.x86_64+debug.img"
title="Rocky Linux (4.18.0-477.10.1.el8_8.x86_64+debug) 8.8 (Green Obsidian)"
id="fe8567dfe90b49c5a4f20a22a12cbc66-4.18.0-477.10.1.el8_8.x86_64+debug"
index=1
kernel="/boot/vmlinuz-4.18.0-425.19.2.el8_7.x86_64"
args="ro crashkernel=auto resume=/dev/mapper/rl-swap rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet console=ttyS0,115200n8 console=tty1"
root="/dev/mapper/rl-root"
initrd="/boot/initramfs-4.18.0-425.19.2.el8_7.x86_64.img"
title="Rocky Linux (4.18.0-425.19.2.el8_7.x86_64) 8.7 (Green Obsidian)"
id="fe8567dfe90b49c5a4f20a22a12cbc66-4.18.0-425.19.2.el8_7.x86_64"
index=2
kernel="/boot/vmlinuz-0-rescue-fe8567dfe90b49c5a4f20a22a12cbc66"
args="ro crashkernel=auto resume=/dev/mapper/rl-swap rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet console=ttyS0,115200n8 console=tty1"
root="/dev/mapper/rl-root"
initrd="/boot/initramfs-0-rescue-fe8567dfe90b49c5a4f20a22a12cbc66.img"
title="Rocky Linux (0-rescue-fe8567dfe90b49c5a4f20a22a12cbc66) 8.6 (Green Obsidian)"
id="fe8567dfe90b49c5a4f20a22a12cbc66-0-rescue"
```
# 查看当前的启动内核

```shell
$ sudo grubby --default-kernel      
/boot/vmlinuz-4.18.0-477.10.1.el8_8.x86_64+debug
$ sudo grubby --default-index 
0
```

# 修改启动内核

```shell
$ sudo grubby --set-default-index=0
The default is /boot/loader/entries/fe8567dfe90b49c5a4f20a22a12cbc66-4.18.0-477.10.1.el8_8.x86_64+debug.conf with index 0 and kernel /boot/vmlinuz-4.18.0-477.10.1.el8_8.x86_64+debug
```
