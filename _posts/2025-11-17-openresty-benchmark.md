---
layout: post
title: "OpenResty Benchmark"
description: "OpenResty Benchmark(性能压测)"
date: 2025-11-17
tags: [Benchmark, OpenResty]
---

# upstream configuration

```nginx
user  nobody;
worker_processes  2;
worker_cpu_affinity 0010 0100;  # bind to CPU1 and CPU2
error_log  logs/error.log error;
pid        logs/nginx.pid;

events {
    accept_mutex off;
    worker_connections  8192;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    server {
        listen 1880;
        location / {
            content_by_lua_block {
                ngx.say("Hello world!")
            }
        }
    }
}
```

# openresty configuration

```nginx
user  nobody;
worker_processes  1;
worker_cpu_affinity auto;  # This will bind to CPU0
pid        logs/nginx.pid;
error_log  logs/error.log error;


events {
    accept_mutex off;
    worker_connections  8192;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    upstream backend_1880 {
        server 127.0.0.1:1880;
        keepalive 32;
    }

    server {
        listen       1881;
        server_name  localhost;

        location / {
            # This following configuration is required to make proxy_pass work with keepalive connections.
            proxy_pass http://backend_1880;
            proxy_http_version 1.1;
            proxy_set_header Connection ""; 
        }
    }
}
```

# benchmark

1. Test against the upstream first, make sure it works correctly.

```shell
curl http://127.0.0.1:1880
```

2. Test against the upstream directly, get the performance of the upstream.
The upstream should not be the bottleneck.

```shell
wrk -d 10s -c 10 -t 2 http://127.0.0.1:1880
```

3. Test against the openresty, make sure it works correctly.

```shell
curl http://127.0.0.1:1881
```

2. Test against the openresty gatewaty, get the performance of the gateway.

```shell
wrk -d 10s -c 10 -t 2 http://127.0.0.1:1881
```
