---
layout: post
title: "使用 OpenTelemetry 采集 Nginx 访问日志/错误日志/指标数据/Trace 数据"
description: "Collect Nginx logs using otelcol-contrib"
date: 2024-09-06
tags: [Nginx, Shared Memory]
---

# 安装 nginx-otel 模块

安装 Nginx 模块参考 https://github.com/nginxinc/nginx-otel/blob/main/README.md



# 配置 Nginx

1. 为了方便日志采集，将日志输出为 json 这种结构化的格式。注意 log_format 中指定 escape 为 json。
2. location /t 配置了推送 trace 信息到 127.0.0.1:4317
3. /status 配置了访问规则，拒绝 127.0.0.1 以后的访问

```nginx
load_module /usr/lib64/nginx/modules/ngx_otel_module.so;

user  nobody;
worker_processes  auto;
worker_cpu_affinity auto;
error_log  logs/error.log;
pid        logs/nginx.pid;

events {
    accept_mutex off;
    worker_connections  256;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format logger-json escape=json '{"source": "nginx", "time": $msec, "resp_body_size": $body_bytes_sent, "host": "$http_host", "address": "$remote_addr", "request_length": $request_length, "method": "$request_method", "uri": "$request_uri", "status": $status,  "user_agent": "$http_user_agent", "resp_time": $request_time, "upstream_addr": "$upstream_addr"}';

    access_log logs/access.log logger-json;

    otel_service_name test:openresty;
    otel_exporter {
        endpoint 127.0.0.1:4317;
        interval    1s;
        batch_size  1024;
        batch_count 16;
    }

    server {
        listen       *:8080 reuseport;
        server_name  localhost;
        server_tokens off;

        location /status {
            allow 127.0.0.1;
            deny  all;
            access_log off;
            stub_status on;
        }

        location = /t {
            otel_trace on;
            otel_trace_context propagate;

            content_by_lua_block {
                ngx.log(ngx.ERR, "test log")
                ngx.exit(403)
            }
        }
    }
}
```

# 采集访问日志

采集访问日志比较简单，因为是结构化的格式，因此只需要指定 json_parser 即可。

```yaml
receivers:
  filelog:
    include:
      - /usr/local/openresty/nginx/logs/access.log
    operators:
      - type: json_parser
        include_file_name: false

processors:
  batch:

exporters:
  debug:
    verbosity: detailed

service:

  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch]
      exporters: [debug]
```

# 采集错误日志

采集错误日志的时候，我们也希望能够将错误消息和元数据区分开来，因为不是结构化数据，因此需要使用正则表达式。

下面这个正则表达式能够匹配三种格式的错误日志，但是也存在一下日志不能匹配的问题。

## 三种支持的错误日志

### 第一种是这种最简单请求访问的错误日志。

```log
2024/09/08 19:50:04 [error] 78809#78809: *52904 [lua] content_by_lua(nginx.conf:124):2: fail to connect to upstream, client: 127.0.0.1, server: localhost, request: "GET /403 HTTP/1.1", host: "127.0.0.1:8080"
```

### 第二种是多行日志，Lua 抛出的异常。

```
2024/09/08 19:50:04 [error] 78809#78809: *52904 lua entry thread aborted: runtime error: content_by_lua(nginx.conf:124):3: abc
stack traceback:
coroutine 0:
	[C]: in function 'error'
	content_by_lua(nginx.conf:124):3: in main chunk, client: 127.0.0.1, server: localhost, request: "GET /403 HTTP/1.1", host: "127.0.0.1:8080"
```

### 第三种是以  context: ngx.timer 结尾的日志

```log
2024/09/08 16:16:05 [error] 69277#69277: *46495 [lua] dynamic.lua:269: ctx is boolean ctx: false, context: ngx.timer
```

## 对应的 yaml 配置

1. 日志先配置 multiline 这个进行分割, 防止正则表达式无法匹配所有的错误日志。
2. 使用 regex_parser 这个运算符将各个字段解析出来

当然，多行日志也可以用 recombine 这个 operator，但是感觉不方便。具体参考 https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/pkg/stanza/docs/operators/recombine.md。

```yaml
receivers:
  filelog:
    include:
      - /usr/local/openresty/nginx/logs/error.log
    multiline:
      line_start_pattern: ^\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}
    operators:
      - type: regex_parser
        regex: '(?s)(?P<time>\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}) \[(?P<level>\w+)\] (?P<pid>\d+)#(?P<tid>\d+): \*(?P<connection_id>\d+) (?P<message>.*?), (?:client: (?P<client>[0-9a-f.]+), server: (?P<server>[^ ]+), request: "(?P<request>.*?)", host: "(?P<host>[0-9a-f:.]+)"|context: (?P<context>[\w.*]+))'
        timestamp:
          parse_from: attributes.time
          layout: '%Y/%m/%d %H:%M:%S'
        severity:
          parse_from: attributes.level
          preset: none
          mapping:
            debug: debug
            info: info
            info1: notice
            warn: warn
            error: error
            fatal: crit
            fatal2: alert
            fatal3: emerg

    nginx:
      endpoint: "http://127.0.0.1:8080/status"
      collection_interval: 60s


processors:
  batch:

exporters:
  debug:
    verbosity: detailed

service:

  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch]
      exporters: [debug]
```

