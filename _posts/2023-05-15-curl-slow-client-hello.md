---
layout: post
title: "curl send client hello after 200ms"
description: "curl send client hello after 200ms"
date: 2023-05-07
tags: [curl, timing, client hello, SSL/TLS]
---

# 反馈问题

同事发现业务的请求有点慢，使用 curl 测试一下指定的网站看看响应时间。结果发现需要 229 ms 才响应。

```shell
time curl -k https://a.b.c.d:443
<!doctype html>
<html lang=en>
<title>404 Not Found</title>
<h1>Not Found</h1>
<p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>

real    0m0.229s
user    0m0.063s
sys     0m0.110s
```

使用 ping 测量延时，发现链路延时只有 10 多毫秒

```shell
$ ping a.b.c.d
PING a.b.c.d (a.b.c.d a.b.c.d) 56(84) bytes of data.
64 bytes from a.b.c.d: icmp_seq=1 ttl=41 time=13.1 ms
64 bytes from a.b.c.d: icmp_seq=2 ttl=41 time=15.3 ms
```

# 使用 curl 自带的时间测量变量

ping 的延时很短，但是 curl 的响应时间很长，应该是服务端的问题。
我们使用 curl 自带的测量时间的工具进行测试

```shell
$ curl -s -o /dev/null -w "%{time_connect} %{time_appconnect} %{time_total}" -k https://a.b.c.d
0.014 0.213 0.228
```

可以看到，建连的时间和 ping 是一致的。但是应用层连接(也就是https连接)成功的时间高达 213 毫秒，
实际的响应时间是 15 ms，差不多就说链路延时。

因此，我们可以判断，问题就是在建连上。

# tcpdump 抓包分析

建连为什么这么长呢？ 是 SSL/TLS 握手很慢导致的码？假设是 SSL/TLS 握手造成的问题，
我们计算一下的话会发现 1 秒只能进行 5次握手，这显然不合常理。所以不会是 SSL/TLS 的握手性能导致的问题。
因为我们也知道服务端的负载很低，所以也不是由于高负载导致的。

因此，我们可以判断，可能是客户端发送 client hello 很慢导致的。
为了验证这个猜测，我们使用 tcpdump 进行抓包分析。

```shell
$ sudo tcpdump -nnn -ttt  -i eth0 host a.b.c.d and tcp port 443
 00:00:00.000000 IP c.l.i.n.32780 > a.b.c.d: Flags [S], seq 4109404002, win 26883, options [mss 8961,sackOK,TS val 1376744720 ecr 0,nop,wscale 7], length 0
 00:00:00.010984 IP a.b.c.d > c.l.i.n.32780: Flags [S.], seq 950036968, ack 4109404003, win 65160, options [mss 1460,sackOK,TS val 1945479284 ecr 1376744720,nop,wscale 7], length 0
 00:00:00.000040 IP c.l.i.n.32780 > a.b.c.d: Flags [.], ack 1, win 211, options [nop,nop,TS val 1376744731 ecr 1945479284], length 0
 00:00:00.159736 IP c.l.i.n.32780 > a.b.c.d: Flags [P.], seq 1:174, ack 1, win 211, options [nop,nop,TS val 1376744891 ecr 1945479284], length 173
 00:00:00.009734 IP a.b.c.d > c.l.i.n.32780: Flags [.], ack 174, win 508, options [nop,nop,TS val 1945479454 ecr 1376744891], length 0
 ...
```

通过以上抓包，我们可以看到，客户端在三次握手成功之后的 159 ms 之后才发送了 client hello 报文出去。
因此，我们确认是客户端问题导致的整个 HTTPS 请求耗时很长。

# strace 分析系统调用

那么客户端在做什么事情，为什么会这么慢呢？
我们在其它机器上测试并不会这么慢，因此我们怀疑是跟系统有关系。我们这个机器是 `CentOS Linux 7 (Core)` 的系统。

想要了解 curl 到底在干啥，可以使用 strace 命令进行跟踪分析。
我们使用了下面的命令进行分析，发现 curl 打开了很多不相关的文件。或许就是这些不相关功能导致的，没有再进一步的深入分析了。

**这里应该加上 `-r` 的命令行选项，这样就有两个系统调用之间的时间差了。**

