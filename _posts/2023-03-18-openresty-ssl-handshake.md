---
layout: post
title: "SSL/TLS 握手开销测试"
description: "SSL/TLS 握手开销测试"
date: 2023-03-18
tags: [OpenResty, SSL/TLS, Nginx, OpenSSL]
---

# TLS 握手的性能问题

TLS 握手开销是很高的，因此就有了会话恢复这样的技术来缓解由于握手带来的额外性能开销。

我们这里搭建了一个 [OpenResty](https://openresty.org) 的服务器来测试一下各种场景下的请求速率。

# 证书生成脚本
搭建的服务器是 2048 bits RSA 签名的证书，使用如下的脚本生成。

```shell
#! /usr/bin/env bash

openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=Dev/CN=root.test.com" \
    -keyout ca.key -out ca.crt

openssl req -new -newkey rsa:2048 -nodes \
    -keyout server.key -out server.csr \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=Dev/CN=server.com"

openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial "0x`openssl rand -hex 8`" -out server.crt
```

# OpenResty 配置

OpenResty 服务器配置如下，这里关闭了访问日志，TLS 协议配置是 TLSv1.1 TLSv1.2。

```nginx
worker_processes  1;
worker_cpu_affinity auto;

http {
    include       mime.types;
    default_type  application/octet-stream;
    error_log  logs/error.log;

    server {
        listen       127.0.0.10:2048 ssl reuseport;
        server_name  localhost;

        access_log off;
        ssl_session_cache off;
        ssl_session_tickets on;
        ssl_certificate ssl/2048/server.crt;
        ssl_certificate_key ssl/2048/server.key;
        ssl_prefer_server_ciphers on;
        ssl_protocols TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA;

        location / {
            keepalive_requests 1000;
            content_by_lua_block {
               ngx.say("Hello world")
            }
        }
    }
}
```

# 各种场景请求速率测试

## 长连接场景下的请求速率

```bash
$ wrk -t 1 -d 5 https://127.0.0.10:2048                            
Running 5s test @ https://127.0.0.10:2048
  1 threads and 10 connections

  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   290.12us   35.79us   2.87ms   92.84%
    Req/Sec    34.02k     3.04k   35.96k    92.00%
  169198 requests in 5.00s, 31.95MB read
Requests/sec:  33828.70
Transfer/sec:      6.39MB
```

## 长链接无会话恢复的请求速率

```shell
$ wrk -t 1 -d 5 --no-session-id --no-ticket https://127.0.0.10:2048
Running 5s test @ https://127.0.0.10:2048
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   280.50us   96.35us   3.03ms   97.41%
    Req/Sec    34.56k     3.74k   37.68k    90.00%
  171748 requests in 5.00s, 32.43MB read
Requests/sec:  34340.55
Transfer/sec:      6.48MB
```


## 短链接场景下的请求速率

```shell
$ wrk -t 1 -d 5 -H "Connection: close" https://127.0.0.10:2048 
Running 5s test @ https://127.0.0.10:2048
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   666.80us  341.93us   8.82ms   97.27%
    Req/Sec     3.07k   444.40     3.72k    69.39%
  15349 requests in 5.00s, 2.83MB read
Requests/sec:   3068.78
Transfer/sec:    578.39KB
```

## 短链接场景下禁用会话复用的请求速率

```shell
$ wrk -t 1 -d 5 --no-session-id --no-ticket -H "Connection: close" https://127.0.0.10:2048 
Running 5s test @ https://127.0.0.10:2048
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.30ms  333.15us   5.38ms   85.94%
    Req/Sec     0.88k   102.02     0.94k    84.00%
  4388 requests in 5.00s, 827.04KB read
Requests/sec:    877.30
Transfer/sec:    165.35KB
```

## 总结

| 场景  | 请求速率 |
|---|---|
| 长链接会话恢复 | 33828 r/s  |
| 长链接禁用会话恢复 | 34340 r/s |
| 短链接会话恢复 | 3068 r/s |
| 短链接禁用会话恢复 | 877 r/s |

通过上面可以看到：
1. 如果是 1000 个请求才切换一次 TCP 长链接，那么会话恢复看起来是没有效果。
2. 如果是短链接，那么是否有会话恢复则相差很大。会话恢复的请求速率是没有会话恢复的请求速率的 3068/877 = 3.5 倍。

# 配置调试

有时候我们并不确认我们的 OpenResty/Nginx 配置是否正确，因此我们需要验证一下配置的正确性，否则基于错误的配置来验证效果也是得到错误的结论。

## 确认 OpenResty 是否支持会话恢复

我们使用 openssl s_client -reconnect 参数验证会话复用是否生效。

```shell
$ openssl s_client -servername test.com -connect 127.0.0.10:2048 -reconnect 2>&1 | grep -i Reused 
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

如果我们要验证的是 Session ID 类型的会话恢复，那么应该加上 -no_ticket 的参数。
比如这个是给 nginx 配置加上 `ssl_session_cache builtin:1000 shared:SSL:10m;` 后验证的效果。

```shell
$ echo | openssl s_client -servername test.com -connect 127.0.0.10:2048 -no_ticket -reconnect 2>&1  | grep -i Reuse
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

一般情况下，为了保证高可用性，网关节点会有多台机器。
如果我们想确认跨机器是否支持 TLS 恢复，那么之间使用 `openssl s_client -reconnect` 就
达不到我们的要求了。这个时候可以通过将会话保存再读取发送的方式来验证。

具体命令如下：

```shell
$ (echo; sleep 0.3) | openssl s_client -servername test.com -sess_out sess.data -connect 127.0.0.10:2048 >/dev/null 2>&1
$ echo | openssl s_client -sess_in sess.data -connect 127.0.0.10:2048 2>&1 | grep Reuse
Reused, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
```

### TLSv1.3 的会话复用

如果配置了 TLS 1.3, 那么就不能使用 `openssl s_client -reconnect` 来验证了。
这个是 openssl s_client 实现的问题。

```shell
openssl s_client -servername test.com -reconnect -connect 127.0.0.10:2048 2>&1 | grep -i Reuse
```

这个时候我们可以采用上面说的先保存会话信息到文件再读取发送的方式。

## 性能影响因素

TLS 的性能收到很多因素的影响，其中一个很重要的因素就是签名算法。
这种里比较 rsa2048 rsa4096, 可见 rsa2048 的签名速度是 rsa4096 的 6.67 倍
，校验速度是 3.5 倍。

```shell
$ openssl speed rsa2048 rsa4096
...
                  sign    verify    sign/s verify/s
rsa 2048 bits 0.000750s 0.000021s   1333.2  47946.7
rsa 4096 bits 0.005005s 0.000073s    199.8  13690.4
```

这边有一个比较完整的测试列表，详情请点击：[openssl spped test](http://wiki.espressobin.net/tiki-index.php?page=Running+OpenSSL+speed+test+on+ESPRESSObin)

另一个性能影响因素就是不同的加密套件的算法，这个等测试清楚再补充。
