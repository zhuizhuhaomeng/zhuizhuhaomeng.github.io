---
layout: post
title: "lua_package_path of the OpenResty"
description: "Lua package path of the OpenResty"
date: 2024-09-23
tags: [Nginx, Search Path]
---

OpenResty 官方发布的包的搜索路径默认包含 /usr/local/openresty/site/lualib，这个是如何做到的呢？

从 C 代码中我们可以看到， lua-nginx-module 的正常发布版本是不会有 LUA_DEFAULT_PATH 的，除非定义了 LUA_DEFAULT_PATH。

```C
#if !defined(LUA_DEFAULT_PATH) && (NGX_DEBUG)
#define LUA_DEFAULT_PATH "../lua-resty-core/lib/?.lua;"                      \
                         "../lua-resty-lrucache/lib/?.lua"
#endif


static lua_State *
ngx_http_lua_new_state(lua_State *parent_vm, ngx_cycle_t *cycle,
    ngx_http_lua_main_conf_t *lmcf, ngx_log_t *log)
{
...
#ifdef LUA_DEFAULT_PATH
#   define LUA_DEFAULT_PATH_LEN (sizeof(LUA_DEFAULT_PATH) - 1)
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0,
                       "lua prepending default package.path with %s",
                       LUA_DEFAULT_PATH);

        lua_pushliteral(L, LUA_DEFAULT_PATH ";"); /* package default */
        lua_getfield(L, -2, "path"); /* package default old */
        lua_concat(L, 2); /* package new */
        lua_setfield(L, -2, "path"); /* package */
#endif

#ifdef LUA_DEFAULT_CPATH
#   define LUA_DEFAULT_CPATH_LEN (sizeof(LUA_DEFAULT_CPATH) - 1)
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0,
                       "lua prepending default package.cpath with %s",
                       LUA_DEFAULT_CPATH);

        lua_pushliteral(L, LUA_DEFAULT_CPATH ";"); /* package default */
        lua_getfield(L, -2, "cpath"); /* package default old */
        lua_concat(L, 2); /* package new */
        lua_setfield(L, -2, "cpath"); /* package */
#endif
}

既然上面有一个例外的情况，那么就需要看看这个宏是哪里定义的。想当然的到发行脚本中去查找结果找不到。
不过终于有一天在 openresty 的 [configure](https://github.com/openresty/openresty/blob/master/util/configure#L889) 脚本中看到了。

```perl
$lualib_prefix = File::Spec->catfile($prefix, "lualib");
        my $site_lualib_prefix = File::Spec->catfile($prefix, "site/lualib");

        {
            my $ngx_lua_dir = auto_complete 'ngx_lua';

            my $outfile = "$ngx_lua_dir/config";

            open my $in, ">>$outfile" or
                die "Cannot open $outfile for appending: $!\n";

            {
                print $in <<"_EOC_";

echo '
#ifndef LUA_DEFAULT_PATH
#define LUA_DEFAULT_PATH "$site_lualib_prefix/?.ljbc;$site_lualib_prefix/?/init.ljbc;$lualib_prefix/?.ljbc;$lualib_prefix/?/init.ljbc;$site_lualib_prefix/?.lua;$site_lualib_prefix/?/init.lua;$lualib_prefix/?.lua;$lualib_prefix/?/init.lua"
#endif

#ifndef LUA_DEFAULT_CPATH
#define LUA_DEFAULT_CPATH "$site_lualib_prefix/?.so;$lualib_prefix/?.so"
#endif
' >> "\$ngx_addon_dir/src/ngx_http_lua_autoconf.h"
_EOC_
            }
```

隐藏得好深，一直找不到找得让我怀疑人生，没想到不经意发现。

lua_package_cpath 的设置也是同样的情况，不再赘述。

有心栽花花不开，无心插柳柳成荫。
