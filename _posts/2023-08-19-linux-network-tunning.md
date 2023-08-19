---
layout: post
title: "Linux 网络调优"
description: ""
date: 2023-08-19
tags: [tunning, Network]
---

为了充分利用机器的性能，我们需要对 Linux 系统参数进行调优。
比如 Nginx/OpenResty 服务器做反向代理，我们就希望单击可以达到百万连接数。

调优的过程中，我们要采用自顶向下，逐层递进的方式来进行调优。这里的自顶向下不是说从应用层到物理层，
而是说我们应该从粗到细，而不应该一下子陷入细节。


在调优之前，我们应该先识别系统的瓶颈，一般都采用模拟业务的压测的方式来制造瓶颈是比较合适的。
那么在压测过程中，我们经常会遇到压不上去，其实这就是遇到瓶颈。我们应该如何才能识别瓶颈并进行调优呢？

# 查询 CPU 的开销和瓶颈

1. top 命令

使用 top 命令有两个目的：

    1. 查看目标进程的 CPU 利用率有多高
    2. 查看系统的各个 CPU 利用率是否均衡，软中断，硬中断，用户态 的CPU 占比各是多少

比如，下面这个例子显示 Luajit 的 CPU 利用率 高达93.8%。同时，我们可以看到 CPU5 的 空闲 CPU 是 0。
并且 CPU5 的用户态开销是 100%，其它的用户态，软中断，硬中断都是 0 。

```shell
$ top -1
top - 18:29:27 up  6:02,  3 users,  load average: 1.01, 1.05, 1.05
Tasks: 176 total,   2 running, 173 sleeping,   1 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  :  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu4  :  0.0 us,  6.2 sy,  0.0 ni, 93.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu5  :100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7762.9 total,    580.3 free,   2150.3 used,   5032.4 buff/cache
MiB Swap:   8092.0 total,   8085.4 free,      6.5 used.   5296.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 155788 ljl       20   0 1064640   1.0g   2248 R  93.8  13.3 113:57.46 luajit
```

再比如，下面这个例子显示，ksoftirq 的 CPU 利用率很高。
因为软中断都集中在单核的上，这就导致单核的性能瓶颈, 同时也造成了CPU利用率不均衡。

```shell
KiB Swap:  8388604 total,  8388604 free,        0 used. 20625841+avail Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    71 root      20   0       0      0      0 S  91.1  0.0   2:32.86 ksoftirqd/12
  2374 resty     20   0 6689840 618448   5044 R  58.1  0.2   0:46.58 nginx
  2416 resty     20   0 6689712 618212   5040 R  40.3  0.2   0:57.41 nginx
  2383 resty     20   0 6689840 618268   5036 R  39.9  0.2   0:12.14 nginx
  2397 resty     20   0 6689712 618244   5040 R  39.9  0.2   0:11.22 nginx
```

在下面这个例子， 软中断分散到 4 个CPU，这样就不会被单核的瓶颈束缚了。

```shell
Tasks: 144 total,   5 running, 139 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.4 us,  0.3 sy,  0.0 ni, 51.1 id,  0.0 wa,  0.0 hi, 48.3 si,  0.0 st
MiB Mem :   7821.2 total,    284.5 free,    384.7 used,   7152.0 buff/cache
MiB Swap:    976.0 total,    976.0 free,      0.0 used.   7130.0 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
     33 root      20   0       0      0      0 R  99.3   0.0   3:38.34 ksoftirqd/4
     11 root      20   0       0      0      0 R  98.3   0.0   0:14.95 ksoftirqd/0
     48 root      20   0       0      0      0 R  82.7   0.0   3:07.15 ksoftirqd/7
     23 root      20   0       0      0      0 R  67.1   0.0   0:12.52 ksoftirqd/2
```

2. pidstat 命令

top 命令查看进程的 CPU 利用率并没有办法区分用户态，内核态的 CPU 占比。可以使用 pidstat 查看进程的 CPU 利用率。

比如：

```shell
$ pidstat -p 161881 -u 1
Linux 4.18.0-372.9.1.el8.x86_64 (rocky8) 	08/19/2023 	_x86_64_	(6 CPU)

10:10:36 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
10:10:37 PM  1000    161881   99.00    0.00    0.00    0.00   99.00     5  luajit
10:10:38 PM  1000    161881   97.00    0.00    0.00    0.00   97.00     5  luajit
10:10:39 PM  1000    161881   98.00    0.00    0.00    0.00   98.00     5  luajit
```

