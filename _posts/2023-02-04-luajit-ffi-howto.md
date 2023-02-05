---
layout: post
title: "通过 LuaJIT FFI 接口集成原有的C模块"
description: "将原有的C模块通过FFI集成到OpenResty"
date: 2023-01-28
feature_image: img/luajit-ffi.png
tags: [luajit, FFI]
---

# LuaJIt FFI 接口和 Lua API 有什么差别

Lua 提供了一些接口用来实现和 C 代码的交互。比如 `void lua_pushnumber(lua_State *L, lua_Number n)`
这样的函数。更多的函数可以参考 Luajit 的 [头文件](https://github.com/LuaJIT/LuaJIT/blob/v2.1/src/lua.h#L162)。

传统上通过 Lua 的调用栈来传递参数的方法来跟 C 交互是非常麻烦的，而且有一些Lua的代码是晦涩难懂。
比如 max(a, b) 这样的一个函数，在 C 代码需要通过类似 `lua_Number lua_tonumber(lua_State *L, int idx)` 这样的函数把值取出来，
再完成计算后需要通过 `void lua_pushnumber(lua_State *L, lua_Number n)` 这样的函数将结果返回给 Lua代码。

举例如下:

## Lua代码

我们要实现如下的 Lua 代码调用


## 传统的实现方式

### Lua 代码

```lua
local c = calc.max(123, 456)
print("c is", c)
return c
```

### C 代码

```C
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdlib.h>
#include <stdio.h>

static int lua_max (lua_State *L)
{
    if (lua_gettop(L) != 2) {
        return luaL_error(L, "expecting 2 arguments, but got %d",
                          lua_gettop(L));
    }

    double a = lua_tonumber(L, 1);  /* get argument a */
    double b = lua_tonumber(L, 2);  /* get argument b */
    double c = a >= b ? a : b;

    lua_pushnumber(L, c);  /* push result */

    return 1;  /* number of results */
}

static void
init(lua_State *L)
{
    lua_newtable(L);

    lua_pushcfunction(L, lua_max);
    lua_setfield(L, -2, "max");

    lua_setglobal(L, "calc");
}

int
main(void)
{
    int status, result, i;
    double sum;
    lua_State *L;

    /*
     * All Lua contexts are held in this structure. We work with it almost
     * all the time.
     */
    L = luaL_newstate();

    luaL_openlibs(L); /* Load Lua libraries */

    init(L);
    /* Load the file containing the script we are going to run */
    status = luaL_loadfile(L, "script.lua");
    if (status) {
        /* If something went wrong, error message is at the top of */
        /* the stack */
        fprintf(stderr, "Couldn't load file: %s\n", lua_tostring(L, -1));
        exit(1);
    }

    /* Ask Lua to run our little script */
    result = lua_pcall(L, 0, 1, 0);
    if (result) {
        fprintf(stderr, "Failed to run script: %s\n", lua_tostring(L, -1));
        exit(1);
    }

    /* Get the returned value at the top of the stack (index -1) */
    sum = lua_tonumber(L, -1);

    printf("Script returned: %.0f\n", sum);

    lua_pop(L, 1);  /* Take the returned value out of the stack */
    lua_close(L);

    return 0;
}
```

## FFI 调用方式

对于 FFI，我们要实现 calc 模块，其代码如下

下述代码为 calc.lua

```lua
local ffi = require "ffi"

local C = ffi.C

local _M = {
    version = 1.0
}


ffi.cdef[[
double lua_max(double a, double b);
]]

function _M.max(a, b)
    local c = C.lua_max(a, b)
    return c
end

return _M
```

下述代码为 script.lua, 该代码通过require 使用 calc 模块

```lua
local calc = require "calc"

local c = calc.max(123, 456)
print("c is", c)
return c
```

对应的C代码的实现

```C
#include <lua.h>
#include <lualib.h>
#include <lauxlib.h>
#include <stdlib.h>
#include <stdio.h>

double lua_max(double a, double b)
{
    double c = a >= b ? a : b;

    return c;
}

int
main(void)
{
    int status, result, i;
    double sum;
    lua_State *L;

    /*
     * All Lua contexts are held in this structure. We work with it almost
     * all the time.
     */
    L = luaL_newstate();

    luaL_openlibs(L); /* Load Lua libraries */

    /* Load the file containing the script we are going to run */
    status = luaL_loadfile(L, "script.lua");
    if (status) {
        /* If something went wrong, error message is at the top of */
        /* the stack */
        fprintf(stderr, "Couldn't load file: %s\n", lua_tostring(L, -1));
        exit(1);
    }

    /* Ask Lua to run our little script */
    result = lua_pcall(L, 0, 1, 0);
    if (result) {
        fprintf(stderr, "Failed to run script: %s\n", lua_tostring(L, -1));
        exit(1);
    }

    /* Get the returned value at the top of the stack (index -1) */
    sum = lua_tonumber(L, -1);

    printf("Script returned: %.0f\n", sum);

    lua_pop(L, 1);  /* Take the returned value out of the stack */
    lua_close(L);   /* Cya, Lua */

    return 0;
}
```

编译 C 代码的命令为 `gcc -rdynamic -I/usr/include/luajit-2.1 -lluajit-5.1 main.c`
如果不使用该命令，那么会出现类似这样的错误。

```
Failed to run script: ./calc.lua:15: /lib64/libluajit-5.1.so.2: undefined symbol: lua_max
```

通过这两个例子对比我们可以看到，FFI 的方式传递参数会更加的自然，在C代码里面不需要从栈上获取参数再将结果压栈。

如果我们要集成的是现有的 C 模块，那么通过 FFI 的方式集成显然是更简单方便的。在理想情况下，C代码完全不需要改动。

除了编码上的便利易于理解以外，通过 FFI 方式调用还有性能更高的优点，因为参数会直接通过寄存器进行传递了。

# 如何集成一个库

OpenResty 是最成功使用 LuaJIT 的一个开源软件。一半我们扩展功能是增加一个库文件，因此直接把现有的库编译成 so 文件。

在使用 so 文件的方式时，在加载上会有一点点的差异。
以 lua-resty-signal 为例子，在将 C 代码编译成 so 库后，Lua代码里面加载 so 的方式如下：

```lua
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


local resty_signal, tried_paths = load_shared_lib("librestysignal.so")
if not resty_signal then
    error("could not load librestysignal.so from the following paths:\n" ..
          table.concat(tried_paths, "\n"), 2)
end

ffi.cdef[[
int resty_signal_signum(int num);
]]

local function signum(name)
    local sig_num
    if type(name) == "number" then
        sig_num = name
    else
        local id = signals[name]
        if not id then
            return nil, "unknown signal name"
        end

        sig_num = tonumber(resty_signal.resty_signal_signum(id))
        if sig_num < 0 then
            error(
                string_format("missing C def for signal %s = %d", name, id),
                2
            )
        end
    end
    return sig_num
end
```

这里的重点是需要主动加载so，在调用C函数时不再使用C这个空间，而是使用加载的so变量， 比如 `resty_signal.resty_signal_signum(id)`。

lua-resty-signal 的项目地址在 https://github.com/openresty/lua-resty-signal

# 其它问题

FFI 传递参数还有很多的注意事项，比如可变参数怎么传递 int，short变量，数组等等。后面我们将进一步提供各种参数传递的例子。

FFI 的细节需要到官网 https://luajit.org/ext_ffi.html 进行查阅。
