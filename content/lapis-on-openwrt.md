+++
title = 'Lapis on OpenWrt'
date = 2015-02-09
[taxonomies]
tags = ["openwrt", "nginx"]
+++

Heard that my former company's product was exhibited at CES. Happy that the remaining team members are doing great. This also reminds me of a framework (lapis) I worked with back then.

<!-- more -->

## Http Service On Openwrt
When I was working on smart routers, I looked at the HTTP service frameworks of Xiaomi and Hiwifi. Both basically use luci - Xiaomi has some integrity with open source code. As for Hiwifi, they supposedly deeply customized luci but somehow don't release their GPL-based source code.

After testing luci's performance at about 5 requests/second, while working on nginx in my previous project, I discovered the lapis framework. I tried setting it up - simple HTTP requests achieved 100 requests/second. The template-based lapis is also easier to edit than the node-based luci...


## Download lapis_on_openwrt
Download [lapis_on_openwrt](https://github.com/gaxxx/lapis_on_openwrt)

## Make Image
Place lapis_on_monet in the OpenWrt build directory and run makeimage.sh


    ./makeimage.sh