2. mpstat 命令

如果不喜欢 top 命令输出的其它信息的干扰，可以使用 mpstat 命令。
该命令位于 sysstat 软件包中。


```shell
$ mpstat -P ALL 1
Linux 4.18.0-372.9.1.el8.x86_64 (rocky8) 	08/19/2023 	_x86_64_	(6 CPU)

06:40:04 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
06:40:05 PM  all   16.17    0.00    0.17    0.00    0.50    0.00    0.00    0.00    0.00   83.17
06:40:05 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
06:40:05 PM    1    0.00    0.00    0.00    0.00    0.99    0.00    0.00    0.00    0.00   99.01
06:40:05 PM    2    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
06:40:05 PM    3    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
06:40:05 PM    4    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
06:40:05 PM    5   97.00    0.00    1.00    0.00    2.00    0.00    0.00    0.00    0.00    0.00
```

3. free 命令

free 命令主要是查看内存大小，有没有使用到 swap 分区。

```
$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7762        1416        1307          20        5039        6030
Swap:          8091           6        8085
```

# 调优网卡的配置

在上面的配置中，我们看到软中断的 CPU 利用率集中在一个网卡上，这样会导致单个 CPU 过载。
我们应该尽可能的将软中断分散到多个不同的 CPU 上，这样才能充分利用多核 CPU 的优势。

## 查看网卡是否丢包

```shell
$ ethtool -S enp67s0
NIC statistics:
     tx_packets: 9087125983
     rx_packets: 10455834650
     tx_errors: 0
     rx_errors: 1
     rx_missed: 0
     align_errors: 0
     tx_single_collisions: 0
     tx_multi_collisions: 0
     unicast: 10405349404
     broadcast: 30878645
     multicast: 19606601
     tx_aborted: 0
     tx_underrun: 0
```

## 查看和修改网卡的队列数量

```shell
ethtool -l enp1s0
Channel parameters for enp1s0:
Pre-set maximums:
RX:		n/a
TX:		n/a
Other:		1
Combined:	4
Current hardware settings:
RX:		n/a
TX:		n/a
Other:		1
Combined:	4
```

从上面的输出可以看到，该网卡有 4 个 组合队列（输入/输出合在一起），还有一个 Others的队列。
因此这个网卡总共 5个队列，对应 5 个中断。上面的 RX 是输入队列， TX是输出队列。这两个的值
都是 n/a, 表示不支持单独是输入/输出队列。

注意，上面一个是 Pre-set maximums, 一个是 Current hardware settings。前者是硬件的能力值，
后者是当前生效的配置。

在可能的情况下，可以将队列的数量调整为和 CPU 的数量一致。
调整的命令为： ethtool -L 。

比如：
```shell
$ sudo ethtool -L enp1s0 rx 8
```

## 查看和修改网卡的接收队列缓冲区大小

如果网卡的缓冲区太小，那么在瞬时收到大量报文时会造成丢包。配置得太大可能造成堆积太多老的数据报文，处理延迟增大。
因此，一般不需要调整该值。

可以通过命令 ethtool -g 命令查看队列的大小。

比如：

```shell
$ sudo ethtool -g enp0s8
Ring parameters for enp0s8:
Pre-set maximums:
RX:		4096
RX Mini:	n/a
RX Jumbo:	n/a
TX:		4096
Current hardware settings:
RX:		256
RX Mini:	n/a
RX Jumbo:	n/a
TX:		256
```

通过 ethtool -G 命令修改队列大小。

比如：

```shell
$ sudo ethtool -G enp0s8 rx 1024
$ sudo ethtool -g enp0s8
Ring parameters for enp0s8:
Pre-set maximums:
RX:		4096
RX Mini:	n/a
RX Jumbo:	n/a
TX:		4096
Current hardware settings:
RX:		1024
RX Mini:	n/a
RX Jumbo:	n/a
TX:		256
```

## 查看和修改网卡的队列权重

一些网卡支持给不同的 queue 设置不同的权重（weight），权重越大， 分发到该队列的包越多。

