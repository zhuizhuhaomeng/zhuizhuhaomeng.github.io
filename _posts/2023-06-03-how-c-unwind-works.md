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
## 查看 elf 文件中的 dwarf 的信息

可以使用命令 llvm-dwarfdump --eh-frame 或者 readelf -wf 查看相关的信息

```shell
$ readelf -wf a.out
readelf -wf a.out
Contents of the .eh_frame section:


00000000 0000000000000014 00000000 CIE
  Version:               1
  Augmentation:          "zR"
  Code alignment factor: 1
  Data alignment factor: -8
  Return address column: 16
  Augmentation data:     1b
  DW_CFA_def_cfa: r7 (rsp) ofs 8
  DW_CFA_offset: r16 (rip) at cfa-8
  DW_CFA_nop
  DW_CFA_nop

00000018 0000000000000010 0000001c FDE cie=00000000 pc=0000000000401040..0000000000401066
  DW_CFA_advance_loc: 4 to 0000000000401044
  DW_CFA_undefined: r16 (rip)

0000002c 0000000000000010 00000030 FDE cie=00000000 pc=0000000000401070..0000000000401075
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000040 0000000000000024 00000044 FDE cie=00000000 pc=0000000000401020..0000000000401040
  DW_CFA_def_cfa_offset: 16
  DW_CFA_advance_loc: 6 to 0000000000401026
  DW_CFA_def_cfa_offset: 24
  DW_CFA_advance_loc: 10 to 0000000000401030
  DW_CFA_def_cfa_expression (DW_OP_breg7 (rsp): 8; DW_OP_breg16 (rip): 0; DW_OP_lit15; DW_OP_and; DW_OP_lit11; DW_OP_ge; DW_OP_lit3; DW_OP_shl; DW_OP_plus)
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000068 000000000000001c 0000006c FDE cie=00000000 pc=0000000000401126..000000000040113b
  DW_CFA_advance_loc: 1 to 0000000000401127
  DW_CFA_def_cfa_offset: 16
  DW_CFA_offset: r6 (rbp) at cfa-16
  DW_CFA_advance_loc: 3 to 000000000040112a
  DW_CFA_def_cfa_register: r6 (rbp)
  DW_CFA_advance_loc: 16 to 000000000040113a
  DW_CFA_def_cfa: r7 (rsp) ofs 8
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000088 ZERO terminator
```

## 将测试程序编译成汇编代码

```shell
gcc -S test.c
```

编译得到的汇编代码在 a.s 中, 摘录部分结果如下:

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

# 调用接口执行调用栈回溯

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

# 参考文章

## unwind 的参考源码

https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind.inc;h=12f62bca7335f3738fb723f00b1175493ef46345;hb=HEAD#l275
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1222
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1494
https://gcc.gnu.org/git/gitweb.cgi?p=gcc.git;a=blob;f=libgcc/unwind-dw2.c;h=b262fd9f5b92e2d0ea4f0e65152927de0290fcbd;hb=HEAD#l1376
