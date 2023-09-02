---
layout: post
title: "OpenResty 拦截并使用 HTTP 代理流量"
description: "stream 模块实现透明代理"
date: 2023-09-02
tags: [iptables, OpenResty, transparent, HTTP proxy]
---

我需要用到 HTTP 用代理，因为一些网络限制的客观原因没有办法直接连接 HTTP 代理。
因此我们通过将流量导入 OpenResty 节点，然后用 OpenResty 的 stream 模块的 content_by_lua 代理流量到 HTTP proxy 节点。

需要注意，网络的很多透明代理是指 OpenResty 网关代理客户端的流量，保持客户端 IP 地址不变。
我这里的透明代理是指对客户端透明，客户端不知道自己的 流量被 OpenResty 代理了。

# 基本网络拓扑如下

Client --------> OpenResty Gateway ---------> HTTP Proxy ------> Original Server

# 系统配置

OpenResty Gateway 节点要把从指定接口收上来的流量重定向到 OpenResty 上服务进程。
因此需要使用 iptables 的 redirect 功能。

```shell
# 防止循环
iptables -t mangle -A OUTPUT -p tcp  -j MARK --set-mark 0x80
iptables -t mangle -I INPUT -p tcp --dport 8880 -m mark --mark 0x80 -j DROP
iptables -t mangle -I INPUT -p tcp --dport 8843 -m mark --mark 0x80 -j DROP

# 报文重定向
iptables -t nat -I PREROUTING -i wg0 -p tcp --dport 80 -j REDIRECT --to-port 8880
iptables -t nat -I PREROUTING -i wg0 -p tcp --dport 443 -j REDIRECT --to-port 8843
```

**这些配置会在系统重启后失效，因此如果想要确保系统重启还生效需要另外配置。**

# OpenResty 配置

```config
#user  root;
worker_processes  1;
worker_cpu_affinity auto;

error_log  logs/error.log  error;

#pid        logs/nginx.pid;

events {
    worker_connections  10240;
}

stream {
    init_by_lua_block {
        local cfg_file = ngx.config.prefix() .. "conf/proxy-config.json";
        local cjson = require "cjson"
        local file = io.open(cfg_file, "r")
        local content = file:read("*a")
        file:close()

        ngx.proxy_cfg = cjson.decode(content)
        ngx.local_addrs = {}

        local ngx_re = require "ngx.re"
        local cmd = "/sbin/ip -4 addr | grep inet"
        local handle = io.popen(cmd)
        local res = handle:read("*a")
        handle:close()

        if not res then
          ngx.log("Error executing command: ", err)
        else
            local lines = ngx_re.split(res, "\r?\n")

            for i, line in ipairs(lines) do
              local m, err = ngx.re.match(line, "inet ([^/]+)", "jo")
              if m ~= nil then
                  ngx.local_addrs[m[1]] = true
              else
                  ngx.log(ngx.ERR, "Failed to match line: ", line)
              end
            end
        end
    }

    server  {
        listen 8880 reuseport;
        listen 8843 reuseport;

        content_by_lua_block {
            local utils = require "resty.core.utils"
            local addr = utils.get_original_addr()

            local m, err = ngx.re.match(addr, [[([0-9.]+):(\d+)]], "jo")
            if m == nil then
                ngx.log(ngx.ERR, "Invalid address ", addr, ", match error: ", err)
                ngx.exit(1)
            end

            local host = m[1]
            local port = tonumber(m[2])

            if ngx.local_addrs[host] ~= nil or host == ngx.var.remote_addr then
                ngx.log(ngx.ERR, "loop detected, invalid address ", addr)
                ngx.exit(1)
            end

            local upstream_sock = ngx.socket.tcp()
            upstream_sock:settimeout(10000)

            local proxy_host = ngx.proxy_cfg["proxy_host"]
            local proxy_port = ngx.proxy_cfg["proxy_port"]
            local ok, err = upstream_sock:connect(proxy_host, proxy_port)
            if not ok then
                ngx.log(ngx.ERR, "failed to connect to ", proxy_host, ":", proxy_port, ", error: ", err)
                upstream_sock:close()
                ngx.exit(1)
            end

            local user = ngx.proxy_cfg["user"]
            local passwd = ngx.proxy_cfg["password"]
            local b64_user_pass = ngx.encode_base64(user .. ":" .. passwd)

            local template = "CONNECT %s:%d HTTP/1.1\r\nHost: %s:%d\r\nProxy-Authorization: Basic %s\r\nUser-Agent: curl/7.61.1\r\nProxy-Connection: Keep-Alive\r\n\r\n"

            local req = string.format(template, host, port, host, port, b64_user_pass)
            local bytes, err = upstream_sock:send(req)
            if err ~= nil then
                ngx.log(ngx.ERR, "failed to send req to ", host, ":", port, ", error: ", err)
                upstream_sock:close()
                ngx.exit(1)
            end

            local reader = upstream_sock:receiveuntil("\r\n\r\n")
            local data, err, partial = reader()
            if not data then
                ngx.log(ngx.ERR, "failed to receive heaer from ", host, ":", port, ", error: ", err)
                upstream_sock:close()
                ngx.exit(1)
            end

            local m, err = ngx.re.match(data, [[^HTTP/1\.1 (\d+) +([^\r\n]+)]], "jo")
            if m == nil then
                ngx.log(ngx.ERR, "Invalid response: ", data)
                upstream_sock:close()
                ngx.exit(1)
            end

            local rep_code = tonumber(m[1])
            local rep_code_name = m[2]
            -- ngx.log(ngx.ERR, "rep_code: ", rep_code, ", rep_code_name: ", rep_code_name, data)
            if rep_code ~= 200 then
                ngx.log(ngx.ERR, "upstream error: ", rep_code)
                upstream_sock:close()
                ngx.exit(1)
            end

            local function downstream_recv_handler(downstream_sock, upstream_sock)
                while true do
                    local data, err = downstream_sock:receiveany(10 * 1024)
                    if not data then
                        if err ~= "closed" then
                            ngx.log(ngx.ERR, "failed to receive from downstream: ", err)
                        end
                            return
                    end

                    local bytes, err = upstream_sock:send(data)
                    if err ~= nil then
                        ngx.log(ngx.ERR, "failed to send to ", host, ":", port, ", error: ", err)
                        return
                    end
                end
            end

            local function downstream_send_handler(downstream_sock, upstream_sock)
                while true do
                    local data, err = upstream_sock:receiveany(10 * 1024)
                    if not data then
                        if err ~= "closed" then
                            ngx.log(ngx.ERR, "failed to receive from upstream: ", err)
                        end
                        return
                    end

                    local bytes, err = downstream_sock:send(data)
                    if err ~= nil then
                        ngx.log(ngx.ERR, "failed to send to downstream: ", err)
                        return
                    end
                end
            end

            local downstream_sock, err = ngx.req.socket()
            if downstream_sock == nil then
                ngx.log(ngx.ERR, "failed to get request socket: ", err)
                return
            end

            local recv_thr = ngx.thread.spawn(downstream_recv_handler, downstream_sock, upstream_sock)
            local send_thr = ngx.thread.spawn(downstream_send_handler, downstream_sock, upstream_sock)
            local ok, res1, res2 = ngx.thread.wait(recv_thr, send_thr)
            upstream_sock:close()
            -- ngx.log(ngx.ERR, "res1: ", res1, ", res2: ", res2)
        }
    }
}
```

