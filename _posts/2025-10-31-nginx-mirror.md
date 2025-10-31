---
layout: post
title: "Nginx Mirror Request Keepalive 为什么不生效"
description: "Why Nginx Mirror Request keepalive Can't Work"
date: 2025-10-31
tags: [Nginx, Mirror]
---

在创建子请求的时候会设置 header_only，所以不会处理上游服务器的响应体

```C
static ngx_int_t
ngx_http_mirror_handler_internal(ngx_http_request_t *r)
{
    ngx_str_t                   *name;
    ngx_uint_t                   i;
    ngx_http_request_t          *sr;
    ngx_http_mirror_loc_conf_t  *mlcf;
    ngx_uint_t                   nelts;

    mlcf = ngx_http_get_module_loc_conf(r, ngx_http_mirror_module);

    name = mlcf->mirror->elts;
    nelts = mlcf->mirror->nelts;

    for (i = 0; i < nelts; i++) {
        if (ngx_http_subrequest(r, &name[i], &r->args, &sr, NULL,
                                NGX_HTTP_SUBREQUEST_BACKGROUND)
            != NGX_OK)
        {
            return NGX_HTTP_INTERNAL_SERVER_ERROR;
        }

        sr->header_only = 1;
        sr->method = r->method;
        sr->method_name = r->method_name;
    }

    return NGX_DECLINED;
}
```

没有处理上游的响应体，因此 u->request_body_sent 是 false，所以下面直接 invalid 跳到 original_free_peer

```C
static void
ngx_http_upstream_free_keepalive_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_keepalive_peer_data_t  *kp = data;
    ngx_http_upstream_keepalive_cache_t      *item;

    ngx_queue_t          *q;
    ngx_connection_t     *c;
    ngx_http_upstream_t  *u;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                   "free keepalive peer");

    /* cache valid connections */

    u = kp->upstream;
    c = pc->connection;

    if (state & NGX_PEER_FAILED
        || c == NULL
        || c->read->eof
        || c->read->error
        || c->read->timedout
        || c->write->error
        || c->write->timedout)
    {
        goto invalid;
    }

    if (c->requests >= kp->conf->requests) {
        goto invalid;
    }

    if (ngx_current_msec - c->start_time > kp->conf->time) {
        goto invalid;
    }

    if (!u->keepalive) {
        goto invalid;
    }

    if (!u->request_body_sent) {
        goto invalid;
    }

    if (ngx_terminate || ngx_exiting) {
        goto invalid;
    }

    if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
        goto invalid;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                   "free keepalive peer: saving connection %p", c);

    if (ngx_queue_empty(&kp->conf->free)) {

        q = ngx_queue_last(&kp->conf->cache);
        ngx_queue_remove(q);

        item = ngx_queue_data(q, ngx_http_upstream_keepalive_cache_t, queue);

        ngx_http_upstream_keepalive_close(item->connection);

    } else {
        q = ngx_queue_head(&kp->conf->free);
        ngx_queue_remove(q);

        item = ngx_queue_data(q, ngx_http_upstream_keepalive_cache_t, queue);
    }

    ngx_queue_insert_head(&kp->conf->cache, q);

    item->connection = c;

    pc->connection = NULL;

    c->read->delayed = 0;
    ngx_add_timer(c->read, kp->conf->timeout);

    if (c->write->timer_set) {
        ngx_del_timer(c->write);
    }
#if (NGX_SSL)
    if (c->async->timer_set) {
        ngx_del_timer(c->async);
    }
#endif

    c->write->handler = ngx_http_upstream_keepalive_dummy_handler;
    c->read->handler = ngx_http_upstream_keepalive_close_handler;

    c->data = item;
    c->idle = 1;
    c->log = ngx_cycle->log;
    c->read->log = ngx_cycle->log;
    c->write->log = ngx_cycle->log;
#if (NGX_SSL)
    c->async->log = ngx_cycle->log;
#endif
    c->pool->log = ngx_cycle->log;

    item->socklen = pc->socklen;
    ngx_memcpy(&item->sockaddr, pc->sockaddr, pc->socklen);

    if (c->read->ready) {
        ngx_http_upstream_keepalive_close_handler(c->read);
    }

invalid:

    kp->original_free_peer(pc, kp->data, state);
}
```
