+++
title = 'lapis on openwrt'
date = 2015-02-09 
[taxonomies]
tags = ["openwrt", "nginx"] 
+++

# Overview
听说前东家的产品在CES上展出了，很高兴这个项目剩下的小伙伴这么给力，同时也记录一下，当时撸过的一个框架(lapis)

<!-- more -->

## Http Service On Openwrt
被抓状丁搞智能路由器的时候，我参考了小米和Hiwifi的http服务框架，基本上都是用的luci,只不过小米稍微有点节操，代码是公开的，至于hiwifi,据说是深度定制了luci，然后不知道为什么就不放开基于gpl的源代码了。

测试过luci的性能，大约在5次/秒的量级，正在上一个项目搞nginx的时候，发现了一个lapis的框架，所以尝试搭建了一个，简单的http请求，性能能达到100次/秒，而且基于模板的lapis,比基于node的luci，编辑起来更简单一些....


## 下载lapis_on_openwrt
下载[lapis_on_openwrt](https://github.com/gaxxx/lapis_on_openwrt)

## make image
将lapis_on_monet放在Openwrt编译目录，运行makeimage.sh


    ./makeimage.sh

