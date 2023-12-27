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

1. Certificate chain：证书链信息可以看到服务端发送的证书链是否完整。
2. Subject: 可以确认证书是否是泛域名证书
3. Peer signing digest: 证书签名的摘要算法
4. Peer signature type: 证书的签名类型
5. Server Temp Key: 
6. Verification: 证书校验是否通过
7. TLSv1.3:  TLS 的版本
8. Cipher：加密协议族
9. Server public key：服务器的公钥长度，这个和上面的签名类型关联

[openssl s_client 测试结果](#openssl-s_client-结果) 中包含了 *.google.com 和 www.baidu.com 的结果。

我们可以看到：
1.  Google 使用的是  ECDSA 256bit 签名 SHA256 摘要的证书，TLSv1.3 TLS_AES_256_GCM_SHA384。
2. 百度使用的是 RSA 2048 bit 签名 SHA512 摘要的证书。TLSv1.2 ECDHE-RSA-AES128-GCM-SHA256

在这里，我想问问 google baidu 那家强？那么让我们先了解一下 TLS 的基本信息吧。

一个 TLS 加密协议族一般包含 4 个部分：

1. 密钥交换算法 - 规定了交换对称密钥的方式。
1. 认证算法 - 规定如何进行服务器认证和客户端认证 (可选)。
1. 块加密算法 - 规定了将使用哪种对称密钥算法来加密实际数据。
1. 消息摘要算法 - 决定了连接将使用何种方法来进行数据完整性检查。

比如：

1. 密钥交换算法：RSA, DH, ECDH, ECDHE;
1. 认证算法：RSA, DSA, ECDSA;
1. 对称加密算法：AES, 3DES, CAMELLIA;
1. 消息摘要算法：SHA, MD5

比如 TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384:

TLS：传输层安全
ECDHE：密钥交换算法是 ECDHE（Elliptic curve Diffie-Hellman，ephemeral）。
ECDSA：认证算法是 ECDSA（椭圆曲线数字签名算法）。证书颁发机构使用 ECDH 密钥来签署公共密钥。
WITH_AES_256_CBC: 这是用来加密信息流的。(AES=高级加密标准，CBC=密码块链）。数字 256 表示块的大小。
SHA_384: 这是所谓的消息验证码（MAC）算法。SHA=安全哈希算法。它用于创建一个消息摘要或消息流块的哈希值。这可以用来验证消息内容是否被改变。数字表示哈希值的大小。较大的是更安全的。

google 使用的加密协议族为 TLS_AES_256_GCM_SHA384。该协议族只包含 对称加密算法和消息摘要算法。
因为 TLS1.3 中 RSA 被取消了，密钥交换是通过 Diffie-Hellman 算法进行。

百度使用的协议族是 ECDHE-RSA-AES128-GCM-SHA256，这个对应的名称是 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256。
该算法的密钥交换算法是 ECDHE, 认证算法是 RSA，对称加密算法是 AES128-GCM, 消息摘要算法是 SHA256.

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

# openssl s_client 的其它用法

## 是否支持 HTTP2 检查

另外 openssl s_client 一个常用的功能是检查网关是否支持 http2。

比如这样子：

```shell
$ openssl s_client -alpn h2 -servername openresty.org -connect openresty.org:443 < /dev/null | grep 'ALPN'
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
verify return:1
depth=0 CN = openresty.org
verify return:1
DONE
ALPN protocol: h2
```

## 是否支持 session id 会话复用的检查

如果想要检查是否支持基于 session id 的会话恢复，可以使用如下命令 -reconnect 的参数

比如，下面的例子中 `drop connection and then reconnect` 就是断开当前连接并使用 获得的 session id 进行连接，`Reuse` 表明复用了链接。
注意，这里的命令行参数一定要有 `-tls1_2`。

```shell
     1	$echo | openssl s_client -tls1_2 -servername openresty.org  -connect openresty.org:443 -reconnect -no_ticket
     2
     3	depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
     4	verify return:1
     5	depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
     6	verify return:1
     7	depth=0 CN = openresty.org
     8	verify return:1
     9	CONNECTED(00000003)
    10	---
    11	Certificate chain
    12	 0 s:CN = openresty.org
    13	   i:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    14	 1 s:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    15	   i:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    16	 2 s:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    17	   i:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
    18	---
    19	Server certificate
    20	-----BEGIN CERTIFICATE-----
    21	MIIGoDCCBIigAwIBAgIQAf8IN3hitNWb/V16aIBRujANBgkqhkiG9w0BAQwFADBL
    22	MQswCQYDVQQGEwJBVDEQMA4GA1UEChMHWmVyb1NTTDEqMCgGA1UEAxMhWmVyb1NT
    23	TCBSU0EgRG9tYWluIFNlY3VyZSBTaXRlIENBMB4XDTIzMDgxNTAwMDAwMFoXDTIz
    24	MTExMzIzNTk1OVowGDEWMBQGA1UEAxMNb3BlbnJlc3R5Lm9yZzCCASIwDQYJKoZI
    25	hvcNAQEBBQADggEPADCCAQoCggEBAOiIE3xiM+FYFjj20Hf4UJ9mkdeFoZBZWZiI
    26	QfE9WEpAGEIXh/OgmRTKuyJIbE5rNSWd+nh1C8XYkv2zYQx26P0ywdjeQM0tCLso
    27	Xa5f+X0hkfmhck4UMgfHYlTGnKLofKbi1qhFeCo1Q7V3ilD4xRSyMzUh8PeP57+E
    28	9AzYq5SYkkG+Z6YtY0ByF+yHzR7ZoDYs35dJNiC9LF3A5fBDKuyAlXyz5q8Ojd5v
    29	4FmuL4or608peh9i2yT2ssvurjTP426LVBXqlOhl12G8BjHh9RzO6tj4dGMZR9vb
    30	S9BkmCbLwcx8VhSdfuIWjSSfwuVmB7nZzo8qspAxwFdBDa4tkwMCAwEAAaOCArEw
    31	ggKtMB8GA1UdIwQYMBaAFMjZeGii2Rlo1T1y3l8KPty1hoamMB0GA1UdDgQWBBRs
    32	dl7F0P1LZe3lj+wgEoPvU/AXKzAOBgNVHQ8BAf8EBAMCBaAwDAYDVR0TAQH/BAIw
    33	ADAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwSQYDVR0gBEIwQDA0Bgsr
    34	BgEEAbIxAQICTjAlMCMGCCsGAQUFBwIBFhdodHRwczovL3NlY3RpZ28uY29tL0NQ
    35	UzAIBgZngQwBAgEwgYgGCCsGAQUFBwEBBHwwejBLBggrBgEFBQcwAoY/aHR0cDov
    36	L3plcm9zc2wuY3J0LnNlY3RpZ28uY29tL1plcm9TU0xSU0FEb21haW5TZWN1cmVT
    37	aXRlQ0EuY3J0MCsGCCsGAQUFBzABhh9odHRwOi8vemVyb3NzbC5vY3NwLnNlY3Rp
    38	Z28uY29tMIIBBAYKKwYBBAHWeQIEAgSB9QSB8gDwAHcArfe++nz/EMiLnT2cHj4Y
    39	arRnKV3PsQwkyoWGNOvcgooAAAGJ9tcFogAABAMASDBGAiEAj6zE+0Y5GArXYi+P
    40	GcZvNl3HJ3lLQbq+AfK71GLIkdICIQCQi9XzpIXKm+h4z2NRflKanaqmG2grHHZQ
    41	jTpbA7buPwB1AHoyjFTYty22IOo44FIe6YQWcDIThU070ivBOlejUutSAAABifbX
    42	BfwAAAQDAEYwRAIgfz03vRCcf0I4aufkiUHLtKeW8dpEOrpUp7uTwMSTs5MCICCB
    43	qkfDujIK13gELAS4e8vuuJ1X8YGuI0AwTDKhzESPMFAGA1UdEQRJMEeCDW9wZW5y
    44	ZXN0eS5vcmeCEWNvbi5vcGVucmVzdHkub3JnghBxYS5vcGVucmVzdHkub3JnghF3
    45	d3cub3BlbnJlc3R5Lm9yZzANBgkqhkiG9w0BAQwFAAOCAgEANf+P2Yg6A5GgwKvw
    46	slzncT6uJRIKMZplMoSwljME8di9iqpXZLGFFoIFWTAjtAoU6wBvJqDbSyaE4nRx
    47	CvKA/Is4k+Chloorn7hzBV2dFHEI18q6OoiLKwvOiaopblGpcMU6/5Myz5uhnF4n
    48	uhGr167BYLgM3+QwJz18crPXl2NmqwJER9f8N2D10RiZuIdpDpFaZaY75Gj8hfhc
    49	cmdtjLYohvJD212y65Og0DaEm0nLxgRurxcz3JenDKZJOEl67IiiaZLentOigYCn
    50	Dw8PjZAuTo4XLZZblBmtXG15QPZEuPbeGtcu1ldqXYTTiWxEReCQsWpU12HdRgwO
    51	2X5gu3HEOjOAsUfY192golooHZxvX7qyiTukbbcQ0tvYOq/rsBh8MSPdiHZrnevj
    52	LTVoNxTLEEQRr9dyGC7zuIVwL06f6h5zG54dNz/QEieaCsLWKF5FSkfxYOAuc6CO
    53	fkHLwHssRWWxOhvP4MF0SCglfdWOoMc+a05hK8yG1JuLyM9BnSzQEVoi8h04NZCA
    54	HFkS34UL16KY37LuyuJrdWj2XZiwNtb3KIB2yMFbOks+GRoujJpsTWxVnKdKOdXz
    55	oqkTiIPJegqpoaGacrGJCTHCUtwUPIs9Pd9l6DOyEXPaMV7O6AOeEG9UZe6eO6J0
    56	4N646F6DdL+6l+vqUaIjFbo9+0Y=
    57	-----END CERTIFICATE-----
    58	subject=CN = openresty.org
    59
    60	issuer=C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    61
    62	---
    63	No client certificate CA names sent
    64	Peer signing digest: SHA256
    65	Peer signature type: RSA-PSS
    66	Server Temp Key: X25519, 253 bits
    67	---
    68	SSL handshake has read 5354 bytes and written 303 bytes
    69	Verification: OK
    70	---
    71	New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
    72	Server public key is 2048 bit
    73	Secure Renegotiation IS supported
    74	Compression: NONE
    75	Expansion: NONE
    76	No ALPN negotiated
    77	SSL-Session:
    78	    Protocol  : TLSv1.2
    79	    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
    80	    Session-ID: 59B452EEE791A0E9FB1972D197835130516F4887877C01B036BC138E9C886C5A
    81	    Session-ID-ctx:
    82	    Master-Key: 20F9646695BC413BEAD805523EE95EA5E1A65C9C9A985F6B46EE10F2BB996C3AAB1F1342DAF2AA07DB0C2F39A4AF9034
    83	    PSK identity: None
    84	    PSK identity hint: None
    85	    SRP username: None
    86	    Start Time: 1692436027
    87	    Timeout   : 7200 (sec)
    88	    Verify return code: 0 (ok)
    89	    Extended master secret: yes
    90	---
    91	drop connection and then reconnect
    92	CONNECTED(00000003)
    93	Verification: OK
    94	---
    95	Reused, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
    96	Secure Renegotiation IS supported
    97	Compression: NONE
    98	Expansion: NONE
    99	No ALPN negotiated
   100	SSL-Session:
   101	    Protocol  : TLSv1.2
   102	    Cipher    : ECDHE-RSA-AES256-GCM-SHA384
   103	    Session-ID: 59B452EEE791A0E9FB1972D197835130516F4887877C01B036BC138E9C886C5A
   104	    Session-ID-ctx:
   105	    Master-Key: 20F9646695BC413BEAD805523EE95EA5E1A65C9C9A985F6B46EE10F2BB996C3AAB1F1342DAF2AA07DB0C2F39A4AF9034
   106	    PSK identity: None
   107	    PSK identity hint: None
   108	    SRP username: None
   109	    Start Time: 1692436027
   110	    Timeout   : 7200 (sec)
   111	    Verify return code: 0 (ok)
   112	    Extended master secret: yes
   113	---
```

如果我们使用 tls1_3 进行测试，可以看到每次都是 `new`，比如 70 行，89 行。
因为 tls1_3 支持的是 session ticket，而不支持 session id 的会话复用

```shell
     1	$ echo | openssl s_client  -servername openresty.org  -connect openresty.org:443 -reconnect -no_ticket > a.log 2>&1
     2	depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
     3	verify return:1
     4	depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
     5	verify return:1
     6	depth=0 CN = openresty.org
     7	verify return:1
     8	CONNECTED(00000003)
     9	---
    10	Certificate chain
    11	 0 s:CN = openresty.org
    12	   i:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    13	 1 s:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    14	   i:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    15	 2 s:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    16	   i:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
    17	---
    18	Server certificate
    19	-----BEGIN CERTIFICATE-----
    20	MIIGoDCCBIigAwIBAgIQAf8IN3hitNWb/V16aIBRujANBgkqhkiG9w0BAQwFADBL
    21	MQswCQYDVQQGEwJBVDEQMA4GA1UEChMHWmVyb1NTTDEqMCgGA1UEAxMhWmVyb1NT
    22	TCBSU0EgRG9tYWluIFNlY3VyZSBTaXRlIENBMB4XDTIzMDgxNTAwMDAwMFoXDTIz
    23	MTExMzIzNTk1OVowGDEWMBQGA1UEAxMNb3BlbnJlc3R5Lm9yZzCCASIwDQYJKoZI
    24	hvcNAQEBBQADggEPADCCAQoCggEBAOiIE3xiM+FYFjj20Hf4UJ9mkdeFoZBZWZiI
    25	QfE9WEpAGEIXh/OgmRTKuyJIbE5rNSWd+nh1C8XYkv2zYQx26P0ywdjeQM0tCLso
    26	Xa5f+X0hkfmhck4UMgfHYlTGnKLofKbi1qhFeCo1Q7V3ilD4xRSyMzUh8PeP57+E
    27	9AzYq5SYkkG+Z6YtY0ByF+yHzR7ZoDYs35dJNiC9LF3A5fBDKuyAlXyz5q8Ojd5v
    28	4FmuL4or608peh9i2yT2ssvurjTP426LVBXqlOhl12G8BjHh9RzO6tj4dGMZR9vb
    29	S9BkmCbLwcx8VhSdfuIWjSSfwuVmB7nZzo8qspAxwFdBDa4tkwMCAwEAAaOCArEw
    30	ggKtMB8GA1UdIwQYMBaAFMjZeGii2Rlo1T1y3l8KPty1hoamMB0GA1UdDgQWBBRs
    31	dl7F0P1LZe3lj+wgEoPvU/AXKzAOBgNVHQ8BAf8EBAMCBaAwDAYDVR0TAQH/BAIw
    32	ADAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwSQYDVR0gBEIwQDA0Bgsr
    33	BgEEAbIxAQICTjAlMCMGCCsGAQUFBwIBFhdodHRwczovL3NlY3RpZ28uY29tL0NQ
    34	UzAIBgZngQwBAgEwgYgGCCsGAQUFBwEBBHwwejBLBggrBgEFBQcwAoY/aHR0cDov
    35	L3plcm9zc2wuY3J0LnNlY3RpZ28uY29tL1plcm9TU0xSU0FEb21haW5TZWN1cmVT
    36	aXRlQ0EuY3J0MCsGCCsGAQUFBzABhh9odHRwOi8vemVyb3NzbC5vY3NwLnNlY3Rp
    37	Z28uY29tMIIBBAYKKwYBBAHWeQIEAgSB9QSB8gDwAHcArfe++nz/EMiLnT2cHj4Y
    38	arRnKV3PsQwkyoWGNOvcgooAAAGJ9tcFogAABAMASDBGAiEAj6zE+0Y5GArXYi+P
    39	GcZvNl3HJ3lLQbq+AfK71GLIkdICIQCQi9XzpIXKm+h4z2NRflKanaqmG2grHHZQ
    40	jTpbA7buPwB1AHoyjFTYty22IOo44FIe6YQWcDIThU070ivBOlejUutSAAABifbX
    41	BfwAAAQDAEYwRAIgfz03vRCcf0I4aufkiUHLtKeW8dpEOrpUp7uTwMSTs5MCICCB
    42	qkfDujIK13gELAS4e8vuuJ1X8YGuI0AwTDKhzESPMFAGA1UdEQRJMEeCDW9wZW5y
    43	ZXN0eS5vcmeCEWNvbi5vcGVucmVzdHkub3JnghBxYS5vcGVucmVzdHkub3JnghF3
    44	d3cub3BlbnJlc3R5Lm9yZzANBgkqhkiG9w0BAQwFAAOCAgEANf+P2Yg6A5GgwKvw
    45	slzncT6uJRIKMZplMoSwljME8di9iqpXZLGFFoIFWTAjtAoU6wBvJqDbSyaE4nRx
    46	CvKA/Is4k+Chloorn7hzBV2dFHEI18q6OoiLKwvOiaopblGpcMU6/5Myz5uhnF4n
    47	uhGr167BYLgM3+QwJz18crPXl2NmqwJER9f8N2D10RiZuIdpDpFaZaY75Gj8hfhc
    48	cmdtjLYohvJD212y65Og0DaEm0nLxgRurxcz3JenDKZJOEl67IiiaZLentOigYCn
    49	Dw8PjZAuTo4XLZZblBmtXG15QPZEuPbeGtcu1ldqXYTTiWxEReCQsWpU12HdRgwO
    50	2X5gu3HEOjOAsUfY192golooHZxvX7qyiTukbbcQ0tvYOq/rsBh8MSPdiHZrnevj
    51	LTVoNxTLEEQRr9dyGC7zuIVwL06f6h5zG54dNz/QEieaCsLWKF5FSkfxYOAuc6CO
    52	fkHLwHssRWWxOhvP4MF0SCglfdWOoMc+a05hK8yG1JuLyM9BnSzQEVoi8h04NZCA
    53	HFkS34UL16KY37LuyuJrdWj2XZiwNtb3KIB2yMFbOks+GRoujJpsTWxVnKdKOdXz
    54	oqkTiIPJegqpoaGacrGJCTHCUtwUPIs9Pd9l6DOyEXPaMV7O6AOeEG9UZe6eO6J0
    55	4N646F6DdL+6l+vqUaIjFbo9+0Y=
    56	-----END CERTIFICATE-----
    57	subject=CN = openresty.org
    58
    59	issuer=C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    60
    61	---
    62	No client certificate CA names sent
    63	Peer signing digest: SHA256
    64	Peer signature type: RSA-PSS
    65	Server Temp Key: X25519, 253 bits
    66	---
    67	SSL handshake has read 5436 bytes and written 387 bytes
    68	Verification: OK
    69	---
    70	New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
    71	Server public key is 2048 bit
    72	Secure Renegotiation IS NOT supported
    73	Compression: NONE
    74	Expansion: NONE
    75	No ALPN negotiated
    76	Early data was not sent
    77	Verify return code: 0 (ok)
    78	---
    79	depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    80	verify return:1
    81	depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
    82	verify return:1
    83	depth=0 CN = openresty.org
    84	verify return:1
    85	drop connection and then reconnect
    86	CONNECTED(00000003)
    87	Verification: OK
    88	---
    89	New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
    90	Secure Renegotiation IS NOT supported
    91	Compression: NONE
    92	Expansion: NONE
    93	No ALPN negotiated
    94	Early data was not sent
    95	Verify return code: 0 (ok)
    96	---
    97	depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
    98	verify return:1
    99	depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
   100	verify return:1
   101	depth=0 CN = openresty.org
   102	verify return:1
```

## session ticket 会话恢复测试

测试 session ticket 的会话恢复，可以使用 -sess_out 先把会话的 ticket 信息写入文件，然后用 -sess_in 的选项指定该 ticket 文件。

举例如下：

```shell
{ sleep 0.3; echo "GET /"; } | openssl s_client  -servername openresty.org  -connect openresty.org:443 -sess_out sess.pem; \
echo | openssl s_client  -servername openresty.org  -connect openresty.org:443 -sess_in sess.pem

CONNECTED(00000003)
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
verify return:1
depth=0 CN = openresty.org
verify return:1
---
Certificate chain
 0 s:CN = openresty.org
   i:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
 1 s:C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA
   i:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
 2 s:C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
   i:C = GB, ST = Greater Manchester, L = Salford, O = Comodo CA Limited, CN = AAA Certificate Services
---
Server certificate
-----BEGIN CERTIFICATE-----
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
4N646F6DdL+6l+vqUaIjFbo9+0Y=
-----END CERTIFICATE-----
subject=CN = openresty.org

issuer=C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA

---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: RSA-PSS
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 5436 bytes and written 391 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: FBEC1F9178735B136229A6696C2660A0FC3D015356F1CA9F103FD038DE216BA5
    Session-ID-ctx:
    Resumption PSK: 1F74A19812D9A8BB818A36214DB130AE9C73D199BCF9C875B928EAAE8A98267C02D6F8386B72B08BE76DD0CE2E172143
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 86400 (seconds)
    TLS session ticket:
    0000 - bb 86 df 4f 6d 0e 2b f7-36 d4 92 37 b5 bf 1b 4a   ...Om.+.6..7...J
    0010 - bf f4 b0 37 ea a1 2e 29-9e 4c 77 04 15 ae 94 3a   ...7...).Lw....:
    0020 - 63 7b 41 6c 8d fc 46 25-60 95 77 8e 3c b4 5e d2   c{Al..F%`.w.<.^.
    0030 - f5 42 ea 18 50 19 50 e4-e0 fe a6 23 0a aa 84 e8   .B..P.P....#....
    0040 - 8f 5b db b8 71 2a 38 97-df 01 0f b9 b7 fe 9f 6e   .[..q*8........n
    0050 - c8 4c 64 9f b8 4e 10 6e-0c 57 6a dc 17 60 b0 e3   .Ld..N.n.Wj..`..
    0060 - 6a d5 87 f6 41 52 a7 36-25 98 e0 93 c7 f3 92 a4   j...AR.6%.......
    0070 - ad 21 ec 37 15 ba 52 26-b4 74 0e 82 20 bb f7 b8   .!.7..R&.t.. ...
    0080 - 4f d9 ed a3 e8 dc 29 41-5b 06 7d 4d 4f e6 e1 75   O.....)A[.}MO..u
    0090 - de 0b 27 4a c0 c7 1e 71-81 d1 4a 2e f7 7e 98 19   ..'J...q..J..~..
    00a0 - 3b 9c 04 78 83 81 3a 4f-33 1a da ef d3 4f d1 3c   ;..x..:O3....O.<
    00b0 - d3 41 96 3b fa b1 9b 23-b8 92 ff d6 9c 29 c6 45   .A.;...#.....).E
    00c0 - b8 95 53 bc c0 bb 3e bb-0d 64 58 94 b5 f0 03 13   ..S...>..dX.....
    00d0 - 65 c0 9c 55 07 a2 ba af-9d cc 22 9c 99 62 2e 11   e..U......"..b..
    00e0 - b3 c7 5e 8b 9d 0f 27 2b-b5 72 e1 c1 23 d6 ff 02   ..^...'+.r..#...

    Start Time: 1692437641
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
---
Post-Handshake New Session Ticket arrived:
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: 17D74167E06A9809A19322B69B7C80B17A4B4D3AC3A9578ABFF64155604CF99D
    Session-ID-ctx:
    Resumption PSK: 0AF25A198A344C9576F9CC2DCE58D50024458CBE1F72593E9FED0C7B26D9A4222AB8E62D9CD669DED647E659C37179C0
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 86400 (seconds)
    TLS session ticket:
    0000 - bb 86 df 4f 6d 0e 2b f7-36 d4 92 37 b5 bf 1b 4a   ...Om.+.6..7...J
    0010 - 70 74 ce ee 8d 41 43 7f-44 28 77 37 16 c3 ef 0a   pt...AC.D(w7....
    0020 - 9f 22 aa 04 59 3b b7 27-88 38 36 7a d8 50 20 96   ."..Y;.'.86z.P .
    0030 - 28 7f ac 99 81 9c d9 b0-15 04 4c 90 91 ca e3 84   (.........L.....
    0040 - 91 76 40 d9 58 3b 43 8d-0d 1f 3a d1 5c df f3 82   .v@.X;C...:.\...
    0050 - 4c 30 62 39 c8 d0 23 9b-59 26 ed fc 7b f1 3a e0   L0b9..#.Y&..{.:.
    0060 - bb 49 e6 07 e1 fd ca 35-4a c3 93 92 bb 72 c1 0e   .I.....5J....r..
    0070 - 40 2a 22 ba fe f9 a8 fc-3e 0f 20 ea 6c 23 26 3a   @*".....>. .l#&:
    0080 - b0 b3 15 63 a7 58 3c d4-bb 0b e6 60 e8 b3 40 ab   ...c.X<....`..@.
    0090 - 23 62 e9 c6 c3 b5 1f 92-be dc f8 77 15 d7 fa a8   #b.........w....
    00a0 - 97 84 21 e5 96 9e 4d 0c-3c 18 26 80 c7 40 6a ff   ..!...M.<.&..@j.
    00b0 - 75 9e 31 04 75 56 9d 42-0f 84 0d 52 0e 45 d4 1c   u.1.uV.B...R.E..
    00c0 - f2 71 e7 03 7e 5f 38 dc-da 77 3b 01 13 03 5a ae   .q..~_8..w;...Z.
    00d0 - f4 9a ca 87 7a ad 92 1f-ed 90 12 45 a8 8c 78 4b   ....z......E..xK
    00e0 - 9a 9d 8f 9e 50 9e f9 26-14 14 19 c5 90 4e 39 95   ....P..&.....N9.

    Start Time: 1692437641
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
    Max Early Data: 0
---
read R BLOCK
DONE
CONNECTED(00000003)
---
Server certificate
-----BEGIN CERTIFICATE-----
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
4N646F6DdL+6l+vqUaIjFbo9+0Y=
-----END CERTIFICATE-----
subject=CN = openresty.org

issuer=C = AT, O = ZeroSSL, CN = ZeroSSL RSA Domain Secure Site CA

---
No client certificate CA names sent
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 245 bytes and written 694 bytes
Verification: OK
---
Reused, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
```

有些术语可能使用的不准确，欢迎指正。请发邮件联系：zhuizhuhaomeng@gmail.com
