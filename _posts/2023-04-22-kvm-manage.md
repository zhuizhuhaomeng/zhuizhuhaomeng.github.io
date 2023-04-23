---
layout: post
title: "KVM manage"
description: "KVM manage"
date: 2023-04-22
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