# 获取原始目的 IP/端口的补丁


## stream-lua-nginx-module 的补丁

```path
diff --git a/src/ngx_stream_lua_api.c b/src/ngx_stream_lua_api.c
index cf01b36..fc918f0 100644
--- a/src/ngx_stream_lua_api.c
+++ b/src/ngx_stream_lua_api.c
@@ -16,6 +16,7 @@
 #define DDEBUG 0
 #endif
 #include "ddebug.h"
+#include <linux/netfilter_ipv4.h>
 
 
 #include "ngx_stream_lua_common.h"
@@ -218,4 +219,43 @@ ngx_stream_lua_shared_memory_init(ngx_shm_zone_t *shm_zone, void *data)
     return NGX_OK;
 }
 
+
+int
+ngx_stream_lua_ffi_req_dst_addr(ngx_stream_lua_request_t *r, char *buf,
+    int *buf_size, u_char *errbuf, size_t *errbuf_size)
+{
+    int                fd;
+    struct sockaddr_in addr;
+    socklen_t          addr_sz = sizeof(addr);
+
+    if (r->session->connection == NULL) {
+        *errbuf_size = ngx_snprintf(errbuf, *errbuf_size, "no connection")
+                       - errbuf;
+        return NGX_ERROR;
+    }
+
+    fd = r->session->connection->fd;
+
+    if (fd < 0) {
+        *errbuf_size = ngx_snprintf(errbuf, *errbuf_size, "invalid fd")
+                       - errbuf;
+        return NGX_ERROR;
+    }
+
+    memset(&addr, 0, addr_sz);
+    addr.sin_family = AF_INET;
+    if (getsockopt(fd, SOL_IP, SO_ORIGINAL_DST, &addr, &addr_sz) != 0) {
+        *errbuf_size
+            = ngx_snprintf(errbuf, *errbuf_size, "failed to get origin addr")
+              - errbuf;
+        return NGX_ERROR;
+    }
+
+    *buf_size = ngx_sock_ntop((struct sockaddr *)&addr, addr_sz,
+                              (u_char *) buf, *buf_size, 1);
+
+    return NGX_OK;
+}
+
+
 /* vi:set ft=c ts=4 sw=4 et fdm=marker: */
```

## lua-resty-core 的补丁

```shell
diff --git a/lib/resty/core/utils.lua b/lib/resty/core/utils.lua
index fda074a..42bc2fe 100644
--- a/lib/resty/core/utils.lua
+++ b/lib/resty/core/utils.lua
@@ -11,8 +11,10 @@ local ffi_copy = ffi.copy
 local byte = string.byte
 local str_find = string.find
 local get_string_buf = base.get_string_buf
+local get_size_ptr = base.get_size_ptr
 local subsystem = ngx.config.subsystem
-
+local ffi_new = ffi.new
+local get_request = base.get_request
 
 local _M = {
     version = base.version
@@ -42,5 +44,29 @@ if subsystem == "http" then
     end
 end
 
+if subsystem == "stream" then
+ffi.cdef[[
+int ngx_stream_lua_ffi_req_dst_addr(ngx_stream_lua_request_t *r, char *buf,
+    size_t *buf_size, char *errbuf, size_t *errbuf_size);
+]]
+
+    local ERR_BUF_SIZE = 256
+
+    local buf = get_string_buf(128, 1)
+    local buf_size = ffi_new("size_t[1]")    
+
+    function _M.get_original_addr()
+	local r = get_request()
+	local err = get_string_buf(ERR_BUF_SIZE)
+	local errlen = get_size_ptr()
+	buf_size[0] = 128
+        local rc = C.ngx_stream_lua_ffi_req_dst_addr(r, buf, buf_size, err, errlen)
+        if tonumber(rc) ~= 0 then
+            return nil, tostring(err, errlen)
+	end
+
+        return ffi_str(buf, buf_size[0])
+    end
+end
 
 return _M
```
