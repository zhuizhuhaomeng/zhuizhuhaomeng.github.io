---
layout: post
title: "使用 GDB 跟踪 Lua 内存分配"
description: "使用 GDB 跟踪 Lua 内存分配"
date: 2024-02-05
tags: [gdb, memory, Lua]
---

我们想要了解一个某一个软件的内存是如何分配的，最好的方式就是看看内存分配的函数的调用栈。
我们可以使用 GDB 来跟踪内存的分配和释放。

# 设置日志输出文件

因为程序都会涉及大量的内存分配和释放，所以我们需要把调试信息导出到文件。
这样我们就可以使用其它工具对导出的日志进一步的分析。

这就涉及 logging 这个命令。

```gdb
set logging file gdb.log
set logging enabled on
```

因为我们不希望输出到一半就卡住，因此还需要加上禁止分页.

```gdb
set pagination off
```

# 监控内存分配

一般情况下，跟踪内存分配和释放，我们只要把函数探针打在特定的函数即可。

比如，把函数探针打在 `luaM_realloc_` 、 `luaM_malloc_` 和 `luaM_free_` 来观察 Lua 的内存分配和释放。
因为我们不能一直手动按回车，因此就需要 `command` 命令来配合。

```gdb
break luaM_realloc_
  commands
    bt
    print "--- luaM_realloc_ end"
    continue
  end

break luaM_malloc_
  commands
    bt
    print "--- luaM_malloc_ end"
    continue
  end

break luaM_free_
  commands
    bt
    print "--- luaM_free_ end"
    continue
  end
```

# 监控内存分配的结果

我们不仅仅要知道分配的调用栈，还希望知道分配的内存指针，那么这时候就需要在代码行上打上探针了。

很幸运的是，lua 的内存分配只有一个返回的地方, 我们只要在一个地方打上探针即可（忽略异常返回 NULL的情况）。

```gdb
break lmem.c:213
  commands
    bt
    info registers rax
    print "--- luaM_malloc_ ret"
    continue
  end

break lmem.c:188
  commands
    bt
    info registers rax
    print "--- luaM_realloc_ ret"
    continue
  end
```

## 探针打在 ret 指令上

有时候打在代码行上并不灵光，这时候需要把函数探针打在指令上，这时候就可以用 `dis` 来查看 `ret` 指令。

因为一个函数又多个 ret 指令，每一个都打上断点是没有问题的，但是这样子显然没有必要。
但是对于不熟悉汇编的人来说，想要判断哪个 `ret` 真是太难了。

因此，这里应该使用 `disasemble /s`, 因为这样子会把对应的代码行给出来，方便我们对照分析。

```gdb
(gdb) disassemble /s luaM_malloc_
Dump of assembler code for function luaM_malloc_:
lmem.c:
201	void *luaM_malloc_ (lua_State *L, size_t size, int tag) {
202	  if (size == 0)
   0x000000000040c999 <+0>:	test   %rsi,%rsi
   0x000000000040c99c <+3>:	je     0x40ca01 <luaM_malloc_+104>

201	void *luaM_malloc_ (lua_State *L, size_t size, int tag) {
   0x000000000040c99e <+5>:	push   %r13
   0x000000000040c9a0 <+7>:	push   %r12
   0x000000000040c9a2 <+9>:	push   %rbp
   0x000000000040c9a3 <+10>:	push   %rbx
   0x000000000040c9a4 <+11>:	sub    $0x8,%rsp
   0x000000000040c9a8 <+15>:	mov    %rdi,%r12
   0x000000000040c9ab <+18>:	mov    %rsi,%rbx

204	  else {
205	    global_State *g = G(L);
   0x000000000040c9ae <+21>:	mov    0x18(%rdi),%r13

206	    void *newblock = firsttry(g, NULL, tag, size);
   0x000000000040c9b2 <+25>:	movslq %edx,%rbp
   0x000000000040c9b5 <+28>:	mov    0x8(%r13),%rdi
   0x000000000040c9b9 <+32>:	mov    %rsi,%rcx
   0x000000000040c9bc <+35>:	mov    %rbp,%rdx
   0x000000000040c9bf <+38>:	mov    $0x0,%esi
   0x000000000040c9c4 <+43>:	call   *0x0(%r13)

207	    if (l_unlikely(newblock == NULL)) {
   0x000000000040c9c8 <+47>:	test   %rax,%rax
   0x000000000040c9cb <+50>:	je     0x40c9dc <luaM_malloc_+67>

211	    }
212	    g->GCdebt += size;
   0x000000000040c9cd <+52>:	add    %rbx,0x18(%r13)

213	    return newblock;
   0x000000000040c9d1 <+56>:	add    $0x8,%rsp
   0x000000000040c9d5 <+60>:	pop    %rbx

214	  }
215	}
   0x000000000040c9d6 <+61>:	pop    %rbp
   0x000000000040c9d7 <+62>:	pop    %r12
   0x000000000040c9d9 <+64>:	pop    %r13
   0x000000000040c9db <+66>:	ret    

208	      newblock = tryagain(L, NULL, tag, size);
   0x000000000040c9dc <+67>:	mov    %rbx,%rcx
   0x000000000040c9df <+70>:	mov    %rbp,%rdx
   0x000000000040c9e2 <+73>:	mov    $0x0,%esi
   0x000000000040c9e7 <+78>:	mov    %r12,%rdi
   0x000000000040c9ea <+81>:	call   0x40c7f4 <tryagain>

209	      if (newblock == NULL)
   0x000000000040c9ef <+86>:	test   %rax,%rax
   0x000000000040c9f2 <+89>:	jne    0x40c9cd <luaM_malloc_+52>

210	        luaM_error(L);
   0x000000000040c9f4 <+91>:	mov    $0x4,%esi
   0x000000000040c9f9 <+96>:	mov    %r12,%rdi
   0x000000000040c9fc <+99>:	call   0x4087b5 <luaD_throw>

203	    return NULL;  /* that's all */
   0x000000000040ca01 <+104>:	mov    $0x0,%eax

214	  }
215	}
   0x000000000040ca06 <+109>:	ret    
End of assembler dump.
```

这里我们选择 `0x000000000040c9db` 这个地址的 `ret` 指令。

我们也可以选择 `0x000000000040c9d1`, 因为从 `0x000000000040c9d1` 开始到 ret 指令属于汇编的 epilogue。
因此我们也可以这么设置断点。

```gdb
break *0x000000000040c9d1
    command
      bt
      info registers rax
      print "--- luaM_malloc_ ret"
      continue
    end
```
# 保存断点，方便重复执行

```gdb
save breakpoints trace-mem.gdb
```

# 总结

以代码行设置断点为例，我们保存上面的指令为如下的文件，假设文件名为 trace-mem.gdb。

```gdb
set logging file gdb.log
set logging enabled on
set pagination off

break lmem.c:188
  commands
    info registers rax
    print "--- luaM_realloc_ ret"
    continue
  end

break lmem.c:213
  commands
    info registers rax
    print "--- luaM_malloc_ ret"
    continue
  end

break luaM_free_
  commands
    bt
    print "--- luaM_free_ end"
    continue
  end
```

那么如何利用这个脚本呢？这个一个简单的例子。

```shell
$ gdb lua
source trace-mem.gdb
run test.lua
```

而且我们也可以在编辑器里面对 trace-mem.lua 进行编辑，而不需要在gdb 里面一个字母一个字母慢慢的敲了。
