---
layout: post
title: "mariner Linux 系统在 ssh 退出后杀死了手动启动的后台进程"
description: ""
date: 2023-07-10
tags: [systemd]
---

# 问题

我们通过下面这样的命令启动了 OpenResty, 在 ssh 登出后，nginx 进程被系统杀死了。

```shell
sudo /usr/local/openresty/nginx/sbin/nginx -p /usr/local/openresty/nginx
```

# 定位是谁杀死了 nginx

要定位谁杀死了 nginx，就需要看看是什么进程发送了信号。这个可以使用 systemstap 这个工具
来快速分析。我们使用这个脚本定位到是 systemd 杀死了 nginx。因此就可以通过这个信息在 google
搜索相关的关键词来定位该问题。

```stap
global execname = "nginx";

probe signal.send {
    exe = execname();

    if (exe == execname || pid_name == execname) {
        printf("[%d] %s %d %s -> %s(%d)\n", gettimeofday_s(), exe, pid(), sig_name, pid_name, sig_pid);
    }
}

probe timer.s(300) {
    abort();
}

probe begin {
    warn("Start tracing...");
}
```

# 解决方案

通过 Google 的搜索，我们可以知道需要修改 /etc/systemd/logind.conf 这个文件。

```text
[Login]
KillUserProcesses=no
```

修改完成后执行命令： `systemctl restart systemd-logind`

这下我们就可以再次愉快的手动启动进程了。
不过 systemd 是怎么知道要杀死哪些进程的呢？
这个需要进一步的分析，留待后续有空再说。
