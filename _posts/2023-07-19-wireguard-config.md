---
layout: post
title: "wireguard 组网配置"
description: ""
date: 2023-07-19
tags: [WireGuard, VPN, Admin]
---

1. 安装 wireguard 客户端

参考 https://www.wireguard.com/install/

```shell
yum install wireguard-tools
```

2. 生成公钥和配置 IP 地址

带图形界面的客户端，新建一个隧道就会自动生成成对的私钥和公钥。

如果是命令行，则使用如下命令：

```shell
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey > publickey
```

将 publickey 内容读取出来，即为对应的公钥

# 配置软件

配置文件为： /etc/wireguard/wg0.conf

配置 [interface] 部分，如下（需要修改私钥和 IP 地址）

假设用的 IP 地址为 10.0.0.0/24 网段，Address 字段掩码固定填 24

注意这里的 Peer 为对端的配置，应该填入对端的公钥

```
[Interface]
PrivateKey = [your private key here]
ListenPort = 10088
Address = 10.0.0.x/24
PostUp = iptables -I FORWARD -i %i -j ACCEPT; iptables -I FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o br0 -j MASQUERADE;iptables -t nat -I POSTROUTING -o %i -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o br0 -j MASQUERADE

[Peer]
PublicKey = xj9iTUL8WSbvMZa3As22K3QBfw1tkp0HvYv/J9HYnkE=
AllowedIPs = 10.0.0.0/24,192.168.80.0/24
Endpoint = ip.address.peer
PersistentKeepalive = 25
```

wireguard 的两个节点是对等的，因此需要登陆到对端把本机的添加上去。
上面的配置中有 PostUp，PostDown 两个配置项，表示对应的网络接口启动和停止时要做的操作。
这里添加的目的就是为了让本机作为软路由，代理其它的设备数据转发，这样就可以做到一台机器配置 wiregurad 的 VPN 隧道，
其它设备将配置了 VPN 的机器作为网关。

# 系统配置

```shell
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sysctl -p
systemctl enable  wg-quick@wg0
systemctl start  wg-quick@wg0
```

# 重启软件

```shell
sudo systemctl restart wg-quick@wg0
```

# 故障和调试

## 调试

使用 systemctl 启动时出错不方便查看错误，可以使用下面的命令进行启动和停止

```shell
wg-quick up wg0
wg-quick down wg0
```

## 关闭防火墙

为了防止防火墙和配置的 iptables 命令冲突，需要关闭防火墙。

```shell
systemctl disable firewalld
```

## 抓包分析

如果是隧道内的报文问题，可以用这样的命令来查看报文

```shell
sudo tcpdump -i wg0 -nnn
```

## 确认 iptable 规则是不是预期的

使用 iptables-save 查看所有规则，防止其它组件的规则配置导致冲突

```shell
iptables-save
```
