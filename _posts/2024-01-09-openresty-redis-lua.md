---
layout: post
title: "OpenResty interacting with Redis from a script"
description: "OpenResty interaction with Redis from a script"
date: 2024-01-09
tags: [OpenResty, Lua Script]
---

# 问题

先向 Redis 服务器获取所有数据，然后在客户端操作过滤需要的数据这种方式显然是低效的。

缺点包过：

1. 需要通过网络传输大量的数据
2. 服务端需要编码大量的数据
3. 客户端还需要解码大量的数据
4. 客户端要自己执行行过滤逻辑

如果能让服务端执行过滤，那么上面的问题就可以解决。 这就是为什么 Redis 服务器提供了执行 Lua 脚本的功能。
可以参考 [Redis 官方的文档](https://redis.io/docs/interact/programmability/eval-intro/#interacting-with-redis-from-a-script)。

# OpenResty 让 Redis server 执行 Lua 脚本的例子

这是一个 Lua 脚本，这个脚本是给 OpenResty 执行的。

```lua
local redis = require "resty.redis"
local red = redis:new()

red:set_timeouts(1000, 1000, 1000) -- 1 sec

local ok, err = red:connect("127.0.0.1", 6379)

if not ok then
    ngx.say("failed to connect: ", err)
    return
end

local script = [[
local i =  100
local function set_key()
    local res
    math.randomseed(10000)
    while (i > 0) do
        local val = "res"
        for i =  1, 1000 do
            val = val .. tostring(math.random())
        end 
        res = redis.call('set', KEYS[1], math.random())
        i = i-1
    end
    return res
end

local res = set_key()
return res
]]
for i = 1, 100000000 do
    ok, err = red:eval(script, 1, "10000")
    if not ok then
        ngx.say("failed to set dog: ", err)
        return
    end
end
```

我们把上述脚本保存为 test.lua, 使用 resty 命令执行。

```shell
resty test.lua
```
