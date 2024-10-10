---
layout: post
title: "LuaJIT FFI 的注意事项"
description: "LuaJIT FFI 的注意事项"
date: 2023-02-23
tags: [LuaJIT, FFI]
---

# NULL 指针 与 nil 比较

我们一般是通过 `ptr == nil` 或者 `ptr ~= nil` 这样的方式来判断是否空指针。
但是下面的表达式却是错误的，无法达到判断 ptr 是否是 NULL 指针的目的：

```lua
if ptr then
    -- do something
end
```

这个是因为 `ptr` 是一个 `cdata` 类型的数据，不同的数据类型的数据比较总是不相等。
只有在 `ptr == nil` 或者 `ptr ~= nil` 这种显示的比较的时候，luajit 才扩展这个语义。

如果是下面这种判断的方式也是错误的:

```lua
if not ptr then
    print("ptr is null")
end
```


# 引用语义和值语义

对于成员是结构体的数组来说，a[i]的结果是一个引用类型，而不是结构对象的副本。
这与标量数组不同，标量数组具有值语义（标量是不可变的）。
这在试图实现泛化元素类型的数据结构时表现出来。
因为不能假设值语义，所以你不能只用 a[i]来弹出一个值或用于交换值（a[i], a[j] = a[j], a[i]不再适用）。

#参考连接

nil-equality-of-pointers: https://luapower.com/luajit-notes#nil-equality-of-pointers

# OpenResty LuaJIT 调试

想查看 luajit 生成的 trace 信息，可以使用下面的命令

```shell
init_worker_by_lua_block { require "jit.dump".on("tbirsm", "/tmp/traces.txt." .. ngx.worker.pid()) }
```

# 案例代码

## 判断 cdata

### C 代码

```C
#include <stdlib.h>

void *get_null(void)
{
    return NULL;
}
```

编译 C 代码的命令: 
```shell
gcc -shared  -o test.so test.c
```

### Lua 代码

```lua
local new_tab = require "table.new"
local ffi = require "ffi"

local load_shared_lib
do
    local string_gmatch = string.gmatch
    local string_match = string.match
    local io_open = io.open
    local io_close = io.close

    local cpath = package.cpath

    function load_shared_lib(so_name)
        local tried_paths = new_tab(32, 0)
        local i = 1

        for k, _ in string_gmatch(cpath, "[^;]+") do
            local fpath = string_match(k, "(.*/)")
            fpath = fpath .. so_name
            -- Don't get me wrong, the only way to know if a file exist is
            -- trying to open it.
            local f = io_open(fpath)
            if f ~= nil then
                io_close(f)
                return ffi.load(fpath)
            end

            tried_paths[i] = fpath
            i = i + 1
        end

        return nil, tried_paths
    end  -- function
end  -- do


local test , tried_paths = load_shared_lib("test.so")
if not test then
    error("could not load librestysignal.so from the following paths:\n" ..
          table.concat(tried_paths, "\n"), 2)
end

ffi.cdef[[
void *get_null(void);
]]

local cdata = test.get_null()
print(tostring(cdata))

if not cdata then
    print("`not cdata` works")
else
    print("`not cdata` can not work")
end

if n == nil then
    print("`cdata == nil` works")
else
    print("`cdata == nil` can not work")
end
```

### 测试结果

```shell
$ luajit t.lua 
cdata<void *>: NULL
`not cdata` can not work
`cdata == nil` works
```
