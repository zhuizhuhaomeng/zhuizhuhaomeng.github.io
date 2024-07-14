
---
layout: post
title: "Lua 协程缓存池的实现"
description: "Lua 缓存池的实现"
date: 2024-07-14
tags: [Lua, coroutine, pool]
---

性能优化的一个关键就是使用缓存。从硬件层面的 CPU 的一级缓存，二级缓存，三级缓存
到操作系统内核的各种文件缓存，目录缓存，文件元数据信息缓存以及应用软件各种缓存都是为了
加速访问。

而我今天遇到了一个没有使用缓存会导致内存膨胀的问题。之所以会导致内存膨胀是因为每创建一个协程，
会创建一个对应的 C 的内存结构，而这个 C 结构体的内存的生命周期和协程的生命周期是不一致的。
因此即使协程被 GC 回收了也无法回收 C 结构体的内存。

针对这个问题，我想到两个方案:

1. 如果可以给创建的协程设置 GC 函数，由 GC 函数来实现 C 结构体的内存回收也是很好的。
1. 另一个就是本次方案，使用协程缓存来解决，这样不会创建新的协程，性能也会更高了。

不过缓存也会导致被缓存数据无法被 GC 回收的问题, 因此使用缓存也要非常的小心。
缓存不宜开得过大，过大浪费内存；更不能开得太小，太小缓存就起不到应有的效果。

下面把 func 和 args 赋值为 nil 就是为了减少缓存带来的影响，让 GC 可以尽快，尽可能的回收内存。

```lua
local co_create = coroutine.create
local co_yield = coroutine.yield
local co_create = coroutine.create
local co_status = coroutine.status
local co_resume = coroutine.resume


local co_pool = {}
local co_pool_cnt = 0
local co_pool_size = 128

local function create_co(func)
    local arg1, arg2, arg3
    local co

    if co_pool_cnt > 0 then
        co  = co_pool[co_pool_cnt]
        co_pool[co_pool_cnt] = nil
        co_pool_cnt = co_pool_cnt - 1
        co_resume(co, func)
    else
        co = co_create(function(...)
            local rets = {func(...)}
            while true do
                if co_pool_cnt < co_pool_size then
                    co_pool_cnt = co_pool_cnt + 1
                    co_pool[co_pool_cnt] = co
                else
                    return unpack(rets)
                end

                func = co_yield(unpack(rets))
                local args = { co_yield() }
                rets = {func(unpack(args))}
                func = nil
                args = nil
            end
        end)
    end

    return co
end

-- examples
local function f(a)
    print(a)
end

local cos = {}
for i = 1, 20 do
    local co = create_co(f)
    table.insert(cos, co)
end

for i = 1, 20 do
    local co = table.remove(cos)
    coroutine.resume(co, i)
end

```
