---
layout: post
title: "openresty 怎么做日志切割"
description: "openresty 怎么做日志切割"
date: 2023-07-23
tags: [openresty, logrotate]
---

# logrotate in nginx

nginx 官方给出的日志切割建议如下： 

```shell
mv access.log access.log.0
kill -USR1 `cat master.nginx.pid`
sleep 1
gzip access.log.0    # do something with access.log.0
```

可以参考官方文档 https://www.nginx.com/resources/wiki/start/topics/examples/logrotation/

那么在 OpenResty 中我们可以怎么做呢？

# logrotate in OpenResty

OpenResty 有特权进程，因此我们可以在特权进程中来进行日志轮转。

## OpenResty 初始化配置

http 模块中需要先启用特权进程，配置如下

```config
http {
    init_by_lua_block {
        local process = require "ngx.process"
        local ok, err = process.enable_privileged_agent()
        if not ok then
            ngx.log(ngx.ERR, "enables privileged agent failed error:", err)
        end
    }
}
```

# 特权进程切割日志

```config
http {
    init_worker_by_lua_block {
        local process = require "ngx.process"

        local function log_rotate_handler(premature)
            local process = require "ngx.process"
            local shell = require "resty.shell"

            if premature then
                return
            end

            if process.type() ~= "privileged agent" then
                return
            end

            local old_file = "/usr/local/openresty/nginx/logs/access.log"
            local t = ngx.localtime()
            t = ngx.re.gsub(t, "[-: ]", "", jo)
            local new_file = old_file .. "." .. t
            os.rename(old_file, new_file)

            ngx.sleep(1)
    
            local resty_signal = require "resty.signal"
            local pid = process.get_master_pid()
            local ok, err = resty_signal.kill(pid, "HUP")
            if not ok then
                ngx.log(ngx.ERR, "failed to send signal(HUP) to pid ", pid, " : ", err)
                return
            end

            local ok, stdout, stderr, reason, status =
            shell.run("gzip " .. new_file , nil, 1000 * 60, 4096)
            if not ok then
                ngx.log(ngx.ERR, "gzip failed: ", stderr, ", reason: ", reason)
            end
        end

        if process.type() == "privileged agent" then
            ngx.timer.every(3600, log_rotate_handler)
        end
    }
}
```
