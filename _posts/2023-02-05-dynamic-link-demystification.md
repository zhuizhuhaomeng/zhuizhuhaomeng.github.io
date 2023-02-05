---
layout: post
title: "使用 GDB 验证动态链接的过程"
description: "使用 GDB 验证 @PLT 的过程，非常的有趣，对动态链接的理解就豁然开朗了"
date: 2023-02-05
tags: [gdb, link]
---

# 背景

对于动态链接的详细工作过程一直很好奇，终于搞明白了其中的缘由，记录一下。

以前一直以为自己明白了，其实不明白。动手验证，细节就是魔鬼。

本次验证使用了 gdb，过程非常有趣，你也可以自己动手试试看。

# 测试代码

这是一个最小化的代码，调用的printf函数是libc提供的，因此就涉及到动态链接的过程了。

```C
#include<stdio.h>

int main()
{
    printf("hello world!\n");
    return 0;
}
```

将源代码编译成可执行文件

```shell
gcc -g t.c
```

# 使用 gdb 分析动态链接过程

```shell
$ /usr/bin/gdb ./a.out
(gdb) start
(gdb) list
1	#include<stdio.h>
2
3	int main()
4	{
5	    printf("hello world!\n");
6	    return 0;
7	}
(gdb) disassemble
Dump of assembler code for function main:
   0x0000000000401126 <+0>:	push   %rbp
   0x0000000000401127 <+1>:	mov    %rsp,%rbp
=> 0x000000000040112a <+4>:	mov    $0x402010,%edi
   0x000000000040112f <+9>:	callq  0x401030 <puts@plt>
   0x0000000000401134 <+14>:	mov    $0x0,%eax
   0x0000000000401139 <+19>:	pop    %rbp
   0x000000000040113a <+20>:	retq
End of assembler dump.
(gdb)
```

我们看到,反汇编代码重并没有 printf 的调用, 取而代之的是puts函数的调用。
这是因为 printf 的调用只有常量字符串没有参数，因此被 gcc 优化成 puts的调用。

=> 0x000000000040112a <+4>:	mov    $0x402010,%edi 这条指令其实就是将 "hello world!\n"
的地址放在%edi。我们可以通过 x 这个 gdb 命令来确认。

```gdb
(gdb) x /s 0x402010
0x402010:	"hello world!"
```

通过该汇编语句 `0x000000000040112f <+9>:	callq  0x401030 <puts@plt>` 我们知道
调用了 0x401030 这个位置的 plt 过程链接的函数。我们查看一下该地址的反汇编代码。

```gdb
(gdb) disassemble 0x401030
Dump of assembler code for function puts@plt:
   0x0000000000401030 <+0>:	jmpq   *0x2fca(%rip)        # 0x404000 <puts@got.plt>
   0x0000000000401036 <+6>:	pushq  $0x0
   0x000000000040103b <+11>:	jmpq   0x401020
End of assembler dump.
```

可以看到这个反汇编很短，只有3个指令。第一个指令的 jmpq 到另一个地址 0x404000 存储的指令地址去执行。
`0x0000000000401030 <+0>:	jmpq   *0x2fca(%rip)` 这个指令的 * 对理解 plt 的原理很重要。这里并不是直接
跳转到 `0x2fca(%rip)` 这个地址，而是跳转到 `0x2fca(%rip)` 这个地址存储的指令地址去执行，这里相当于是C语言的双重指针。
第二条指令是将一个常数参数压栈，第三个指令跳转到另一个地址 0x401020 去执行。

0x404000 这个地址是如何得到的呢？这个是通过 0x2fca(%rip)计算得到的，而 %rip 的值是下一条指令的地址,
也就是 0x401036。所以 0x2fca + 0x401036 = 0x404000。我们来看看 0x404000 这个地址存储的是什么值。
因为 0x404000 存储的是一个地址，因此我们使用 `x /a` 的方式来查看该地址存储的值。

