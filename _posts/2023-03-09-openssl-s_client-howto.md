---
layout: post
title: "Google Baidu 哪家强"
description: "openssl s_client 查看服务器的 TLS 配置"
date: 2023-03-05
tags: [OpenSSL]
---

openssl s_client 可以将握手信息打印出来，因此可以得到不少的信息。
通过 wireshark 抓包虽然也可以看到类似的信息，但是显然多一道手续。

我们需要关注那些信息呢？

1. Certificate chain： 证书链信息可以看到服务端发送的证书链是否完整。
2. Subject: 可以确认证书是否是泛域名证书
3. Peer signing digest: 证书签名的摘要算法
4. Peer signature type: 证书的签名类型
5. Server Temp Key: 
6. Verification: 证书校验是否通过
7. TLSv1.3:  TLS 的版本
8. Cipher： 加密协议族
9. Server public key：服务器的公钥长度, 这个和上面的签名类型关联

[openssl s_client测试结果](#openssl-s_client-结果) 中包含了 *.google.com 和 www.baidu.com 的结果。

我们可以看到:
1.  Google 使用的是  ECDSA 256bit 签名 SHA256 摘要的证书，TLSv1.3 TLS_AES_256_GCM_SHA384。
2. 百度使用的是 RSA 2048 bit 签名 SHA512 摘要的证书. TLSv1.2 ECDHE-RSA-AES128-GCM-SHA256

在这里，我想问问 google baidu 那家强？那么让我们先了解一下 TLS 的基本信息吧。

一个 TLS 加密协议族一般包含 4 个部分：

1. 密钥交换算法 - 规定了交换对称密钥的方式。
1. 认证算法 - 规定如何进行服务器认证和客户端认证(可选)。
1. 块加密算法 - 规定了将使用哪种对称密钥算法来加密实际数据。
1. 消息摘要算法 - 决定了连接将使用何种方法来进行数据完整性检查。

比如：

1. 密钥交换算法: RSA, DH, ECDH, ECDHE;
1. 认证算法: RSA, DSA, ECDSA;
1. 对称加密算法: AES, 3DES, CAMELLIA;
1. 消息摘要算法: SHA, MD5

比如 TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384:

TLS：传输层安全
ECDHE：密钥交换算法是 ECDHE（Elliptic curve Diffie-Hellman，ephemeral）。
ECDSA：认证算法是ECDSA（椭圆曲线数字签名算法）。证书颁发机构使用 ECDH 密钥来签署公共密钥。
WITH_AES_256_CBC: 这是用来加密信息流的。(AES=高级加密标准，CBC=密码块链）。数字256表示块的大小。
SHA_384: 这是所谓的消息验证码（MAC）算法。SHA=安全哈希算法。它用于创建一个消息摘要或消息流块的哈希值。这可以用来验证消息内容是否被改变。数字表示哈希值的大小。较大的是更安全的。

google 使用的加密协议族为 TLS_AES_256_GCM_SHA384。 该协议族只包含 对称加密算法和消息摘要算法。
因为 TLS1.3 中 RSA 被取消了，密钥交换是通过 Diffie-Hellman 算法进行。

百度使用的协议族是 ECDHE-RSA-AES128-GCM-SHA256， 这个对应的名称是 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256。
该算法的密钥交换算法是 ECDHE, 认证算法是 RSA， 对称加密算法是 AES128-GCM, 消息摘要算法是 SHA256.

查询标准名称可以使用这样的命令 `openssl ciphers -stdname | grep ECDHE-RSA-AES128-GCM-SHA256`

关于如何选择加密协议族，Mozilla 的建议如下：[wiki.mozilla.org/Security/Server_Side_TLS](https://wiki.mozilla.org/Security/Server_Side_TLS)

对于普通的服务器，为了更好的兼容性，其推荐如下的配置

```text
Cipher suites (TLS 1.3): TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
Cipher suites (TLS 1.2): ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
Protocols: TLS 1.2, TLS 1.3
TLS curves: X25519, prime256v1, secp384r1
Certificate type: ECDSA (P-256) (recommended), or RSA (2048 bits)
DH parameter size: 2048 (ffdhe2048, RFC 7919)
HSTS: max-age=63072000 (two years)
Certificate lifespan: 90 days (recommended) to 366 days
Cipher preference: client chooses
```

如果要之间放在 Nginx / OpenResty 服务器上，那么可以参考这个连接：[ssl-config.mozilla.org](https://ssl-config.mozilla.org/)

如果选择加密协议族其实是一件很有挑战的事情，如下是一些考虑的因素：

1. 服务器、客户端和证书颁发机构的兼容性；
    - 对外的网站需要与所有主要的客户端兼容
    - 内部网站可以控制客户端的范围
1. 加密/解密性能
1. 加密强度；密钥和哈希值的类型和长度
1. 所需的加密功能；如防止重放攻击、前向保密性

**我们通过实验来比较一下 google 和 baidu 实际性能差异。**

# 生成证书

## 生成 google 的 ECC 证书

```shell
openssl ecparam -name prime256v1 -genkey -noout -out server.key
openssl ec -in server.key -pubout -out server.pem
openssl req -new -x509 -key server.key -out server.crt -days 360
```

## 生成 baidu 的 RSA 证书

```
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
	    -subj "/C=CN/ST=Beijing/L=Beijing/O=Dev/CN=root.com" \
	        -keyout ca.key  -out ca.crt
openssl req -new -newkey rsa:2048 -nodes \
	    -keyout server.key -out server.csr \
	        -subj "/C=CN/ST=Beijing/L=Beijing/O=Dev/CN=server.com"

openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial "0x`openssl rand -hex 8`" -out server.crt
```

# 配置 Nginx

```nginx
    access_log off;

    server {
        listen       127.0.0.10:4256 ssl reuseport;
        server_name  localhost;

        ssl_session_tickets on;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_certificate ssl/ecc/server.crt;
        ssl_certificate_key ssl/ecc/server.key;
        location / {
            keepalive_requests 1000;
            content_by_lua_block {
               ngx.say("Hello world")
            }
        }
    }

    server {
        listen       127.0.0.10:2048 ssl reuseport;
        server_name  localhost;

        ssl_session_tickets on;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256;
        ssl_certificate ssl/2048/server.crt;
        ssl_certificate_key ssl/2048/server.key;
        ssl_protocols TLSv1.1 TLSv1.2;
        location / {
            keepalive_requests 1000;
            content_by_lua_block {
               ngx.say("Hello world")
            }
        }
    }
```

# 测试 Nginx 性能

```shell
wrk --no-session-id --no-ticket -H "Connection: close" -t1 -d5 https://127.0.0.10:425656
Running 5s test @ https://127.0.0.10:4256
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.39ms  320.30us   8.22ms   89.81%
    Req/Sec     1.48k   183.38     1.58k    85.71%
  7324 requests in 5.00s, 9.34MB read
Requests/sec:   1464.18
Transfer/sec:      1.87MB
```


```shell
wrk -t 1 -d 5 --no-ticket --no-session-id -H "Connection: close" https://127.0.0.10:2048 

Running 5s test @ https://127.0.0.10:2048
  1 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.29ms  287.98us   5.66ms   83.54%
    Req/Sec     0.89k    75.76     0.95k    90.00%
  4418 requests in 5.00s, 832.69KB read
Requests/sec:    883.29
Transfer/sec:    166.48KB
```

# 结论

使用 google 的配置下，服务器的性能明显比使用 baidu 配置的服务器的性能高很多，差点翻倍了。

当然这种对比只是比较短链接下的很小的一个场景，握手是占据了绝大多数的 CPU 开销。
其它更多的比较还有待进一步的展开。

# openssl s_client 结果

## google 的结果

```shell
$ openssl s_client -connect google.com:443
depth=2 C = US, O = Google Trust Services LLC, CN = GTS Root R1
verify return:1
depth=1 C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
verify return:1
depth=0 CN = *.google.com
verify return:1
---
Certificate chain
 0 s:CN = *.google.com
   i:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
 1 s:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R1
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R1
   i:C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIOOzCCDSOgAwIBAgIQcykDwrxnmlwKpO8sLjVYmzANBgkqhkiG9w0BAQsFADBG
...
NdV0oacHdBU0s/uF+N5VNMVQ63Gn6Nd/gYuC2A6VVNRtMKroTBlcr1aTjPqdgx8=
-----END CERTIFICATE-----
subject=CN = *.google.com

issuer=C = US, O = Google Trust Services LLC, CN = GTS CA 1C3

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 6779 bytes and written 384 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
```

## 百度的结果

```shell
$ openssl s_client -connect www.baidu.com:443
CONNECTED(00000003)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
verify return:1
depth=1 C = US, O = DigiCert Inc, CN = DigiCert Secure Site Pro CN CA G3
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC\E5\B8\82, O = "BeiJing Baidu Netcom Science Technology Co., Ltd", CN = www.baidu.cn
verify return:1
---
Certificate chain
 0 s:C = CN, ST = \E5\8C\97\E4\BA\AC\E5\B8\82, O = "BeiJing Baidu Netcom Science Technology Co., Ltd", CN = www.baidu.cn
   i:C = US, O = DigiCert Inc, CN = DigiCert Secure Site Pro CN CA G3
 1 s:C = US, O = DigiCert Inc, CN = DigiCert Secure Site Pro CN CA G3
   i:C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIHSjCCBjKgAwIBAgIQB8YyshMp/Wi3t2Ddh9FfIDANBgkqhkiG9w0BAQsFADBQ
...
zpMevdWiKj8+Ev5tMGLWTeX+8+sn/8XHUSTO/ezhTxvICaOF2kilUnGRFFUWzA==
-----END CERTIFICATE-----
subject=C = CN, ST = \E5\8C\97\E4\BA\AC\E5\B8\82, O = "BeiJing Baidu Netcom Science Technology Co., Ltd", CN = www.baidu.cn

issuer=C = US, O = DigiCert Inc, CN = DigiCert Secure Site Pro CN CA G3

---
No client certificate CA names sent
Peer signing digest: SHA512
Peer signature type: RSA
Server Temp Key: ECDH, P-256, 256 bits
---
SSL handshake has read 3839 bytes and written 429 bytes
Verification: OK
---
New, TLSv1.2, Cipher is ECDHE-RSA-AES128-GCM-SHA256
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.2
    Cipher    : ECDHE-RSA-AES128-GCM-SHA256
    Session-ID: 81C3DD7EB31FE99C97B0E5ED79A213B033C24EE9D61A91CCFCAE828A166858DE
    Session-ID-ctx: 
    Master-Key: CF8718D6C4C186CCBAAD6004B6FF40C9808820D8D71FA83642E7C80E43DB9C882F61DC831C311CF109589082B9B0CFF1
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 72000 (seconds)
    TLS session ticket:
    0000 - 3a d6 e6 6e eb 38 c2 ac-38 6e 3f 6c ff 74 fb e0   :..n.8..8n?l.t..
    0010 - 6d 8e e8 5c 4e 4d 7e 87-85 da ca 80 7b da f0 e2   m..\NM~.....{...
    0020 - 08 38 4f de 61 a7 9d ca-45 ba 94 5e 8b 4e 00 91   .8O.a...E..^.N..
    0030 - aa f5 52 91 4e 6b 0f e8-ed 96 cc a4 6c e5 95 39   ..R.Nk......l..9
    0040 - 65 d7 49 a1 7d 36 0c 36-f3 ef 2f b7 b9 14 27 22   e.I.}6.6../...'"
    0050 - 17 40 11 c7 9d 20 eb 47-5c 09 5e 9f f7 c4 b3 ea   .@... .G\.^.....
    0060 - ac b7 40 a5 0c aa d2 6b-37 b8 0d b7 7f 93 20 1a   ..@....k7..... .
    0070 - fa 3d 6a f7 ef a6 02 3e-c2 ca d0 a3 4f 66 cc f6   .=j....>....Of..
    0080 - b6 e1 dd 62 f4 2f 39 9c-b1 1b 00 88 9f 46 5a cd   ...b./9......FZ.
    0090 - 8b 94 72 4d 9f f3 70 fd-64 46 1a 6b e2 df 1a 4d   ..rM..p.dF.k...M
    00a0 - fb a6 c4 e0 bc a7 d7 08-ca 45 71 94 c3 c0 a0 91   .........Eq.....

    Start Time: 1679022939
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
---
```

有些术语可能使用的不准确，欢迎指正。请发邮件联系： zhuizhuhaomeng@gmail.com