# 采集 Trace 信息

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 127.0.0.1:4317
      http:
        endpoint: 127.0.0.1:4318
processors:
  batch:

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```
# 采集指标数据

```yaml
receivers:
  nginx:
    endpoint: "http://127.0.0.1:8080/status"
    collection_interval: 3s

processors:
  batch:

exporters:
  debug:
    verbosity: detailed

service:

  pipelines:
    metrics:
      receivers: [nginx]
      processors: [batch]
      exporters: [debug]
```


# 将所有配置合在一个文件

```yaml
receivers:
  filelog/access:
    include:
      - /usr/local/openresty/nginx/logs/access.log
    operators:
      - type: json_parser
        include_file_name: false

  filelog/error:
    include:
      - /usr/local/openresty/nginx/logs/error.log
    multiline:
      line_start_pattern: ^\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}
    operators:
      - type: regex_parser
        regex: '(?s)(?P<time>\d{4}/\d{2}/\d{2} \d{2}:\d{2}:\d{2}) \[(?P<level>\w+)\] (?P<pid>\d+)#(?P<tid>\d+): \*(?P<connection_id>\d+) (?P<message>.*?), (?:client: (?P<client>[0-9a-f.]+), server: (?P<server>[^ ]+), request: "(?P<request>.*?)", host: "(?P<host>[0-9a-f:.]+)"|context: (?P<context>[\w.*]+))'
        timestamp:
          parse_from: attributes.time
          layout: '%Y/%m/%d %H:%M:%S'
        severity:
          parse_from: attributes.level
          preset: none
          mapping:
            debug: debug
            info: info
            info1: notice
            warn: warn
            error: error
            fatal: crit
            fatal2: alert
            fatal3: emerg

  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  nginx:
    endpoint: "http://127.0.0.1:8080/status"
    collection_interval: 3s

processors:
  batch:

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    logs/access:
      receivers: [filelog/access]
      processors: [batch]
      exporters: [debug]

    logs/error:
      receivers: [filelog/error]
      processors: [batch]
      exporters: [debug]

    metrics:
      receivers: [nginx]
      processors: [batch]
      exporters: [debug]

    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

## 运行 otelcol-contrib

```shell
otelcol-contrib  --config=file:error.yaml
```

发送一个请求，得到结果如下：



