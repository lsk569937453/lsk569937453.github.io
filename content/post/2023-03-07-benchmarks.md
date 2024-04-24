---
title: Proxy Benchmarks

description: how to generate pem file for https
date: 2023-03-07T07:07:07+01:00

categories:
 - https
 - RSA
 
tags:
 - https
 - RSA  

---
## Command
The testing command is like following:
```
.\hey_windows_amd64.exe -n 100000 -c 250 -m GET http://localhost:3360
```
## Benchmarks
### Rust-Proxy
```
Summary:
  Total:        4.4908 secs
  Slowest:      0.4010 secs
  Fastest:      0.0004 secs
  Average:      0.0112 secs
  Requests/sec: 22267.7514

  Total data:   13700000 bytes
  Size/request: 137 bytes

Response time histogram:
  0.000 [1]     |
  0.040 [99717] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.081 [32]    |
  0.121 [0]     |
  0.161 [0]     |
  0.201 [0]     |
  0.241 [0]     |
  0.281 [0]     |
  0.321 [0]     |
  0.361 [227]   |
  0.401 [23]    |


Latency distribution:
  10% in 0.0079 secs
  25% in 0.0088 secs
  50% in 0.0100 secs
  75% in 0.0114 secs
  90% in 0.0131 secs
  95% in 0.0145 secs
  99% in 0.0184 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0004 secs, 0.0004 secs, 0.4010 secs
  DNS-lookup:   0.0031 secs, 0.0000 secs, 0.0157 secs
  req write:    0.0001 secs, 0.0000 secs, 0.0110 secs
  resp wait:    0.0050 secs, 0.0004 secs, 0.0897 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0123 secs

Status code distribution:
  [200] 100000 responses
```
### Envoy
```
Summary:
  Total:        72.7533 secs
  Slowest:      0.4988 secs
  Fastest:      0.0017 secs
  Average:      0.1818 secs
  Requests/sec: 1374.5084

  Total data:   25000000 bytes
  Size/request: 250 bytes

Response time histogram:
  0.002 [1]     |
  0.051 [24]    |
  0.101 [14428] |■■■■■■■■■■■■■■■■■■■■■■
  0.151 [24247] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.201 [26549] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.250 [7407]  |■■■■■■■■■■■
  0.300 [19927] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.350 [7204]  |■■■■■■■■■■■
  0.399 [33]    |
  0.449 [77]    |
  0.499 [103]   |


Latency distribution:
  10% in 0.0916 secs
  25% in 0.1232 secs
  50% in 0.1579 secs
  75% in 0.2564 secs
  90% in 0.2943 secs
  95% in 0.3058 secs
  99% in 0.3196 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0017 secs, 0.4988 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0120 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0035 secs
  resp wait:    0.1817 secs, 0.0016 secs, 0.4894 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0019 secs

Status code distribution:
  [200] 100000 responses
```
## NGINX
```

Summary:
  Total:        23.4456 secs
  Slowest:      2.4671 secs
  Fastest:      0.0008 secs
  Average:      0.0577 secs
  Requests/sec: 4265.1998

  Total data:   14400000 bytes
  Size/request: 144 bytes

Response time histogram:
  0.001 [1]     |
  0.247 [99749] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.494 [101]   |
  0.741 [34]    |
  0.987 [23]    |
  1.234 [21]    |
  1.481 [16]    |
  1.727 [15]    |
  1.974 [15]    |
  2.220 [13]    |
  2.467 [12]    |


Latency distribution:
  10% in 0.0468 secs
  25% in 0.0481 secs
  50% in 0.0495 secs
  75% in 0.0721 secs
  90% in 0.0750 secs
  95% in 0.0769 secs
  99% in 0.0850 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0007 secs, 0.0008 secs, 2.4671 secs
  DNS-lookup:   0.0002 secs, 0.0000 secs, 0.0114 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0058 secs
  resp wait:    0.0474 secs, 0.0008 secs, 2.1455 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0012 secs

Status code distribution:
  [200] 100000 responses
```