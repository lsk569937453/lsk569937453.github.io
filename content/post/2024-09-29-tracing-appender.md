---
title: tracing的tracing-appender的NonBlocking影响性能      

description: the waiting time of XHR request
date: 2024-09-29T14:29:07+01:00

categories:
 - backend
 - rust
 - tracing
 
tags:
 - backend

---


# tracing的tracing-appender的NonBlocking影响性能(CPU使用率极高)

## 使用的crates版本,机器规格
```
tracing = "0.1.4"
tracing-subscriber = "0.3.18"
tracing-appender = "0.2.3"
```
测试的机器为**2核CPU,在docker-compose的环境下做的测试**。
### 有问题的代码
```
let app_file = rolling::daily("./logs", "access.log");
    let (non_blocking_appender, guard) = NonBlockingBuilder::default()
        .buffered_lines_limit(1000)
        .finish(app_file);
    let file_layer = tracing_subscriber::fmt::Layer::new()
        .with_target(true)
        .with_ansi(false)
        .with_writer(non_blocking_appender)
        .with_filter(tracing_subscriber::filter::LevelFilter::INFO);
```
### 写日志线程的CPU使用率极高
```
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    8 root      20   0  211164   9952   5084 R  71.3   0.1   0:20.38 tokio-runtime-w
    7 root      20   0  211164   9952   5084 R  70.3   0.1   0:20.32 tokio-runtime-w
    9 root      20   0  211164   9952   5084 R  58.3   0.1   0:16.94 tracing-appende
```
### 优化前的压测结果
```
 hey -n 5000000 -c 250 -m GET http://backend:80

Summary:
  Total:        42.4888 secs
  Slowest:      0.2562 secs
  Fastest:      0.0000 secs
  Average:      0.0106 secs
  Requests/sec: 117678.1926

  Total data:   765000000 bytes
  Size/request: 765 bytes

Response time histogram:
  0.000 [1]     |
  0.026 [988915]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.051 [10845] |
  0.077 [0]     |
  0.102 [69]    |
  0.128 [0]     |
  0.154 [0]     |
  0.179 [155]   |
  0.205 [5]     |
  0.231 [0]     |
  0.256 [10]    |


Latency distribution:
  10% in 0.0008 secs
  25% in 0.0011 secs
  50% in 0.0015 secs
  75% in 0.0019 secs
  90% in 0.0023 secs
  95% in 0.0026 secs
  99% in 0.0283 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0000 secs, 0.2562 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0799 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0032 secs
  resp wait:    0.0104 secs, 0.0000 secs, 0.2474 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0049 secs

Status code distribution:
  [200] 1000000 responses


```


## 优化后的代码如下
```
let app_file = rolling::daily("./logs", "access.log");
let file_layer = tracing_subscriber::fmt::Layer::new()
    .with_target(true)
    .with_ansi(false)
    .with_writer(app_file)
    .with_filter(tracing_subscriber::filter::LevelFilter::INFO);

```
## 优化后的压测结果
```
hey -n 5000000 -c 200 -m GET http://backend:80

Summary:
  Total:        31.8442 secs
  Slowest:      0.0411 secs
  Fastest:      0.0000 secs
  Average:      0.0064 secs
  Requests/sec: 157014.5325

  Total data:   765000000 bytes
  Size/request: 765 bytes

Response time histogram:
  0.000 [1]     |
  0.004 [999220]        |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.008 [574]   |
  0.012 [5]     |
  0.016 [0]     |
  0.021 [0]     |
  0.025 [0]     |
  0.029 [0]     |
  0.033 [0]     |
  0.037 [0]     |
  0.041 [200]   |


Latency distribution:
  10% in 0.0007 secs
  25% in 0.0009 secs
  50% in 0.0012 secs
  75% in 0.0014 secs
  90% in 0.0017 secs
  95% in 0.0019 secs
  99% in 0.0024 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0000 secs, 0.0411 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0042 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0042 secs
  resp wait:    0.0062 secs, 0.0000 secs, 0.0395 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0387 secs

Status code distribution:
  [200] 1000000 responses
```

## 优化后的应用的cpu占比
```
PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    8 root      20   0  143444   9984   4976 R  99.0   0.1   0:44.52 tokio-runtime-w
    7 root      20   0  143444   9984   4976 R  98.7   0.1   0:44.54 tokio-runtime-w
```