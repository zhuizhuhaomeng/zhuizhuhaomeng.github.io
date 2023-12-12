---
layout: post
title: "Add gnu BuildID for Go program"
description: "Add gnu BuildID for Go program"
date: 2023-12-12
tags: [Go, BuildID]
---

我们可以使用 `file` 命令查看 ELF 文件的 `BuildID`。比如：

```shell
$ file /usr/bin/ls                          
/usr/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=cedbe8d7fb5757dd39992c1524f8d362adafcf41, for GNU/Linux 3.2.0, stripped
```

但是我发现 Go 编译的程序默认不带 GNU BuildID，而是 Go BuildID。比如：

```shell
$ file chat-service 
chat-service: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=jxRHR3IMC5wjCJ3s_QCM/gqNKIARj6EYdPdS2nr1y/OmtMZzsjlNwxaAewNKv6/Qn2Ez4Yh8_cVLeQWtUcH, with debug_info, not stripped
```

这可不是什么好消息，因为这会造成其它依赖 BuildID 的工具因为缺少 BuildID 而不能很好的工作。比如

```shell
$ eu-unstrip -n --core core.353189  
0x400000+0x92c000 - . . /home/go-chat-service/chat-service
0x7ffff7fc5000+0x1000 9804e73e726b222229e63daf02212f1337c226b8@0x7ffff7fc5548 . /usr/lib/debug/usr/lib/modules/5.14.0-362.8.1.el9_3.x86_64/vdso/vdso64.so.debug linux-vdso.so.1
0x7ffff7fc7000+0x37218 46d8c77d92436bbcafcf16f6737ab9d6a16f3a38@0x7ffff7fc72d8 /lib64/ld-linux-x86-64.so.2 . ld-linux-x86-64.so.2
0x7ffff7c00000+0x208fb0 65a29d0b1cd88c15b44cfda562dd904e2cf35435@0x7ffff7c00390 /lib64/libc.so.6 /usr/lib/debug/usr/lib64/libc.so.6-2.34-83.el9.7.x86_64.debug libc.so.6
```

上面的 eu-unstrip 得到的第一行结果的第二列是 `-`，也就是缺少对应的值。那么分析coredump 的时候，就不知道这个core 对应哪个版本的而二进制程序了。

增加给 go 程序添加 BuildID 呢？只需要给 -ldflags 增加 -B 参数即可。可以参考这个 [例子](https://gitlab.com/gitlab-org/gitlab/-/merge_requests/78721/diffs#0b7c17c837be4e82948dcc0b07b3e07c95d46a2b_58_60)。


这里是我自己写的一个小例子：

```Makefile
BUILDID=$(shell openssl rand -hex 32 | sha1sum | cut -d' ' -f1)

all: compile

compile:
	gofmt -w main.go models routers services utils download
	go mod tidy
	go build -o chat-service -gcflags "all=-N -l" -ldflags "-B 0x$(BUILDID)"

clean:
	rm chat-service
```
