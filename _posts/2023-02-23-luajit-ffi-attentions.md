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
