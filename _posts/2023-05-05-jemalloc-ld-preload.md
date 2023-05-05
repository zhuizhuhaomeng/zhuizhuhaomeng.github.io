---
layout: post
title: "使用 Jemalloc 作为 nginx 的默认内存分配器"
description: "使用 Jemalloc 作为 nginx 的默认内存分配器"
date: 2023-05-05
tags: [Jemalloc, systemd, LD_PRELOAD]
---

我们希望能够使用 Jemalloc 作为 nginx 默认的内存分配器，
但是我我们不想要重新编译 Nginx。这个时候可以在启动 Nginx 的时候加上 LD_PRELOAD 来实现目标。

1. 我们先通过如下命令查找开启启动的配置文件。

    ```shell
    $ rpm -ql openresty | grep system
    /usr/lib/systemd/system/openresty.service
    ```

1. 我们在开启启动的配置文件中加入 LD_PRELOAD 的配置

    ```config
    [Service]
    Type=forking
    Environment="LD_PRELOAD=/usr/lib64/libjemalloc.so.2"
    PIDFile=/usr/local/openresty/nginx/logs/nginx.pid
    ExecStartPre=/usr/local/openresty/nginx/sbin/nginx -t
    ExecStart=/usr/local/openresty/nginx/sbin/nginx
    ExecReload=/bin/kill -s HUP $MAINPID
    ExecStop=/bin/kill -s QUIT $MAINPID
    PrivateTmp=true
    ```

1. 重新加载配置

    ```shell
    systemctl daemon-reload
    ```

1. 重启 nginx 服务

    ```shell
    systemctl restart openresty
    ```