```gdb
(gdb) x /a 0x404000
0x404000 <puts@got.plt>:	0x401036 <puts@plt+6>
```

从上面的输出可以看到 0x404000 存储的指令地址是 `0x401036 <puts@plt+6>`, 这个是上面的 plt 函数的下一条要执行的指令。
也就是说 `jmpq   *0x2fca(%rip)` 这个指令是跳转到另一个地址上存储的指令地址去执行，而这个存储的地址就是下一条指令。
如果不是跳转指令，那么上一条指令执行结束就是执行下一条指令。这里为什么要多此一举，通过 `jmp` 指令跳转到下一条指令呢?
这就是 plt 动态链接的关键之处了。 `0x2fca(%rip)` 原来存储的是 plt 的下一条指令，第一次执行会解析 puts 的函数地址，
将解析得到的函数地址存储在 `0x2fca(%rip)` 这个位置，下一次执行的时候就直接跳转到 puts函数了。

``` shell
(gdb) disassemble 0x401030
Dump of assembler code for function puts@plt:
   0x0000000000401030 <+0>:	jmpq   *0x2fca(%rip)        # 0x404000 <puts@got.plt>
   0x0000000000401036 <+6>:	pushq  $0x0
   0x000000000040103b <+11>:	jmpq   0x401020
End of assembler dump.
```

我们再回顾一下上面的 plt 函数。可以看到下一条指令把 常数 0 压栈，然后跳转到 0x401020 去执行。我们看看 0x401020 这个地方的指令。
因为查看的是指令，因此使用 `x /i` 这样的gdb命令，下面的6表示打印6条指令。

```gdb
(gdb) x /6i 0x401020
   0x401020:	pushq  0x2fca(%rip)        # 0x403ff0
   0x401026:	jmpq   *0x2fcc(%rip)        # 0x403ff8
   0x40102c:	nopl   0x0(%rax)
   0x401030 <puts@plt>:	jmpq   *0x2fca(%rip)        # 0x404000 <puts@got.plt>
   0x401036 <puts@plt+6>:	pushq  $0x0
   0x40103b <puts@plt+11>:	jmpq   0x401020
(gdb)
```

通过上面的指令，我们看到 把 `0x2fca(%rip)        # 0x403ff0` 这个值压栈了，然后跳转到
`*0x2fcc(%rip)        # 0x403ff8` 所存储的指令去执行了。 压栈的这两个参数分别是代表什么呢？
第一个代表的是 puts 这个动态链接函数的索引，第二个代表 link_map 的结构。具体的可以参考 https://ypl.coffee/dl-resolve/ 这篇文章。

我们接下来看看 0x403ff8 这个位置存储的是指令地址是什么。

```shell
(gdb) x /a 0x403ff8
0x403ff8:	0x7ffff7dd0b20 <_dl_runtime_resolve_xsave>
```

可以看到 0x403ff8 地址存储的是 _dl_runtime_resolve_xsave , 用来解析 puts 的函数。关于该函数的原理可以参考 https://ypl.coffee/dl-resolve/ 。
我们接下来用 gdb 来分析一下 _dl_runtime_resolve_xsave 是如何获取 puts 这个字符串的。因为上面的参数一个是 0， 一个是 link_map，并没有 puts这个字符串参数。

```shell
[ljl@rocky8 openresty-develop]$ readelf -r ./a.out

Relocation section '.rela.dyn' at offset 0x458 contains 4 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000403fc8  000100000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000403fd0  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000403fd8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000403fe0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0

Relocation section '.rela.plt' at offset 0x4b8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000404000  000200000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
```

我们使用 `readelf -r ./a.out` 这个命令查询所有的 plt 函数，可以看到就只有一个 puts。
puts排在第一个，以 0 作为起始值的索引计算，puts 的索引值为 0。


