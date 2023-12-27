---
layout: post
title: "How to debug linux crash"
description: "How to debug linux crash"
date: 2023-11-28
tags: [crash, linux]
---

# 总结

Linux 内核崩溃应该怎么调试呢？这篇文件挺好的，包含了分析的实例，https://wiki.whamcloud.com/display/LNet/Crash+course+on+Crash。

总结下来，关键的步骤就是：

1. 要先配置 kdump 才能保存崩溃的现场。
2. 需要安装 kernel-debuginfo
3. 需要安装 crash 程序（当然也要 gdb）
4. 使用 crash 分析崩溃的内核
5. 我们需要具备汇编语言的知识

关于怎么安装 debuginfo 包可以参考 [如何安装 debuginfo](./2023-04-25-how-to-install-debuginfo.md)。

# 案例

下面是使用 crash 命令的一个例子

```shell
$ cd /var/crash/127.0.0.1-2023-11-28-15:44:54
$ ls
kexec-dmesg.log  vmcore  vmcore-dmesg.txt
$ crash /usr/lib/debug/lib/modules/5.14.0-284.18.1.el9_2.x86_64/vmlinux vmcore
# Display stack trace for crashed task
(crash) bt
  
# gives you stack trace for all the CPUs
(crash) bt -a
  
# gives you task list in condensed form
(crash) ps
  
# give you more info on each call, including stack addresses.
# 记住堆栈是从高地址到低地址, 压栈的时候第一个压入的在高地址，后面压入的在低地址
(crash) bt -f
  
# print back trace with line numbers
# -l might not always work if the wrong debuginfo rpms or the wrong debug symbols are loaded
(crash) bt -l
  
# print stack traces for all tasks
(crash) foreach bt | less
  
# print the stack trace for wanted task
(crash) bt [<PID> | <task pointer>]

# to examine type definitions
(crash) whatis <type name>

# disassemble function
(crash) dis -l <function name>
```

可以看到在 crash 的目录下存在 vmcore-dmesg.txt 的文件。该文件的内容如下：

