---
layout: post
title: "使用 ipset 打造低成本的 IP 防火墙"
description: "虽然 ebpf，XDP 才是未来的方向，但是当前用户的使用门槛却仍然是比较高的"
date: 2023-03-05
tags: [OpenResty, Nginx, DDOS, ipset]
---

# 问题

我们的 DNS 服务器遭到攻击了，每秒高达几万 qps，这个时候怎么办呢？

如果在应用层识别攻击流量并将其丢弃，虽然可以缓解部分问题问题，但是这对于 DDOS 的大流量攻击还是力不从心。
显然拦截报文应该在越靠近网卡收包的位置越好。目前比较理想的是用 ebpf，XDP 来实现报文的过滤。
虽然ebpf，XDP 性能很高，但是他们不是简单的命令行操作就可以搞定的事情。

# 简单的重现

这个是利用 OpenResty 创建的一个简单的 UDP 服务器

```nginx
stream {
    server {
        listen 2023 udp reuseport;
        content_by_lua_block {
            local s = ngx.req.socket(true)
            local bytes, err = s:send("1: received:\n")
            if not bytes then
                ngx.log(ngx.ERR, "server: failed to send: ", err)
                return
            end
        }
    }
}
```

我们使用这个仓库 [udp-flood](https://github.com/araujo88/udpflood) 的代码稍微修改一下
固定源 IP 和 目的 IP 地址。

可以看到，nginx 的 CPU 被打到 99.7%， 收包速率高达 113306 rxpck/s。

```shell
$ top
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 611706 nobody    20   0   70832  11524   7812 R  99.7   0.0   0:44.37 nginx

$ sar -n DEV 1 10
 06:36:53 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
06:36:54 PM    enp1s0 113306.00  39142.00   6086.60   2108.95      0.00      0.00      0.00      0.00
```

下面介绍一种基于 ipset 进行报文过滤的解决方案。

# ipset 拦截攻击报文

首先创建一个名称为 black_list 的 ipset 表，在 iptables 的 PREROUTING 阶段 添加一个过滤规则。

```shell
sudo ipset create black_list hash:ip timeout 3600
sudo iptables -I PREROUTING -t raw -m set --match-set black_list src -j DROP
```

其次我们将黑名单中的 IP 地址添加到 ipset 中

```shell
sudo ipset add black_list 192.168.0.181
sudo ipset add black_list 192.168.0.199
```

再次查看 CPU 利用率，可以看到 nginx 的CPU 利用率已经为0，报文全部被拦截了。
这时候就有查看软中断的开销了。

```shell
top - 18:44:02 up  9:34,  4 users,  load average: 0.07, 0.16, 0.17
Tasks: 638 total,   3 running, 633 sleeping,   2 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni, 83.5 id,  0.0 wa,  3.3 hi, 13.2 si,  0.0 st
```

可以看到 CPU 0 的软中断开销为13.2%。如果使用bpf让网卡直接处理报文，不需要拷贝到 skb，
那么这部分的 CPU 开销也可以进一步降低了。

## 批量添加操作

如果每一个 IP 执行一条 ipset add 命令，显然很低效。
在首次初始化的时候可以使用 ipset resotre 的命令实现批量的添加。

```shell
cat ips.txt | sudo ipset restore
```

而这个文件的格式是怎么样的？我们可以看看 `ipset save` 的一个输出样例。

```shell
$ sudo ipset save black_list
create black_list hash:ip family inet hashsize 1024 maxelem 65536 timeout 3600
add black_list 192.168.0.199 timeout 3598
add black_list 192.168.0.181 timeout 2831
```

如果是为了匹配添加而不是首次添加，那么不需要 第一条 create 命令。

## ipset 的一些操作

如果想要查看添加了哪些 IP,可以使用下面的命令

```shell
$ sudo ipset list black_list
Name: black_list
Type: hash:ip
Revision: 4
Header: family inet hashsize 1024 maxelem 65536 timeout 3600
Size in memory: 216
References: 1
Number of entries: 1
Members:
192.168.0.181 timeout 3297
```

# 参考链接

https://ipset.netfilter.org/ipset.man.html
