---
layout: post
title: "Google Baidu 哪家强"
description: "openssl s_client 查看服务器的 TLS 配置"
date: 2023-03-05
tags: [openssl s_client]
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

那么 google baidu 那家强？我们通过实验来验证一下实际的效果。

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
