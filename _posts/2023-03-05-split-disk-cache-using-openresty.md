---
layout: post
title: "如何利用多块不同规格的磁盘来打造高效的缓存"
description: "如果仅仅使用一块巨大的磁盘，那么磁盘 IO 就会成为性能瓶颈；如果使用多块盘就必须得做RAID,而扩展容量则很不方便。"
date: 2023-03-05
tags: [OpenResty, Cache, Nginx]
---

# 问题

大型网站都需要通过 CDN 实现就近访问来提升客户的服务满意度。很多客户直接使用 Nginx/OpenResty 来打造自己的 CDN 服务器。
这个时候就会出现一个问题：磁盘 IO 成为了服务器的性能瓶颈。

不假思索的方法就是使用更高性能的磁盘，但是高性能磁盘显然不是一个好的解决方案。
我们还可以通过将多块磁盘通过 RAID0 的方式组成磁盘阵列的方式来提升性能。但是 Nginx 提供了一个更简单的方式，直接把文件分在
多块不同的磁盘中。

# Nginx 的解决方案

## 常规的 Nginx 缓存配置

常规的 Nginx 的缓存配置如下所示。该配置通过 `proxy_cache_path` 指令配置一个缓存目录，
通过 `proxy_cache` 指令指示 Nginx 服务器将文件缓存到上面配置的目录。

这种情况下只有一个目录，因此也就只会用到一个磁盘。如何将缓存文件分到多个不同的目录呢？
我们可以通过 Nginx 提供的 `split_clients` 的指令来将请求的根据 URL 参数映射到不同的目录。
这就是 [Nginx 官网介绍的方案](https://www.nginx.com/blog/nginx-caching-guide/)。


```nginx
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g 
                 inactive=60m use_temp_path=off;

server {
    # ...
    location / {
        proxy_cache my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502
                              http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;

        proxy_pass http://my_upstream;
    }
}
```

## split_client 方案

```nginx
proxy_cache_path /path/to/hdd1 levels=1:2 keys_zone=my_cache_hdd1:10m
                 max_size=10g inactive=60m use_temp_path=off;
proxy_cache_path /path/to/hdd2 levels=1:2 keys_zone=my_cache_hdd2:10m
                 max_size=10g inactive=60m use_temp_path=off;

split_clients $request_uri $my_cache {
              50%          "my_cache_hdd1";
              50%          "my_cache_hdd2";
}

server {
    # ...
    location / {
        proxy_cache $my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502
                              http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;

        proxy_pass http://my_upstream;
    }
}
```

我们将 Nginx 官网的配置稍微整理之后就如上面的配置所示。

可以看到，这个示例中通过 `proxy_cache_path` 指令配置了两个缓存目录： `my_cache_hdd1` 和 `my_cache_hdd2`。通过 `split_clients` 指令将 `$request_uri` 变量映射为 `$my_cache`变量。该变量取值分别是 `my_cache_hdd1` 和 `my_cache_hdd2`, 并且两者的比率各占据 50%。最后通过 `proxy_cache` 指令指示应该使用的缓存目录。

其实该配置的核心就是 `proxy_cache` 指令跟的是一个 Nginx 变量，而不是常规配置下的字符串值。

因为 `proxy_cache` 指令的参数可以是变量，那么这个变量就可以通过 OpenResty 来设置，这样就可以更加灵活的控制缓存文件的分布方式了。

下面给出 OpenResty 下的解决方案。

# 使用 OpenResty 实现多个磁盘缓存

```nginx
proxy_cache_path /path/to/hdd1 levels=1:2 keys_zone=my_cache_hdd1:10m
                 max_size=10g inactive=60m use_temp_path=off;
proxy_cache_path /path/to/hdd2 levels=1:2 keys_zone=my_cache_hdd2:10m
                 max_size=10g inactive=60m use_temp_path=off;

set $my_cache my_cache_hdd1;

server {
    # ...
    location / {
        access_by_lua_block {
            -- choose_cache 函数根据需要来分配缓存
            ngx.var.my_cache = choose_cache()
        }

        proxy_cache $my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502
                              http_503 http_504;
        proxy_cache_background_update on;
        proxy_cache_lock on;

        proxy_pass http://my_upstream;
    }
}
```

我们通过 `ngx.var.my_cache = choose_cache()` 来设置缓存配置项的名称。

`choose_cache` 就是我们需要实现的函数了。比如我们的需求是把热门的文件缓存在高性能的磁盘中，而把冷门的文件缓存在普通的机械硬盘中。如果仅仅使用 `split_clients` 指令我们是无法实现该需求的，但是借助 Lua，我们则可以充分发挥我们的想象力来实现该功能了。



# 参考链接

https://www.nginx.com/blog/nginx-caching-guide/