```log
2024-09-08T21:52:18.207+0800	info	LogsExporter	{"kind": "exporter", "data_type": "logs", "name": "debug", "resource logs": 1, "log records": 1}
2024-09-08T21:52:18.208+0800	info	ResourceLog #0
Resource SchemaURL:
ScopeLogs #0
ScopeLogs SchemaURL:
InstrumentationScope
LogRecord #0
ObservedTimestamp: 2024-09-08 13:52:18.065184632 +0000 UTC
Timestamp: 1970-01-01 00:00:00 +0000 UTC
SeverityText:
SeverityNumber: Unspecified(0)
Body: Str({"source": "nginx", "time": 1725803538.050, "resp_body_size": 150, "host": "127.0.0.1:8080", "address": "127.0.0.1", "request_length": 81, "method": "GET", "uri": "/403", "status": 403,  "user_agent": "curl/7.76.1", "resp_time": 0.000, "upstream_addr": ""})
Attributes:
     -> resp_body_size: Double(150)
     -> address: Str(127.0.0.1)
     -> uri: Str(/403)
     -> log.file.name: Str(access.log)
     -> request_length: Double(81)
     -> method: Str(GET)
     -> host: Str(127.0.0.1:8080)
     -> time: Double(1725803538.05)
     -> upstream_addr: Str()
     -> source: Str(nginx)
     -> user_agent: Str(curl/7.76.1)
     -> resp_time: Double(0)
     -> status: Double(403)
Trace ID:
Span ID:
Flags: 0
	{"kind": "exporter", "data_type": "logs", "name": "debug"}


2024-09-08T21:52:18.809+0800	info	LogsExporter	{"kind": "exporter", "data_type": "logs", "name": "debug", "resource logs": 1, "log records": 1}
2024-09-08T21:52:18.810+0800	info	ResourceLog #0
Resource SchemaURL:
ScopeLogs #0
ScopeLogs SchemaURL:
InstrumentationScope
LogRecord #0
ObservedTimestamp: 2024-09-08 13:52:18.664561882 +0000 UTC
Timestamp: 2024-09-08 13:52:18 +0000 UTC
SeverityText: error
SeverityNumber: Error(17)
Body: Str(2024/09/08 21:52:18 [error] 247444#247444: *2 [lua] content_by_lua(nginx.conf:133):2: 403, client: 127.0.0.1, server: localhost, request: "GET /403 HTTP/1.1", host: "127.0.0.1:8080")
Attributes:
     -> time: Str(2024/09/08 21:52:18)
     -> context: Str()
     -> pid: Str(247444)
     -> server: Str(localhost)
     -> log.file.name: Str(error.log)
     -> client: Str(127.0.0.1)
     -> request: Str(GET /t HTTP/1.1)
     -> host: Str(127.0.0.1:8080)
     -> level: Str(test error)
     -> connection_id: Str(2)
     -> message: Str([lua] content_by_lua(nginx.conf:133):2: 403)
     -> tid: Str(247444)
Trace ID:
Span ID:
Flags: 0
	{"kind": "exporter", "data_type": "logs", "name": "debug"}



2024-09-08T21:52:19.011+0800	info	TracesExporter	{"kind": "exporter", "data_type": "traces", "name": "debug", "resource spans": 1, "spans": 1}
2024-09-08T21:52:19.011+0800	info	ResourceSpans #0
Resource SchemaURL:
Resource attributes:
     -> service.name: Str(test:openresty)
ScopeSpans #0
ScopeSpans SchemaURL:
InstrumentationScope nginx 1.27.1
Span #0
    Trace ID       : 0ab6bfb877fb06edec782e43af3fc202
    Parent ID      :
    ID             : b9a488acb3c8d559
    Name           : /403
    Kind           : Server
    Start time     : 2024-09-08 13:52:18.05 +0000 UTC
    End time       : 2024-09-08 13:52:18.05 +0000 UTC
    Status code    : Unset
    Status message :
Attributes:
     -> http.method: Str(GET)
     -> http.target: Str(/403)
     -> http.route: Str(/403)
     -> http.scheme: Str(http)
     -> http.flavor: Str(1.1)
     -> http.user_agent: Str(curl/7.76.1)
     -> http.request_content_length: Int(0)
     -> http.response_content_length: Int(150)
     -> http.status_code: Int(403)
     -> net.host.name: Str(localhost)
     -> net.host.port: Int(8080)
     -> net.sock.peer.addr: Str(127.0.0.1)
     -> net.sock.peer.port: Int(28688)
	{"kind": "exporter", "data_type": "traces", "name": "debug"}


2024-09-08T22:16:08.046+0800	info	MetricsExporter	{"kind": "exporter", "data_type": "metrics", "name": "debug", "resource metrics": 1, "metrics": 4, "data points": 7}
2024-09-08T22:16:08.048+0800	info	ResourceMetrics #0
Resource SchemaURL:
ScopeMetrics #0
ScopeMetrics SchemaURL:
InstrumentationScope github.com/open-telemetry/opentelemetry-collector-contrib/receiver/nginxreceiver 0.108.1
Metric #0
Descriptor:
     -> Name: nginx.connections_accepted
     -> Description: The total number of accepted client connections
     -> Unit: connections
     -> DataType: Sum
     -> IsMonotonic: true
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 10
Metric #1
Descriptor:
     -> Name: nginx.connections_current
     -> Description: The current number of nginx connections by state
     -> Unit: connections
     -> DataType: Sum
     -> IsMonotonic: false
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
Data point attributes:
     -> state: Str(active)
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 1
NumberDataPoints #1
Data point attributes:
     -> state: Str(reading)
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 0
NumberDataPoints #2
Data point attributes:
     -> state: Str(writing)
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 1
NumberDataPoints #3
Data point attributes:
     -> state: Str(waiting)
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 0
Metric #2
Descriptor:
     -> Name: nginx.connections_handled
     -> Description: The total number of handled connections. Generally, the parameter value is the same as nginx.connections_accepted unless some resource limits have been reached (for example, the worker_connections limit).
     -> Unit: connections
     -> DataType: Sum
     -> IsMonotonic: true
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 10
Metric #3
Descriptor:
     -> Name: nginx.requests
     -> Description: Total number of requests made to the server since it started
     -> Unit: requests
     -> DataType: Sum
     -> IsMonotonic: true
     -> AggregationTemporality: Cumulative
NumberDataPoints #0
StartTimestamp: 2024-09-08 14:16:04.039538984 +0000 UTC
Timestamp: 2024-09-08 14:16:08.043710279 +0000 UTC
Value: 20
	{"kind": "exporter", "data_type": "metrics", "name": "debug"}
```
