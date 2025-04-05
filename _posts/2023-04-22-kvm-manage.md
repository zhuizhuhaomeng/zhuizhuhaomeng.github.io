---
layout: post
title: "KVM manage"
description: "KVM manage"
date: 2023-04-22
modified: 2024-08-10
tags: [KVM]
---

# Shut down all vms

Every day I need to shut down all the virtual machines, which is a lot of work.

The script below is used to shut down all the running machines.

```shell
#!/bin/bash

for i in `sudo virsh list | grep running | awk '{print $2}'`
do
    sudo virsh shutdown $i
done

while [ 1 -gt 0 ]
do
    n=`virsh list | wc -l`
    if [ $n -eq 3 ]; then
        sudo shutdown -P now
    fi
    sleep 1
done
```

# Generate connect config on startup

I want to connect to windows kvm from the host machine via desktop viewer.
I need to lanuch the Cockpit web console and then download the desktop viewer config file.
After double click the config file, it was auto-deleted. This is just too much trouble

I generate the config via the following script.

`/home/ljl` is my home directory, you need to change to your own.

```shell
#!/bin/bash

port=$(/usr/bin/virsh vncdisplay win10 | /usr/bin/awk -F: '/127.0.0.1/{print $2+5901}')

cat > /home/ljl/Downloads/win10 << EOF
[virt-viewer]
type=spice
host=127.0.0.1
port=$port
delete-this-file=1
fullscreen=1

[...............................GraphicsConsole]
EOF

chown ljl:ljl /home/ljl/Downloads/win10
```

Run `sudo crontab -e` and add the following line to the crontab.

```text
* * * * * /bin/bash /bin/get-win10-spice.sh
```

# config disk as write through mode

``` xml
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='writethrough' discard='unmap'/>
      <source file='/kvm-pools/build-farm/fed32-dev-130g.qcow2'/>
      <target dev='sda' bus='scsi'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
```

# deal with the permission deny error


Unfortunately, I've encountered this error.
I can not add another disk for my VM.

```log
$ cat /var/log/syslog | grep qemu
Aug 10 10:17:02 ljl-X99 libvirtd[1667]: internal error: qemu unexpectedly closed the monitor: 2024-08-10T02:17:02.234897Z qemu-system-x86_64: -blockdev {"driver":"file","filename":"/1t/200GB.qcow2","node-name":"libvirt-1-storage","auto-read-only":true,"discard":"unmap"}: Could not open '/1t/200GB.qcow2': Permission denied
```

Fortunately, I found the following link that solved my problem.
[qemu with pool/volume storage: Could not open ‘xxxxxxx’: Permission denied](https://zhensheng.im/2019/02/06/qemu-with-pool-volume-storage-could-not-open-xxxxxxx-permission-denied.meow)


Modify file /etc/libvirt/qemu.conf, change the security_driver to "none".

```config
security_driver = "none"
```

After finishing the above step, restart libvirtd.

```shell
systemctl restart libvirtd
```

# query state

```shell
virsh domstate fed32-ktest
```

# snapshot

```shell
virsh snapshot-delete vm-name --snapshotname init
virsh snapshot-create-as vm-name --atomic --name init
```
