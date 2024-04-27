---
layout: post
title: "使用 OpenResty 共享字典的注意事项"
description: "使用 OPenResty 共享字典的注意事项"
date: 2024-04-27
tags: [OpenResty, Shdict, Memory]
---

OpenResty 共享内存的字典如果设置得太大，那么就会进程占用太多的宝贵内存；
如果内存设置得过小，那么内存不足会导致申请失败或者提前淘汰未过期的表项。

如何用好共享字典其实是很有讲究的。我们来看看共享字典的一些问题。

# OpenResty 共享内存字典如何淘汰表项

OpenResty 共享字典淘汰表项主要可以分为淘汰过期表项和淘汰未过期表项。
淘汰过期表项是每次操作都会进行的，而淘汰未过期表项是内存申请失败时候进行的。

`ngx_http_lua_shdict_expire(ctx, 1)` 调用就是执行未过期表项的淘汰。在 `DICT:set`, `DICT:get`, `DICT:lpush` 等操作的入口都是先执行淘汰过期表项的动作。之所以这么做是为了让过期表项更加均匀的被淘汰而不至于让某一次的执行时间过长。

## DICT:set 如何淘汰表项

[ngx.shared.DICT.set](https://github.com/openresty/lua-nginx-module/?tab=readme-ov-file#ngxshareddictset)
接口签名是 `syntax: success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)`。

这里我们重点关注 `exptime` 这个参数以及 `success`, `forcible` 两个返回结果。

`exptime` 这个参数设置表项的过期时间，表项过期即可被回收。
如果共享内存满了，那么 `set` 接口就会强制淘汰未过期表项，如果淘汰的表项后成功分配到内存，那么这时候返回 的 `forcible` 就是 `true`。

而 `success` 表示是否成功，虽然 `set` 接口会强制淘汰未过期的内存，但是只是淘汰一定数量的表项，而不是无限制的淘汰。

从下面的代码可以看到，最多会调用 30 次的 ngx_http_lua_shdict_expire 去淘汰表项，而 `ngx_http_lua_shdict_expire(ctx, 0)` 每次调用最多淘汰3个表项。如果 LRU 链表末尾上的都是未过期的表项，那么 `ngx_http_lua_shdict_expire(ctx, 0)` 就只会淘汰 1 个表项。

```C
    if (node == NULL) {

        if (op & NGX_HTTP_LUA_SHDICT_SAFE_STORE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            *errmsg = "no memory";
            return NGX_ERROR;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: overriding non-expired items "
                       "due to memory shortage for entry \"%*s\"", key_len,
                       key);

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

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        *errmsg = "no memory";
        return NGX_ERROR;
    }
```

虽然 `DICT:set` 会强制淘汰表项，但是这不代表执行一定会成功。因为虽然会强制淘汰多次，但是这些被淘汰的内存不一定是连城一片的，因此最终申请可能还是会因为内存碎片而无法申请成功。

## DICT:safe_set 如何淘汰表项
`DICT:safe_set` 一样在入口就会执行淘汰过期表项的动作。同 `DICT:set` 相比， `DICT:safe_set` 并不会强制淘汰未过期的表项，因此如果申请不到内存就会返回失败。

因此虽然接口名称有个 `safe`, 但是使用这个接口其实是很不安全的，一定要判断接口的返回值。这里的 `safe` 其实只是对于已经存在的表项的 `safe` 的，而对本次操作并不保证成功，因此也就是操作本身并不  `safe`。

# OpenResty 共享字典的内存碎片

共享字典的内存分配使用的是跟 Linux 系统的 Slab 分配器一样的伙伴分配算法，
会按照 2^n 的尺寸对内存进行切割。 不管多好的内存分配算法，总会遇到内存碎片的问题。
那么如何更好的避免内存碎片呢？这个就要了解碎片是如何产生的。

内存分配具有随机性，这些随机的内存分配的内存的生命周期和大小都不一致。这样的特性导致大块小块的内存混杂在一起，有的被释放了，有的却长期存在。随着内存分配的进行，内存块越来越细碎，不连续的细小空闲内存无法聚合到一起导致无法服务大块的内存申请, 这些小内存就可以称为内存碎片。

虽然内存分配器对于内存碎片无能为力，但是应用层是可以通过对应用进行优化来实现减少甚至消除内存碎片的。
比如：

1. 将单一共享内存改为多个共享内存，按照不同的内存大小到不同的共享内存申请内存。当然存在的问题就是写入共享内存的时候知道总的大小，但是查询的时候要到哪个共享内存字典查询，如果遍历多个显然就比较低效。因此这种情况更适合于能够按照 key 区分共享内存大小的情况。
2. 将表项大小归一化：很多情况下，我们共享内存存储的 value 是数值，而 key 是不同大小的。比如：我们想按照一定特征进行限速，可能是 url， cookie 等参数的组合。如果直接使用这些原始值，那么 key 的大小可能很大，并且尺寸很不统一。如果把这些 key 做一个 md5，那么就可以得到大小都是 16 字节的 key，这样就不存在内存碎片的问题了。
3. 二级存储方式：将元数据存储在共享内存中，将实际数据存储在硬盘上。这样既可以减少共享内存的大小，又能够降低共享内存碎片发生的概率。只不过这个时候要额外进行硬盘数据的清理。

# 共享字典如何释放内存

**共享字典的内存只会逐步增长，不会降低。** 

因此配置共享内存的时候要设置合理值，否则可能导致进程使用的内存太大，在系统内存不足时进程被系统的 OOM Killer 杀死的情况。

为什么共享字典的内存不会被释放？这是因为共享内存是采用 mmap 的方式而不像 malloc 分配的内存还会被 free进而在适当的机会归还给操作系统。

# 否进程启动后就会占用共享字典分配的内存

这个是不会的。Linux 系统的内存是按需分配的，只有在实际写入共享字典的时候操作系统才会真正分配物理内存。
因此设置比较大的共享内存并不会立即分配对应大小的物理内存。

```nginx
lua_shared_dict dogs 1000m;
```

上面是一个配置了1G的共享内存。我们看看在没有写入任何数据情况下的内存占用情况。

```shell
# master process
$ pmap -x -p 1730 | egrep "(RSS|zero)"
Address           Kbytes     RSS   Dirty Mode  Mapping
00007fff86070000 1024000    5976    5976 rw-s- /dev/zero (deleted)

# worker process
$ pmap -x -p 385165 | egrep "(RSS|zero)"
Address           Kbytes     RSS   Dirty Mode  Mapping
00007fff86070000 1024000       0       0 rw-s- /dev/zero (deleted)
```

从上面可以看到， master 进程的 RSS 是 5976 Kbytes， 而 worker 进程则是 0。
master 进程是因为对共享内存进行了一定的初始化，因此会占用一定的物理内存。
worker 进程是 master 进程 fork 出来的，共享内存的布局跟 master 进程是完全一致的，
为什么 pmap 展示的共享内存部分是 0 呢？ 这个应该是因为 worker 进程没有任何访问，连对应的页表也是空的，所以统计的 RSS 就是0。

# 共享内存的过期时间

共享内存淘汰表项是淘汰 LRU 链表上的表项，我们需要了解这个 LRU 链表是如何形成的。

比如我们连续插入 1,2,3,4,5,6 六个表项，那么 LRU 链表如下：

```
6 <- 5 <- 4 <- 3 <- 2 <- 1
```

如果这时候查询一下表项 3， 那么 LRU 链表变成

```
3 <- 6 <- 5 <- 4 <- 2 <- 1
```

如果再查询一下 1，那么链表变成

```
1 <- 3 <- 6 <- 5 <- 4 <- 2
```

也就是说，查询的时候会把表项从 LRU 链表摘除再放在链表头。

假设表项 1，2, 3 是过期表项。 如果这时候要淘汰过期表项，那么从链表末尾开始。
第一个表项 2 是可以被淘汰的，第二个是表项 4 因为还未过期，因此无法被淘汰。
虽然 1， 3 也是过期的，但是因为在后面还有未过期的表项，因此也就不会被淘汰。

这里存在一个问题就是虽然存在过期表项，但是过期表项也不一定会被淘汰，因为他们不在链表的末尾。
而在内存不足时，未过期的表项可能就会被强制淘汰了,因为他们在链表的末尾。

:open_mouth: 惊不惊喜，意不意外。:open_mouth:
