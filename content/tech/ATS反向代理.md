+++
title = "Apache Traffic Server 反向代理"
date = 2014-07-25
[taxonomies]
tags = ["cdn"]

+++

## 简介
Apache出的一个高效CDN服务器，应用案例有：
[Comcast](http://www.bizety.com/2014/02/23/comcasts-internal-cdn/)
[Taobao](http://www.infoq.com/cn/presentations/apache-traffic-server-and-cdn-practice)

<!-- more -->


## 安装
Ubuntu:
apt-get install trafficserver

Manual:
https://docs.trafficserver.apache.org/en/latest/admin/getting-started.en.html


## 配置调试参数

```
//添加VIA字段用于调试  
traffic_line -s  proxy.config.http.insert_response_via_str -v 2  
traffic_line -s  proxy.config.http.insert_request_via_str -v 1  

//修改默认的cache策略  
traffic_line -s proxy.config.http.cache.required_headers -v 0

//commit 操作  
traffic_line -x
```

## 配置需要加速的网站

```
//配置remap.config,  默认的ats端口是8080,
//如果需要配置加速 mirrors.163.com
//本地镜像地址为 mirrors.gaxxx.me
map http://mirrors.gaxxx.me:8080/ http://mirrors.163.com/
```
 

## 测试配置效果

```
curl http://mirrors.gaxxx.me:8080/  2>&1 | less
```

可以看到:
  Via: http/1.1 tv140002 (ApacheTrafficServer/5.1.0 [cRs f ])


//查看VIA代码,cache命中在内存  
```
traffic_line --decode_via  "cRs f" 
Via Header Details:
Result of Traffic Server cache lookup for URL          :in cache, fresh Ram hit (a cache "HIT")
Response information received from origin server       :no server connection needed
Result of document write-to-cache:                     :no cache write performed
```

## 常用命令
```
trafficserver ATS服务器运行脚本
traffic_line 配置ATS，重载ATS服务
traffic_logstats ATS日志查看，可以查看cache命中等等
```

## 集群控制

ATS有两种集群模式， Manager Only 和 Full Cluster
* ManagerOnly: 同步配置文件，每个node的cache是分开的
* Full Cluster: 所有node的cache合并成一个逻辑上的cache，节点之间可能通过网络传输cache object, 因此对于内部网络有一个较高的要求

[配置传送门](https://docs.trafficserver.apache.org/en/latest/admin/cluster-howto.en.html)

## 日志管理
ATS有独立的日志系统，跟nginx一样，有错误日志(error.log),事件日志(access.log),系统日志(message.log)
还能根据配置生成summary log

在日志的格式上，可以采用常用的Squid格式，也能采用ATS自定义的xml格式，存储类型可以是binary或者ASCII，支持 '翻滚吧，日志'(logrotate) 功能

支持集群式部署日志

具体查看 [ATS日志](https://docs.trafficserver.apache.org/en/latest/admin/working-log-files.en.html)

## 缓存管理
