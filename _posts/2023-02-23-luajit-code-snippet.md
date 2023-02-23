---
layout: post
title: "LuaJIT 代码片段"
description: "LuaJIT 代码片段"
date: 2023-02-23
tags: [luajit, FFI]
---

# 如何判断 IPv4 地址是内网地址

下面是openresty的一个片段代码，用于展示如何判断一个 IP 地址是否是内网地址。
这个比通过字符串判断是否是 `127.` `10.` `172.16. ~ 172.31.` `192.168.` 的这些网段高效多了。

```lua
local bit_and = require "bit".band
local addr = ngx.var.binary_remote_addr
local fb = string.byte(addr, 1)
local sb = string.byte(addr, 2)

if #addr == 4 and (fb == 10
   or fb == 127
   or (fb == 172 and bit_and(sb, 0xFF) == 16)
   or (fb == 192 and sb == 168)) then
    ngx.say("it is local")
end
```
