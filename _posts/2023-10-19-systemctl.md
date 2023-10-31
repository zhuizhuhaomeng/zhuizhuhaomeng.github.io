---
layout: post
title: "systemctl examples"
description: "systemctl examples"
date: 2023-10-19
tags: [systemctl]
---

# 设置进程的调度属性

```shell
[Unit]
Description=Machine Check Exception Logging Daemon

[Service]
ExecStart=/usr/sbin/mcelog --ignorenodev --daemon --foreground
CPUSchedulingPolicy=fifo
CPUSchedulingPriority=20

[Install]
WantedBy=multi-user.target
```

如果命令行修改调度属性，应该使用 chrt，比如

```shell
for pid in `pgrep -f 'nginx: worker'`; do chrt -r -p 1 $pid; done

sysctl -w kernel.sched_rr_timeslice_ms=3
```


# 设置环境变量

```config
[Unit]
Description=The Talent Program
Wants=network-online.target
After=network-online.target nss-lookup.target

[Service]
LimitNOFILE=2048
Type=forking
WorkingDirectory=/usr/local/talent-calc
PIDFile=/usr/local/talent-calc/logs/nginx.pid
ExecStartPre=/usr/local/talent-calc/bin/talent-master -p /usr/local/talent-calc -t
ExecStartPre=/bin/rm -f /usr/local/talent-calc/agent.sock
ExecStart=/usr/local/talent-calc/bin/talent-master -p /usr/local/talent-calc
ExecReload=/usr/local/talent-calc/bin/talent-master -p /usr/local/talent-calc -t
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
Environment="HOSTNAME=%H"
Environment="ASAN_OPTIONS=log_path=/var/asan/asan,log_exe_name=true,detect_leaks=0,abort_on_error=1"
TimeoutStopSec=10s
PrivateTmp=false
CPUQuota=50%
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```
