---
layout: post
title: "An exmaple of ssl_certificate_by_lua"
description: "An example of ssl_certificate_by_lua"
date: 2023-10-27
tags: [OpenResty]
---


# 1.  the lua code

save the following code to file /usr/local/openresty/site/lualib/dynamic_cert.lua

```lua
local lru = require "resty.lrucache"
local ssl = require "ngx.ssl"
local io_open = io.open

local cert_cache = lru.new(10000)
local priv_cache = lru.new(10000)

local _M = {}

local cert_path = "/usr/local/openresty/nginx/conf/"
local function get_pem_cert(host)
    local filename = cert_path .. host .. ".crt";
    -- local filename = cert_path .. "ssl_cert.pem"
    local file, err = io_open(filename, "r")
    
    if err ~= nil then
        return nil, err
    end

    local contents = file:read("*a")
    io.close(file)
    return contents;
end

local function get_pem_priv_key(host)
    local filename = cert_path .. host .. ".key";
    -- local filename = cert_path .. "ssl_key.pem"
    local file, err = io_open(filename, "r")
    
    if err ~= nil then
        return nil, err
    end

    local contents = file:read("*a")
    io.close(file)
    return contents;
end

local function get_cert(host)
    local cert_data, err

    local cert = cert_cache:get(host)
    if cert ~= nil then
        return cert
    end

    cert_data, err = get_pem_cert(host)
    if not cert_data then
        return nil, err
    end
    
    cert, err = ssl.parse_pem_cert(cert_data)
    if not cert then
        return nil, err
    end

    cert_cache:set(host, cert)
    return cert
end

local function get_pkey(host)
    local pkey_data, err
    local pkey = priv_cache:get(host)
    if pkey ~= nil then
        return pkey 
    end

    local pkey_data, err = get_pem_priv_key(host)
    if not pkey_data then
        return nil, err
    end
    
    pkey, err = ssl.parse_pem_priv_key(pkey_data)
    if not pkey then
        return nil, err
    end

    priv_cache:set(host, pkey)
    return pkey
end

function _M.set_cert_priv()
    local host, err = ssl.server_name()
    if not host or host == "" then
        ngx.log(ngx.ERR, "not server name found: ", err)
        return ngx.exit(ngx.ERROR)
    end

    local ok, err = ssl.clear_certs()
    if not ok then
        ngx.log(ngx.ERR, "failed to clear existing (fallback) certificates:", err)
        return ngx.exit(ngx.ERROR)
    end

    local cert, err = get_cert(host)
    if err ~= nil then
        ngx.log(ngx.ERR, "failed to get certificate: ", err)
        return ngx.exit(ngx.ERROR)
    end

    local pkey, err = get_pkey(host)
    if err ~= nil then
        ngx.log(ngx.ERR, "failed to get private key: ",  err)
        return ngx.exit(ngx.ERROR)
    end

    local ok, err = ssl.set_cert(cert)
    if not ok then
        ngx.log(ngx.ERR, "failed to set cert: ", err)
        return ngx.exit(ngx.ERROR)
    end

    ok, err = ssl.set_priv_key(pkey)
    if not ok then
        ngx.log(ngx.ERR, "failed to set private key: ", err)
        return ngx.exit(ngx.ERROR)
    end
end

return _M
```

# config of nginx

```conf
    server {
        listen 4443 ssl reuseport http2;
        server_name  localhost;
        ssl_certificate ssl_cert.pem;
        ssl_certificate_key ssl_key.pem;

        ssl_certificate_by_lua_block {
            local dynamic_cert = require "dynamic_cert"
            dynamic_cert.set_cert_priv()
        }

        location / {
            access_log off;

            content_by_lua_block {
               ngx.say("Hello world.")
            }
        }
    }
```

# test the config

```shell
for i in {1..10000}; do
    curl -k --resolve test$i.com:4443:127.0.0.1 https://test$i.com:4443/http_504
done
```

模拟对数曲线的发送，6 个小时发送 10000 个不同域名的请求。
```lua
local log = math.log
local last = 0
local total_req = 10000
local total_sec = 6 * 3600

for i = 1, total_sec do
    local v = math.floor(1000 * log(i)) -- 10000 / math.log(6 * 3600) = 1001.958970696
    if v ~= last then
        for j = last + 1, v do
	    local cmd = "curl -k --resolve test" .. j ..".com:4443:127.0.0.1 https://test" .. j .. ".com:4443/http_504"
            os.execute(cmd)
        end
        last = v
    end

    os.execute("sleep 1")
end
```
