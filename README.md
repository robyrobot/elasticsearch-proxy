<img src="https://travis-ci.org/robyrobot/elasticsearch-proxy.svg?branch=master" />

PROXY, a lightweight elasticsearch proxy written in golang.

# Features
- Auto handling upstream failure while indexing, aka nonstop indexing
- Auto detect the upstream failure in search
- Multiple write mechanism, one indexing request map to multi remote elasticsearch clusters
- Support TLS/HTTPS, generate the cert files automatically
- Support run background as daemon mode(only available on linux and macOS)
- Auto merge indexing operations to single bulk operation(WIP)
- Load balancing(indexing and search request), algorithm configurable(WIP)
- A controllable query cache layer, use redis as backend
- Index throttling or buffering, via disk based indexing queue(limit by queue length or size)
- Search throttling, limit concurrent connections to upstream(WIP)
- Builtin stats API and management UI(WIP)

# How to use

- First, setup upstream config in the `proxy.yml`.

```
api:
  enabled: true
  network:
    binding: 0.0.0.0:2900
  tls:
    enabled: true
    
elasticsearch:
- name: default
  enabled: true
  endpoint: http://localhost:9200

plugins:
- name: proxy
  enabled: true
  upstream:
  - name: primary
    enabled: true
    rate_limit:
      max_qps: 10000
    queue_name: primary
    max_queue_depth: -1
    timeout: 60s
    elasticsearch: default
```
- Start the PROXY.

```
➜  elasticsearch-proxy ✗ ./bin/proxy
___  ____ ____ _  _ _   _
|__] |__/ |  |  \/   \_/
|    |  \ |__| _/\_   |
[PROXY] An elasticsearch proxy written in golang.
0.1.0_SNAPSHOT,  430bd60, Sun Apr 8 09:44:38 2018 +0800, medcl, seems good to go

[04-05 19:30:13] [INF] [instance.go:23] workspace: data/APP/nodes/0
[04-05 19:30:13] [INF] [api.go:147] api server listen at: http://0.0.0.0:2900

```

- Done! Now you are ready to rock with it.

```
➜ curl  -XGET http://localhost:2900/
{
  "name": "PROXY",
  "tagline": "You Know, for Proxy",
  "upstream": {
    "backup": "http://localhost:9201",
    "primary": "http://localhost:9200"
  },
  "uptime": "1m58.019165s",
  "version": {
    "build_commit": "430bd60, Sun Apr 8 09:44:38 2018 +0800, medcl, seems good to go ",
    "build_date": "Sun Apr  8 09:58:29 CST 2018",
    "number": "0.1.0_SNAPSHOT"
  }
}
➜ curl  -XGET -H'UPSTREAM:primary'  http://localhost:2900/
{
  "name" : "XZDZ8qc",
  "cluster_name" : "my-application",
  "cluster_uuid" : "FWt_UO6BRr6uBVhkVrisew",
  "version" : {
    "number" : "6.2.3",
    "build_hash" : "c59ff00",
    "build_date" : "2018-03-13T10:06:29.741383Z",
    "build_snapshot" : false,
    "lucene_version" : "7.2.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
➜ curl  -XGET -H'UPSTREAM:backup'  http://localhost:2900/
{
  "name" : "zRcp1My",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "FWt_UO6BRr6uBVhkVrisew",
  "version" : {
    "number" : "5.6.8",
    "build_hash" : "688ecce",
    "build_date" : "2018-02-16T16:46:30.010Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
➜ curl  -XPOST http://localhost:2900/myindex/doc/1 -d'{"msg":"hello world!"}'
{ "acknowledge": true }
➜ curl  -XGET http://localhost:2900/myindex/doc/1
{"_index":"myindex","_type":"doc","_id":"1","_version":1,"found":true,"_source":{"msg":"hello world!"}}
➜ curl  -XPUT http://localhost:2900/myindex/doc/1 -d'{"msg":"i am a proxy!"}'
{ "acknowledge": true }
➜ curl  -XGET http://localhost:2900/myindex/doc/1
{"_index":"myindex","_type":"doc","_id":"1","_version":2,"found":true,"_source":{"msg":"i am a proxy!"}}
➜ curl  -XDELETE http://localhost:2900/myindex/doc/1
{ "acknowledge": true }
➜ curl  -XGET http://localhost:2900/myindex/doc/1
{"_index":"myindex","_type":"doc","_id":"1","found":false}
```

Have fun!

# Options

- Additional request headers
  1. `UPSTREAM`, manually choose which upstream are going to query against(read/search requests)

    ```
    ➜ curl -v -XGET -H'UPSTREAM:primary'  http://localhost:2900/index/doc/1
    Note: Unnecessary use of -X or --request, GET is already inferred.
    *   Trying 127.0.0.1...
    * TCP_NODELAY set
    * Connected to localhost (127.0.0.1) port 2900 (#0)
    > GET /index/doc/1 HTTP/1.1
    > Host: localhost:2900
    > User-Agent: curl/7.54.0
    > Accept: */*
    > UPSTREAM:primary
    >
    < HTTP/1.1 200 OK
    < Upstream: primary
    < Date: Sat, 07 Apr 2018 13:00:30 GMT
    < Content-Length: 86
    < Content-Type: text/plain; charset=utf-8
    <
    * Connection #0 to host localhost left intact
    {"_index":"index","_type":"doc","_id":"1","_version":5,"found":true,"_source":{"a":6}}%
    ```

# API

- Status
```
curl -XGET localhost:2900/_proxy/stats
```
```
curl -XGET localhost:2900/_proxy/queue/stats
```
- Resume Queue
```
curl -XPOST http://localhost:2900/_proxy/queue/resume -d'{"queue":"primary"}'
```
- Get Error requests
```
curl  -XGET http://localhost:2900/_proxy/requests/?from=0&size=20&upstream=primary&status=1
```
- Replay Error log
```
curl  -XPOST http://localhost:2900/_proxy/request/redo -d'{"ids":["bb6t4cqaukihf1ag10q0","bb6t4daaukihf1ag10r0"]}'
```

# Smoking Benchmark

MacBook Pro (13-inch, 2017, Four Thunderbolt 3 Ports), 3.5 GHz Intel Core i7, 16 GB 2133 MHz LPDDR3

- Https 2900, query
```
~$ wrk -c 1000 -d 3m -t 10 -H --latency  https://localhost:2900/index/_search?q=customer:A
Running 3m test @ https://localhost:2900/index/_search?q=customer:A
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    19.13ms    9.07ms 247.10ms   75.79%
    Req/Sec     2.63k   699.58    10.55k    61.93%
  4707681 requests in 3.00m, 1.89GB read
  Socket errors: connect 417, read 290, write 0, timeout 0
Requests/sec:  26140.53
Transfer/sec:     10.72MB
```

- Https 2900, query with cache
```
~$ wrk -c 1000 -d 3m -t 10 -H --latency  https://localhost:2900/index/_search?q=customer:A
Running 3m test @ https://localhost:2900/index/_search?q=customer:A
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     7.59ms    8.47ms 338.09ms   87.25%
    Req/Sec     6.88k     3.08k   38.47k    78.65%
  12275137 requests in 3.00m, 4.90GB read
  Socket errors: connect 387, read 676, write 0, timeout 0
Requests/sec:  68158.61
Transfer/sec:     27.84MB
```


License
=======
Released under the [Apache License, Version 2.0](https://github.com/medcl/elasticsearch-proxy/blob/master/LICENSE) .

