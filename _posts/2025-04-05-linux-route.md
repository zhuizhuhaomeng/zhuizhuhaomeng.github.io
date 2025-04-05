---
layout: post
title: "Linux 路由"
description: ""
date: 2025-04-05
modified: 2025-04-05
tags: [Linux, route table]
---

最近搞了一个通过 wiregurad 对 openvpn 进行加速分流的功能，这里做一个总结。
系统使用的是 RockyLinux-9 的操作系统。

# 安装软件

```shell
yum install -y openvpn wireguard-tools
```

# 配置 openvpn

该文件应该放在 /etc/openvpn/client/vpn2.conf，注意文件名以 conf 结尾。

## 配置文件的修改

因为默认的 配置文件不能满足要求，因此要做一下微小的调整。
主要是修改接口名称和禁用默认路由。

禁用默认路由很重要，因为如果不禁用默认路由，那么远程 SSH 连接的服务器就断开了。

```config
# 每一个 vpn 分配一个不同的接口。比如 vpn3，vpn4
dev tun-vpn2

resolv-retry infinite
nobind
persist-key
persist-tun
client
verb 3

# 禁用默认路由
route-noexec
#auth-user-pass

# vpn 连接成功时执行的脚本
up "/opt/openvpn/tun-vpn2-up.sh"

# vpn 关闭后执行的脚本
down "/opt/openvpn/tun-vpn2-down.sh"
```

## tun-vpn2-up.sh 的内容

```shell
iptables -t nat -I POSTROUTING -o tun-vpn2 -j MASQUERADE
ip route add default dev tun-vpn2 table 10
ip rule add from 192.168.100.0/24 lookup 10
#这里的 10 是 table id，跟上面的 tun 接口一一对应。
# 比如 vpn3 就可以对应 11，vpn4 对应 12
```

## tun-vpn2-down.sh 的内容是

```shell
iptables -t nat -D POSTROUTING -o tun-vpn2 -j MASQUERADE
ip rule del from 192.168.100.0/24 lookup 10
ip route del default dev tun-vpn2 table 10
```

### 启动 VPN

```shell
systemctl start openvpn-client@vpn2
```

# Wireguard 配置

## 生成密钥
```shell
wg genkey | tee privatekey | wg pubkey > publickey
```

## 配置 服务端的 wireguard

这个是 /etc/wireguard/wg0.conf 的配置示例

```config
[Interface]
PrivateKey = 2GHTXxxxxxxJWNjdz8N+vFvYYVc=
ListenPort = 51821
Address = 10.0.100.1/24

[Peer]
PublicKey = IPBvlUhuRHIdsjXkH/6SM9tI/Rvsihn8rT2ZRuOzxhE=
AllowedIPs = 10.0.100.2/32, 192.168.100.0/24
PersistentKeepalive = 25
```

## 启动 wireguard

```shell
systemctl restart wg-quick@wg0
```

# 配置操作系统

```shell
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sysctl -p

# 默认的 iptables 规则
# 这条的作用是防止 wg0 的接口不走 vpn，直接转发出去。
iptables -I FORWARD -i wg0 -o eth0 -j DROP
```