我们可以通过 readelf 来查看各个 section 的加载地址。因为这里是 exe 并且没有编译成共享类型的，因此加载的地址是不变的。
比如 我们可以看到 .rela.plt 这个段的加载地址是 0x4004b8。这些段的信息也存储在 link_map 中。

```shell
[ljl@rocky8 openresty-develop]$ readelf -S --wide ./a.out
There are 35 section headers, starting at offset 0x63c0:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        00000000004002a8 0002a8 00001c 00   A  0   0  1
...
  [ 5] .dynsym           DYNSYM          0000000000400328 000328 000090 18   A  6   1  8
  [ 6] .dynstr           STRTAB          00000000004003b8 0003b8 000073 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          000000000040042c 00042c 00000c 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         0000000000400438 000438 000020 00   A  6   1  8
  [ 9] .rela.dyn         RELA            0000000000400458 000458 000060 18   A  5   0  8
  [10] .rela.plt         RELA            00000000004004b8 0004b8 000018 18  AI  5  22  8
  [11] .init             PROGBITS        0000000000401000 001000 00001b 00  AX  0   0  4
  [12] .plt              PROGBITS        0000000000401020 001020 000020 10  AX  0   0 16
  [13] .text             PROGBITS        0000000000401040 001040 000175 00  AX  0   0 16
  [14] .fini             PROGBITS        00000000004011b8 0011b8 00000d 00  AX  0   0  4
  [15] .rodata           PROGBITS        0000000000402000 002000 00001d 00   A  0   0  8
...
  [20] .dynamic          DYNAMIC         0000000000403df8 002df8 0001d0 10  WA  6   0  8
  [21] .got              PROGBITS        0000000000403fc8 002fc8 000020 08  WA  0   0  8
  [22] .got.plt          PROGBITS        0000000000403fe8 002fe8 000020 08  WA  0   0  8
  [23] .data             PROGBITS        0000000000404008 003008 000004 00  WA  0   0  1
  [24] .bss              NOBITS          000000000040400c 00300c 000004 00  WA  0   0  1
...
  [32] .symtab           SYMTAB          0000000000000000 005548 000720 18     33  56  8
  [33] .strtab           STRTAB          0000000000000000 005c68 0005fc 00      0   0  1
  [34] .shstrtab         STRTAB          0000000000000000 006264 000159 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

## 使用 readelf 得到的加载地址来分析获取 puts 符号的过程

使用下面两个宏得到 symbol的索引和符号的类型。

gdb 查看 符号。因为 .rela.plt 这个段的加载地址是 0x4004b8，而 puts 这个函数的 plt 索引是 0。
因此我们使用下面命令得到 put 函数的 Elf64_Rela 结构体。然后我们使用下面这两个宏对 r_info 字段
做运算，得到 puts 在 .dynsym 的索引是 2。 然后我们打印 Elf64_Sym 结构体，通过st_name字段知道
puts 函数在字符串 .dynstr 段的偏移量是 1。

```C
#define ELF64_R_SYM(i)                  ((i) >> 32)
#define ELF64_R_TYPE(i)                 ((i) & 0xffffffff)
```

```gdb
(gdb) print *(Elf64_Rela *) 0x4004b8
$5 = {
  r_offset = 4210688,
  r_info = 8589934599, # = 0x2,00000007
  r_addend = 0
}

(gdb) print ((Elf64_Sym *)0x400328)[2]
$7 = {
  st_name = 1,
  st_info = 18 '\022',
  st_other = 0 '\000',
  st_shndx = 0,
  st_value = 0,
  st_size = 0
}

