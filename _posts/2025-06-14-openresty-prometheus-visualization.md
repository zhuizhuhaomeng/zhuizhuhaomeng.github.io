---
layout: post
title: "prometheus 监控 nginx 请求"
description: ""
date: 2025-06-14
modified: 2025-06-14
tags: [prometheus, nginx, openresty]
---

# nginx/OpenResty 吐出数据

nginx 直接输出 prometheus 的格式可比再通过 [nginx-prometheus-exporter](https://github.com/nginx/nginx-prometheus-exporter) 中转一次好多了。

nginx/OpenResty 吐出 Prometheus 格式的数据使用 https://github.com/knyar/nginx-lua-prometheus 这个库。但是也可以使用 https://github.com/bjne/lua-resty-p8s 这个库，因为它使了 string.buffer, 可以减少临时 GC 对象，提升整体性能。lua-resty-p8s 和 nginx-lua-promethus 具体是否有功能的差异则没有深入分析。

# 容器部署全套例子

## nginx 准备

```shell
mkdir -p site/lualib
git clone git@github.com:knyar/nginx-lua-prometheus.git .
cp nginx-lua-prometheus/*.lua site/lualib
```

### nginx 配置文件

nginx-server.conf

```nginx
init_worker_by_lua_block {
  prometheus = require("prometheus").init("prometheus_metrics")

  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_connections = prometheus:gauge(
    "nginx_http_connections", "Number of HTTP connections", {"state"})
}

log_by_lua_block {
  metric_requests:inc(1, {ngx.var.server_name, ngx.var.status})
  metric_latency:observe(tonumber(ngx.var.request_time), {ngx.var.server_name})
}

server {
    listen       *:80;
    server_name  _;

    location / {
        content_by_lua_block {
            ngx.say("hello")
        }
    }

    location /metrics {
      content_by_lua_block {
        metric_connections:set(ngx.var.connections_reading, {"reading"})
        metric_connections:set(ngx.var.connections_waiting, {"waiting"})
        metric_connections:set(ngx.var.connections_writing, {"writing"})
        prometheus:collect()
      }

      access_log off;
      allow all; # 实际上应该只允许 prometheus server的 IP 地址
    }
}
```

## prometheus 配置

prometheus.yaml 示例

```yaml
global:
  scrape_interval: 5s
  external_labels:
    monitor: "my-monitor"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "lua-resty-prometheus"
    static_configs:
      - targets: ["openresty-1.27.1.2:10080"]
```

## docker compose 配置

deployment.yaml

```yaml
version: "3.8"
services:
  nginx:
    image: docker pull openresty/openresty:1.27.1.2-2-bullseye-fat
    container_name: openresty-1.27.1.2
    volumes:
      - ./nginx-server.conf:/etc/nginx/conf.d/default.conf
      - ./site:/usr/local/openresty/site
    ports:
      - 10080:80

  prometheus:
    image: prom/prometheus:v2.35.0
    container_name: prometheus
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yaml
      - ./prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yaml"
    ports:
      - "9090:9090"

  renderer:
    image: grafana/grafana-image-renderer
    environment:
      BROWSER_TZ: Asia/Shanghai
    ports:
      - "8082:8081"

  grafana:
    image: grafana/grafana
    container_name: grafana
    volumes:
      - ./grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: pass
      GF_RENDERING_SERVER_URL: http://renderer:8082/render
      GF_RENDERING_CALLBACK_URL: http://grafana:3007/
      GF_LOG_FILTERS: rendering:debug
    depends_on:
      - prometheus
      - renderer
    ports:
      - "3007:3000"
```

# 启动容器

```shell
docker-compose -f ./deployment.yaml up -d
```

# 压测 OpenResty

```shell
wrk -d 1000 http://127.0.0.1:10080
```

通过如下命令查看 nginx 吐出的数据是否正常。

```shell
$ curl http://127.0.0.1:10080/metrics
# HELP nginx_http_connections Number of HTTP connections
# TYPE nginx_http_connections gauge
nginx_http_connections{state="reading"} 0
nginx_http_connections{state="waiting"} 4
nginx_http_connections{state="writing"} 8
# HELP nginx_http_request_duration_seconds HTTP request latency
# TYPE nginx_http_request_duration_seconds histogram
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.005"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.01"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.02"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.03"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.05"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.075"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.1"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.2"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.3"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.4"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.5"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="0.75"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="1"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="1.5"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="2"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="3"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="4"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="5"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="10"} 30672965
nginx_http_request_duration_seconds_bucket{host="localhost",le="+Inf"} 30672965
nginx_http_request_duration_seconds_count{host="localhost"} 30672965
nginx_http_request_duration_seconds_sum{host="localhost"} 140.779
# HELP nginx_http_requests_total Number of HTTP requests
# TYPE nginx_http_requests_total counter
nginx_http_requests_total{host="localhost",status="200"} 29802107
nginx_http_requests_total{host="localhost",status="499"} 3
nginx_http_requests_total{host="localhost",status="502"} 870855
# HELP nginx_metric_errors_total Number of nginx-lua-prometheus errors
# TYPE nginx_metric_errors_total counter
nginx_metric_errors_total 0
```

# 查看 Prometheus 数据

在浏览器中打 http://localhost:9090，你可以查看 Prometheus 的数据。

# Grafana

Prometheus 的查看数据只适合用例查看采集数据的正确性。
应该使用 Grafana 查看数据才比较友好。

打开链接 http://localhost:3007/ 查看 Grafana 的数据。

不过 Grafana 的配置比较麻烦，请参考下面参考资料的配置方式进行配置。


# 参考资料

1. https://www.markkulab.net/prometheus-monitoring-nginx-requests/#%E5%BB%BA%E7%AB%8B%E5%8F%8A%E5%95%9F%E5%8B%95%E7%9B%B8%E9%97%9C%E5%AE%B9%E5%99%A8%E6%87%89%E7%94%A8
