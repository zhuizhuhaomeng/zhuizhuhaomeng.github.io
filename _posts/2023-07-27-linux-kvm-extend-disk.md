---
layout: post
title: "给 kvm 虚拟机扩展磁盘分区"
description: "给 kvm 虚拟机扩展磁盘分区"
date: 2023-07-27
tags: [kvm, disk]
---

使用 virt-filesystems 查看当前的虚拟机的磁盘分区的情况。
可以看到磁盘 /dev/sda 分成两个分区，分别是 /dev/sda1, /dev/sda2。

```shell
$ virt-filesystems -h --all --logical-volumes --long -a  centos9.qcow2 
Name                 Type       VFS  Label MBR Size  Parent
/dev/sda1            filesystem xfs  -     -   1014M -
/dev/cs_centos9/home filesystem xfs  -     -   19G   -
/dev/cs_centos9/root filesystem xfs  -     -   38G   -
/dev/cs_centos9/swap filesystem swap -     -   5.9G  -
/dev/cs_centos9/home lv         -    -     -   19G   /dev/cs_centos9
/dev/cs_centos9/root lv         -    -     -   38G   /dev/cs_centos9
/dev/cs_centos9/swap lv         -    -     -   5.9G  /dev/cs_centos9
/dev/cs_centos9      vg         -    -     -   63G   /dev/sda2
/dev/sda2            pv         -    -     -   63G   -
/dev/sda1            partition  -    -     83  1.0G  /dev/sda
/dev/sda2            partition  -    -     8e  63G   /dev/sda
/dev/sda             device     -    -     -   64G   -
```

我们使用下面的命令将整个虚拟机磁盘扩大 2G。

```shell
$ qemu-img resize centos9.qcow2 +2G
Image resized.
```

将新增的磁盘扩展到 /dev/sda2。注意，上面添加的是 2G 的空间，这里如果还用 2G 会提示空间不足。
举例如下：

```shell
root@dragon:/var/lib/libvirt/images# virt-resize --resize /dev/sda2=+2G --expand /dev/sda2 --LV-expand /dev/cs_centos9/root centos9.qcow2 centos9-n.qcow2
[   0.0] Examining centos9.qcow2
virt-resize: error: You cannot use --expand when there is no surplus space 
to expand into.  You need to make the target disk larger by at least 1.2M.

If reporting bugs, run virt-resize with debugging enabled and include the 
complete output:

  virt-resize -v -x [...]
```

因此，我们我们下面的参数用的是 1800M，而不是 2G。
但是可能会遇到下面这样的错误

```shell
$ sudo virt-resize --resize /dev/sda2=+1800M  --expand /dev/sda2 --LV-expand /dev/cs_centos9/root  centos9.qcow2 centos9-n.qcow2
[   0.0] Examining centos9.qcow2
virt-resize: error: /dev/sda2: this partition has already been marked for 
resizing

If reporting bugs, run virt-resize with debugging enabled and include the 
complete output:

  virt-resize -v -x [...]
```

如果遇到这样的错误，可以去掉 --resize 

```shell
sudo virt-resize  --expand /dev/sda2 --LV-expand /dev/cs_centos9/root  centos9.qcow2 centos9-n.qcow2
[   0.0] Examining centos9.qcow2
**********

Summary of changes:

/dev/sda1: This partition will be left alone.

/dev/sda2: This partition will be resized from 63.0G to 67.0G.  The LVM PV 
on /dev/sda2 will be expanded using the ‘pvresize’ method.

/dev/cs_centos9/root: This logical volume will be expanded to maximum size. 
 The filesystem xfs on /dev/cs_centos9/root will be expanded using the 
‘xfs_growfs’ method.

**********
[   4.3] Setting up initial partition table on centos9-n.qcow2
```

