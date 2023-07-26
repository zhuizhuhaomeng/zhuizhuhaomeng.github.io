---
layout: post
title: "openresty 集成 lfs 模块"
description: "openresty 怎么集成 lfs 模块"
date: 2023-07-23
tags: [openresty, logrotate]
---


# 为什么需要 lfs

文件操作是所有软件都需要具备的一个非常基本的操作。因为 Lua 本身没提供足够的文件操作接口。
因此就有了不少第三方模块。

Lua lfs 模块就是为了解决文件操作接口不足的问题。那么 OpenResty 应该如何集成 lfs 模块呢？

# 编译 lfs 模块

## lfs 模块打 patch

在使用全局变量会导致 OpenResty 打印告警信息，为了消除这个告警信息，我们增加了如下的补丁。

```patch
diff --git a/src/lfs.c b/src/lfs.c
index e5e5ee4..464f76d 100644
--- a/src/lfs.c
+++ b/src/lfs.c
@@ -1160,8 +1160,10 @@ LFS_EXPORT int luaopen_lfs(lua_State * L)
   dir_create_meta(L);
   lock_create_meta(L);
   new_lib(L, fslib);
+#ifdef ENABLE_LFS_GLOBAL
   lua_pushvalue(L, -1);
   lua_setglobal(L, LFS_LIBNAME);
+#endif
   set_info(L);
   return 1;
 }
```

## 编译 lfs

```shell
make LUA_INC=-I/usr/local/openresty/luajit/include/luajit-2.1/ LUA_LIB=B=-L/usr/local/openresty/luajit/lib/
sudo make install LUA_LIBDIR=/usr/local/openresty/lualib
```

# lfs 模块的使用

具体的接口 API 参考： https://lunarmodules.github.io/luafilesystem/manual.html#reference

```Lua
local lfs = require "lfs"

    local dir, err = lfs.currentdir()
```
