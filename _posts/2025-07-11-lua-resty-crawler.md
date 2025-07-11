---
layout: post
title: "crawler by lua-resty-http"
description: "crawler by lua-resty-http"
date: 2025-07-11
modified: 2025-07-11
tags: [crawler, spider, lua-resty-http]
---

# The code of this crawler

The crawler can not work in the real world.
Just for fun!

```lua
local http = require "resty.http"
local re_match = ngx.re.match
local re_gsub = ngx.re.gsub
local sleep = ngx.sleep
local random = math.random
local str_find = string.find
local ngx_log = ngx.log
local LOG_WARN = ngx.WARN
local LOG_ERR = ngx.ERR
local is_exiting = ngx.worker.exiting
local str_cat = table.concat
local thread_spawn = ngx.thread.spawn
local thread_wait = ngx.thread.wait

local _M = {}

-- A set of visited URLs to avoid duplicate crawling
local visited_urls = {}

-- Define a function to fetch web page content
local function fetch_page(url, refer)
    -- Random delay of 0.1 to 0.3 seconds
    local delay = random(100, 300) / 1000
    sleep(delay)

    local httpc = http.new()
    local headers = {
        ["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    }

    if refer then
        headers["refer"] = refer
    end

    local res, err = httpc:request_uri(url, { 
        method = "GET",
        ssl_verify = false,
        headers = headers
    })

    if not res then
        ngx_log(LOG_ERR, "Request ", url, " failed: ", err)
        return nil
    end

    if res.status >= 300 and res.status < 400 then
        local location = res.headers["Location"]
        if re_match(location, "^//.+", "jo") then
            if re_match(url, "^https://") then
                location = "https:" .. location
            else
                location = "http:" .. location
            end
        end

        if location then
            ngx_log(LOG_WARN, "Redirected to: ", location)
            return fetch_page(location)
        end
    elseif res.status >= 400 then
        ngx_log(LOG_ERR, "Request ", url, " failed with status code: ", res.status)
        return nil
    end

    if res.status == 200 and res.body == nil then
       ngx_log(LOG_ERR, "=== Got Response 200, but the body is empty: ", url)
    end

    return url, res.body, res.headers
end

-- Extract href links from HTML using regular expression matching
local function extract_href_links(url, html)
    local m, err = re_match(url, "((https?)://[^/]+)", "jo")
    if not m then
        ngx_log(LOG_ERR, "Cannot match host from ", url, ":", err)
        return nil, nil
    end

    local host = m[1]
    local scheme = m[2]
    -- ngx_log(LOG_WARN, "host is ", host)
    if not html then
        return {}
    end
    local links = {}
    local iterator, err = ngx.re.gmatch(html, [=[\bhref=["']([^"']+)["']]=], "jo")
    if not iterator then
        ngx_log(LOG_ERR, "Regular expression matching failed: ", err)
        return links
    end

    while true do
        local m, err = iterator()
        if err then
            ngx_log(LOG_ERR, "Error during matching: ", err)
            break
        end
        if not m then
            break
        end

        local url = m[1]
        url = re_gsub(url, "&amp;", "&", "jo")
        if re_match(url, "^https://", "jo") ~= nil then
            table.insert(links, url)
        elseif re_match(url, "^//.+", "jo") then
            table.insert(links, scheme .. ":" .. url)
        elseif re_match(url, "^/", "jo") then
            table.insert(links, host .. url)
        end
    end
    return links
end

local function worker_helper(ctx, task)
    local url, refer, depth = task.url, task.refer, task.depth
    if not visited_urls[url] then
        visited_urls[url] = true
        ngx_log(LOG_WARN, "Crawling link: ", url)

        local url, body, headers = fetch_page(url, refer)
        if depth <= 1 then
            return
        end

        if not url or not body then
            return
        end

        local content_type = headers and headers["content-type"]
        if type(content_type) == "table" then
            content_type = str_cat(content_type, " ")
        end

        if content_type and str_find(content_type, "text/html", nil, true) ~= nil and body then
            local links = extract_href_links(url, body)
            depth = depth -1
            if links then
                for _, link in ipairs(links) do
                    table.insert(ctx.url_queue, {url = link, refer = url, depth = depth})
                end
            end
        end
    end
end

-- Worker function for concurrent crawling
local function worker(ctx)
    local is_idle = false
    while not is_exiting() do
        if #ctx.url_queue == 0 then
            sleep(1)
            if is_idle == false then
                is_idle = true
                ctx.idle_thread = ctx.idle_thread + 1
            end

            if ctx.idle_thread >= ctx.concurrency then
                break
            end

            goto continue
        end

        if is_idle == true then
            is_idle = false
            ctx.idle_thread = ctx.idle_thread - 1
        end

        local task = table.remove(ctx.url_queue, 1)
        local ok, err = pcall(worker_helper, ctx, task)
        if not ok then
            ngx_log(LOG_ERR, "worker_helper failed: ", err)
        end

        ::continue::
    end
end

-- Function to start concurrent crawling
local function start_concurrent_crawl(start_url, ctx)
    table.insert(ctx.url_queue, {url = start_url, depth = ctx.max_depth})
    local threads = {}
    for i = 1, ctx.concurrency do
        threads[i] = thread_spawn(worker, ctx)
    end

    for _, thread in ipairs(threads) do
        thread_wait(thread)
    end
end

function _M.run(url)
    -- Example usage
    local start_url = url or "https://sina.com.cn/"
    local ctx = {
        url_queue = {},
        concurrency = 4,
        idle_thread = 0,
        max_depth = 8,
    }

    start_concurrent_crawl(start_url, ctx)
end

return _M
```