```shell
ethtool -x enp68s0
RX flow hash indirection table for enp68s0 with 2 RX ring(s):
    0:      0     0     0     0     0     0     0     0
    8:      0     0     0     0     0     0     0     0
   16:      0     0     0     0     0     0     0     0
   24:      0     0     0     0     0     0     0     0
   32:      0     0     0     0     0     0     0     0
   40:      0     0     0     0     0     0     0     0
   48:      0     0     0     0     0     0     0     0
   56:      0     0     0     0     0     0     0     0
   64:      1     1     1     1     1     1     1     1
   72:      1     1     1     1     1     1     1     1
   80:      1     1     1     1     1     1     1     1
   88:      1     1     1     1     1     1     1     1
   96:      1     1     1     1     1     1     1     1
  104:      1     1     1     1     1     1     1     1
  112:      1     1     1     1     1     1     1     1
  120:      1     1     1     1     1     1     1     1
RSS hash key:
Operation not supported
RSS hash function:
    toeplitz: on
    xor: off
    crc32: off
```

第一列是该行的第一个哈希值，冒号后面的每个哈希值对应的 RX queue。例如，

哈希值是 0~63，对应 RX queue 0；
哈希值是 64~127，对应 RX queue 1。

在前两个 RX queue 之间均匀的分发接收到的包

```shell
$ sudo ethtool -X eth0 equal 2
```

设置自定义权重：给 rx queue 0 和 1 不同的权重：6 和 2

```shell
$ ethtool -X enp68s0 weight 6 2
$ ethtool -x enp68s0
RX flow hash indirection table for enp68s0 with 2 RX ring(s):
    0:      0     0     0     0     0     0     0     0
    8:      0     0     0     0     0     0     0     0
   16:      0     0     0     0     0     0     0     0
   24:      0     0     0     0     0     0     0     0
   32:      0     0     0     0     0     0     0     0
   40:      0     0     0     0     0     0     0     0
   48:      0     0     0     0     0     0     0     0
   56:      0     0     0     0     0     0     0     0
   64:      0     0     0     0     0     0     0     0
   72:      0     0     0     0     0     0     0     0
   80:      0     0     0     0     0     0     0     0
   88:      0     0     0     0     0     0     0     0
   96:      1     1     1     1     1     1     1     1
  104:      1     1     1     1     1     1     1     1
  112:      1     1     1     1     1     1     1     1
  120:      1     1     1     1     1     1     1     1
RSS hash key:
Operation not supported
RSS hash function:
    toeplitz: on
    xor: off
    crc32: off
```

我们再调整为相等的权重

```shell
$ ethtool -x enp68s0
RX flow hash indirection table for enp68s0 with 2 RX ring(s):
    0:      0     1     0     1     0     1     0     1
    8:      0     1     0     1     0     1     0     1
   16:      0     1     0     1     0     1     0     1
   24:      0     1     0     1     0     1     0     1
   32:      0     1     0     1     0     1     0     1
   40:      0     1     0     1     0     1     0     1
   48:      0     1     0     1     0     1     0     1
   56:      0     1     0     1     0     1     0     1
   64:      0     1     0     1     0     1     0     1
   72:      0     1     0     1     0     1     0     1
   80:      0     1     0     1     0     1     0     1
   88:      0     1     0     1     0     1     0     1
   96:      0     1     0     1     0     1     0     1
  104:      0     1     0     1     0     1     0     1
  112:      0     1     0     1     0     1     0     1
  120:      0     1     0     1     0     1     0     1
RSS hash key:
Operation not supported
RSS hash function:
    toeplitz: on
    xor: off
    crc32: off
```

可以看到，这次 0 ~ 127 分别是 0，1交叉的队列。


## 查看调整 RSS RX 哈希字段（ethtool -n/-N）

可以用 ethtool 调整 RSS 计算哈希时所使用的字段。

例子：查看 UDP RX flow 哈希所使用的字段：

```shell
$ sudo ethtool -n eth0 rx-flow-hash udp4
UDP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
```

可以看到只用到了源 IP（SA：Source Address）和目的 IP。我们压测的时候经常是用两台机器，这时候经常遇到压力不均衡的问题。 **这是因为源 IP 和 目的 IP 地址一直保持不变，导致都报文都被分发到同一个队列了。**

我们修改一下，加入源端口和目的端口：

```shell
$ sudo ethtool -N eth0 rx-flow-hash udp4 sdfn
```

## Flow 绑定到 CPU：ntuple filtering（ethtool -k/-K, -u/-U）

一些网卡支持 “ntuple filtering” 特性。该特性允许用户（通过 ethtool ）指定一些参数来 在硬件上过滤收到的包，然后将其直接放到特定的 RX queue。例如，用户可以指定到特定目 端口的 TCP 包放到 RX queue 1。

