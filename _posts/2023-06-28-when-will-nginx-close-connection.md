---
layout: post
title: "When Will Nginx Send 'Connection: close' Header"
description: "Nginx 什么时候会发送 Connection: close 的头部"
date: 2023-06-28
tags: [Nginx, connection close]
---

长链接对于降低请求延时，减少服务器开销是非常有好处的。然而有时候由于配置不当，导致
Nginx 服务器发送响应头包含 'Connection: close'。客户端在收到带有 'Connection: close'
的响应后会关闭连接。

那么什么情况下会导致 Nginx 发送 'Connection: close' 呢？

对于 Nginx，我们可以通过搜索 r->keepalive 来查找将连接复用关闭的代码。
以下是我们找到的一些情况

# ngx_http_special_response_handler

在该函数中，对于 HTTP 响应码是 NGX_HTTP_BAD_REQUEST(400)，
NGX_HTTP_REQUEST_ENTITY_TOO_LARGE(413), NGX_HTTP_REQUEST_URI_TOO_LARGE(414),
NGX_HTTP_TO_HTTPS(497), NGX_HTTPS_CERT_ERROR(495), NGX_HTTPS_NO_CERT(496),
NGX_HTTP_INTERNAL_SERVER_ERROR(500), NGX_HTTP_NOT_IMPLEMENTED(501) 这些错误会关闭连接复用。

这些错误码为什么要关闭连接复用呢？其实也很好理解，比如含有错误的请求头的请求，那么这个请问无法正常解析，
就不应该再复用这个连接，这时候的错误码就是  NGX_HTTP_BAD_REQUEST。
其它的状态码也是一样的道理，都表示该状态下连接已经不具备复用的基础。

另外就是如果请求携带请求体，但是调用 ngx_http_discard_request_body() 设置丢弃请求体失败也不能复用连接了。

```C
ngx_int_t
ngx_http_special_response_handler(ngx_http_request_t *r, ngx_int_t error)
{
    ngx_uint_t                 i, err;
    ngx_http_err_page_t       *err_page;
    ngx_http_core_loc_conf_t  *clcf;

    ngx_log_debug3(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http special response: %i, \"%V?%V\"",
                   error, &r->uri, &r->args);

    r->err_status = error;

    if (r->keepalive) {
        switch (error) {
            case NGX_HTTP_BAD_REQUEST:
            case NGX_HTTP_REQUEST_ENTITY_TOO_LARGE:
            case NGX_HTTP_REQUEST_URI_TOO_LARGE:
            case NGX_HTTP_TO_HTTPS:
            case NGX_HTTPS_CERT_ERROR:
            case NGX_HTTPS_NO_CERT:
            case NGX_HTTP_INTERNAL_SERVER_ERROR:
            case NGX_HTTP_NOT_IMPLEMENTED:
                r->keepalive = 0;
        }
    }
    ...
    if (ngx_http_discard_request_body(r) != NGX_OK) {
        r->keepalive = 0;
    }
}
```

## 怎么选择响应码

如果我们没有正确选择响应码，那么就会导致连接被关闭了。比如在限流的情况下，我们希望客户的延迟重发请求而不是关闭连接重新建立连接再发请求。这时候我们就不应该使用 400 的状态码。可以使用默认的 NGX_HTTP_SERVICE_UNAVAILABLE(503) 或者使用 NGX_HTTP_TOO_MANY_REQUESTS(429).

# ngx_http_chunked_header_filter()

如果请求头没有 Content-Length, 也不是 chunked 编码，那么也不能复用请求。

```C
static ngx_int_t
ngx_http_chunked_header_filter(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t       *clcf;
    ngx_http_chunked_filter_ctx_t  *ctx;

    if (r->headers_out.status == NGX_HTTP_NOT_MODIFIED
        || r->headers_out.status == NGX_HTTP_NO_CONTENT
        || r->headers_out.status < NGX_HTTP_OK
        || r != r->main
        || r->method == NGX_HTTP_HEAD)
    {
        return ngx_http_next_header_filter(r);
    }

    if (r->headers_out.content_length_n == -1
        || r->expect_trailers)
    {
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

        if (r->http_version >= NGX_HTTP_VERSION_11
            && clcf->chunked_transfer_encoding)
        {
            if (r->expect_trailers) {
                ngx_http_clear_content_length(r);
            }

            r->chunked = 1;

            ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_chunked_filter_ctx_t));
            if (ctx == NULL) {
                return NGX_ERROR;
            }

            ngx_http_set_ctx(r, ctx, ngx_http_chunked_filter_module);

        } else if (r->headers_out.content_length_n == -1) {
            r->keepalive = 0;
        }
    }

    return ngx_http_next_header_filter(r);
}
```

# ngx_http_handler()

http 1.0 以及以下， 如果没有 'Connection: keepalive' 的头部，那么默认也是不复用连接。
如果客户端发送了 'Connection: close' 的头部，那么就直接不复用链接。

