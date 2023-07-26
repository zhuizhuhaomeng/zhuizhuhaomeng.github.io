---
layout: post
title: "ocserv 组网配置"
description: ""
date: 2023-07-23
tags: [ocserv, VPN, Admin]
---


# 参考文档

搭建过程主要基于官方文档： https://ocserv.gitlab.io/www/manual.html 

# 主要参数

这里假设域名为 `my.com`, 请根据实际需要修改。后面会反复用到 `my.com` 这个域名

| 参数名称 | 参数值 | 说明 |
|---|---|---|
| 认证方式  | 证书 | |
| 端口 | 1443 |  |
| 域名 | ocserv.my.com |  |


# 安装 ocserv 软件

```shell
yum -y install ocserv
mkdir /etc/ocserv/ssl
cd /etc/ocserv/ssl
```

# 生成相关证书

## 生成CA证书

执行以下脚本生成CA证书

```shell
cd /etc/ocserv/ssl
certtool --generate-privkey --outfile ca-key.pem
cat << _EOF_ >ca.tmpl
cn = "VPN CA"
organization = "my.com"
serial = 1
expiration_days = 400000
ca
signing_key
cert_signing_key
crl_signing_key
_EOF_

certtool --generate-self-signed --load-privkey ca-key.pem \
           --template ca.tmpl --outfile ca-cert.pem
```

## 生成服务器证书

执行以下命令生成服务端证书

```shell
cd /etc/ocserv/ssl
certtool --generate-privkey --outfile server-key.pem
cat << _EOF_ >server.tmpl
cn = "VPN server"
dns_name = "ocserv.my.com"
organization = "my.com"
expiration_days = 4000
signing_key
encryption_key #only if the generated key is an RSA one
tls_www_server
_EOF_

certtool --generate-certificate --load-privkey server-key.pem \
           --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem \
           --template server.tmpl --outfile server-cert.pem
```

# 添加用户

将下述下述脚本保存为 add_user.sh,  然后执行 ./add_user.sh user1 这样的形式创建用户证书。

```shell
#!/bin/bash

if [ $# != 1 ];then
   exit 1
fi

ocserv_user=$1
certtool --generate-privkey --outfile ${ocserv_user}-key.pem
cat <<EOF >${ocserv_user}.tmpl
dn = "cn=${ocserv_user},O=my.com,UID=${ocserv_user}"
unit = "admins"
#if usernames are SAN(rfc822name) email addresses
#email = "username@example.com"
expiration_days = 3650
signing_key
tls_www_client
EOF

certtool --generate-certificate --load-privkey ${ocserv_user}-key.pem \
 --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem \
 --template ${ocserv_user}.tmpl --outfile ${ocserv_user}-cert.pem
```

# 服务端配置

配置文件的路径是：/etc/ocserv/ocserv.conf

该文件重点修改 auth，tcp-port，udp-port 以及 server-cert， server-key，ca-cert，ipv4-network， route-add-cmd， route-del-cmd, config-per-user 这些参数。

route-add-cmd 主要是为了添加路由并做 SNAT，实现访问内网其它网段的机器。

```text
auth = "certificate"
tcp-port = 1443
udp-port = 1443
run-as-user = ocserv
run-as-group = ocserv
socket-file = ocserv.sock
chroot-dir = /var/lib/ocserv
isolate-workers = true
max-clients = 16
max-same-clients = 2
rate-limit-ms = 100
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = false
server-cert = /etc/ocserv/ssl/server-cert.pem
server-key = /etc/ocserv/ssl/server-key.pem
ca-cert = /etc/ocserv/ssl/ca-cert.pem
cert-user-oid = 0.9.2342.19200300.100.1.1
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 50
ban-reset-time = 300
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
default-domain = example.com
ipv4-network = 10.0.3.0/24
ping-leases = false
config-per-user = /etc/ocserv/config-per-user/
route-add-cmd = "ip route add %{R} dev %{D}; iptables -t nat -I POSTROUTING -d %{R} -j MASQUERADE"
route-del-cmd = "ip route delete %{R} dev %{D}; iptables -t nat -D POSTROUTING -d %{R} -j MASQUERADE"
cisco-client-compat = true
dtls-legacy = true
user-profile = profile.xml
route=192.168.50.0/24
```

## 服务器配置中针对特定客户端的配置

route 表示该客户端连接过来时，通告客户端应该添加那些路由。在主配置文件中配置了 route 后，如果个人配置又配置了 route，那么以个人配置为主。

iroute表示客户端连接后，这些路由会指向该客户端。

route 和 iroute的方向是相反的。

如果要配置固定IP地址，使用 explicit-ipv4。

```shell
$ cat config-per-user/sz-intel.dev 
explicit-ipv4=10.0.3.6
# The client is already under 192.168.50.0/24, so we only add 10.234.3.0/24 here
route=10.234.3.0/24
iroute=192.168.50.0/24
```

## 启动服务器

```shell
systemctl enable ocserv
systemctl start ocserv
```

# 客户端配置

这个证书是上面生成的 CA 证书, 需要保存到客户端上。如果是使用公开签名的证书，那么不需要指定 CA 证书。

```text
-----BEGIN CERTIFICATE-----
MIIDCDCCAfCgAwIBAgIBATANBgkqhkiG9w0BAQsFADAkMQ8wDQYDVQQDEwZWUE4g
Q0ExETAPBgNVBAoTCEJpZyBDb3JwMCAXDTIzMDIwMzAyNTM0MloYDzk5OTkxMjMx
...
acitxhxQPYfqprwaAIFXOSjhGR1+Eq1H0FyX/U87lvOvmCqAcyqW0VTB1oVBV4Ra
YHef4V2vMRPcqhCuYCvA1ZmoPqJkOUIxum9WNezBtupI1HZ438zO3t1OuTxXg6Q4
tNlidJC1VimcFtNFEpKdE4ZiD/4LXPvsSxHYtGFILUuB9RaDa9XNfoiS18HuPe9E
/UEUiONlt5Wk0EHp
-----END CERTIFICATE-----
```

## Windows

下载 openconnect-gui：

https://github.com/openconnect/openconnect-gui/releases/download/v1.5.3/openconnect-gui-1.5.3-win32.exe

新增 profile：

```
Name：office

Gateway：https://ocserv.my.com:1443
```

## Linux

客户端手动连接命令

```shell
sudo openconnect -b -c user1-cert.pem -k user1-key.pem \
    --cafile ./ca-cert.pem https://ocserv.my.com:1443
```

### 客户端 systemd 脚本

该脚本可以再断开时重连。

需要将相关的个人证书和 CA 证书放到 /etc/openconnect 下。

将下述配置保存为 openconnect.service，然后拷贝到 /usr/lib/systemd/system/openconnect.service

```
[Unit]
Description=OpenConnect  VPN
Wants=network-online.target
After=network-online.target nss-lookup.target

[Service]
Type=simple
User=root
ExecStart=openconnect --cafile /etc/openconnect/ca-cert.pem -c /etc/openconnect/sz-intel.dev-cert.pem -k /etc/openconnect/sz-intel.dev-key.pem ocserv.openresty.com.cn:1443
KillSignal=SIGINT
Restart=always
RestartSec=10

StartLimitIntervalSec=200
StartLimitBurst=10

[Install]
WantedBy=multi-user.target
```

## MacOS

客户端下载：https://github.com/openconnect/openconnect-gui/releases/download/v1.5.3/openconnect-gui-1.5.3.high_sierra.bottle.tar.gz

命令行：

```shell
brew install openconnect
```