Intel 的网卡上这个特性叫 Intel Ethernet Flow Director，其他厂商可能也有他们的名字 ，这些都是出于市场宣传原因，底层原理是类似的。

ntuple filtering 其实是 Accelerated Receive Flow Steering (aRFS) 功能的核心部分之一， 这个功能在原理篇中已经介绍过了。aRFS 使得 ntuple filtering 的使用更加方便。

适用场景：最大化数据局部性（data locality），提高 CPU 处理网络数据时的 缓存命中率。例如，考虑运行在 80 口的 web 服务器：

1. webserver 进程运行在 80 口，并绑定到 CPU 2
1. 和某个 RX queue 关联的硬中断绑定到 CPU 2
1. 目的端口是 80 的 TCP 流量通过 ntuple filtering 绑定到 CPU 2
1. 接下来所有到 80 口的流量，从数据包进来到数据到达用户程序的整个过程，都由 CPU 2 处理
1. 监控系统的缓存命中率、网络栈的延迟等信息，以验证以上配置是否生效

检查 ntuple filtering 特性是否打开：

```shell
$ethtool -k enp68s0
Features for enp68s0:
rx-checksumming: on
tx-checksumming: on
	tx-checksum-ipv4: off [fixed]
	tx-checksum-ip-generic: on
	tx-checksum-ipv6: off [fixed]
	tx-checksum-fcoe-crc: off [fixed]
	tx-checksum-sctp: on
scatter-gather: on
	tx-scatter-gather: on
	tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on
ntuple-filters: off
receive-hashing: on
highdma: on [fixed]
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: on
tx-gre-csum-segmentation: on
tx-ipxip4-segmentation: on
tx-ipxip6-segmentation: on
tx-udp_tnl-segmentation: on
tx-udp_tnl-csum-segmentation: on
tx-gso-partial: on
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: on
tx-gso-list: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off [fixed]
rx-all: off
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: on
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: off [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
```

可以看到，上面的 ntuple 是关闭的。

打开：

```shell
$ ethtool -K enp68s0 ntuple on
```

打开 ntuple filtering 功能，并确认打开之后，可以用 ethtool -u 查看当前的 ntuple rules：

```shell
$ sudo ethtool -u enp68s0
2 RX rings available
Total 0 rules
```

可以看到当前没有 rules。

我们来加一条：目的端口是 80 的放到 RX queue 2：

```shell
$ sudo ethtool -U enp68s0 flow-type tcp4 dst-port 80 action 0
```

也可以用 ntuple filtering 在硬件层面直接 drop 某些 flow 的包。 当特定 IP 过来的流量太大时，这种功能可能会派上用场。更多关于 ntuple 的信息，参考 ethtool man page。

ethtool -S <DEVICE> 的输出统计里，Intel 的网卡有 fdir_match 和 fdir_miss 两项， 是和 ntuple filtering 相关的。关于具体、详细的统计计数，需要查看相应网卡的设备驱 动和 data sheet。

来自： http://arthurchiao.art/blog/linux-net-stack-tuning-rx-zh/

## 查看和配置网卡中断合并（Interrupt coalescing，ethtool -c/-C）

中断频率太高会导致 CPU 一直被打断。合并中断可以提升 CPU 的处理效率，但是降低响应的及时性。
所以如果支持自适应中断响应 `ethtool -C enp68s0 adaptive-rx on` 就更好了。

```shell
ethtool -c enp68s0
Coalesce parameters for enp68s0:
Adaptive RX: n/a  TX: n/a
stats-block-usecs: n/a
sample-interval: n/a
pkt-rate-low: n/a
pkt-rate-high: n/a

rx-usecs: 3
rx-frames: n/a
rx-usecs-irq: n/a
rx-frames-irq: n/a

tx-usecs: 3
tx-frames: n/a
tx-usecs-irq: n/a
tx-frames-irq: n/a

rx-usecs-low: n/a
rx-frame-low: n/a
tx-usecs-low: n/a
tx-frame-low: n/a

rx-usecs-high: n/a
rx-frame-high: n/a
tx-usecs-high: n/a
tx-frame-high: n/a
```

## 查看和设置中断的亲和性

一个网卡队列对应一个中断号，一个中断号可以绑定到多个不同的 CPU 上。
irqbalance 进程可以自动调整中断的 CPU，但是有时候自动调整不是我们想要的。
因此需要手动设置中断的 CPU 亲和性。注意需要关闭 irqbalance 进程，命令为 `systemctl stop irqbalance`。

