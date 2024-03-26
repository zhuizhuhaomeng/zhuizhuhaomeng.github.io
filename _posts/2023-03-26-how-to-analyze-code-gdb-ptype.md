---
layout: post
title: "如何阅读代码 -- LuaJIT GC 对象代码分析"
description: "如何阅读代码序列 -- LuaJIT GC 对象代码分析"
date: 2023-03-26
tags: [GCC, gdb]
---

# 应该配置哪些参数

在使用 GDB 手动分析问题的时候，我不希望 GDB 的主动分页功能干扰，
也不希望密密麻麻的信息揉在一起影响分析过程。因此，我一般会执行下面的命令来禁用分页并开启
格式化打印的功能。

```gdb
set pagination off
set print pretty on
```

# LuaJIT GC 对象

LuaJIT 的 GC 对象的定义如下所示。我们可以看到，Lua 中包含了 字符串，upvalue，函数，
协程， cdata，userdata，table，以及 proto 这 8 种GC 对象。

```C
typedef union GCobj {
  GChead gch;
  GCstr str;
  GCupval uv;
  lua_State th;
  GCproto pt;
  GCfunc fn;
  GCcdata cd;
  GCtab tab;
  GCudata ud;
} GCobj;

/* Macros to convert a GCobj pointer into a specific value. */
#define gco2str(o)	check_exp((o)->gch.gct == ~LJ_TSTR, &(o)->str)
#define gco2uv(o)	check_exp((o)->gch.gct == ~LJ_TUPVAL, &(o)->uv)
#define gco2th(o)	check_exp((o)->gch.gct == ~LJ_TTHREAD, &(o)->th)
#define gco2pt(o)	check_exp((o)->gch.gct == ~LJ_TPROTO, &(o)->pt)
#define gco2func(o)	check_exp((o)->gch.gct == ~LJ_TFUNC, &(o)->fn)
#define gco2cd(o)	check_exp((o)->gch.gct == ~LJ_TCDATA, &(o)->cd)
#define gco2tab(o)	check_exp((o)->gch.gct == ~LJ_TTAB, &(o)->tab)
#define gco2ud(o)	check_exp((o)->gch.gct == ~LJ_TUDATA, &(o)->ud)


#ifdef LUA_USE_ASSERT
#define lj_assertX(c, ...)	lj_assert_check(NULL, (c), __VA_ARGS__)
#define check_exp(c, e)		(lj_assertX((c), #c), (e))
#else
#define lj_assertX(c, ...)	((void)0)
#define check_exp(c, e)		(e)
#endif

#define gcref(r)	((GCobj *)(r).gcptr64)
#define gcV(o)		check_exp(tvisgcv(o), gcval(o))
#define strV(o)		check_exp(tvisstr(o), &gcval(o)->str)


/* String object header. String payload follows. */
typedef struct GCstr {
  GCHeader;
  uint8_t reserved;	/* Used by lexer for fast lookup of reserved words. */
  uint8_t hashalg;	/* Hash algorithm. */
  StrID sid;		/* Interned string ID. */
  StrHash hash;		/* Hash of string. */
  MSize len;		/* Size of string. */
} GCstr;

#define strref(r)	(&gcref((r))->str)
#define strdata(s)	((const char *)((s)+1))
#define strdatawr(s)	((char *)((s)+1))
#define strVdata(o)	strdata(strV(o))

```

在上面的代码中，有一个宏 `strVdata(o)`, 这个宏的意思是什么呢？它和宏 `strdata(s)` 又是什么关系呢？

可以看到，这些宏的参数名字是不同的，宏参数名字 `o` 应该是 `object` 的缩写，代表 `GCobj`,
参数 `s` 代表 `GCStr` 对象。`strref(r)` 这个就是把一个指针转换为 `GCStr` 类型的指针。

知道上面这些套路以后，我们就知道其它的 GC 对象的宏定义是怎么回事了。


```C
/* C data object. Payload follows. */
typedef struct GCcdata {
  GCHeader;
  uint16_t ctypeid;	/* C type ID. */
} GCcdata;

/* Prepended to variable-sized or realigned C data objects. */
typedef struct GCcdataVar {
  uint16_t offset;	/* Offset to allocated memory (relative to GCcdata). */
  uint16_t extra;	/* Extra space allocated (incl. GCcdata + GCcdatav). */
  MSize len;		/* Size of payload. */
} GCcdataVar;

#define cdataptr(cd)	((void *)((cd)+1))
#define cdataisv(cd)	((cd)->marked & 0x80)
#define cdatav(cd)	((GCcdataVar *)((char *)(cd) - sizeof(GCcdataVar)))
#define cdatavlen(cd)	check_exp(cdataisv(cd), cdatav(cd)->len)
#define sizecdatav(cd)	(cdatavlen(cd) + cdatav(cd)->extra)
#define memcdatav(cd)	((void *)((char *)(cd) - cdatav(cd)->offset))


typedef struct GCfuncC {
  GCfuncHeader;
  lua_CFunction f;	/* C function to be called. */
  TValue upvalue[1];	/* Array of upvalues (TValue). */
} GCfuncC;

typedef struct GCfuncL {
  GCfuncHeader;
  GCRef uvptr[1];	/* Array of _pointers_ to upvalue objects (GCupval). */
} GCfuncL;

typedef union GCfunc {
  GCfuncC c;
  GCfuncL l;
} GCfunc;
```

# 每次的堆栈都不一样怎么办

地址空间布局随机化（ASLR）是一种在大多数现代操作系统中实施的漏洞缓解技术。
Linux 上默认是开启该功能的，因此我们每次执行一个程序，他的栈空间，so 加载的地址都是不一样的。
如果是调试程序，那么我们系统每次都完全一样，这样方便下次一分析时候使用上次的分析结果，比如某一个变量的地址。

我们可以通过这样的命令来查看默认值：

```shell
$ cat /proc/sys/kernel/randomize_va_space            
2
```

如果我们想关闭该功能，可以使用如下的命令:

```shell
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
# or
sudo sysctl -w kernel.randomize_va_space=0
```


两种不同的调用栈回溯方式

1. "dynamic registration", with frame pointers in each activation record, organized as a linked list. This makes stack unwinding fast at the expense of having to set up the frame pointer in each function that calls other functions. This is also simpler to implement.

2. "table-driven", where the compiler and assembler create data structures alongside the program code to indicate which addresses of code correspond to which sizes of activation records. This is called “Call Frame Information” (CFI) data in e.g. the GNU tool chain. When an exception is generated, the data in this table is loaded to determine how to unwind. This makes exception propagation slower but the general case faster.
