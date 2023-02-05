---
layout: post
title: "LuaJIT 的参数传递"
description: "LuaJIT 参数传递的注意事项"
date: 2023-02-05
feature_image: img/luajit-ffi.png
tags: [luajit, FFI]
---

FFI 的方式调用 C 函数可以说是很简单，但是有时候缺个例子也是非常的难。
本文通过几个例子来展示 FFI 调用 C 函数。相信对于需要用到可变参数函数
以及需要传递数组参数的会有帮助。

# 传递数值类型参数

传递数值类型的参数可以说是最简单的了。FFI 会自动将数值转换为需要的类型。

比如：

```lua
local ffi = require "ffi"

local C = ffi.C

local _M = {
    version = 1.0
}

ffi.cdef[[
double lua_max_d(double a, double b);
double lua_int_i(int a, int b);
]]

function _M.max_d(a, b)
    local c = C.lua_max_d(a, b)
    return c
end

function _M.max_i(a, b)
    local c = C.lua_max_i(a, b)
    return c
end

return _M
```

# 传递字符串

传递字符串参数需要把 C 语言的参数定义为 const 类型的指针，并且不能修改该参数。

比如：

```lua
local ffi = require("ffi")

ffi.cdef[[
int puts(const char *s);
]]

ffi.C.puts("Hello world! I am lijunlong\n")
```

# 传递字符串数组

Unix 的命令行参数刚好是一个字符串数组，因此我们用 `execv` 做一个例子。注意，这里要分配字符串数组时要用 `#arg + 1`。因为 execv 要求最后一个以 NULL 结尾作为定界符。


```lua
ffi.cdef[[
int execv(const char *path, char *const argv[]);
]]

function execv(...)
    local arg = {...}
    arg = ffi.new("const char*[?]", #arg+1, arg)
    arg = ffi.cast("char *const*", arg)
    return ffi.C.execv(arg[0], arg)
end

execv("/usr/bin/echo", "hello", "world")
```

这里有一个可以优化的地方是不需要每一次都给参数分配内存。因此可以使用类似下面这样的优化方法。这里面通过将参数数组缓存在 cached_args 这个 upvalue 中消除了每次分配内存的开销。

这里面要注意，FFI 的索引是从 0 开始，而不是像Lua从 1 开始。

```lua
ffi.cdef[[
int execv(const char *path, char *const argv[]);
]]

local cached_args= ffi.new("const char*[?]", 100)


function execv(...)
    local args_cnt = select('#', ...)
    args = cached_args

    if args_cnt >= 100 then
        args = ffi.new("const char*[?]", args_cnt)
    end

    for i = 1, args_cnt do
        args[i - 1] = select(i, ...)
    end
    args[args_cnt] = nil

    args = ffi.cast("char *const*", args)
    return ffi.C.execv(args[0], args)
end

execv("/usr/bin/echo", "hello", "world")
```

# 传递指针作为输出参数

因为 Lua 没有取地址的运算符，如果要输入指针作为函数的输出参数，那么就应该传递元素个数为1的数组这样的小技巧。

```lua
local ffi = require("ffi")
local C = ffi.C

ffi.cdef[[
struct timeval {
    long tv_sec;     /* seconds */
    long tv_usec;    /* microseconds */
};

struct timezone {
    int tz_minuteswest;     /* minutes west of Greenwich */
    int tz_dsttime;         /* type of DST correction */
};

int gettimeofday(struct timeval *tv, struct timezone *tz);
]]


local tv = ffi.new("struct timeval[1]")
local rc = C.gettimeofday(tv, nil)
print(type(rc))
print(type(tv[0].tv_sec))
print(type(tv[0].tv_usec))
print("sec:", tv[0].tv_sec, "usec:", tv[0].tv_usec)
```

执行以上代码，输出类似结果类似下面这样子

```shell
$ luajit script.lua
number
cdata
cdata
sec:	1675580196LL	usec:	623550LL
```

可见 LuaJIT 自动把返回值类型为 int 函数的返回值转换为了 Lua 数值类型。
但是对于通过 tv[0].tv_sec 这样的方式访问 long 类型时并不会自动转换。
但是如果方位 cdata 结构体的成员是int类型时候，是否会自动转换为 Lua 的数值类型呢？ 经过实验是可以的。

实验代码如下：

```lua
local c = ffi.new("int [1]")
print(type(c[0]))
```

# 传递字符串参数

我们以经典的 `printf` 函数为例子来说明传递可变参数。

如果我们要给 `printf` 传递整数，那么不能像上面的传递数值类型一样直接把

```lua
local ffi = require("ffi")
local math = require "math"


ffi.cdef[[
int printf(const char *fmt, ...);
]]
local C = ffi.C

C.printf("Hello %s!\nI am %s.\nI am %d years old.\nI am %dcm.\n",
         "world", "lijunlong", ffi.new("int", 28), ffi.new("int", 178))
C.printf("Pi %f %lf\n", ffi.new("float", math.pi), ffi.new("double", math.pi))
```

# 后续

后面我们进一步介绍 ffi.typeof 的使用以及 ffi.gc 的使用