通过 /proc/interrupt 查看网卡的中断编号

```shell
$ cat /proc/interrupts | grep enp68
 105:          0          0          0          0          0   IR-PCI-MSI 35651584-edge      enp68s0
 106:          5      54446      23185      74710      45450   IR-PCI-MSI 35651585-edge      enp68s0-rx-0
 107:          5      35245      21530      76345      57965   IR-PCI-MSI 35651586-edge      enp68s0-rx-1
 108:         10      31310      14290      58920      68140   IR-PCI-MSI 35651587-edge      enp68s0-tx-0
 109:          0      42830      36705      74465      48460   IR-PCI-MSI 35651588-edge      enp68s0-tx-1
```

我们查看一下 106 号中断的配置

```shell
$ cat /proc/irq/106/smp_affinity
00000000,00000000,00020000,00000000
```

修改中断的 CPU 亲和性

```shell
echo 00000000,00000000,00000003,00000003 > /proc/irq/106/smp_affinity
```

## /proc/net/softnet_stat 各字段说明

如果 budget 或者 time limit 到了而仍有包需要处理，那 net_rx_action 在退出 循环之前会更新统计信息。这个信息存储在该 CPU 的 struct softnet_data 变量中。

这些统计信息打到了/proc/net/softnet_stat，但不幸的是，关于这个的文档很少。每一 列代表什么并没有标题，而且列的内容会随着内核版本可能发生变化，所以应该以内核源码为准， 下面是内核 5.10，可以看到每列分别对应什么：

// https://github.com/torvalds/linux/blob/v5.10/net/core/net-procfs.c#L172

static int softnet_seq_show(struct seq_file *seq, void *v)
{
    ...
    seq_printf(seq,
           "%08x %08x %08x %08x %08x %08x %08x %08x %08x %08x %08x %08x %08x\n",
           sd->processed, sd->dropped, sd->time_squeeze, 0,
           0, 0, 0, 0, /* was fastroute */
           0,    /* was cpu_collision */
           sd->received_rps, flow_limit_count,
           softnet_backlog_len(sd), (int)seq->index);
}

```shell
$ cat /proc/net/softnet_stat
6dcad223 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6f0e1565 00000000 00000002 00000000 00000000 00000000 00000000 00000000 00000000 00000000
660774ec 00000000 00000003 00000000 00000000 00000000 00000000 00000000 00000000 00000000
61c99331 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6794b1b3 00000000 00000005 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6488cb92 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

每一行代表一个 struct softnet_data 变量。因为每个 CPU 只有一个该变量，所以每行其实代表一个 CPU； 数字都是 16 进制表示。字段说明：

第一列 sd->processed：处理的网络帧数量。如果用了 ethernet bonding，那这个值会大于总帧数， 因为 bond 驱动有时会触发帧的重处理（re-processed）；
第二列 sd->dropped：因为处理不过来而 drop 的网络帧数量；具体见原理篇；
第三列 sd->time_squeeze：由于 budget 或 time limit 用完而退出 net_rx_action() 循环的次数；原理篇中有更多分析；
接下来的 5 列全是 0；
第九列 sd->cpu_collision：为发送包而获取锁时冲突的次数；
第十列 sd->received_rps：当前 CPU 被其他 CPU 唤醒去收包的次数；
最后一列，flow_limit_count：达到 flow limit 的次数；这是 RPS 特性。
5.2 调整 softirq 收包预算：sysctl netdev_budget/netdev_budget_usecs
权威解释见 内核文档。

netdev_budget：一个 CPU 单次轮询所允许的最大收包数量。 单次 poll 收包时，所有注册到这个 CPU 的 NAPI 变量收包数量之和不能大于这个阈值。
netdev_budget_usecs：每次 NAPI poll cycle 的最长允许时间，单位是 us。
触发二者中任何一个条件后，都会导致一次轮询结束。

查看当前配置：

```shell
$ sudo sysctl -a | grep netdev_budget
net.core.netdev_budget = 300         # kernel 5.10 默认值
net.core.netdev_budget_usecs = 2000  # kernel 5.10 默认值
```

修改配置：

```shell
$ sudo sysctl -w net.core.netdev_budget=3000
$ sudo sysctl -w net.core.netdev_budget_usecs = 10000
```

要保证重启不丢失，需要将这个配置写到 /etc/sysctl.conf。


本文主要自己实践 http://arthurchiao.art/blog/linux-net-stack-tuning-rx-zh/ 这个博文并加上自己的一些经验。