```shell
$ strace -tt curl -s -o /dev/null -w "%{time_connect} %{time_appconnect} %{time_total}" -k https://a.b.c.d > c.log 2>&1
$ grep -w sleep c.log
16:10:56.216763 open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
16:10:56.217161 open("/lib64/libcurl.so.4", O_RDONLY|O_CLOEXEC) = 3
16:10:56.217877 open("/lib64/libssl3.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.218568 open("/lib64/libsmime3.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.219322 open("/lib64/libnss3.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.220209 open("/lib64/libnssutil3.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.221536 open("/lib64/libplds4.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.222480 open("/lib64/libplc4.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.223231 open("/lib64/libnspr4.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.223938 open("/lib64/libpthread.so.0", O_RDONLY|O_CLOEXEC) = 3
16:10:56.224960 open("/lib64/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.225765 open("/lib64/libz.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.226579 open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
16:10:56.227626 open("/lib64/libidn.so.11", O_RDONLY|O_CLOEXEC) = 3
16:10:56.228515 open("/lib64/libssh2.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.229375 open("/lib64/libgssapi_krb5.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.230538 open("/lib64/libkrb5.so.3", O_RDONLY|O_CLOEXEC) = 3
16:10:56.231272 open("/lib64/libk5crypto.so.3", O_RDONLY|O_CLOEXEC) = 3
16:10:56.231672 open("/lib64/libcom_err.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.232631 open("/lib64/liblber-2.4.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.233409 open("/lib64/libldap-2.4.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.234440 open("/lib64/librt.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.235325 open("/lib64/libssl.so.10", O_RDONLY|O_CLOEXEC) = 3
16:10:56.236245 open("/lib64/libcrypto.so.10", O_RDONLY|O_CLOEXEC) = 3
16:10:56.237543 open("/lib64/libkrb5support.so.0", O_RDONLY|O_CLOEXEC) = 3
16:10:56.238469 open("/lib64/libkeyutils.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.239260 open("/lib64/libresolv.so.2", O_RDONLY|O_CLOEXEC) = 3
16:10:56.240246 open("/lib64/libsasl2.so.3", O_RDONLY|O_CLOEXEC) = 3
16:10:56.241191 open("/lib64/libselinux.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.242199 open("/lib64/libcrypt.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.243129 open("/lib64/libpcre.so.1", O_RDONLY|O_CLOEXEC) = 3
16:10:56.244002 open("/lib64/libfreebl3.so", O_RDONLY|O_CLOEXEC) = 3
16:10:56.254924 open("/etc/pki/tls/legacy-settings", O_RDONLY) = -1 ENOENT (No such file or directory)
16:10:56.256168 open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
16:10:56.256694 open("/home/lijunlong/.curlrc", O_RDONLY) = -1 ENOENT (No such file or directory)
16:10:56.269587 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.270662 open("/lib64/libsoftokn3.so", O_RDONLY|O_CLOEXEC) = 4
16:10:56.271482 open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 4
16:10:56.271914 open("/lib64/libsqlite3.so.0", O_RDONLY|O_CLOEXEC) = 4
16:10:56.274046 open("/lib64/libfreeblpriv3.so", O_RDONLY|O_CLOEXEC) = 4
16:10:56.275283 open("/dev/urandom", O_RDONLY) = 4
16:10:56.276363 open("/dev/urandom", O_RDONLY) = 4
16:10:56.277094 open("/etc/passwd", O_RDONLY) = 4
16:10:56.277986 open("/tmp", O_RDONLY)  = 4
16:10:56.278406 open("/var/tmp", O_RDONLY) = 4
16:10:56.278828 open("/usr/tmp", O_RDONLY) = 4
16:10:56.279211 open("/dev/urandom", O_RDONLY) = 4
16:10:56.287094 open("/lib64/libfreeblpriv3.chk", O_RDONLY) = 4
16:10:56.288361 open("/lib64/libfreeblpriv3.so", O_RDONLY) = 4
16:10:56.316105 open("/lib64/libsoftokn3.chk", O_RDONLY) = 4
16:10:56.318139 open("/lib64/libsoftokn3.so", O_RDONLY) = 4
16:10:56.329741 open("/etc/pki/nssdb/pkcs11.txt", O_RDONLY) = 4
16:10:56.330824 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.331676 open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 4
16:10:56.332158 open("/lib64/libnsssysinit.so", O_RDONLY|O_CLOEXEC) = 4
16:10:56.333191 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.334547 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.371159 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.372245 open("/home/lijunlong/.pki/nssdb/pkcs11.txt", O_RDONLY) = -1 ENOENT (No such file or directory)
16:10:56.372482 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 4
16:10:56.407273 open("/etc/pki/nssdb/cert9.db", O_RDONLY|O_CLOEXEC) = 4
16:10:56.410721 open("/dev/urandom", O_RDONLY|O_CLOEXEC) = 5
16:10:56.444059 open("/etc/pki/nssdb/key4.db", O_RDONLY|O_CLOEXEC) = 5
16:10:56.487055 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 6
16:10:56.487662 open("/etc/pki/nss-legacy/nss-rhel7.config", O_RDONLY) = 6
16:10:56.488312 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 6
16:10:56.489217 open("/proc/sys/crypto/fips_enabled", O_RDONLY) = 6
16:10:56.489762 open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 6
16:10:56.490219 open("/lib64/libnsspem.so", O_RDONLY|O_CLOEXEC) = 6
16:10:56.580671 open("/dev/null", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 6
```