(gdb) x /s (0x4003b8 + 1)
0x4003b9:	"puts"
```

# 通过 link_map 来解析 puts 这个符号

上面说过在调用 _dl_runtime_resolve_xsave 前向堆栈压入两个参数。第一个参数是0，第二个参数是 0x403ff0。
我们通过 `info symbol 0x403ff0` 可以知道该地址是 got.plt 的第二个表项，这个表项存储的是 `struct link_map *` 的指针。
因此我们可以通过 `x /a 0x403ff0` 获取 `struct link_map *` 的指针值。

```shell
(gdb) info symbol 0x403ff0
_GLOBAL_OFFSET_TABLE_ + 8 in section .got.plt of /home/ljl/a.out
(gdb) x /a 0x403ff0
0x403ff0:	0x7ffff7ffe1f0
`

解下来我们通过 link_map 来获取 puts 这个符号。

```shell
(gdb) set print pretty on
(gdb) set pagination off
(gdb) print *(struct link_map *)0x7ffff7ffe1f0
$8 = {
  l_addr = 0,
  l_name = 0x7ffff7ffe790 "",
  l_ld = 0x403df8,
  l_next = 0x7ffff7ffe7a0,
  l_prev = 0x0,
  l_real = 0x7ffff7ffe1f0,
  l_ns = 0,
  l_libname = 0x7ffff7ffe778,
  l_info = {0x0, 0x403df8, 0x403ed8, 0x403ec8, 0x0, 0x403e78, 0x403e88, 0x403f08, 0x403f18, 0x403f28, 0x403e98, 0x403ea8, 0x403e08, 0x403e18, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x403ee8, 0x403eb8, 0x0, 0x403ef8, 0x0, 0x403e28, 0x403e48, 0x403e38, 0x403e58, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x403f48, 0x403f38, 0x0 <repeats 13 times>, 0x403f58, 0x0 <repeats 25 times>, 0x403e68},
  l_phdr = 0x400040,
  l_entry = 4198496,
...
  l_audit = 0x7ffff7ffe670
}

```

```C
...
#define DT_STRTAB        5                /* Address of string table */
#define DT_SYMTAB        6                /* Address of symbol table */
...
#define DT_JMPREL        23               /* Address of PLT relocs */
```

获取 plt 段的值, 这里使用到 23 这个值，定义可以参考上面。
因为 puts 在 plt 段的索引值是 0，下面用到的 0 索引就是这个原因。

```shell
(gdb) print *((struct link_map *)0x7ffff7ffe1f0)->l_info[23]
$13 = {
  d_tag = 23,
  d_un = {
    d_val = 4195576, # 0x4004F8
    d_ptr = 4195576
  }
}

(gdb) print ((Elf64_Rela *)4195576)[0]
$14 = {
  r_offset = 4210688,
  r_info = 8589934599,
  r_addend = 0
}

// #define ELF64_R_SYM(i)                  ((i) >> 32) 高 32 bit得到在 dynsym表中的索引值

(gdb) print ((Elf64_Rela *)4195576)[0].r_info >> 32
$15 = 2

// 获取 dynsym 段的值 DT_SYMTAB 6

(gdb) print *((struct link_map *)0x7ffff7ffe1f0)->l_info[6]
$16 = {
  d_tag = 6,
  d_un = {
    d_val = 4195112,
    d_ptr = 4195112
  }
}

// 获取索引值为 2 的表项

(gdb) print ((Elf64_Sym*)4195112)[2]
$17 = {
  st_name = 6,
  st_info = 18 '\022',
  st_other = 0 '\000',
  st_shndx = 0,
  st_value = 0,
  st_size = 0
}

// 获取 dynstr段  DT_STRTAB 5

(gdb) print *((struct link_map *)0x7ffff7ffe1f0)->l_info[5]
$20 = {
  d_tag = 5,
  d_un = {
    d_val = 4195304,
    d_ptr = 4195304
  }
}

(gdb) x /s (4195304 + 6)
0x4003ee:	"puts"

```

上面的 gdb 命令我们一步步执行，通过 plt 的索引值 和 got.plt 的link_map地址得到了 `puts` 的值。

# 总结

上面我们通过两种方法来获取 puts 这个字符串的来源。
一个是 gdb 结合 readelf 的方法， 一个是 gdb 结合 link_map 字段的方法。
两种方法我们最终都得到代理 puts这个字符串。

