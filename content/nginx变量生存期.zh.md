+++
title = "nginx变量生存期"
date = 2014-08-05
[taxonomies]
tags = ["nginx", "lua", "cdn"]

+++

在nginx中，变量有多种形式，一般来说，在同一个location里面，变量是一致的，但是经常出现的情况是，在从一个location转移到另外一个location的时候，变量会发生变化，有些变量会改变、有些变量会结束生存期。

如果看看location跳转的类型和变量类型的组合，那么变量的生存期就变得非常的复杂和有趣...

<!-- more -->

## 从一个location转移到另外一个location有如下几种方法：
* rewrite (rewrite_by_lua, rewrite_by_luafile)
* proxy_pass 
------------ 
* 以下为ngx_openresty提供的模块
* ngx.exec
* ngx.location.capture

## 变量类型
* [nginx core 变量](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)
* 在nginx配置文件里面自定义变量  set $foo 'bar'
* Headers相关


## 变化规律  
|location change | nginx core | custom | headers| 
|----------------|------------|--------|--------|
|rewrite | uri相关变量 | 不变或者被覆盖 | 不变 |
|proxy_pass | uri相关变化，args不变，$remote_addr变成 127.0.0.1 | 为空 |[proxy_pass](http://nginx.org/cn/docs/http/ngx_http_proxy_module.html#proxy_pass_header),自定义header不变 |
|ngx.exec | uri相关变量，args丢失 | 不变或者被覆盖 | 不变 | 
|ngx.location.capture | uri相关变量，args丢失 |  为空 | 不变|


参考：
* http://wiki.nginx.org/HttpLuaModule
* http://nginx.org/en/docs/http/ngx_http_core_module.html
