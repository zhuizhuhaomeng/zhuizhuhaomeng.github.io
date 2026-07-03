---
layout: post
title: "vs code 使用"
description: "vs code 配置"
date: 2024-04-11
tags: [vscode]
---

# 自动跳转

vscode 插件好多好多，多到让人头花脑胀。而我最讨厌的就是 C、C++ 无法自动跳转了。
不过还好现在有了 clangd 插件，可以更好的自动跳转了。

虽然插件很好，但是还是配置还是有门槛。

如果是 cmake 系统，应该使用这样的方式

```shell
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=1
```

如果就只有 Makefile，那么应该使用

```shell
bear -- make
```

# Lua 配置

## Navigate to a directory where you want to store the type definitions

```shell
mkdir -p ~/.local/share/lua-language-server/meta
cd ~/.local/share/lua-language-server/meta
git clone https://github.com/LuaCATS/openresty
```

## Config vscode

Press `ctrl + shift + p`  and then type `Preferences: Open User Settings (JSON)`

```json
{
    "cSpell.userWords": [
        "cjson",
        "lrucache",
        "resty",
        "aliyun",
        "addrs",
        "httpc",
        "HTTPDNS",
        "receiveany",
        "preread",
        "NOAUTH",
        "nbytes",
        "plen",
        "atyp",
        "mbits",
        "agentzh",
        "lshift",
        "rshift",
        "tohex",
        "cdef",
        "settimeout",
        "settimeouts",
        "shdict",
        "lmdb",
        "nkeys"
    ],
    "Lua.workspace.library": [
      "${3rd}/openresty/library"
    ],
    "Lua.diagnostics.globals": ["ngx"],
    "Lua.runtime.version": "LuaJIT",
}
```