```C
void
ngx_http_handler(ngx_http_request_t *r)
{
    ngx_http_core_main_conf_t  *cmcf;

    r->connection->log->action = NULL;

    if (!r->internal) {
        switch (r->headers_in.connection_type) {
        case 0:
            r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
            break;

        case NGX_HTTP_CONNECTION_CLOSE:
            r->keepalive = 0;
            break;

        case NGX_HTTP_CONNECTION_KEEP_ALIVE:
            r->keepalive = 1;
            break;
        }

        r->lingering_close = (r->headers_in.content_length_n > 0
                              || r->headers_in.chunked);
        r->phase_handler = 0;

    } else {
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
        r->phase_handler = cmcf->phase_engine.server_rewrite_index;
    }

    r->valid_location = 1;
#if (NGX_HTTP_GZIP)
    r->gzip_tested = 0;
    r->gzip_ok = 0;
    r->gzip_vary = 0;
#endif

    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```

# ngx_http_update_location_config(ngx_http_request_t *r)

1. 如果 keepalive_timeout 配置为 0，那么也不复用连接。
1. 如果 连接复用的次数超过配置的 keepalive_requests 的数量，那么也不再复用连接
1. 对于上古时代的 IE6.0 的浏览器的 POST 请求也不复用连接
1. safari 浏览器如果配置的禁用复用连接，那么也不复用连接


```C
void
ngx_http_update_location_config(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    if (r->method & clcf->limit_except) {
        r->loc_conf = clcf->limit_except_loc_conf;
        clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    }

    if (r == r->main) {
        ngx_set_connection_log(r->connection, clcf->error_log);
    }

    if ((ngx_io.flags & NGX_IO_SENDFILE) && clcf->sendfile) {
        r->connection->sendfile = 1;

    } else {
        r->connection->sendfile = 0;
    }

    if (clcf->client_body_in_file_only) {
        r->request_body_in_file_only = 1;
        r->request_body_in_persistent_file = 1;
        r->request_body_in_clean_file =
            clcf->client_body_in_file_only == NGX_HTTP_REQUEST_BODY_FILE_CLEAN;
        r->request_body_file_log_level = NGX_LOG_NOTICE;

    } else {
        r->request_body_file_log_level = NGX_LOG_NOTICE;
    }

    r->request_body_in_single_buf = clcf->client_body_in_single_buffer;

    if (r->keepalive) {
        if (clcf->keepalive_timeout == 0) {
            r->keepalive = 0;

        } else if (r->connection->requests >= clcf->keepalive_requests) {
            r->keepalive = 0;

        } else if (r->headers_in.msie6
                   && r->method == NGX_HTTP_POST
                   && (clcf->keepalive_disable
                       & NGX_HTTP_KEEPALIVE_DISABLE_MSIE6))
        {
            /*
             * MSIE may wait for some time if an response for
             * a POST request was sent over a keepalive connection
             */
            r->keepalive = 0;

        } else if (r->headers_in.safari
                   && (clcf->keepalive_disable
                       & NGX_HTTP_KEEPALIVE_DISABLE_SAFARI))
        {
            /*
             * Safari may send a POST request to a closed keepalive
             * connection and may stall for some time, see
             *     https://bugs.webkit.org/show_bug.cgi?id=5760
             */
            r->keepalive = 0;
        }
    }

    if (!clcf->tcp_nopush) {
        /* disable TCP_NOPUSH/TCP_CORK use */
        r->connection->tcp_nopush = NGX_TCP_NOPUSH_DISABLED;
    }

    if (clcf->handler) {
        r->content_handler = clcf->handler;
    }
}
```

# ngx_http_finalize_connection(）

如果还在读取请求体，那么也不复用连接。这个比较复杂，需要看看具体的调用栈。

```C
static void
ngx_http_finalize_connection(ngx_http_request_t *r)
{
    ngx_http_core_loc_conf_t  *clcf;

    ...
    if (r->reading_body) {
        r->keepalive = 0;
        r->lingering_close = 1;
    }
}
```


# ngx_http_set_keepalive()

这里其实不是不复用连接，而是为了防止递归调用。

# ngx_http_upstream_upgrade()

比如是 websocket 的连接转换，那么也不能复用该连接做下一次的 HTTP 请求连接。

```C
static void
ngx_http_upstream_upgrade(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ngx_connection_t          *c;
    ngx_http_core_loc_conf_t  *clcf;

    c = r->connection;
    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    /* TODO: prevent upgrade if not requested or not possible */

    if (r != r->main) {
        ngx_log_error(NGX_LOG_ERR, c->log, 0,
                      "connection upgrade in subrequest");
        ngx_http_upstream_finalize_request(r, u, NGX_ERROR);
        return;
    }

    r->keepalive = 0;
    c->log->action = "proxying upgraded connection";
    ...
}
```

# ngx_http_upstream_finalize_request()

这里的情况不是特别清楚，后续再分析。
