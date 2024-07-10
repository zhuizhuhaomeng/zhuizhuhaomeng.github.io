---
layout: post
title: "cosocket 的上游元表是如何生效的"
description: "上游元表的实现方法"
date: 2024-07-08
tags: [cosocket, metatable, upstream]
---

回收资源是一个很重要的问题。函数之间的调用非常复杂，所以一个资源的生命周期可能很难确定。
如果有一个统一的入口和统一的出口实现资源的分配和回收那么就可以极大的简化问题。
Lua 个 gc 回调就是一个很好的时机，可以作为一种最终的兜底方案。

下面看看 ngx_http_lua_socket_tcp_upstream_t 这个 GC 对象是如何设置 gc 函数回收资源的。

# 元表的创建

在 `ngx_http_lua_inject_socket_tcp_api` 函数中创建了以 `ngx_http_lua_upstream_udata_metatable_key`
作为 key 的元表，该元表存储在注册表（LUA_REGISTRYINDEX）中。该元表中存在成员 `__gc`。
`gc` 对应的回调函数是 `ngx_http_lua_socket_tcp_upstream_destroy`。


```C
static char ngx_http_lua_upstream_udata_metatable_key;

void
ngx_http_lua_inject_socket_tcp_api(ngx_log_t *log, lua_State *L)
{
    /* {{{upstream userdata metatable */
    lua_pushlightuserdata(L, ngx_http_lua_lightudata_mask(
                          upstream_udata_metatable_key));
    lua_createtable(L, 0 /* narr */, 1 /* nrec */); /* metatable */
    lua_pushcfunction(L, ngx_http_lua_socket_tcp_upstream_destroy);
    lua_setfield(L, -2, "__gc");
    lua_rawset(L, LUA_REGISTRYINDEX);
    /* }}} */
}
```

# 创建上游对象时设置元表

可以看到，在 `lua_newuserdata` 创建 GC 对象之后，就给该 GC 对象设置了元表。

static int
ngx_http_lua_socket_tcp_connect(lua_State *L)
{
        u = lua_newuserdata(L, sizeof(ngx_http_lua_socket_tcp_upstream_t));
        if (u == NULL) {
            return luaL_error(L, "no memory");
        }

#if 1
        lua_pushlightuserdata(L, ngx_http_lua_lightudata_mask(
                              upstream_udata_metatable_key));
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_setmetatable(L, -2);
#endif

        lua_rawseti(L, 1, SOCKET_CTX_INDEX);
}
```

