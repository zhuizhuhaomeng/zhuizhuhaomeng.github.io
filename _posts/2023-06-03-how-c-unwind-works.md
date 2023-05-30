---
layout: post
title: "C/C++/Rust 的堆栈回溯是怎么实现的"
description: "C/C++/Rust 的堆栈回溯是怎么实现的"
date: 2023-06-03
tags: [unwind, eh_frame, debug_frame, eh_frame_header]
---

# 好文推荐

使用 -O2 编译的程序如果没有传递 -fno-omit-framepointer 这个编译参数，那么 rbp 寄存器就不会被用来作为调用栈回溯的栈帧寄存器。这个时候如果需要执行 unwind 应该如何处理呢？

这个文章通过一步步的手动解码告诉我们 C/C++ 的调用栈回溯是怎么处理的。
https://lesenechal.fr/en/linux/unwinding-the-stack-the-hard-way

# 测试程序

```C
#include <stdio.h>

int main() {
    printf("Hello, world!\n");

    return 0;
}
```

## 将测试程序编译成二进制

```
gcc -g -O2 test.c
# or
gcc -g test.c
```

## 将测试程序编译成汇编代码


```shell
gcc -S test.c
```

编译得到的汇编代码在 a.s中, 摘录部分结果如下:

下面的汇编代码中有各种 .cfi_ 开头的指令，想要了解这些指令的意义可以查阅 https://sourceware.org/binutils/docs/as/CFI-directives.html。

```asm
main:
.LFB7:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$32, %rsp
	movl	%edi, -20(%rbp)
	movq	%rsi, -32(%rbp)
	movl	$0, %edi
	call	sbrk
	movq	%rax, -8(%rbp)
	movq	stderr(%rip), %rax
	movq	-8(%rbp), %rdx
	movl	$.LC4, %esi
	movq	%rax, %rdi
	movl	$0, %eax
	call	fprintf
	call	print_file_map
	subq	$-128, -8(%rbp)
	movq	stderr(%rip), %rax
	movq	-8(%rbp), %rdx
	movl	$.LC5, %esi
	movq	%rax, %rdi
	movl	$0, %eax
	call	fprintf
	movq	-8(%rbp), %rax
	movq	%rax, %rdi
	call	brk
	movl	$0, %edi
	call	sbrk
	movq	%rax, -16(%rbp)
	movq	stderr(%rip), %rax
	movq	-16(%rbp), %rdx
	movl	$.LC4, %esi
	movq	%rax, %rdi
	movl	$0, %eax
	call	fprintf
	call	print_file_map
	movl	$0, %eax
	leave
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE7:
	.size	main, .-main
	.ident	"GCC: (GNU) 8.5.0 20210514 (Red Hat 8.5.0-16)"
	.section	.note.GNU-stack,"",@progbits
```

# 查看 dwarf 的信息

## 需要的工具

llvm-dwarfdump --eh-frame and readelf -wf can dump the section.

```shell
readelf -wf a.out
```

# 术语

DIE: Debugging Information Entry
CFA: Canonical Frame Address
CFI: Call Frame Information
CIE: Common Information Entry
FDE: Frame Description Entry

.eh_frame is composed of Common Information Entry (CIE) and Frame Description Entry (FDE). CIE has these fields:

 - length: The size of the length field plus the value of length must be an integral multiple of the address size.
 - CIE_id: Constant 0. This field is used to distinguish CIE and FDE. In FDE, this field is non-zero, representing CIE_pointer
 - version: Constant 1
 - augmentation: A NUL-terminated string describing the CIE/FDE parameter list.
   - z: augmentation_data_length and augmentation_data fields are present and provide arguments to interpret the remaining bytes
   - P: retrieve one byte (encoding) and a value (length decided by the encoding) from augmentation_data to indicate the personality routine pointer
   - L: retrieve one byte from augmentation_data to indicate the encoding of language-specific data area (LSDA) in FDEs. The augmentation data of a FDE stores LSDA
   - R: retrieve one byte from augmentation_data to indicate the encoding of initial_location and address_range in FDEs
   - S: an associated FDE describes a signal frame (used by unw_is_signal_frame)
 - code_alignment_factor: Assuming that the instruction length is a multiple of 2 or 4 (for RISC), it affects the multiplier of parameters such as DW_CFA_advance_loc
 - data_alignment_factor: The multiplier that affects parameters such as DW_CFA_offset DW_CFA_val_offset
 - return_address_register
 - augmentation_data_length: only present if augmentation contains z.
 - augmentation_data: only present if augmentation contains z. This field provides arguments describing augmentation. For P, the argument specifies the personality. For R, the argument specifies the encoding of FDE initial_location.
 - initial_instructions: bytecode for unwinding, a common prefix used by all FDEs using this CIE
 - padding