```text
[  188.458550] RIP: 0010:xray_unwind+0x136/0x39e [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.460972] Code: 83 b8 90 00 00 00 00 0f 84 f1 01 00 00 48 c7 c2 c0 83 b5 c0 be 68 06 00 00 48 c7 c7 20 d9 ae c0 e8 1f ac ff ff bd 00 00 00 00 <4c> 8b 4d 28 4c 8b 45 20 48 8b 8d 98 00 00 00 ff 75 18 48 c7 c2 02
[  188.466421] RSP: 0000:ffffa7ac419ebca0 EFLAGS: 00010246
[  188.467962] RAX: 0000000000000000 RBX: ffffa7ac42ec0178 RCX: 0000000000000000
[  188.470042] RDX: ffff8b287fd00000 RSI: ffffa7ac40d13053 RDI: ffffa7ac419ebbe0
[  188.472138] RBP: 0000000000000000 R08: ffffa7ac40d13030 R09: 0000000000000003
[  188.474228] R10: ffffffffffffffff R11: ffffa7ac40d13030 R12: ffffffffc0b8f360
[  188.476311] R13: 0000000000401118 R14: ffffffffc0b44e41 R15: 0000000000000001
[  188.478402] FS:  00007ffff7fa9740(0000) GS:ffff8b287fd00000(0000) knlGS:0000000000000000
[  188.480753] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[  188.482454] CR2: 0000000000000028 CR3: 00000001a2a20005 CR4: 0000000000370ee0
[  188.484540] DR0: 0000000000000000 DR1: 0000000000000000 DR2: 0000000000000000
[  188.486633] DR3: 0000000000000000 DR6: 00000000fffe0ff0 DR7: 0000000000000400
[  188.488715] Call Trace:
[  188.489469]  <TASK>
[  188.490144]  _stp_stack_unwind_one_user_nd+0x77/0x237 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.492644]  _stp_stack_user_get_nd+0x11f/0x1b7 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.494944]  _stp_stack_user_print_nd+0x83/0xe0 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.497232]  function___global_print_ubacktrace_xray_nd__overload_0+0x3a/0x95 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.500201]  function___global_func_handle__overload_0+0x238/0x27f [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.502927]  probe_6376+0xef/0x1c6 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.504918]  stapiu_probe_handler+0x185/0x35c [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.507262]  stapiu_probe_prehandler+0x16c/0x170 [stap_d004c0a720e496417bd755fb0ca654a_33669]
[  188.509572]  handler_chain+0x56/0x210
[  188.510610]  uprobe_notify_resume+0x197/0x370
[  188.511731]  ? arch_uprobe_exception_notify+0x41/0x50
[  188.513015]  exit_to_user_mode_loop+0x7a/0x130
[  188.514166]  exit_to_user_mode_prepare+0xb6/0x100
[  188.515309]  ? exc_int3+0x2c/0xc0
[  188.516120]  irqentry_exit_to_user_mode+0x5/0x30
[  188.517224]  asm_exc_int3+0x35/0x40
[  188.518072] RIP: 0033:0x401118
[  188.518828] Code: c3 90 c3 66 66 2e 0f 1f 84 00 00 00 00 00 0f 1f 40 00 f3 0f 1e fa eb 8a 55 48 89 e5 89 7d fc 83 45 fc 01 8b 45 fc 83 c0 01 5d <cc> 55 48 89 e5 48 83 ec 08 89 7d fc 83 45 fc 01 8b 45 fc 83 c0 01
[  188.522893] RSP: 002b:00007fffffffd040 EFLAGS: 00000206
[  188.523995] RAX: 0000000000000005 RBX: 0000000000000000 RCX: 0000000000403e38
[  188.525482] RDX: 00007fffffffd188 RSI: 00007fffffffd178 RDI: 0000000000000003
[  188.526978] RBP: 00007fffffffd050 R08: 00007ffff7dfbf10 R09: 00007ffff7fd0d50
[  188.528466] R10: 00007fffffffcdd0 R11: 0000000000000206 R12: 00007fffffffd178
[  188.531068] R13: 0000000000401137 R14: 0000000000403e38 R15: 00007ffff7ffd000
[  188.533643]  </TASK>
[  188.535230] Modules linked in: stap_d004c0a720e496417bd755fb0ca654a_33669(OE) snd_seq_dummy snd_hrtimer veth xt_nat nf_conntrack_netlink xt_addrtype xt_CHECKSUM br_netfilter xt_MASQUERADE xt_conntrack ipt_REJECT nf_reject_ipv4 nft_compat nft_chain_nat nf_nat nf_conntrack nf_defrag_ipv6 nf_defrag_ipv4 nft_counter nf_tables nfnetlink bridge stp llc qrtr rfkill overlay sunrpc intel_rapl_msr intel_rapl_common snd_hda_codec_generic ledtrig_audio snd_hda_intel kvm_intel snd_intel_dspcfg snd_intel_sdw_acpi snd_hda_codec kvm snd_hda_core snd_hwdep snd_seq irqbypass rapl snd_seq_device iTCO_wdt snd_pcm iTCO_vendor_support snd_timer lpc_ich snd i2c_i801 virtio_balloon soundcore i2c_smbus pcspkr joydev tcp_bbr xfs libcrc32c virtio_gpu virtio_dma_buf drm_shmem_helper drm_kms_helper ahci libahci syscopyarea sysfillrect sysimgblt fb_sys_fops libata drm crct10dif_pclmul crc32_pclmul crc32c_intel virtio_net virtio_blk ghash_clmulni_intel virtio_console net_failover failover serio_raw dm_mirror
[  188.535323]  dm_region_hash dm_log dm_mod fuse [last unloaded: orxray_c_on_cpu_X_29824]
[  188.561258] CR2: 0000000000000028

```

我们重点关注第一行的信息：
```text
RIP: 0010:xray_unwind+0x136/0x39e 
```

# 如何加载 kernel module

```crash
# 加载默认路径下的kernel module
(crash) mod -S

# 如果不在默认路径下，那么可以使用
(crash) mod -s <name> /path/to/name.ko
```

# 查看 ko 中函数指定偏移对应的代码行

可以使用下面的例子

```shell
$ gdb xxx.ko
(gdb) l *lnet_cpt_of_md+223
```

# 其它一些文章

1. 另一个案例 https://walac.github.io/kernel-crashes/
2. 哪里结合技术要点说明 https://www.dedoimedo.com/computers/crash-analyze.html
3. https://www.suse.com/ja-jp/support/kb/doc/?id=000016171
