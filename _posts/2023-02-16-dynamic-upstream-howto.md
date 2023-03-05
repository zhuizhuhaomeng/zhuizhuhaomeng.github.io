---
layout: post
title: "LuaJIT 的参数传递"
description: "LuaJIT 参数传递的注意事项"
date: 2023-02-16
tags: [upstream, OpenResty]
---

# 社区的问题

OpenResty 相比于 Nginx 的一大优点就是动态特性。社区上有人问想要在 404 的时候重试上游应该如何实现。
事实上这个功能 nginx 自身也可以实现，只不过 OpenResty 可以更加灵活的控制请求。

# 如何实现

想要动态控制上游就需要使用 balancer_by_lua* 这个指令。

为了实现 404 时候的重试，需要 nginx 的 [proxy_next_upstream](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) 指令来配合。

当然 [proxy_next_upstream_tries](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream_tries) 和 [proxy_next_upstream_timeout](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream_timeout) 两个指令也值得考虑。

不过 `proxy_next_upstream_tries` 也可以不配置，通过 Lua 接口来控制。

配置 `proxy_next_upstream` 指令的时候需要特别注意，如果业务本身是 `POST` 请求，但是还想要重试，
那么 应该加上 `non_idempotent` 这个参数。

# 配置案例

下面给从相关的重点配置，这里面虽然没有判断上游的结果是不是 404，但是给出了判断的途径。
既通过 `ngx.var.upstream_status` 获取上游的返回状态。

```nginx
    upstream backend {
        server 0.0.0.1;
        balancer_by_lua_block {
            local balancer = require "ngx.balancer"
            local host = "127.0.0.2"
            local port = 80
            if ngx.ctx.upstream_retries == nil then
                ngx.ctx.upstream_retries = 1
                balancer.set_more_tries(1)
            else
                ngx.ctx.upstream_retries = ngx.ctx.upstream_retries + 1
                host = "127.0.0.3"
                -- ngx.log(ngx.INFO, "upstream status: ", ngx.var.upstream_status)
            end

            local ok, err = balancer.set_current_peer(host, port)
            if not ok then
                ngx.log(ngx.ERR, "failed to set the current peer: ", err)
                return ngx.exit(500)
            end
        }

        keepalive 10;  # connection pool
    }

    server {
        location /test404 {
            proxy_next_upstream error timeout http_404;
            proxy_pass http://backend;
        }
    }
```

# 思考题

如果重试上游的时候需要使用不同的参数，那么又该如何处理呢？

这个时候就得派上我们的 [recreate_request](https://github.com/openresty/lua-resty-core/blob/master/lib/ngx/balancer.md#recreate_request).

有空我们再给一个案例。
