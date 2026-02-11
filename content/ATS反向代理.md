+++
title = "Apache Traffic Server Reverse Proxy"
date = 2014-07-25
[taxonomies]
tags = ["cdn"]
+++

A high-performance CDN server from Apache, with use cases including:
[Comcast](http://www.bizety.com/2014/02/23/comcasts-internal-cdn/)
[Taobao](http://www.infoq.com/cn/presentations/apache-traffic-server-and-cdn-practice)

<!-- more -->

## Installation
Ubuntu:
apt-get install trafficserver

Manual:
https://docs.trafficserver.apache.org/en/latest/admin/getting-started.en.html

## Configure Debug Parameters

```
//Add VIA field for debugging
traffic_line -s  proxy.config.http.insert_response_via_str -v 2
traffic_line -s  proxy.config.http.insert_request_via_str -v 1

//Modify default cache policy
traffic_line -s proxy.config.http.cache.required_headers -v 0

//Commit changes
traffic_line -x
```

## Configure Sites to Accelerate

```
//Configure remap.config, default ATS port is 8080
//To accelerate mirrors.163.com
//Local mirror address is mirrors.gaxxx.me
map http://mirrors.gaxxx.me:8080/ http://mirrors.163.com/
```

## Test Configuration

```
curl http://mirrors.gaxxx.me:8080/  2>&1 | less
```

You can see:
  Via: http/1.1 tv140002 (ApacheTrafficServer/5.1.0 [cRs f ])

//Check VIA codes, cache hit in memory
```
traffic_line --decode_via  "cRs f"
Via Header Details:
Result of Traffic Server cache lookup for URL          :in cache, fresh Ram hit (a cache "HIT")
Response information received from origin server       :no server connection needed
Result of document write-to-cache:                     :no cache write performed
```

## Common Commands
```
trafficserver    ATS server run script
traffic_line     Configure ATS, reload ATS service
traffic_logstats ATS log viewer, check cache hits etc.
```

## Cluster Control

ATS has two cluster modes: Manager Only and Full Cluster
* ManagerOnly: Syncs configuration files, each node's cache is separate
* Full Cluster: All nodes' caches are merged into one logical cache; nodes may transfer cache objects over the network, requiring high internal network bandwidth

[Configuration guide](https://docs.trafficserver.apache.org/en/latest/admin/cluster-howto.en.html)

## Log Management
ATS has an independent logging system, similar to nginx, with error logs (error.log), event logs (access.log), and system logs (message.log). It can also generate summary logs based on configuration.

For log formats, you can use the common Squid format or ATS's custom XML format. Storage types can be binary or ASCII, with logrotate support.

Supports clustered log deployment.

See [ATS Logging](https://docs.trafficserver.apache.org/en/latest/admin/working-log-files.en.html) for details.

## Cache Management
