+++
title = "Nginx Variable Lifetime"
date = 2014-08-05
[taxonomies]
tags = ["nginx", "lua", "cdn"]
+++

In nginx, variables come in multiple forms. Generally within the same location, variables are consistent. However, when transferring from one location to another, variables may change - some will be modified, others will end their lifetime.

Considering the combinations of location jump types and variable types, variable lifetimes become quite complex and interesting...

<!-- more -->

## Methods to transfer from one location to another:
* rewrite (rewrite_by_lua, rewrite_by_luafile)
* proxy_pass
------------
* The following are modules provided by ngx_openresty:
* ngx.exec
* ngx.location.capture

## Variable Types
* [nginx core variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)
* Custom variables defined in nginx config: set $foo 'bar'
* Header-related variables


## Behavior Rules
|location change | nginx core | custom | headers|
|----------------|------------|--------|--------|
|rewrite | URI-related variables change | unchanged or overwritten | unchanged |
|proxy_pass | URI-related change, args unchanged, $remote_addr becomes 127.0.0.1 | empty |[proxy_pass](http://nginx.org/cn/docs/http/ngx_http_proxy_module.html#proxy_pass_header), custom headers unchanged |
|ngx.exec | URI-related variables, args lost | unchanged or overwritten | unchanged |
|ngx.location.capture | URI-related variables, args lost | empty | unchanged|


References:
* http://wiki.nginx.org/HttpLuaModule
* http://nginx.org/en/docs/http/ngx_http_core_module.html
