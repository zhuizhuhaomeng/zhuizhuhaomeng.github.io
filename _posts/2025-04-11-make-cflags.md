---
layout: post
title: "pass string macro via cflags in shell"
description: "在 shell 中使用 cflags 传递字符串的宏定义"
date: 2025-04-05
modified: 2025-04-05
tags: [shell, cflags, macro]
---

宏定义是 C 语言的一个特色，也是一个陷阱。
宏定义替换中的变量运算需要加上括号以防止优先级结合错误的问题想必大家都知道了。

今天遇到的是宏定义是字符串，需要在 shell 中传递该字符串给 Makefile 的问题。

比如存在如下的宏定义，我们想通过命令行传递宏定义来修改该宏定义的值，那么该如何传递呢？

# gcc 命令行传递宏定义

```C
#ifndef DEFAULT_BASE
#define DEFAULT_BASE "/usr/local/openresty"
#endif

const char *base = DEFAULT_BASE;
```

如果是 直接 gcc，那么我们尝试这样子传递：gcc -E -DDEFAULT_BASE="bcd" t.c

```shell
$ gcc -E -DDEFAULT_BASE="bcd" t.c
# 0 "t.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "t.c"




const char *base = bcd;
```

可以看到，替换的结果缺少了双引号。因此，我们需要让替换的字符串包括双引号。
上面的字符串子所以不想，是因为在 shell 中 abc"def" 其实和 abcdef 是一样的，我们可以用 echo 命令来验证这个问题。
shell 中双引号包围起来主要是为了处理空格的问题，否则空格的情况下不知道是要分成几个不同的参数。

```shell
gcc -E -DDEFAULT_BASE="\"bcd\"" t.c
# 0 "t.c"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "t.c"




const char *base = "bcd";
```

当然上面的命令还可以进一步的简化为使用单引号，因为单引号里面不会执行转义。

```shell
gcc -E -DDEFAULT_BASE='"bcd"' t.c
```

# make 命令行传递宏定义

make 传递参数不像 shell 直接调用 gcc 是 shell 直接传递参数给 gcc，而是 shell 传递给 make，make 处理后再传递给 gcc 参数。
因此参数首先是 shell 的参数，其次是 make 的参数，最后是 gcc 的参数。

shell 的参数要按照 shell 的规则执行转义，make 解释参数则也要执行转义，最终再传递给 GCC。

```shell
make V=1 CFLAGS='-DDEFAULT_BASE="\\\"/usr/local/dragonresty\\\"" -DTARGET_LEVEL=100'
```

比如上面这个命令，shell 将命令拆分成如下三个部分，第一个部分就是命令。

```
make
V=1
CFLAGS='-DDEFAULT_BASE="\\\"/usr/local/dragonresty\\\"" -DTARGET_LEVEL=100'
```

make 命令实际上接收到的参数是

```
V=1
CFLAGS=-DDEFAULT_BASE="\\\"/usr/local/dragonresty\\\"" -DTARGET_LEVEL=100
```

make 需要对第二个参数执行拆分，CXXFLAGS 这个如何拆分成多个不同的参数呢？这个其实也是跟 shell 的参数规则是一致的。也就是把这部分按照 shell 的规则来转义。

```
-DDEFAULT_BASE="\\\"/usr/local/dragonresty\\\"" -DTARGET_LEVEL=100
```
可以用 echo -- 来输出转义后的结果

```shell
echo -- -DDEFAULT_BASE="\\\"/usr/local/dragonresty\\\"" -DTARGET_LEVEL=100
```

因此 CFLAGS 会被拆分成这样的参数再传递给 gcc。注意这里传递的就是下面的字符串，没有再次经过转义

```
-DDEFAULT_BASE=\"/usr/local/dragonresty\"
-DTARGET_LEVEL=100
```