In .debug_frame version 4 or above, address_size (4 or 8) and segment_selector_size are present. .eh_frame does not have the two fields.

Each FDE has an associated CIE. FDE has these fields:

- length: The length of FDE itself. If it is 0xffffffff, the next 8 bytes (extended_length) record the actual length. Unless specially constructed, extended_length is not used
- CIE_pointer: Subtract CIE_pointer from the current position to get the associated CIE
- initial_location: The address of the first location described by the FDE. The value is encoded with a relocation referencing a section symbol
- address_range: initial_location and address_range describe an address range
- instructions: bytecode for unwinding, essentially (address,opcode) pairs
- augmentation_data_length
- augmentation_data: If the associated CIE augmentation contains L characters, language-specific data area will be recorded here
- padding

A CIE may optionally refer to a personality routine in the text section (.cfi_personality directive). A FDE may optionally refer to its associated LSDA in .gcc_except_table (.cfi_lsda directive). The personality routine and LSDA are used in Level 2: C++ ABI of Itanium C++ ABI.

```C
#include <libunwind.h>
#include <stdio.h>

void backtrace() {
  unw_context_t context;
  unw_cursor_t cursor;
  // Store register values into context.
  unw_getcontext(&context);
  // Locate the PT_GNU_EH_FRAME which contains PC.
  unw_init_local(&cursor, &context);
  size_t rip, rsp;
  do {
    unw_get_reg(&cursor, UNW_X86_64_RIP, &rip);
    unw_get_reg(&cursor, UNW_X86_64_RSP, &rsp);
    printf("rip: %zx rsp: %zx\n", rip, rsp);
  } while (unw_step(&cursor) > 0);
}

void bar() {backtrace();}
void foo() {bar();}
int main() {foo();}
```

```shell
$CC a.c -Ipath/to/include -Lpath/to/lib -lunwind
```

如果使用nongnu.org/libunwind，两种选择：(a) #include <libunwind.h>前添加#define UNW_LOCAL_ONLY (b) 多链接一个库，x86-64上是-l:libunwind-x86_64.so。 使用Clang的话也可用clang --rtlib=compiler-rt --unwindlib=libunwind -I path/to/include a.c，除了提供unw_*外，能确保不链接libgcc_s.so

unw_getcontext: 获取寄存器值(包含PC)
unw_init_local
使用dl_iterate_phdr遍历可执行文件和shared objects，找到包含PC的PT_LOAD program header
找到所在module的PT_GNU_EH_FRAME(.eh_frame_hdr)，存入cursor
unw_step
二分搜索PC对应的.eh_frame_hdr项，记录找到的FDE和其指向的CIE
执行CIE中的initial_instructions
执行FDE中的instructions。维护一个location、CFA，初始指向FDE的initial_location，指令中DW_CFA_advance_loc增加location；DW_CFA_def_cfa_*更新CFA；DW_CFA_offset表示一个寄存器的值保存在CFA+offset处
location大于等于PC时停止。也就是说，执行的指令是FDE instructions的一个前缀
Unwinder根据program counter找到适用的FDE，执行所有在program counter之前的CFI instructions。

有几种重要的

DW_CFA_def_cfa_*
DW_CFA_offset
DW_CFA_advance_loc
一个-DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86的clang，.text 51.7MiB、.eh_frame 4.2MiB、.eh_frame_hdr 646、2个CIE、82745个FDE

# 参考文章

## unwind 的参考源码

https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind.inc;h=12f62bca7335f3738fb723f00b1175493ef46345;hb=HEAD#l275
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1222
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1494
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1376

## 介绍 unwind 的文章

https://eli.thegreenplace.net/2015/programmatic-access-to-the-call-stack-in-c/
https://github.com/simpleton/stack-unwind-samples
https://maskray.me/blog/2020-11-08-stack-unwinding
https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html

https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc

