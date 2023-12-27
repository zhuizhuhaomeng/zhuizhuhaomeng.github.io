---
layout: post
title: "How to use OpenResty shared dictionary"
description: "How ot use OpenResty shared dictionary"
date: 2023-10-01
tags: [OpenResty, ngx.shared]
---

# 遇到的错误

使用 OpenResty 共享内存字典的同学偶尔会遇到 `out of memory` 的错误。那么这种错误是如何产生的，要怎么解决呢？

# 共享内存基本情况回顾

首先，我们来了解一下共享内存都有哪些特性。

1. 从是否过期的角度，分成会过期和不会过期两种情况。
2. 从在内存满的时候是否删除未过期表项可以分为强制删除和不删除未过期表项
3. 从是否覆盖已经存在的表项分为覆盖和不覆盖

比如下面的接口中，每个接口都有一个 `exptime` 的参数。如果 `exptime` 是 0 或者未指定，那么就不会过期。

```Lua
success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)

success, err, forcible = ngx.shared.DICT:add(key, value, exptime?, flags?)

ok, err = ngx.shared.DICT:safe_add(key, value, exptime?, flags?)```

`ngx.shared.DICT:set` 接口会覆盖或者添加表项，而 `ngx.shared.DICT:add` 只在表项不存在或者已经过期时才会添加表项。在内存不足时，这两个接口会淘汰 LRU 链表末尾上的表项。从这里可以看到，最多调用`ngx_http_lua_shdict_expire` 30 次。而 `ngx_http_lua_shdict_expire` 每次最多淘汰三个表项。

```C
/*
 * n == 1 deletes one or two expired entries
 * n == 0 deletes oldest entry by force
 *        and one or two zero rate entries
 */
static int
ngx_http_lua_shdict_expire(ngx_http_lua_shdict_ctx_t *ctx, ngx_uint_t n)；

int
ngx_http_lua_ffi_shdict_store(ngx_shm_zone_t *zone, int op, u_char *key,
    size_t key_len, int value_type, u_char *str_value_buf,
    size_t str_value_len, double num_value, long exptime, int user_flags,
    char **errmsg, int *forcible)
{
    ...

        for (i = 0; i < 30; i++) {
            if (ngx_http_lua_shdict_expire(ctx, 0) == 0) {
                break;
            }

            *forcible = 1;

            node = ngx_slab_alloc_locked(ctx->shpool, n);
            if (node != NULL) {
                goto allocated;
            }
        }
    }
}
```

# 如何导致内存不足

如果添加表项的时候不设置过期时间，那么一直往共享内存字典添加表项就会用完共享字典的内存空间。

如果设置了表项的过期时间，但是过期时间不合理，也会导致很快用完共享字典的内存空。

在用完共享字典的内存空间后，需要淘汰 LRU 链表上旧的表项。

如果此时的接口是 `ngx.shared.DICT:safe_set` 或者 `ngx.shared.DICT:set` 这样的接口，那么是不会淘汰未过期的表项，也就导致分配不到内存返回 "out of memory"。

如果此时用的是 `ngx.shared.DICT:set` 或者是 `ngx.shared.DICT:add` 这种接口，会强制淘汰 LRU 链表末尾的未过期表项，最多淘汰 30 个。如果淘汰了 30 个依然内存不足，那么此时也会返回 "out of memory"。

如果链表的末尾都是过期的表项，那么也是会有 "out of memory" 这样的错误产生的。情况如下：

1. 因为要分配的内存太大，连续 30 次调用 `ngx_http_lua_shdict_expire(ctx, 0)` 后，依然分配不到足够的内存。
1. 因为内存碎片的原因，连续 30 次调用 `ngx_http_lua_shdict_expire(ctx, 0)` 后，依然分配不到足够的内存。

举例如下：
1. 本次要分配 30 KB 的内存空间，但是 30 次内存回收只回收了 12 KB 的内存。
1. 本次需要分配 30 KB 的内存空间，30 次回收了 60KB 的内存空间，但是连续的内存块只有 6KB，依然无法分配成功。

但是，现实场景还有其它更加复杂的问题。比如 LRU 链表上大部分是过期的表项，但是由于过期时间不一致导致过期表项不是存在于链表的末尾。
这种情况下就会导致 `ngx.shared.DICT:safe_set` 或者 `ngx.shared.DICT:set` 这种不能淘汰未过期的表项即使是有很多空闲空间也无法利用的问题。

举例如下：

像这个头尾都是过期时间 1 小时，而中间都是过期时间 5 秒的情况下，会导致无法回收中间的已经过期的表项。

```text
过期时间 1h，过期时间 5s, 过期时间 5s, ...... 过期时间 1h。
```

# 使用共享内存的注意事项

总结一下上面的情况，在以下情况下容易导致内存不足

1. 未设置过期时间导致表项内存无法过期回收
1. 共享内存空间配置太小，无法分配内存
1. 共享内存的表项尺寸大小不一致，内存碎片导致缺乏连续内存而无法分配内存
1. 共享内存过期时间不一致，LRU 链表末尾也是未过期表项而无法回收过期表项的内存进而导致无法分配内存

因此，我们在使用共享内存的时候需要设置合理的过期时间，同时需要让过期时间尽可能的一致。
如果过期时间差别太大，最好是使用不同的共享内存来存储这些表项。
如果共享内存的表项占用空间大小也很不均衡，那么也需要按照大小来配置共享字典，将不同大小的表项分配到不同的字典。

当然，这么多要求会导致共享字典很难使用，那么应该如何处理呢？
如果上面的缺点都遇到了，那么可以说你的场景不适合用共享内存来实现。
可以考虑用 redis 来存储这些数据，通过 RPC 的方式来存取数据。


OpenResty XRay 可以透视 `ngx.shared` 的共享内存字典，精确分析你的共享内存不足是由于什么原因导致的。更多的共享碎片分析可以参考文章[Nginx/OpenResty 中的共享内存碎片](https://blog.openresty.com/en/nginx-shm-frag/?q=memory)
