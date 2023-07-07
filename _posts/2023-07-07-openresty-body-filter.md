---
layout: post
title: "OpenResty 过滤响应体"
description: "body filter by lua 的注意事项"
date: 2023-07-07
tags: [OpenResty]
---

# 响应体过滤的问题

很多初学者在使用 OpenResty 过滤响应体的时候总是会遇到错误而调试半天。

最常见的就是没有先把 Content-Length 的响应头清空。
因为过滤响应体基本上是会修改响应体的大小, 如果不删除响应头的 Content-Length，
那么响应体大小和 Content-Length 的值就对不上，这就导致协议错误。

另外一个是要对响应体进行修改，最好是未压缩的响应体，否则还需要进行解压才有办法执行替换。
但是具体的策略需要根据情况进行分析后进行选择，因为未压缩的文件可能是压缩后的好几倍，
这个不仅会增加网络传输的耗时并且会增加服务器的宽带费用。

当然，我们这里进行过滤之后会导致数据是未压缩的，因此也会导致数据的膨胀。
如果要处理该问题，那么可以使用子请求进行全缓冲，然后再发送响应。

# 服务器返回未压缩的数据

如果让服务器之间返回未压缩的报文，那么这时候就需要在发送给上游请求的时候把 Accept-Encoding
的头部去掉。


下面这是一个配置示例,展示了对上游未压缩的响应体进行替换的例子。

```lua
location ~ /css {
    proxy_set_header Accept-Encoding "";
    proxy_set_header Connection "";
    proxy_http_version 1.1;
    proxy_set_header Host $host;

    header_filter_by_lua_block {
        ngx.header.content_length = ni
    }

    body_filter_by_lua_block {
        -- ngx.arg[1] 代表输入的 chunk
        local chunk, eof = ngx.arg[1], ngx.arg[2]
        local buffers = ngx.ctx.buffers
        if not buffered then
           buffers = {}
           ngx.ctx.buffers = buffers
           ngx.ctx.buffers_cnt = 0
        end

        if chunk ~= "" then
           local cnt = ngx.ctx.buffers_cnt + 1
           ngx.ctx.buffers_cnt = cnt
           buffers[cnt] = chunk
           ngx.arg[1] = nil
        end

        if eof then
           local body = table.concat(buffers)
           ngx.ctx.buffers = nil

           body = ngx.re.gsub(body, "<img name='xxx'>",  "<img name='xxx', attr='hide'>", "jo")
           ngx.arg[1] = body
        end
    }
}
```

# 服务器返回压缩后的数据

下面这个示例设置为接受gzip类型的压缩数据。

使用到的第三方库：https://github.com/brimworks/lua-zlib。编译命令如下：

```shell
git clone git@github.com:brimworks/lua-zlib.git
cd lua-zlib
mkdir build
cmake -DLUA_INCLUDE_DIR=/usr/local/openresty/luajit/include/luajit-2.1 -DLUA_LIBRARIES=/usr/local/openresty/luajit/lib -DUSE_LUAJIT=ON -DUSE_LUA=OFF  ../
sudo cp zlib.so /usr/local/openresty/lualib
```

```lua
location ~ /css {
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Accept-Encoding "gzip";
    proxy_set_header Host $host;

    proxy_pass http://upstream;

    header_filter_by_lua_block {
        ngx.header.content_length = ni
        ngx.header.content_encoding = nil
    }

    body_filter_by_lua_block {
        -- ngx.arg[1] 代表输入的 chunk
        local zlib = require 'zlib'
        local chunk, eof = ngx.arg[1], ngx.arg[2]
        local buffers = ngx.ctx.buffers
        if not buffers then
           buffers = {}
           ngx.ctx.buffers = buffers
           ngx.ctx.buffers_cnt = 0
        end

        if chunk ~= "" then
           local cnt = ngx.ctx.buffers_cnt + 1
           ngx.ctx.buffers_cnt = cnt
           buffers[cnt] = chunk
           ngx.arg[1] = nil
        end

        if eof then
            local body = table.concat(buffers)
            ngx.ctx.buffers = nil

            local inflate = zlib.inflate()

            local body, eos, bytes_in, bytes_out = inflate(body)
            if not eos then
                ngx.log(ngx.ERR, "not end of gzip stream found") -- just for debug
            end

            body = ngx.re.gsub(body, "<img name='xxx'>",  "<img name='xxx', attr='hide'>", "jo")
            ngx.arg[1] = body
        end
    }
}
```
